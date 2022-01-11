# Managing CPU and Memory in OpenShift

January 11th, 2022 | by Ryan Devlin

[![Gauges](https://github.com/ryandevlin-redhat/ResourcesBlog/blob/main/AirplaneDashboard.jpg "Capacity planning can be complicated.")](#)

*Capacity planning can be complicated, courtesy of [Arie Wubben](https://unsplash.com/@condorito1953?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/MHIw0nSxCR4)*
## Introduction
Requests and limits are mechanisms in OpenShift that are often overlooked and poorly understood. Ironically, they are critical not only for achieving optimal performance in a cluster, but also for extracting the greatest dollar value from the underlying hardware. When properly leveraging requests and limits, organizations can achieve excellent performance and scalability; therefore it is important that time is spent understanding these mechanisms in depth.

## Requests
Requests are the guaranteed resources a container or pod will be allocated at any time. The container/pod may use less than this value, but the scheduler will always assume this % of the total node resources are claimed by the container/pod. It is critical to understand that the value of a request is directly factored into the scheduling algorithm. CPU requests behave differently than memory requests. These differences are outlined below.

### CPU Requests
CPU requests are the amount of CPU a container or pod is guaranteed to run with, measured in millicores. 1 core = 1000 millicores = 100ms of computing time every 100ms of real-world time. Basically millicores are fractional divisions of CPU compute time. The container/pod may use less than this value. The container/pod also is free to use infinitely more CPU than it requested, unless there is a CPU limit added to it. What’s tricky about CPU requests, is that each request contributes to a container/pod’s cpu_shares value. ***This is a fractional weight relative to other requests on a node which determines how excess CPU is distributed [(1)](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)***.

The cpu_shares value is calculated as follows (Note: the request value is measured in CPU cores, this algorithm guarantees at least 2 for the cpu_shares value):
```
	max(spec.containers[].resources.requests.cpu * 1024, 2)
```
**As an example, if a pod requests 500 millicores of CPU we have:**
```
	max(500m * 1024, 2) = max(0.5 * 1024, 2) = 512
```
The implications of this are that requests directly correlate to a weight, the cpu_shares value. In periods of high CPU contention, the cpu_shares weight is used to determine how CPU time is distributed to the pods.

**For example:**
```
Pod #1 requests 1 core
Pod #2 requests 500m
Pod #3 requests 500m
```
**Therefore:**
```
cpu_shares_1 = max(1000m * 1024, 2) = max(1 * 1024, 2) = 1024
cpu_shares_2 = max(500m * 1024, 2) = max(0.5 * 1024, 2) = 512
cpu_shares_3 = max(500m * 1024, 2) = max(0.5 * 1024, 2) = 512
```
The majority of the time, some of the pods idle while others run. In this case any of the running pods are free to use up the excess CPU at will. One day in the future, all three pods try to burst up to 100% CPU usage. The CPU is divided between the pods as follows [(2)](https://docs.docker.com/engine/reference/run/#cpu-share-constraint):

<pre>
<b>Pod #1 gets:</b>  1024 / (1024 + 512 + 512) = 0.5 = 50% of the total CPU time available
<b>Pod #2 gets:</b>  512 / (1024 + 512 + 512) = 0.25 = 25% of the total CPU time available
<b>Pod #3 gets:</b>  512 / (1024 + 512 + 512) = 0.25 = 25% of the total CPU time available
</pre>

In a multicore system, the process is the same, but the % of CPU time received is spread over multiple cores. Thus on a node with 8 cores, the pod #2 and #3 could each use 100% of a CPU core because this is less than their cpu_shares value of 25% of the total available compute resources.

What this shows is that ***the requests control into how much CPU time containers will receive in times of cpu contention***. It is important to understand that the weighting is relative to all the other containers on the node. Thus, pods requesting large amounts of CPU will dominate pods requesting smaller values of CPU during periods where there is a competition for CPU time.

### Memory Requests
Memory requests are the amount of memory a container or pod is guaranteed to have available for use. The container/pod may use less than this value. It may also infinitely exceed this value unless there is a memory limit in place to cap it’s consumption. ***Once a container/pod exceeds its memory request OpenShift can no longer guarantee that the node has the memory capacity for the pod***. If the memory on the node is exhausted (ie. from several other pods exceeding their requests), there is a possibility that the pod will be terminated. The factor that determines which pods are terminated in a low memory situation is called the Quality of Service class. Pods are prioritized based on the table shown here, which is covered in more detail below. If a limit is in place on the pod, and the pod exceeds that limit, it will be immediately terminated. This behavior is due to the fact that, unlike CPU, ***memory cannot be throttled***.

## The Scheduling Process
Before we cover how OpenShift limits work, it is important to discuss the scheduling process for pods. This is because, as stated above, requests in OpenShift directly influence how pods are scheduled in a cluster. **Figure 1** illustrates the Kubernetes scheduling algorithm [(3)](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/). Don’t worry about anything other than the filtering section.

[![Scheudling Algorithm](https://github.com/ryandevlin-redhat/ResourcesBlog/blob/main/SchedulingDiagram.png "Kubernetes scheduling algorithm.")](#)

*__Figure 1:__ The Kubernetes Scheduling Algorithm*

It is in the Filtering stage of the scheduling cycle that ***the set of nodes available for pod placement is reduced by applying several filtering plugins***. One of these plugins is called **`NodeResourcesFit`** which analyzes each node to determine if it has the capability to support the resources the pod has requested. This includes scalar resources such as counts of certain kubernetes objects, but more importantly, it checks for availability of the hardware resources CPU, memory, and storage. Since OpenShift and Kubernetes are open source, we can see exactly how this works [(4)](https://github.com/openshift/kubernetes/blob/68104e8f731c05015b0581c849f11f170a2e8855/pkg/scheduler/framework/plugins/noderesources/fit.go#L251).
<p align="center">
  <img src="https://github.com/ryandevlin-redhat/ResourcesBlog/blob/main/podRequestCode.png" />
</p>

*__Figure 2:__ The NodeResourcesFit filter applied to nodes during scheduling*

Translated to English, the conditional statements above say:
<p align="center">
  <img src="https://github.com/ryandevlin-redhat/ResourcesBlog/blob/main/StyledTextBox.png" />
</p>

Large requests reserve broad chunks of resources, regardless of whether or not those resources are utilized by the pods that requested them.

## Why Do We Care?
The implications of the above information are that ***requests attached to pods or containers in a cluster have a direct effect on the density of pods on a node and the resource utilization on that node***. Each request made in a cluster will carve out a piece of the resource pie on the node. ***Large requests reserve broad chunks of resources, regardless of whether or not those resources are utilized by the pods that requested them***. It is this concept that makes it crucial for developers and administrators to work together to come up with appropriate request values for an application that reconcile nicely with the context of the cluster they are running in.
	
## Limits
Limits are the *maximum* resources a container can use at any time. The container may use less than this value, but it may never use more than this value. It is important to understand that trying to exceed limits set on containers will have different effects on the application depending on the type of limit. These differences are outlined below.

### CPU Limits
CPU limits are the maximum CPU, measured in millicores, that a container may use. In the same fashion as requests, millicores are a fraction of CPU time where 1 core = 1000 millicores = 100ms of computing time every 100ms of real-world time. While a container will not be killed for exceeding its limits, it will be prevented from doing so for extended periods of time via throttling [(5)](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run).

### Memory Limits
Memory limits are the maximum amount of memory a container can use at any time. As stated before, because it is not possible to throttle memory, any container exceeding its memory limits will be immediately terminated.

### Overcommitted Nodes
Limits relate to a Kubernetes concept called ***overcommitment***. This is a state that manifests from one of the following occurrences:

1. **A pod that specifies no requests has been scheduled onto the node**
2. **The sum of the limits for all the pods on the node exceed the capacity for a resource on the node**

Essentially, being in an ***overcommitted state*** means there is a possibility that all resources of a specific type on the node may be exhausted in the future. If the overcommitted resource is ever exhausted, then pods will be prioritized based on their [Quality of Service class](https://docs.openshift.com/container-platform/4.9/nodes/clusters/nodes-cluster-overcommit.html#nodes-cluster-overcommit-qos-about_nodes-cluster-overcommit) as mentioned earlier. Pods are designated a QoS class based on the following table.

| Priority            | Class Name      | Description |
| ------------------- | --------------- | ------------|
| 1 (highest)         | **Guaranteed**  | If limits and optionally requests are set (not equal to 0) for all resources and they are equal, then the container is classified as **Guaranteed**. |
| 2                   | **Burstable**   | If requests and optionally limits are set (not equal to 0) for all resources, and they are not equal, then the container is classified as **Burstable**. |
| 3 (lowest)          | **Best Effort** | If requests and limits are not set for any of the resources, then the container is classified as **BestEffort**. |

*__Figure 3:__ Quality of Service Classes*

The key takeaway here is that, based on how requests and limits are set on pods, nodes can enter overcommitted states. Further, when a resource (mainly CPU or memory) is exhausted on a node, OpenShift will begin to target pods for elimination to avoid OOM errors. It will target these pods for termination based on their priority which is determined by their QoS classification, which is directly derived from their requests and limits. This is another reason why setting requests and limits is critical for successful cluster operation.

## How Is This Managed?
At this point it should be clear not only how requests and limits work, but also why they’re important to your cluster. By properly specifying these values, organizations can pull optimal performance from their hardware and save a significant amount of money. Unfortunately simply setting and forgetting these values on pods is usually not enough to actually achieve the performance gains most organizations desire. Workloads are subject to change not just at the individual pod level, but also as new applications are onboarded or removed from the cluster. In order to properly rein in the applications in a cluster, one must rely on OpenShift administrative tooling as well as metrics gathered from applications.

### Resource Quotas
A Resource Quota in OpenShift is an administrative object that places limitations on the consumption of resources in an OpenShift cluster. Resource Quotas can be scoped on a per-project basis, across several projects, or across an entire cluster. Resource quotas can limit many different kinds of resources. They can be used to put caps on the number of Kubernetes objects (secrets, configMaps, pods, etc.) created in projects or across the cluster. Similarly they can limit the sum total of CPU or memory requests/limits in a project or across projects. More detailed information about resource quotas can be found [here](https://docs.openshift.com/container-platform/4.6/applications/quotas/quotas-setting-per-project.html).

### Limit Ranges
A Limit Range in OpenShift is an administrative object that is typically used to place limitations on resource consumption at the resolution of an individual container or pod. Limit ranges are often used by administrators to specify a range (min and max values) for CPU and memory requests and/or limits placed on containers and/or pods. Limit ranges can also specify a default value for requests and limits on containers or pods in a namespace. More detailed information about limit ranges can be found [here](https://docs.openshift.com/container-platform/4.6/nodes/clusters/nodes-cluster-limit-ranges.html).

### Recommendations From the Vertical Pod Autoscaler
The Vertical Pod Autoscaler Operator can help developers determine the optimal request and limit settings for their pods by watching current and historic resource consumption on pods. The VPA can be configured to automatically redeploy pods with new requests and limit values, or it can simply supply recommendations and rely on developers to manually make the change. For more information about the VPA, see the official documentation [here](https://docs.openshift.com/container-platform/4.9/nodes/pods/nodes-pods-vertical-autoscaler.html).

### Managing Overcommitment
By leveraging clusterResourceQuotas that span several projects, administrators are able to place restrictions on how much CPU and memory an entire application can request. Remembering what was said above, this process gives administrators the power to constrain how much of the “resources pie” in the cluster a specific application is allowed to reserve for itself. Similarly, administrators can use clusterResourceQuotas to place restrictions on the total limits an application can set on itself. By keeping applications from setting their limit ceilings too high, administrators can avoid cluster nodes entering an overcommitted state.

Administrators can also rely on limit ranges added to each project in the cluster to constrain all pods in the cluster to realistic ranges for their requests and limits. Limit ranges also offer the benefit of forcing pods to use default request and limit values, which in turn prevent any pods from being run without requests (which would place a node in an overcommitted state). Finally limit ranges offer administrators the opportunity to specify a maxLimitRequestRatio. This parameter ensures that the ratio of limits to requests is no greater than the value specified. ***A suitable value for maxLimitRequestRatio varies by resource and cluster***. In cases where it is desirable to have many pods burst up CPU usage at certain peak times, the requests and limits on these pods could have a larger delta. If this is the case in a cluster, then the maxLimitRequestRatio for CPU would be a larger value, eg. “10”. If a cluster runs many workloads, and has a lower tolerance for risk of pods being OOM killed, then the maxLimitRequestRatio for memory would be more conservative, eg. “2”.

## Selecting Request and Limit Values
The most difficult aspect of utilizing requests and limits is determining the appropriate values to select for each container, pod, or deployment. While there is no one-size-fits-all solution, there are several steps organizations can take to move closer to their target utilization.

1. Administrators should be responsible for managing node-level commitment. By leveraging resource quotas and limit ranges, it is the job of the administrators to determine the maximum resources that should be carved out for each application. Of course, administrators cannot simply guess the resources an application will need so they must work with the development teams to understand their requirements and set ceilings appropriately. Development teams sometimes may request more resources than they need. After all, their job is to create working applications, not worry about cluster utilization. This is why it is desirable to have development teams supply administrators with application profiling or metrics data, demonstrating the ***peak and mean resource consumption*** of their application.

2. Determine the risk appetite for eviction. If resources are plentiful and the concern is more closely related to application performance, then your risk appetite is high. In this case, requests for resources can be set closer to peak application requirements. If resources are scarce, or the organization is more concerned about the potential for pods to be evicted, then the risk appetite is low. In this case, requests should be set closer to mean application requirements. Again, it’s possible that development teams know their applications well enough to understand what these values are, but ***it is a best practice to back these assumptions with empirical data***.

3. As mentioned previously, the Vertical Pod Autoscaler Operator can assist with finding optimal recommendations for request and limit values.

4. ***CPU can be throttled***. This means that CPU can be allowed to burst up with limits that are more liberal than those needed for memory. A common use case for this is when using Java-based containers. Oftentimes these applications require a lot of CPU to start up, but their steady-state, mean CPU usage is in fact much lower during their runtime. The best way to manage these types of applications is to set the CPU requests on their pods close to, or slightly above their mean CPU usage. Then set the CPU limits on their pods to a more liberal value, such as 2 cores. ***By keeping CPU request values low, the pods are guaranteed the CPU they need to continuously run (their mean CPU usage), but they don’t hoard 2 cores for a workload that only needs 500m to continuously run***. By having CPU limits higher, these pods are allowed to burst upwards on startup and use available CPU on the node to start quickly.

5. ***Memory cannot be throttled***. Thus one must be more careful about how requests and limits are set on memory. In terms of requests, these should be determined by your organization's risk appetite and the empirical data gathered from load testing. If the requests are set too low, the application risks being targeted for eviction when it exceeds it’s requests. ***If requests are set too high, then the application hoards memory it won’t use, and your organization will waste money***. Memory limits allow for a “fail fast” approach to resource management, alerting development teams quickly of the memory ceiling for their application. If limits are set on memory, they should be set above the expected peak memory for those pods/containers, with a margin of safety to avoid pod eviction. Out of the scope of this document, but still relevant to the conversation, is a note about ensuring applications are properly tuned. When applications pool memory, such as with the JVM, tuning is critical.

## Conclusion
There is by no means a “silver bullet” to optimize cluster utilization and performance, but there are paradigms and philosophies organizations can follow that will guide them to discover a solution that best fits their unique situation. Solving this problem comes down to the relationship between the application and infrastructure teams, and their ability to organize around this goal.
