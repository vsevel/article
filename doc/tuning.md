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

* the virtualization service provider may not know the type of workload that is going to run on its ESX. As a result it will be difficult for him to figure out if he can be agressive on the overcommitting ratio or should be conservative.
* the VM owner has an expectation on resources given by the provider, and is not necessarily aware of the use, the level or even the impact of the overcommitting ratio configured on the virtualization cluster that its VM is running on.

If we are talking in Kubernetes terminology, the VM owner expresses a resource request, when the VM provider is only offerring a resource limit:

* the VM is guaranteed to not go over the number of vCPU it is configured for
* but unless there is a 1:1 ratio, it is not guaranteed to always be able to use of all these vCPUs at any point in time (this will depend on the behavior of the other VMs sharing the same underlying physical host).

Of course VM owners and providers are encouraged to collaborate to make sure there is a good mutual understanding. However, the expectation of the VM owner on one side, and partial workload knowledge for the provider, will end up usually in conservative choices, such as a low overcommitting ratio. As a result, resources may stay idle, leading to inefficiencies, most likely resulting in suboptimal investements (e.g. paying for cores that you don't use).

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

The next observation we can make is that resource availability is dependent on all processes defining appropriate requests. In the above example, if all 3 existing pods were running at 333 millicores most of the time, and they had requested 333 instead of 250 millicores, the forth pod would have never been scheduled on that same host, leaving it to battle with the CFS to grab the 250 it requires. It is not enough for a particular process to define appropriate values, all workloads need to play nice. Under-evaluating requests, will lead to poor placement decisions, and resource availability lower than requested. Conversely, Over-evaluating requests (setting requests above actual usage) will lead to under-scheduled nodes and low resource efficiency.

# Kubenetes Quality of Service Classes (QoS)




# References

[Kubernetes Resources Management – QoS, Quota, and LimitRange](https://www.cncf.io/blog/2020/06/10/kubernetes-resources-management-qos-quota-and-limitrangeb/)

[Performance Best Practices for
Kubernetes with VMware Tanzu](https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/performance/vsphere-tanzu-kubernetes-perf.pdf)

[Understanding Linux Container Scheduling](https://engineering.squarespace.com/blog/2017/understanding-linux-container-scheduling)

[Everything you Need to Know about Kubernetes Quality of Service (QoS) Classes](https://www.replex.io/blog/everything-you-need-to-know-about-kubernetes-quality-of-service-qos-classes)