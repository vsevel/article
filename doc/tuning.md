This article aims providing a good understanding of container resource sizing in kubernetes, including (but not limited to) workloads that exhibit a resource spike at startup (such as SpringBoot).

# Running workload on traditional VMs

Running multiple workloads on traditional VMs requires to share resources between the different processes on a given VM. Processes are not protected against each other. If a process decides at some point to go through a lot of processing, it will increase its usage on, say memory and cpu, possibly starving the other processes running on the same VM. This might have an impact on these other processes, which might stop meeting their service level objectives. Conversely, the application requiring more resources might fail absorbing its peak of activity, if the other applications running on the VM are themselves in a phase where they need additional resources.

Several strategies can be used to limit the impact or probability of this happening:

* isolation: move workload that needs a higher level of guarantee, or is likely to be a bad neighbor to dedicated VMs
* carefully and progressively add new workload, making sure "it fits in"
* capacity planning: rely on observability (monitoring and metrics) to figure out if a VM has space left for new workload. If not, optionally offload some workload to other VMs.
* overprovisioning resources such as CPU

One aspect that has worked well with traditional workload is that workloads are static: it is either installed on the VM, or not. Obviously the usage can be dynamic, but new workload does not appear suddenly. It is the result of provisioning. This makes it
somehow more manageable and predictable than if workloads came and left as it pleased themselves.

Isolation and overprovisioning have a tendency however to spoil resources by reserving more than really necessary just in case.
To solve this issue, we typically use overcommitting at the virtualization layer, hosting on a given ESX a bundle of VMs, whose sum of resources exceed the underlying pyhsical resources. There are 2 issues with this:

* The virtualization service provider may not know the type of workload that is going to run on its ESX. As a result it will be difficult for him to figure out if he can be agressive on the overcommitting ratio or should be conservative.
* The VM owner has an expectation on resources given by the provider, and is not necessarily aware of the use, the level or even the impact of the overcommitting ratio configured on the virtualization cluster that its VM is running on.

If we are talking in Kubernetes terminology, the VM owner expresses a resource request, when the VM provider is only offerring a resource limit:

* The VM is guaranteed to not go over the number of vCPU it is configured for
* But unless there is a 1:1 ratio, it is not guaranteed to always be able to use of all these vCPUs at any point in time (this will depend on the behavior of the other VMs sharing the same underlying physical host).

