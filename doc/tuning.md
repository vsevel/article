This article aims providing a good understanding of container resource sizing in kubernetes, including (but not limited to) workloads that exhibit a resource spike at startup (such as SpringBoot).

This is a "Work in Progress". Important topics have yet to be covered, such as:

* Recommendation engines: [Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler), [Harness](https://docs.harness.io/article/ikxjmkqi03-recommendations)
* [Memory swap](https://kubernetes.io/blog/2021/08/09/run-nodes-with-swap-alpha/) in Kubernetes 1.22
* Scaling out: How to mix Horizontal Pod Autoscaling with VPA

# From bare metal to virtualization

Running multiple workloads on bare metal requires to share resources between the different processes on a given host. Processes are not protected against each other (for CPU, it depends on the OS scheduler). If a process decides at some point to go through a lot of processing, it will increase its usage on, say memory and cpu, possibly starving the other processes running on the same host. This might have an impact on these other processes, which might stop meeting their service level objectives. Conversely, the application requiring more resources might fail absorbing its peak of activity, if the other applications running on the host are themselves in a phase where they need additional resources.

Several strategies can be used to limit the impact or probability of this happening:

* Isolation: move workload that needs a higher level of guarantee, or is likely to be a bad neighbor to dedicated hosts.
* Carefully and progressively add new workload, making sure "it fits in".
* Capacity planning: rely on observability (monitoring and metrics) to figure out if a VM has space left for new workload. If not, optionally offload some workload to other hosts.

One aspect that has worked well with traditional workload is that it is static: it is either installed on the host, or not. Obviously, usage can be dynamic, but new workload does not appear suddenly. It is the result of provisioning. This makes it somehow more manageable and predictable than if workloads came and left as it pleased.

But one fundamental characteristic of running workload on bare metal is that unused resources (such as memory and CPU) get lost. As the industry shifted to virtualization, people quickly realized that beside agility, running workload on VMs provided a way to not waste those resources through overprovisioning. However there are 2 challenges with this:

* The virtualization service provider may not know the type of workload that is going to run on its ESX. As a result it will be difficult for him to figure out if he can be agressive on the overcommitting ratio or should be conservative.
* The VM owner has an expectation on resources given by the provider, and is not necessarily aware of the use, the level or even the impact of the overcommitting ratio configured on the virtualization cluster that its VM is running on.

The only guarantee that virtualization service providers will offer is that VMs will not be able to go over the number of vCPU it is configured for. But unless there is a 1:1 ratio, it is not guaranteed to always be able to use of all these vCPUs at any point in time (this will depend on the behavior of the other VMs sharing the same underlying physical host).

Of course VM owners and providers are encouraged to collaborate, to make sure there is a good mutual understanding. However, the expectation of the VM owner on one side, and partial workload knowledge for the provider on the other, will end up usually in conservative choices, such as a low overcommitting ratio. As a result, resources may stay idle, leading to inefficiencies, most likely resulting in suboptimal investments (e.g. paying for cores that you don't use).

Still, overcommitting at the virtualization layer should be an implementation detail, whose responsibility lies into the hands of the infrastructure team, which will treat its cluster as a whole, trying to maximize efficiency of resources, to lower the cost of the vcpus, but with a primary focus on making available the requested resources (i.e. near-guaranteed resources). The corollary, is that virtualization overcommit should not be used as a substitute to proper capacity management at the application level.

# Running workload in containers

Traditional workload offer little protection in terms of resource access, or ability to limit noisy neighbors. Container technology was introduced more than 10 years ago, and offered significant improvements by exposing some features available in linux cgroups.

Containers offer the added ability of configuring a resource access and limit options: `-m` for memory and `--cpus`, `--cpu-period`, `--cpu-quota` and `--cpu-shares` for cpu.

The `-m` memory parameter is a limit that the container cannot go over. Since memory is not a compressible resource, if the container tries to go over, it will go OOM and get killed.

The cpu shares defines a proportional weight for host cpu access, relative to the other containers. The default value in docker is 1024, but it is important to note that this is not equal to a number of millicores. Let's take an example to illustrate:

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

Now let's run the same container with a cpu shares of `150` and another container with `50`:
```
docker run -it --rm -d --cpu-shares=50 --name=container_50 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
docker run -it --rm -d --cpu-shares=150 --name=container_150 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
```

We can see that the `50` container is using `25%` of the total capacity (i.e. `50/200`), and the other container uses the other `75%`. The total `200%` because we are using a 2 cores docker host:
```
NAME            CPU %     MEM USAGE / LIMIT     MEM %
container_50    49.94%    8.988MiB / 7.776GiB   0.11%
container_150   151.81%   8.992MiB / 7.776GiB   0.11%
```

Without waiting for the 2 containers to stop, launch a third container with cpus shares equal to `300`:
```
docker run -it --rm -d --cpu-shares=300 --name=container_300 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
```

Stats now show:
```
NAME            CPU %     MEM USAGE / LIMIT     MEM %
container_50    19.99%    8.984MiB / 7.776GiB   0.11%
container_150   61.01%    8.965MiB / 7.776GiB   0.11%
container_300   122.93%   8.969MiB / 7.776GiB   0.11%
```

We can see that the 2 first containers were readjusted to consume a fraction of the total number of shares defined on all containers:

* `50/500: 10% of total capacity (i.e. 2 cores)`
* `150/500: 30%`
* `300/500: 60%`

The results would have been the same if shares had been `500`, `1500` and `3000`, instead of `50`, `150`, and `300`. So whatever the values may be, they need to be consistent with one another.

CPU shares are used when the system has contention (i.e. processes ask for more than the total capacity). Without contention, processes are free to use whatever capacity is available on the host. As a result CPU shares cannot be used to limit access to cpu, but only to guarantee some access to it. Recognizing this as an issue, Paul Turner, Bharata B. Rao and Nikhil Rao introduced the _CPU bandwidth control for CFS_ in the 2010 Linux Symposium, as a mean to limit access to cpu for processes: cpu quota and cpu period.

The cpu period is by default 100000 microseconds (i.e. 100 ms). The cpu quota defines the number of microseconds per cpu periods that the container is limited to. In docker, `--cpu-period="100000" --cpu-quota="150000"` means that the process is limited to `1.5` cpus, which can be expressed more conveniently with `--cpus="1.5"` in docker `1.13`.

Run again the 2 first as before, and add a limit of 900 millicores on the third one:
```
docker run -it --rm -d --cpu-shares=50 --name=container_50 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
docker run -it --rm -d --cpu-shares=150 --name=container_150 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
docker run -it --rm -d --cpu-shares=300 --cpus=0.9 --name=container_300 alexeiled/stress-ng --cpu 2 --timeout 60s --metrics-brief
```

The stats show:
```
NAME                CPU %               MEM USAGE / LIMIT     MEM %
container_50    26.96%    8.996MiB / 7.776GiB   0.11%
container_150   80.76%    8.973MiB / 7.776GiB   0.11%
container_300   90.58%    8.969MiB / 7.776GiB   0.11%
```

In terms of millicores, the allocation is:

* `container_50`: `27/200*2000 = 270m`
* `container_150`: `81/200*2000 = 810m`
* `container_450`: `91/200*2000 = 910m`

The first thing we see is that `container_300` is effectively limited to roughly `900m`. But where do the limits come from for the 2 other containers?

When `container_300` reaches its limit, the other containers had received:

* `container_150`: half of `container_300`, which is roughly `450m`.
* `container_50`: a third of `container_15`, which is `150m`.

So we end up with `900 + 450 + 150 = 1500m` being used out of a total capacity of `2000m`. This leaves `500m` to be shared between `container_50` (25%) and `container_150` (75%):

* `container_50`: `0.25*500 + 150 = 275m`
* `container_150`: `0.75*500 + 450 = 825m`

As we can see, the container runtime engine provides options to a guarantee (through the cpu shares) and limit (through the cpu quota) access to CPU. Guaranteed access does not mean that cpu cycles will go wasted however if not used. If a container reserves 1 core, but stay idle, those cpu cycles will go to a global shared pool, and get redistributed to whoever needs them by the CFS.

These mechanisms offer a huge improvement compared to running workload on VMs, as now it is possible to tune access to resources. They can be used to define lower and upper bounds for cpu usage, and help with different scenarios:

* Provide dedicated resources, and limit the process to the resources.
* Provide a minimal amount of guaranteed resources, and define an upper bound allowing the process to go above the minimal amount.
* Provide no minimal amount, and define an upper bound.

This brings a lot of flexibility. However, running static containers on hosts, can be tedious (e.g. translating cpu shares into millicores) and workload cannot be moved easily. For that reason, the industry has started working on container orchestrators, to allow running containers at scale, while allowing resource efficiency, and ease of resource configuration for the individual workloads.

# Running workload in Kubernetes

One of the fundamental goal of Kubernetes is to be able to place workload dynamically, while preserving efficiency on resource usage (i.e. avoid wasting available resources). As we have seen, dynamic placement is only possible if a given workload is providing information about its resource usage. Kubernetes introduces `request` and `limit`:

* `request`: the minimum amount of resource made available to the container
* `limit`: the maximum amount of resource made available to the container

As we have seen previously, at the linux kerne level, there are only 2 mechanisms:

* cpu shares to guarantee a relative distribution of cpu cycles
* cpu quota to throttle access to resources above a certain threshold

There is actually a direct relationship between the Kubernetes request and limit, and the underlying linux features. Let's take this pod for instance:

```
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  containers:
  - name: hello
    image: alexeiled/stress-ng
    args: ['--cpu', '1', '--timeout', '60s']
    resources:
      requests:
        cpu: "0.3"
      limits:
        cpu: "0.7"
```

We can display the associated shares and quota that have been defined on the container:
```
$ docker inspect 848589738a9f --format 'shares={{.HostConfig.CpuShares}} quota={{.HostConfig.CpuQuota}} period={{.HostConfig.CpuPeriod}}'
shares=307 quota=70000 period=100000
```

Kubernetes will calculate shares using: `0.3 * 1024`. This is the reason why we have `307` and not `300`.
The quota is calculated using: `0.7 * 100000`.

If requests are not set, then they are automatically set to the limit itself. So in the above example we would have: `shares=716 quota=70000`. 

If there are no requests or limits we end up with: `shares=2 quota=0`.

And finally if there are requests but no limits, the container is created with: `shares=307 quota=0`.

If the cfs cpu quota is a straightforward translation of the Kubernetes limit, there is a mismatch in semantic between the cfs cpu share and the Kubernetes request. The former is a relative weight, whereas the later is an absolute quantity of a physical resource.

There are 2 consequences:

* The container gets more guaranteed resources than it thinks.
* The container are not treated equally with respect to allocation when they go above their request.

Let's take a 2 cores hosts running 1 container with 300 millicores request, and another one with 600 millicores. As we have seen they will respectively get 307 (1/3 of total shares) and 614 (2/3 of total shares) shares. If some contention arises, the first container will be guaranteed 1/3 of 2 cores: 667 millicores. The 2nd one will receive 1333 millicores, far more than what was requested. As the host is filling up with containers, the requests and the shares will look more and more alike. Let's say we are running a 3rd container with 1100 millicores request, filling up our 2-cores host. Then the shares for the 3 containers will be `307/614/1126` for a total of `2047` shares, and the guaranteed amount of cpu will be `300/600/1100`.

If the sum of requests of all pods is equal to the exact host capacity, and all pods are asking to consume their requests or more, they will all get exactly their requests in millicores. In all other situations they will get more than their requests, which will happen actually most of the time.

The Kubernetes request gives the false impression that this represents a lower bound, and after that all containers are treated equally. As we have seen, allocation is driven by the number of shares given to the container. In the first example where we have 2 containers with `300` and `600` shares respectively, 2 cpu cycles will be offered to the 2nd container if needed, when only one will be dedicated to the first one. If the 2nd container does not get limited, the first one will only be able to grab an additional 367 millicores on top of the 300 requested (33% of the remaining 1100), when the 2nd one will be able to grab 733 on top of the 600 requested (67% of the remaining 1100). We can see that the ratios `1/3` and `2/3` are respected above the configured requests.

A direct consequence is that containers with large requests, will tend to be privileged over low request containers when contention arises. Take one application deployed 2 times in 2 different pods on the same 2-cores host. If the first pod is configured with 300 request, and the other one 600, the second instance of the application will be largely privileged over the first one during peaks. If the first team has done a good job at tuning the application when coming up with a 300 request, and the other team has done a loosy job on the 600, the loosy team will be favored over the serious one.

It is worth noting that limits are a way to reintroduce some fairness in the distribution of cpu cycles. In the prevous example, if the containers had been run with a limit of 1 core, then both containers would burst up to 500 and 1000 millicores. At that point container 2 would get throttled, and container 1 would be free to grab the remaining 500, ending up with both consuming 1000 millicores in the end.

As a result, it is recommended to:

* Install a culture of sizing requests and limits appropriately in all teams.
* Monitor those settings and retrofit more appropriate settings regularly (monthly, or ideally weekly).
* Generalize the use of limits to introduce some fairness in the way the host capacity is distributed among the different processes.

It is important to understand that Resource availability is dependent on all processes defining appropriate requests, not just one good citizen. It is not enough for a particular process to define appropriate values; all workloads need to play nice:

* Under-evaluating requests, will lead to poor placement decisions, and resource availability lower than requested, plus non fair access to the extra capacity, which will be distributed to high requestors in a proportional manner during bursts.
* Over-evaluating requests (setting requests above actual usage) will lead to under-scheduled nodes and low resource efficiency.

# Kubernetes Quality of Service Classes (QoS)

Kubernetes does not force all container in all pods to define appropriate values, or even values at all. But it will treat workloads differently depending on what has been set (or not set).

There are 3 classes defined in Kubernetes for a pod:

* Best effort: the pod does not define cpu requests or limits on any of its pods. Those pods may take valuable resources out of other pods running on the same node.
* Burstable: the pod has at least one container with either a memory or a cpu request.
* Guaranteed: every container in the pod have cpu and memory limits. Kubernetes will automatically assign requests that match the limits. For extra predictability, guaranteed pods may be allocated exclusive use of CPU cores with a static CPU management policy.

QoS comes into play when a node runs low on memory, triggering the kubelet to start evicting pods to avoid an OOM. Pods will be evicted in the following order until enough memory could be regained:

* BestEffort and Burstable pods usings resources above their requests.
* Guaranteed and Burstable pods using resources below their requests, based on priority.

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

* Protect pods from workload that is unpredictable or did not go through the effort of carefully describing its resource requirements (e.g. best effort workload).
* Dedicate nodes to best effort workloads, and achieve capacity planning at the node level, rather than at the pod level. When a set of dedicated nodes have max-ed out their resources usage (e.g. 80% of either cpu or memory), just span new dedicated nodes. This will relieve the application team from doing extensive capacity planning on their workload, while maximizing resource usage, which is here the primary driver for allocation. However, this approach will probably lead to behavior unpredictability because Kubernetes spreading looks at the number of pods per node, rather than the actual resource usage. You could end up with 10 cpu-intensive pods on node 1, and 10 idle pods on node 2, leading to a lot of wasted resources. For that reason, this strategy works only if workload exhibits similar resource consumption patterns. An example could be to fit all jvm workloads on a set of dedicated nodes.

With isolation, you can sort of replicate the approach that was used on traditional workload, while retaining the automatic spreading offered by Kubernetes, move pods through eviction in case of maintenance or host saturation, ability to scale easily nodes. This limits however the value you get from Kubernetes, which can be smarter about placement if resource requirements are expressed through requests.

Another drawback of this approach is cluster fragmentation when nodes need to be dedicated according to several axes, such as security on top of technology for instance. You could end up with nodes dedicated to jvm workload with a security sensitivity vs regular jvm workload. If you multiply the axes of workload specilization, you may end up with a large cluster, and a tiny set of nodes for each corner case.

# Assessing resource requirements

A key aspect of working with Kubernetes compared to traditional workload is specifically the ability to describe resource requirements, which allow making better informed decisions in terms of placement. Without it, you can still run in Kubernetes, but you are missing a lot of its value.

What are the resource requirements for your application?

First it is worth making a distinction between memory and cpu, simply because the effects are very differents if you are running short of any of them:

* If you are running out of memory, the pod will get killed and restarted, leading to interruption of in-flight activity.
* If you are running out of cpu, your containers will be throttled by the CFS, leading to slowness.

For that reason, it is a good idea to not over-commit memory, that is, setting a memory limit equal to the memory request. This setting should be enough for the application to run for its entire normal life. If you set a limit different than the request, you risk getting the pod killed if it is trying to increase its memory usage, when the node is near OOM. For that strategy to be effective, you have to make sure also that your containers are able to give memory back. Until Java 11, the JVM would only consume additional memory, and never give it back, even if parts of it were not actually used, for instance in the heap. G1 was improved in the context of JEP 346 to allow returning to the OS unused committed memory. ZGC followed in Java 13 through JEP 351. Shenandoah might even be a better option, since it was backported to Java 11. VMWare clusters are licensed by the amount of allocated memory. If the business and operational objectives can be met with a non guaranteed memory, then this may be an option to lower the cost of the infrastructure. Note however that most of the products are priced using CPU allocation rather than memory. So the effect on the TOC will be limited.

If you want to stay on the safe side, and assuming your application does not have a memory leak, assessing memory requirements is a matter of running a simulation of the expected load on the targetted service, with the target application configuration, and make sure the container does not go OOM. When you found your memory requirement, make it a request=limit, which will guarantee the resource availability, and avoid altogether the risk of getting a node OOM.

For CPU you have 3 choices, intituively mapping to the 3 Kubernetes QoS:

* Not defining cpu requirements (aka Best effort).
* Using a limit equal to the request (aka Guaranteed).
* Something in the middle (aka Burstable).

The more you define precise requirements on your application, the better will be the predictability of its behavior. But this may also lead to a lower CPU efficiency, since unused resources reserved by a specific process will not be usable by neighboring workloads. Guaranteed CPU makes sense when you cannot compromise on predictability, or if your application has a continuous resource consumption near its limit.

For many workloads however, we will want to find the best compromise between behavior predictability and resource usage efficiency.

# The percentile approach

CPU requests should be set to the minimum amount of resources needed by the application. One approach consists in looking at past behavior and assess requirements based how much the application _most of the time_. In other words, what is the value of `y` in the following sentence for different values of `x`: The application consume less than `y` cores `x%` of the time. This measure is the _nearest-rank_ method of the so-called _Percentile_ function. Compared to average, it has 2 extra benefits:

* Captures volatility, as opposed to average that treats the same [50, 50, 50, 50] and [0, 25, 75, 100].
* Offers a cursor to characterize the `most of the time` variable. Well known percentiles are 99 (e.g. The application consumes less than `y` cores 99% of the time), 90, 95, ... 50 being also know as the median value.

When the requests are calculated using a high percentile (e.g. 90, 95, 99), it lowers the density on worker nodes, which lowers the CPU usage, but brings high predictability on the workload behavior because there is a high probability that it will receive the resources it needs. There will be some gambling on spikes only.

When the requests are calculated using a low percentile (e.g. 65, 70, 75, 80), it increases the pod density on nodes, which improves CPU usage, but brings less predictability for the workload, because a bigger portion of resource usage will have to be provided through overcommit (i.e. the assumption that the other pods leave some room for others to handle their spikes by accessing resources above their requests).

Tuning is about finding the sweet spot between workload resource availability (hence behavior predictability) and CPU usage, which is a measure of resource efficiency.

Research show that even a low percentile (such as 75%) guarantees desired access to resource with a very high probability on typical workload (> 95%), even when the desired level is well above the configured requests.


# References

[1] [Kubernetes Resources Management – QoS, Quota, and LimitRange](https://www.cncf.io/blog/2020/06/10/kubernetes-resources-management-qos-quota-and-limitrangeb/)

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

[How Pods with resource limits are run](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)

[Running Kubernetes and the dashboard with Docker Desktop](https://andrewlock.net/running-kubernetes-and-the-dashboard-with-docker-desktop/)

[Getting Started with Cloud Cost Optimization](https://harness.io/blog/cloud-cost-management/cloud-cost-optimization/)

[VERTICAL POD AUTOSCALING: THE DEFINITIVE GUIDE](https://povilasv.me/vertical-pod-autoscaling-the-definitive-guide/)