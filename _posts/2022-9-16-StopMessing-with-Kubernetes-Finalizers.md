---
layout: post
title: Stop Messing with Kubernetes Finalizers
---

We've all been there - it's frustrating seeing deletion of Kubernetes resource getting stuck, hang or take a very long time. 
You might have "solved" this using the terrible advice of removing finalizers or running `kubectl delete ... --force --grace-period=0` to force immediate deletion. 
99% of the time this is a horrible idea and in this article I will show you why.

# Finalizers

Before we get into why force-deletion is a bad idea, we first need to talk about **_finalizers_**.

Finalizers are values in resource metadata that signal required pre-delete operations - they tell resource controller what operations need to be performed before object is deleted.

The most common one would be:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  finalizers:
  - kubernetes.io/pvc-protection
...
```

Their purpose is to stop a resource from being deleted, while controller or Kubernetes Operator cleanly and gracefully cleans-up any dependant objects such as underlying storage devices.

When you delete an object which has a finalizer, `deletionTimestamp` is added to resource metadata making the object read-only. 
Only exception to the read-only rule, is that finalizers can be removed. Once all finalizers are gone, the object is queued to be deleted.

It's important to understand that finalizers are just items/keys in resource metadata. Finalizers don't specify the code to execute. They have to be added/removed by the resource controller.

Also, don't confuse finalizers with _Owner References_. `.metadata.OwnerReferences` field specify parent/child relations between objects such as _Deployment -> ReplicaSet -> Pod_. 
When you delete an object such as Deployment a whole tree of child objects can be deleted. This process (deletion) is automatic, unlike with finalizers, where controller needs to take some action and remove the finalizer field.

# What Could Go Wrong?

As mentioned earlier, the most common finalizer you might encounter is the one attached to _Persistent Volume (PV)_ or _Persistent Volume Claim (PVC)_. 
This finalizer protects the storage from being deleted while it's in use by a Pod. Therefore, if the PV or PVC doesn't want to delete, it most likely means that it's still mounted by a Pod. 
If you decide force-delete PV, be aware that backing storage in Cloud or any other infrastructure might not get deleted, therefore you might leave a dangling resource, which still costs you money.

Another example is a Namespace which can get stuck in `Terminating` state because resources still exist in the namespace that the namespace controller is unable to remove. 
Forcing deletion of namespace can leave dangling resources in your cluster which include for example Cloud provider's load balancer which might be very hard to track down later.

While not necessarily related to finalizers, it's good to mention that resources can get stuck for many other reasons other than waiting for finalizers:

The simplest example would be Pod being stuck in _Terminating_ state, which usually signals issue with Node on which the Pod runs. "_Solving_" this with `kubectl delete pod --grace-period=0 --force ...` will remove the Pod from API server (etcd), but it might still be running on the Node, which is definitely not desirable.

Another example would be a _StatefulSet_, where Pod force-deletion can create problems because Pods have fixed identities (pod-0,pod-1). A distributed system might depend on these names/identities - if the Pod is force-deleted, 
but still runs on the node, you can end-up with 2 pods with same identity when StatefulSet controller replaces the original "deleted" Pod. 
These 2 Pods might then attempt to access same storage, which can lead to corrupted data. More on this in [docs](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod).

# Finalizers in The Wild

 We now know that we shouldn't mess with resources that have finalizers attacked to them, but which resources are these?

The 3 most common ones you will encounter in "*vanilla*" Kubernetes are `kubernetes.io/pv-protection` and `kubernetes.io/pvc-protection` related to *Persistent Volumes* and *Persistent Volume Claims* respectively (plus [couple more](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolume-deletion-protection-finalizer) introduced in v1.23) as well as *kubernetes* finalizer present on *Namespaces*. The last one however isn't in *.metadata.finalizers* field but rather in *.spec.finalizers* - this special case is described in architecture document.

Besides these *"vanilla"* finalizers, you might encounter many more if you install Kubernetes Operators which often perform pre-deletion logic on their custom resources. A quick search through code of some popular projects turn up the following:

[Istio](https://github.com/istio/istio/blob/master/operator/pkg/controller/istiocontrolplane/istiocontrolplane_controller.go#L65) - `istio-finalizer.install.istio.io`

[Cert Manager](https://github.com/cert-manager/cert-manager/blob/ee8ec69fadff165afa96c2dd22264c16fdb7d065/internal/apis/acme/v1beta1/const.go#L20) - `finalizer.acme.cert-manager.io`

[Strimzi (Kafka)](https://github.com/strimzi/strimzi-kafka-operator/commit/69e77ce8d5918c25048a253f91f4bca8e89028d9#diff-0f711d9ed233c37fbe749fd6c4aadce73849f48de3c414d86d9af89d51ea5ef7R317) - `service.kubernetes.io/load-balancer-cleanup`

[Quay](https://github.com/quay/quay-operator/pull/405/files#diff-db06dd075ea792819f15dcbfb9c2376eea2e17832c2bd64ae6b381d3c947b57eR56) - `quay-operator/finalizer`

[Ceph/Rook](https://github.com/rook/rook/blob/master/Documentation/ceph-teardown.md#removing-the-cluster-crd-finalizer) - `ceph.rook.io/disaster-protection`

[ArgoCD](https://github.com/argoproj-labs/argocd-operator/pull/247/files#diff-1078fd6d90631dae21aebe2e5cb7b8f2e559f568d61b8277117dd19344462d47R188) - `argoproj.io/finalizer`

[Litmus Chaos](https://github.com/litmuschaos/chaos-operator/blob/master/pkg/controller/chaosengine/chaosengine_controller.go#L57) - `chaosengine.litmuschaos.io/finalizer`
