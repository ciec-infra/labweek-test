# Networking

XVP Experience APIs are deployed to AWS with VPC networks managed by the Comcast
[Tenant Security Framework (TSF)](https://onecloud.comcast.net/docs/user-guide/tsf/).
For information about network provisioning, management, architecture diagrams, and
Infrastructure as Code, please see
[`xvp/xvp-infra-core`](https://github.com/comcast-xvp/xvp-infra-core).

## VPC Network Resource Planning
### Overview

XVP is deployed to the AWS cloud where resources are elastic, but its
APIs are publicly exposed on the Internet (via the Comcast "DMZ") and connect
directly to service dependencies on Comcast's internal network. This approach
is easy to reason about and performant but has some important implications.

* Each resource provisioned in the VPC consumes _at least one_ real Comcast
  IP address for each Availability Zone (AZ) in which it is deployed. Many
  resources (ALBs and EKS worker nodes, for example) consume more adresses,
  typically 4-8.
* VPC subnets are of a discrete size and are not elastic. [More subnets can
  be added to the VPC](https://onecloud.comcast.net/docs/user-guide/tsf/#editing-an-aws-network) but existing subnets cannot be grown.
* Each ALB [consumes eight (8) IP addresses per subnet / Availability Zone](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#subnets-load-balancer).
* Each ECS Fargate _task_ [consumes one (1) IP address](https://docs.aws.amazon.com/AmazonECS/latest/userguide/fargate-task-networking.html). Tasks can be composed of several
  containers.
* Each EKS Kubernetes _pod_ [consumes one (1) IP address](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html). Pods can be composed of several containers.
* Each EKS Kubernetes control plane (master node) [consumes one (1) IP address.](https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html)
* Each EKS Kubernetes worker node [consumes one (1) or more IP addresses](https://aws.amazon.com/blogs/compute/announcing-improved-vpc-networking-for-aws-lambda-functions/).
  This is configurable with settings such as `WARM_IP_TARGET` and `WARM_ENI_TARGET`
  but there is a tradeoff between pod scale-out latency and warm IPs/ENIs.
* Each Lambda [consumes at least (1) IP address but more as the Lambda scales.](https://aws.amazon.com/blogs/compute/announcing-improved-vpc-networking-for-aws-lambda-functions/)

We are working toward more efficient addressing strategies leveraging a
combination of reserved (RFC1918) addressing, CNI Custom Networking, and NAT
but that is a work-in-progress.

### Back of Envelope Estimate

| Rail      | IP Addresses        |
| --------- | ------------------- |
| Public    | 8/service/AZ        |
| Protected | 10/service/instance |

For dev, QA, and staging workloads, you will want to provision at least two (2)
instances to allow for zero-downtime rolling deployment and fault tolerance in
case of an Availability Zone failure.

When provisioning subnets in AWS, please note that five (5) addresses per
[subnet CIDR range](https://dnsmadeeasy.com/support/subnet/) are reserved
and cannot be used.

Assume:

* 10 services
* 10 IP addresses per service
* Dev/Stg
  * 2 instances in each env per service
  * 2 AZs in each env
* Prod
  * 50 instances/service in prod (a completely arbitrary guess)
  * 3 AZs in prod
  * 33% headroom in prod to deal with an AZ outage

**Protected Rail Recommendation:**

| XVP Env | Account         | IPs/subnet | Subnets | Subnet Size    |
| ------- | --------------- | -----------| ------- | -------------- |
| dev     | xvp-exp-us-dev  | 100        | 2       | -              |
| stg     | xvp-exp-us-dev  | 100        | 2       | -              |
| *total* | xvp-exp-us-dev  | 200        | 2       | /24 (256  ea.) |
| prod    | xvp-exp-us-prod | 2216       | 3       | /20 (4096 ea.) |

Note that the estimate above includes load balancer, application, and container
runtime. It does not include additional infrastructure components such as
databases, caches, Lambdas, and such.

### Detailed Breakdown

| Component                       | IP Count      | Rail      | Notes |
| ------------------------------- | ------------- | --------- | ----- |
| ALB                             | 8 per AZ      | Public    ||
| EKS Master                      | 1 per AZ      | Protected ||
| EKS Worker                      | 1 per worker  | Protected ||
| `aws-node`                      | 1 per worker  | Protected ||
| `coredns`                       | 1 per worker  | Protected ||
| `kube-proxy`                    | 1 per worker  | Protected ||
| `node-exporter`                 | 1 per worker  | Protected ||
| `fluent-bit`                    | 1 per worker  | Protected ||
| `xvp-${service}-${env}`         | 1 per worker  | Protected ||
| `cert-manager`                  | 1             | Protected ||
| `cert-manager-cainjector`       | 1             | Protected ||
| `cert-manager-webhook`          | 1             | Protected ||
| `aws-load-balancer-controller`  | 1             | Protected ||


## See Also

* https://onecloud.comcast.net/docs/using-aws/network-connectivity/#comcast-network-connectivity
* https://onecloud.comcast.net/docs/user-guide/tsf/
* https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html
* https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html
* https://aws.amazon.com/blogs/containers/optimize-ip-addresses-usage-by-pods-in-your-amazon-eks-cluster/
