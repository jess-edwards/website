---
layout: blog
title: "A Custom Kubernetes Scheduler to Orchestrate Highly Available Applications"
date: 2020-09-30
slug: custom-k8s-schduler-to-orchestrate-highly-available-apps
---

# A Custom Kubernetes Scheduler to Orchestrate Highly Available Applications

As long as you're willing to follow the rules, deploying on Kubernetes and air travel can be quite pleasant. More often than not, things will "just work". However, if one is interested in travelling with an alligator that must remain alive or scaling a database that must remain available, the situation is likely to become a bit more complicated. It may even be easier to build one's own plane or database for that matter. Travelling with reptiles aside, scaling a highly available stateful system is no trivial task.

Scaling any system has two main components: 
1. Adding or removing infrastructure that the system will run on and 
2. Ensuring that the system knows how to handle additional instances of itself being added and removed. 

Most stateless systems, web servers for example, are created without the need to be aware of peers. Stateful systems, which includes databases like CockroachDB, have to coordinate with their peer instances and shuffle around data. As luck would have it, CockroachDB handles data redistribution and replication. The tricky part is being able to tolerate failures during these operations by ensuring that data and instances are distributed across many failure domains (availability zones).

One of Kubernetes' responsibilities is to place "resources" (e.g, a disk or container) into the cluster and satisfy the constraints they request (e.g, "I must be in availability A" [docs](https://kubernetes.io/docs/setup/best-practices/multiple-zones/#nodes-are-labeled)} or "I can't be on the same machine as this container" [docs](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-isolation-restriction)). As an addition to these constraints Kubernetes' offers [Statefulsets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) which provide identity to pods and persistent disks that "follow" these identified pods. Identity is handled by an increasing integer at the end of a pod's name. It's important to note that this integer must always be contiguous, if pods 1 and 3 exist then pod 2 must also exist.

