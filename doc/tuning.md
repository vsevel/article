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

Compared to workload running on VMs, containers offer the added option of configuring a resource limit (option `-m` in docker for memory and `--cpus` for cpu). The primary benefit, is that we can guarantee that workloads will not go above the configured limits. This is essentially a protection for the other workloads running on the same host. If the host was configured to match the sum of the resource limits of all containers, this will guarantee that each container will be able to use al resources, at any time, up to the limits it configured. But this comes with a steep price: if containers are not running steadily near their limits, those resources will be wasted (unless the host is a VM, and overcommitting is used efficiently, with the limitations it has, as we have seen). This could be an option if resources need to be guaranteed and the associated inefficiencies are balanced by business opportunities, or if the workloads have all a stable resource consumption. But often this will not be the case. As a result, the host will be configured to hosts a bundle of containers, whose resource limit sum is greater than the host capacity; doing at the process level the same overcommitting that virtualization clusters do at the VM level.

Still, this is a huge improvement compared to running workload directly on VMs:

* There is a protection against a faulty process attempting to grab all the host's resources
* The host capacity is easier to assess, because we can reason in terms of overcommitting ratio (e.g. sum of container limits divided by the host capacity), much like VMs resources compared to the physical host capacity.

However, running containers on hosts without an orchestrator, suffers from some of the limitations discussed previously:

* Workload cannot be moved easily
* A great level of attention is required when adding new workload, to make sure the host will have enough capacity
* Conservative choices in (or lack of) capacity planning will lead to inefficient resource usage

Essentially, we would need containers to tell how much resources they wish, if we wanted to solve some of these above issues.

# Running workload in Kubernetes

One of the fundamental goal of Kubernetes is to be able to place workload dynamically, while preserving efficiency on resource usage (i.e. avoid wasting available resources). As we have seen, dynamic placement is only possible if a given workload is providing information about its resource usage. Kubernetes introduces `request` as a mean to express what the workload needs. It is an information that is used by the Kubernetes scheduler when a new pod needs to be assigned to a worker. Once the pod is assigned, the request does not play a role anymore (except in one case discussed later). This contrasts with the `limit`, which is not used for placement (TODO except for the feature that tries to find a node that can currently accomodate the pod's limit), but is used by the container runtime level to configure cgroup quotas.

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