Of course VM owners and providers are encouraged to collaborate to make sure there is a good mutual understanding. However, the expectation of the VM owner on one side, and partial workload knowledge for the provider, will end up usually in conservative choices, such as a low overcommitting ratio. As a result, resources may stay idle, leading to inefficiencies, most likely resulting in suboptimal investements (e.g. paying for cores that you don't use).

Still, overcommitting at the virtualization layer should be an implementation detail, whose responsibility lies into the hands of the infrastructure team, which will treat its cluster as a whole trying to maximize efficiency of resources, to lower the cost of the vcpus, but with a primary focus on making available the requested resources (i.e. near-guaranteed resources). The corollary, is that virtualization overcommit should not be used as a substitute to propre capacity management at the application level.

# Running workload in containers

Compared to workload running on VMs, containers offer the added option of configuring a resource limit (option `-m` in docker for memory and `--cpus`, `--cpu-period`, `--cpu-quota` and `--cpu-shares` for cpu).

The `-m` memory parameter is a limit that the container cannot go over. Since memory is not a compressible resource, if the container tries to go over, it will go OOM and get killed.

The cpu shares defines a proportional weight for host cpu access, relative to the other containers. The default value in docker is 1024, but it is important to note that this is equal to a number of millicores. Let's take an example to illustrate:

Start launching `docker stats` to observe resource consumption:
```
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"
```

Assuming a docker host with 2 cores, let's launch a stress test that will consume all 2 cores for 60 seconds:
```
docker run -it --rm alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
```

While executing, the stats show:
```
NAME                CPU %               MEM USAGE / LIMIT     MEM %
sleepy_torvalds     202.42%             7.059MiB / 7.776GiB   0.09%
```

Now let's run the same container with a cpu shares of `300` and another container with `100`:
```
docker run -it --rm -d --cpu-shares=100 --name=container_100 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
docker run -it --rm -d --cpu-shares=300 --name=container_300 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
```

We can see that the `100` container is using 25% of the total capacity (i.e. `100/400`), and the other container uses the other `75%`:
```
NAME                CPU %               MEM USAGE / LIMIT     MEM %
container_100       51.39%              7.039MiB / 7.776GiB   0.09%
container_300       152.03%             7.09MiB / 7.776GiB    0.09%
```

Without waiting for the 2 containers to stop, launch a third container with cpus shares equal to `400`:
```
docker run -it --rm -d --cpu-shares=400 --name=container_400 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
```


We can see that the 2 first containers were readjusted to consume a fraction of the total number of shares defined on all containers:
- `100/800: 12.5% of total capacity (i.e. 2 cores)`
- `300/800: 37.5%`
- `400/800: 50%`

```
NAME                CPU %               MEM USAGE / LIMIT     MEM %
container_100       24.78%              7.02MiB / 7.776GiB    0.09%
container_300       72.62%              7.008MiB / 7.776GiB   0.09%
container_400       96.21%              7.027MiB / 7.776GiB   0.09%
```

The results would have been the same if shares had been `1000`, `3000` and `4000`, instead of `100`, `300`, and `400`. So whatever the values may be, they need to be consistent with one another.

Cpu shares are used may the system has contention (i.e. processes ask for more than the total capacity). Without contention, processes are free to use the entire host capacity. As a result cpu shares cannot be used to limit access to cpu, but only to guarantee some access to it. Recognizing this as an issue, Paul Turner, Bharata B. Rao and Nikhil Rao introduced the _CPU bandwidth control for CFS_ in the 2010 Linux Symposium, as a mean to limit access to cpu for processes: cpu quota and cpu period.

The cpu period is by default 100000 microseconds (i.e. 100 ms) defines the cpu period. The cpu quota defines the number of microseconds per cpu periods that the container is limited to. In docker, `--cpu-period="100000" --cpu-quota="150000"` means that the process is limited to `1.5` cpus, which can be expressed more conveniently with `--cpus="1.5"` in docker `1.13`.

Run again the first container with the `--cpus` option:
```
docker run -it --rm --cpus=1.5  alexeiled/stress-ng --cpu 2 --timeout 20s --metrics-brief
```

You can see that the container is being imited to 75% of the 2 cores total capacity:
```
NAME                CPU %               MEM USAGE / LIMIT     MEM %
brave_mayer         154.90%             6.988MiB / 7.776GiB   0.09%
```

As we can see, the container runtime engine provides options to a guarantee (through the cpu shares) and limit (through the cpu quota) access to cpu. Guaranteed access does not mean that cpu cycles will go wasted however if not used. If a container reserves 1 core, but stay idle, those cpu cycles will go to a shared pool, and get distributed to whoever needs them by the CFS.

These mechanisms offer a huge improvement compared to running workload on VMs, as now it is possible to tune access to resources. They can be used to define lower and upper bounds for cpu usage, and help with different scenarios:

- provide dedicated resources, and limit the process to the resources
- provide a minimal amount of guaranteed resources, and define an upper bound allowing the process to go above the minimal amount
- provide no minimal amount, and define an upper bound

This provides a lot of flexibility. However, running static containers on hosts, can be tedious (e.g. translating cpu shares into millicores) and workload cannot be moved easily. For that reason, the industry has started working on container orchestrators, to allow running containers at scale, while allowing resource efficiency, and ease of resource configuration for the individual workloads.

# Running workload in Kubernetes

One of the fundamental goal of Kubernetes is to be able to place workload dynamically, while preserving efficiency on resource usage (i.e. avoid wasting available resources). As we have seen, dynamic placement is only possible if a given workload is providing information about its resource usage. Kubernetes introduces `request` as a mean to express what the workload needs. It is an information that is used by the Kubernetes scheduler when a new pod needs to be assigned to a worker. Once the pod is assigned, . This contrasts with the `limit`, which is not used for placement (TODO except for the feature that tries to find a node that can currently accomodate the pod's limit), but is used by the container runtime level to configure cgroup quotas.

In other words:

* `request`: used by the Kubernetes scheduler for placement
* `limit`: used by the container runtime engine for enforcement

It is easy to understand that requests are not guaranteed - on only limits are. Let's assume a worker node with just 1 core and 3 pods already running. All pods have a 250 millicores request and a 500 millicores limit. Kubernetes may schedule a forth pod on the same node, even if the 3 existing pods were all consuming each 333 millicores, well above their request. Actual cpu availability would be determined by the CFS and the behavior of the other processes running on the same node.

The next observation we can make is that resource availability is dependent on all processes defining appropriate requests, not just one good citizen. In the above example, if all 3 existing pods were running at 333 millicores most of the time, and they had requested 333 instead of 250 millicores, the forth pod would have never been scheduled on that same host, leaving it to battle with the CFS to grab the 250 it requires. It is not enough for a particular process to define appropriate values, all workloads need to play nice:

* Under-evaluating requests, will lead to poor placement decisions, and resource availability lower than requested.
* Over-evaluating requests (setting requests above actual usage) will lead to under-scheduled nodes and low resource efficiency.

# Kubernetes Quality of Service Classes (QoS)

Kubernetes does not force all container in all pods to define appropriate values, or even values at all. But it will treat workloads differently depending on what has been set (or not set).

There are 3 classes defined in Kubernetes for a pod:

* Best effort: the pod does not define cpu requests or limits on any of its pods. Those pods may take valuable resources out of other pods running on the same node.
* Burstable: the pod has at least one container with either a memory or a cpu request.
* Guaranteed: every container in the pod have cpu and memory limits. Kubernetes will automatically assign requests that match the limits. For extra predictability, guaranteed pods may be allocated exclusive use of CPU cores with a static CPU management policy.

QoS comes into play when a node runs low on memory, triggering the kubelet to start evicting pods to avoid an OOM. Pods will be evicted in the following order until enough memory could be regained:

* BestEffort and Burstable pods usings resources above their requests
* Guaranteed and Burstable pods using resources below their requests, based on priority

It is important to note that pods are not evicted because of CPU pressure, which is a compressible ressource. This means that lack of cpu requests (such as in BestEffort), or under-evaluated cpu requests will lead to nodes accepting more load than they can handle, which will lead to starved processes and behaviors low in predictability. This is a situation that is very similar to stuffing workloads onto a VM without looking at the individual needs of the different processes, with the difference that traditional workload on VMs is static, whereas Kubernetes placement is dynamic by default. This means that lack of capacity planning on Kubernetes will bring more unpredictability in behaviors, compared to traditional workload running on VM.

# Node selection in Kubernetes

By default Kubernetes will do placement on resource requirements (i.e. requests) and natural spreading across nodes. There are times however when you may want to be more specific about what should go where. There are 2 sets of features available in Kubernetes:

* Assigning pods to nodes:
    * node selector
    * node affinity
* Restricting pods from running on particular nodes:
    * taints and tolerations
    * node isolation/restriction

With those features, it is possible to force specific workload to run on specific nodes, or conversely prevent some workload from running on particular nodes. When those features are used together, we have a way to dedicate a specific type of workload to a specific set of nodes.

This may be useful for different reasons:

* Security
* Access to specific hardware (e.g. cpu, storage)
* Regulatory
* Isolation

Isolation could work in different ways:

* Protect pods from workload that is unpredictable or did not go through the effort of carefully describing its resource requirements (e.g. best effort workload)
* Dedicate nodes to best effort workloads, and achieve capacity planning at the node level, rather than at the pod level. When a set of dedicated nodes have max-ed out their resources usage (e.g. 80% of either cpu or memory), just span new dedicated nodes. This will relieve the application team from doing extensive capacity planning on their workload, while maximizing resource usage, which is here the primary driver for allocation. However, this approach will probably lead to behavior unpredictability because Kubernetes spreading looks at the number of pods per node, rather than the actual resource usage. You could end up with 10 cpu-intensive pods on node 1, and 10 idle pods on node 2, leading to a lot of wasted resources. For that reason, this strategy works only if workload exhibits similar resource consumption patterns. An example could be to fit all jvm workloads on a set of dedicated nodes.

With isolation, you can sort of replicate the approach that was used on traditional workload, while retaining the automatic spreading offered by Kubernetes, move pods through eviction in case of maintenance or host saturation, ability to scale easily nodes. This limits however the value you get from Kubernetes, which can be smarter about placement if resource requirements are expressed through requests.

Another drawback of this approach is cluster fragmentation when nodes need to be dedicated according to several axes, such as security on top of technology for instance. You could end up with nodes dedicated to jvm workload with a security sensitivity vs regular jvm workload. If you multiply the axes of workload specilization, you may end up with a large cluster, and a tiny set of nodes for each corner case.

# Assessing resource requirements

A key aspect of working with Kubernetes compared to traditional workload is specifically the ability to describe resource requirements, which allow to make better informed decisions in terms of placement. Without it, you can still run in Kubernetes, but you are missing a lot of its value.

What are the resource requirements for your application?

First it is worth making a distinction between memory and cpu, simply because the effects are very differents if you are running short of any of them:

* If you are running out of memory, the pod will get killed and restarted, leading to interruption of in-flight activity.
* If you are running out of cpu, your containers will be throttled by the CFS, leading to slowness.

For that reason, it is a good idea to not over-commit memory, that is setting a memory limit equal to the memory request. This setting should be enough for the application to run for its entire normal life. If you set a limit different than the request, you risk getting the pod killed if it is trying to increase its memory usage, when the node is near OOM. For that strategy to be effective, you have to make sure also that your containers are able to give memory back. Until Java 11, the JVM would only consume additional memory, and never give it back, even if parts of it were not actually used, for instance in the heap. G1 was improved in the context of JEP 346 to allow returning to the OS unused committed memory. ZGC followed in Java 13 through JEP 351. Shenandoah might even be a better option, since it was backported to Java 11. VMWare clusters are licensed by the amount of allocated memory. If the business and operational objectives can be met with a non guaranteed memory, then this may be an option to lower the cost of the infrastructure. Note however that most of the products are priced using cpu allocation rather than memory. So the effect on the TOC will be limited.

If you want to stay on the safe side, and assuming your application does not hve a memory leak, assessing memory requirements is a matter of running a simulation of the expected load on the targetted service, with the target application configuration, and make sure the container does not go OOM. When you found your memory requirement, make it a request=limit, which will guarantee the resource availability, and avoid altogether the risk of getting a node OOM.

For cpu you have 3 choices, intituively mapping to the 3 Kubernetes QoS:

* Not defining cpu requirements (aka Best effort)
* Using a limit equal to the request (aka Guaranteed)
* Something in the middle (aka Burstable)

The more you define precise requirements on your application, the better will be the predictability of its behavior. But this may also lead to a lower cpu efficiency, since unused resources reserved by a specific process will not be usable by neighboring workloads. Guaranteed cpu makes sense when you cannot compromise on predictability, or if your application has a continuous resource consumption near its limit.

For many workloads however, we will want to find the best compromise between behavior predictability and resource usage efficiency.





# References

[Kubernetes Resources Management – QoS, Quota, and LimitRange](https://www.cncf.io/blog/2020/06/10/kubernetes-resources-management-qos-quota-and-limitrangeb/)

[Performance Best Practices for
Kubernetes with VMware Tanzu](https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/performance/vsphere-tanzu-kubernetes-perf.pdf)

[Understanding Linux Container Scheduling](https://engineering.squarespace.com/blog/2017/understanding-linux-container-scheduling)

[Everything you Need to Know about Kubernetes Quality of Service (QoS) Classes](https://www.replex.io/blog/everything-you-need-to-know-about-kubernetes-quality-of-service-qos-classes)

[Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

[JEP 346: Promptly Return Unused Committed Memory from G1](https://openjdk.java.net/jeps/346)

[JEP 351: ZGC: Uncommit Unused Memory (Experimental)](https://openjdk.java.net/jeps/351)

[Release memory back to the OS with Java 11](https://thomas.preissler.me/blog/2021/05/02/release-memory-back-to-the-os-with-java-11.html)


[Understanding resource limits in kubernetes: cpu time](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)

[Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

[Understanding container CPU requests](https://docs.openshift.com/container-platform/4.7/nodes/clusters/nodes-cluster-overcommit.html#understanding-container-CPU-requests_nodes-cluster-overcommit)

[Docker cpu resource limits](https://nodramadevops.com/2019/10/docker-cpu-resource-limits/)