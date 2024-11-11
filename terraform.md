# Terraform Providers

## Kubernetes provider
Most of the Kubernetes infrastructure is built using Terraform with the [Hashicorp Kubernetes provider](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs).

This is the standard provider and offers APIs that correspond to many of the standard Kubernetes resources and APIs.

## Corresponding state
One of the challenges with using this provider is that for the context of resources managed by Terraform, 
Terraform state must reflect what's in the state of the Kubernetes control plane. The Kubernetes state is easily updated by something other than Terraform such as the Kubernetes control plane itself, a daemon process or another pod. As complexity in the environment grows, this issue can become more difficult to manage, because it's not always clear either that a change has occurred or what has triggered the change.

### A Real Life Example
Istio-operator is a pod that manages deployment and configuration of Istio within a Kubernetes cluster. It takes responsibility for deploying istiod - the istio daemon, istio-ingress-gateway and istio-egress-gateway within the cluster. 

It also relies on a namespace called `istio-system` into which it deploys these resources and which must be created prior to deployment. It also creates other resources needed for the operation of those resources into that namespace.

This all works fine on apply - but the Terraform state has no knowledge of all of the resources that the operator has created in `istio-system`. When the environment is destroyed, the job fails when trying to destroy the namespace with the error `context deadline exceeded.` This seems mysterious, but a read of the Kubernetes documentation and Terraform issue posts seems to indicate that the failure is related to Kubernetes finalizers not being able to complete.

### What's a finalizer?

Finalizers execute pre-delete hooks on resources. They are keys that describe a set of actions for Kubernetes controllers to take before deletion. Many types of resource may have finalizers. Namespaces in particular will often have finalizers and can be particularly problematic to delete. ALBs created by the aws-ingress-loadbalancer-controller may also have finalizers.

You can read more about finalizers [here](https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/).

## Kubernetes-alpha provider
The [Kubernetes-alpha provider](https://registry.terraform.io/providers/hashicorp/kubernetes-alpha/latest/docs) is a somewhat experimental provider that implements "updates in place" or "patching" with a Kubernetes mechanism called "server-side apply."

Server-side apply manages the "fields" associated with a resource and tracks updates to them and from which "source." The rule ([from kubernetes documentation](https://kubernetes.io/docs/reference/using-api/server-side-apply/)) is that when two or more appliers set a field to the same value, they share ownership of that field. Any subsequent attempt to change the value of the shared field, by any of the appliers, results in a conflict. 

While the update in place strategy is powerful, it's very easy to run into conflicts between "client-side applied" changes from the standard kubernetes provider and server-side applied changes from kubernetes-alpha.

## kubectl provider
The [kubectl provider](https://registry.terraform.io/providers/gavinbunney/kubectl/latest/docs) supports the application of kubernetes yaml via a Terraform provider and executes updates in place. In a sense, kubectl gives precedence to kubernetes' tracking of state, because if a resource is not currently known to Terraform state, it will resolve the resource an apply updates to it. It also attempts to update in-place without the destruction and re-creation of the resource.