Under the hood, CockroachCloud deploys each region of CockroachDB as a Statefulset in its own Kubernetes cluster [docs](https://www.cockroachlabs.com/docs/stable/orchestrate-cockroachdb-with-kubernetes.html). We'll be looking at an individual region, one Statefulset and one Kubernetes cluster which is distributed across at least three availability zones. A three-node CockroachCloud cluster would look something like this:

![3-node, multi-zone cockroachdb cluster](RackMultipart20200925-4-151p2tz_html_cd5ed90acb1580db.gif)

When adding additional resources to the cluster we also distribute them across zones. For the speediest user experience, we add all Kubernetes nodes at the same time and then scale up the StatefulSet.

![illustration of phases: adding Kubernetes nodes to the multi-zone cockroachdb cluster](RackMultipart20200925-4-151p2tz_html_f11ec4331d788b65.gif)

Note that anti-affinites are satisfied no matter the order in which pods are assigned to Kubernetes nodes. In the example, pods 0, 1 and 2 were assigned to zones A, B, and C respectively, but pods 3 and 4 were assigned in a different order, to zones B and A respectively. The anti-affinity is still satisfied because the pods are still placed in different zones.

To remove resources from a cluster, we perform these operations in reverse order.

We first scale down the StatefulSet and then remove from the cluster any nodes lacking a CockroachDB pod.

![illustration of phases: scaling down pods in a multi-zone cockroachdb cluster in Kubernetes](RackMultipart20200925-4-151p2tz_html_ebca54fd9fc4c5e2.gif)

Now, remember that pods in a StatefulSet of size N must have ids in the range [0,N). When scaling down a StatefulSet by M, Kubernetes removes the M pods with the M highest with ordinals from highest to lowest, [the reverse in which they were added](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#deployment-and-scaling-guarantees). Consider the cluster topology below:

![illustration: cockroachdb cluster: 6 nodes distributed across 3 availability zones](RackMultipart20200925-4-151p2tz_html_5ea59655cb72dc42.gif)

As ordinals 5 through 3 are removed from this cluster, the statefulset continues to have a presence across all 3 availability zones.

![illustration: removing 3 nodes from a 6-node, 3-zone cockroachdb cluster](RackMultipart20200925-4-151p2tz_html_756d28d73507725c.gif)

However, Kubernetes' scheduler doesn't guarantee the placement above as we expected at first.

> Pods in a replication controller or service are automatically spread across zones. [docs](https://kubernetes.io/docs/setup/best-practices/multiple-zones/#pods-are-spread-across-zones)

> For a StatefulSet with N replicas, when Pods are being deployed, they are created sequentially, in order from {0..N-1}. [docs](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#deployment-and-scaling-guarantees)

Consider the following topology:

![illustration: 6-node cockroachdb cluster distributed across 3 availability zones](RackMultipart20200925-4-151p2tz_html_ad9e57152c3ab5a8.gif)

These pods were created in order and they are spread across all availability zones in the cluster. When ordinals 5 through 3 are terminated, this cluster will lose its presence in zone C!

![illustration: terminating 3 nodes in 6-node cluster spread across 3 availability zones, where 2/2 nodes in the same availability zone are terminated, knocking out that AZ](RackMultipart20200925-4-151p2tz_html_b01c88a3a588a547.gif)

Worse yet, our automation, at the time, would remove Nodes A-2, B-2, and C-2. Leaving CRDB-1 in an unscheduled state as persistent volumes are only available in the zone they are initially created in.

To correct the latter issue, we now employ a "hunt and peck" approach to removing machines from a cluster. Rather than blindly removing kubernetes nodes from the cluster, only nodes without a CockroachDB pod would be removed. The much more daunting task was to wrangle the kubernetes scheduler.

## A session of brainstorming left us with 3 options:

### 1. Upgrade to kubernetes 1.18 and make use of [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/).

While this seems like it could have been the perfect solution, at the time of writing kubernetes 1.18 was unavailable on EKS and GKE. Furthermore, pod topology spread constraints were still a beta feature which meant that it wasn't guaranteed to be available on hosted solutions such as EKS and GKE. The entire endeavour was concerningly reminiscent of checking [caniuse.com](https://caniuse.com/) when IE 8 was still around.

### 2. Deploy a statefulset _per_ zone.

Rather than having one statefulset distributed across all availability zones, a single statefulset with node affinities per zone would allow manual control over our zonal topology. Our team had considered this as an option in the past which made it particularly appealing. Ultimately, we decided to forgo this option as it would have required a massive overhaul to our codebase and performing the migration on existing customer clusters would have been an equally massive undertaking.

### 3. [Write a custom kubernetes scheduler.](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)

At its core, this entire issue was a misunderstanding with the guarantees that kube-scheduler provided us with. Why not provide ourselves with the original guarantee that we were looking for? 

Thanks to an example from [Kelsey Hightower](https://github.com/kelseyhightower/scheduler) and a blog post from [Banzai Cloud](https://banzaicloud.com/blog/k8s-custom-scheduler/), we decided to dive in head first and write our own custom [Kubernetes scheduler]((https://github.com/cockroachlabs/crl-scheduler)). Once our POC was deployed and running, we quickly discovered that the kubernetes scheduler is also responsible for mounting persistent volumes to the pods that it schedules, which isn't mentioned in the [scheduler documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/) nor evident from inspecting [kubectl get events](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/#verifying-that-the-pods-were-scheduled-using-the-desired-schedulers). In our journey to find the component responsible for disk mounting, we discovered the [scheduler plugin system](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/). Our next POC was a Filter plugin that determined the appropriate availability zone by pod ordinal, and it worked flawlessly!

We open sourced [our scheduler plugin](https://github.com/cockroachlabs/crl-scheduler), and it is now running in all of our CockroachCloud clusters! Having control over how our statefulset pods are being scheduled has let us scale out statefulsets with confidence. We may look into retiring our plugin once pod topology spread constraints are available in GKE and EKS but the maintenance overhead has been surprisingly low. Better still integration into our existing deployments and codebase was minor modification to the statefulset definition.
