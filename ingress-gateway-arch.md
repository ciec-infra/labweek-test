# Target Ingress-Gateway Architecture

> Note this is a living document meant to reflect our current strategy and understanding of the target architecture. It will be used as a reference point to implement the prototypical architecture in the plat-dev environment. As we validate or disprove assumptions in that environment, this document will be updated.
>
> Also, this document covers the Ingress architecture to the service. The egress architecture including failover for upstream services is not included, but this document does give some strong suggestions about how that will work.

## Overview

The diagram below depicts a selective view of the ingress-gateway component architecture. This configuration is designed to be deployed per Availability Zone so that failover at the ingress happens first at the Zone level (by default) and then at the regional level. The ingress-gateway itself is scaled using a Horizontal Pod Autoscaler (HPA) configuration, and the minimal configuration will include one ingress-gateway pod per AWS availability zone.

The Network Load Balancer (NLB), however, load balances across the ingress-gateway pods. That load balancer serves all services in the region. TLS termination occurs at the NLB, which communicates with the ingress-gateway over TCP. (This is a [concern](#tls-termination) with this architecture for the moment, and is discussed further below.)

The ingress-gateway is consumes service-specific Istio custom resources - the gateway, the virtual service and destination rules. These resources will be managed and deployed by each microservice team. The gateway identifies the incoming ports for the service. The virtual service binds the public host name to a set of Kubernetes services and/or a set of destination rules. As the name implies, the destination rules route traffic to the appropriate Kubernetes service endpoints. In addition, the destination rules initiate Istio mutual TLS between the ingress-gateway and the destination pods via sidecar proxies (Envoy) in both pods.

![Ingress Gateway overview](./img/istio-ingress-gateway.png)

## Failure/Failover

Failover occurs at two levels: zonal and regional.

### Zonal failover

By default, zonal failover is handled by the relationship between the service's istio virtual-service and the Kubernetes service it points to. Istio Virtual-Service depend on resolving the named destination service defined in the registry. Assuming that the actual service has been deployed with distribution across all available zones, i.e., those found in the service registry. Kubernetes passes this information to the virtual-service, which has a set of internal configuration for how to distribute requests across those named destination service. (These are typical load balancer options.) If one of the pods fails - Kubernetes will identify the failure via liveness and readiness probes and remove the pod from the endpoint list.

#### Ingress-gateway zonal failure

This, of course, begs the question of what happens when the ingress-gateway fails. From a Kubernetes point of view, access to the ingress-gateway pods is governed by a `LoadBalancer` type service that maps a series of node ports to the cluster ip endpoints of each ingress-gateway pod. The AWS NLB points to these node ports and contains its own health checks to determine when any of the "targets" are in an unhealthy state. If the ingress is unhealthy, then the NLB will not direct traffic to the unhealthy ingress until it comes back on-line.

### Regional failover

Given the separation of the service from the loadbalancer by inserting the ingress-gateway in between them, there are two types of regional failover needed - one when there is a true regional outage and one when a service is out in the region, but the ingress-gateways are not. The latter uses Istio functions to failover to alternate instances of the service running in a different region. The former uses Route 53 failover as XVP does today.

#### Role of Route 53 - ingress gateway is not healthy (region outage)

The current use of Route 53 to fail over a region when Route 53 health checks fail does not change. Route 53 is looking at the NLB in this scenario - and if the NLB has no healthy targets, Route 53 will shift traffic. That is not the same thing as a service failure in a region.

## Component Architecture

### Istio System and Related Components

#### Istio Operator

Istio Operator is no longer used to deploy Istio. The installation and management of Istio components, such as istiod, the Istio daemon (note this is not a daemonset), and the ingress-gateway and egress-gateway, are performed via Helm. Helm charts allow for customization by configuring parameters in the associated `values.yaml` for the chart. This can be used to set mesh wide settings or configure listener ports for gateways for example.

#### Mutual TLS - Cert-manager and Hashicorp Vault

With the migration to Multi-cluster service mesh, AWS Private CA Controller usage has been replaced by Hashicorp Vault. AWS PCA has a regional limitation which makes it impossible to issue certs from different regions with a unified Root CA. As such, we have migrated to Vault provided by the Secrets-as-a-Service team.

Within the mesh, Istio is able to use mutual TLS. When calls are routed from the ingress-gateway to the service pod - they occur over mutual TLS. The purpose of mutual TLS here is to allow each end of the communication to verify the other as trusted. By default, Istio will configure Peer Authentication policies as `PERMISSIVE` meaning, proxies will prefer mTLS when available, but not require it. Lack of mTLS within the cluster is fine, however communicating via eastwest gateway will require `STRICT` mTLS and a proper [trust relationship](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#identity-and-trust-models) across the clusters.

![mtls-cert](./img/cert-manager.drawio.svg "mtls cert")

While Istio provides [documentation](https://preliminary.istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/#plug-in-certificates-and-key-into-the-cluster) for using self managed certificates for mTLS, it is recommended to use an Enterprise CA such as Hashicorp Vault. This is the direction that we have settled on, and documentation about that setup can be found [HERE](./cert-manager-istio-csr.md).

A single Root CA and individual intermediate CAs is recommended, however due to the manual nature of provisioning these endpoints in vault, we have chosen to use a single Intermediate CA to sign/issue certs for all workloads in all clusters. This, however, does not degrade the security of the infrastructure nor does it imply services can communicate across meshes.

##### Enabling mTLS

The mTLS mode is configured by the `PeerAuthentication` policy that is set:

```bash
$ k get peerauthentication -A
NAMESPACE                   NAME          MODE         AGE
istio-system                default       PERMISSIVE   33d
xvp-disco-api-dev           mtls-policy   STRICT       17h
xvp-linear-api-dev          mtls-policy   STRICT       17h
xvp-mindata-api-dev         mtls-policy   STRICT       17h
xvp-ping-api-dev            mtls-policy   STRICT       17h
xvp-playback-api-dev        mtls-policy   STRICT       17h
xvp-rights-api-dev          mtls-policy   STRICT       17h
xvp-saved-api-dev           mtls-policy   STRICT       17h
xvp-session-api-dev         mtls-policy   STRICT       17h
xvp-sports-api-dev          mtls-policy   STRICT       17h
xvp-sportscast-dev          mtls-policy   STRICT       17h
xvp-transactional-api-dev   mtls-policy   STRICT       17h
```

The policy assigned to the `istio-system` namespace will be applied globally and can be overridden by policies applied to individual namespaces and workloads respectively. In the example above, the mTLS policy applied to the cluster is set to `PERMISSIVE` to allow but not require mTLS. This is because not all workloads in the cluster will be istio injected and have support for mTLS. To enforce mTLS on the XVP services, we've applied workload specific peer authentication policies:

```yaml
$ k get peerauthentication -n xvp-ping-api-dev mtls-policy -oyaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: mtls-policy
  namespace: xvp-ping-api-dev
spec:
  mtls:
    mode: STRICT
  portLevelMtls:
    "8081":
      mode: DISABLE
  selector:
    matchLabels:
      app.kubernetes.io/component: ping-api
```

As you can see, this `PeerAuthentication` policy has a selector to match the labels associated with `ping-api` workloads. While the mTLS mode is set to `STRICT` we have selectively disabled mTLS for port 8081. This specifically was done to facilitate scraping from Prometheus, however there may be other use cases for selectively disabling mTLS for individual workloads and ports. Read more about [Authentication Policy](https://istio.io/latest/docs/tasks/security/authentication/authn-policy/).

### AWS Components

#### AWS Certificate Manager (ACM)

The configuration of services in ACM will change from the current state where each service has its own certificate. Instead, there will be a single wildcard certificate for each environment. The wild card certificate will be associated with the Network Load Balancer in that environment. The routing of actual public host names to Kubernetes services is handled by the ingress-gateway where Virtual Services identify the host name of the service so that the gateway can route requests using destination rules. (See below.)

#### Network Load Balancer (NLB)

The current Application Load Balancer, a Layer 7 construct, is replaced by the combination of the Level 4 Network Load Balancer from AWS and the ingress-gateway. The Network load balancer will point at targets where the ingress-gateway is listening.

It is still up to the service to define their protected DNS endpoints and control when the R53 records point to the NLB rather than the ALB. This enables migration from the current ALB configuration to the new NLB + Istio configuration.

##### TLS termination

The NLB will terminate public TLS at the Network Load balancer. Communication between the NLB and ingress-gateway will occur over TCP. This is an undesirable part of this architecture. Ideally, we could terminate public TLS at the ingress-gateway. However, because we use ACM and not a full Certificate Authority (CA) for the public certificates, we have no way to access the private key Istio would require to decrypt the certificate.

In a typical scenario, this would not be the case - Istio would terminate the public TLS at the ingress-gateway and then initiate mutual TLS internally.

We are still investigating alternatives.

### Ingress-Gateway Components (deployed by service)

Ingress-gateway makes use of three custom resources that will be provisioned for each service. Service teams will be responsible for maintaining these resources, but each service will be provisioned with an initial set of these resources.

While [this document](https://istio.io/latest/docs/ops/configuration/traffic-management/tls-configuration/) talks about TLS configuration, it is also useful to understand the roles of each of these components.

#### Gateway

The gateway resource describes the "edge" of the load balancing functions of the ingress-gateway. To be clear, it is not the same as the ingress-gateway itself, but describes the very edge of it: how requests get to it. It typically identifies the ports exposed externally to the NLB - most of the time this will be ports 80, 443.

Here's the example of the gateway implemented for the ping service in plat-dev:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ping-gateway
  namespace: xvp-ping-api-dev
spec:
  servers:
  - hosts:
    - '*.plat-dev.exp.us-east-2.aws.xvp.xcal.tv'
    port:
      name: http
      number: 8080
      protocol: HTTP
```

In practice, we opt to deploy a generic Gateway that covers the configuration for all services. As such, you'll only find a single Gateway resource that is applied to the ingress-gateway pods.

#### Virtual Service

The virtual service maps hosts and downstream clients to a set of routing rules (the destination rules). Here's an example of one created for the ping service in plat-dev:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: xvp-ping-api-plat-dev-rollout
  namespace: xvp-ping-api-plat-dev
spec:
  gateways:
  - istio-ingress/istio-nlb-gateway
  hosts:
  - ping-plat-dev.exp.xvp.na-1.xcal.tv
  - ping.plat-dev.exp.us-east-2.aws.xvp.xcal.tv
  - ping-nlb.plat-dev.exp.us-east-2.aws.xvp.xcal.tv
  http:
  - name: http-rollout
    route:
    - destination:
        host: xvp-ping-api-plat-dev-canary
        port:
          number: 8080
      weight: 0
    - destination:
        host: xvp-ping-api-plat-dev-stable
        port:
          number: 8080
      weight: 100

```

#### Destination Rule

The Destination Rule defines a series of policies to apply to traffic for a service after the routing has occurred. It is applied based on the matching that occurs in the virtual service.

Here is the destination rule that was set up for ping in plat-dev and that is used to route the traffic to the ping service:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ping-destination
  namespace: xvp-ping-api-dev
spec:
  host: xvp-ping-api-dev.xvp-ping-api-dev.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    portLevelSettings:
    - port:
        number: 8080
      loadBalancer:
        simple: ROUND_ROBIN
        localityLbSetting:
          enabled: true
          failover:
            - from: us-east-2
              to: us-east-1
      outlierDetection:
        consecutive5xxErrors: 1
        interval: 1s
        baseEjectionTime: 1m
  subsets:
  - name: ping
    labels:
      component: xvp-ping-api-dev
      
```

This rule sets up the port level settings, mutual tls and failover.

## Useful resources

* [https://aws.amazon.com/blogs/security/tls-enabled-kubernetes-clusters-with-acm-private-ca-and-amazon-eks-2/](https://aws.amazon.com/blogs/security/tls-enabled-kubernetes-clusters-with-acm-private-ca-and-amazon-eks-2/)
* [https://aws.amazon.com/blogs/containers/setting-up-end-to-end-tls-encryption-on-amazon-eks-with-the-new-aws-load-balancer-controller/](https://aws.amazon.com/blogs/security/tls-enabled-kubernetes-clusters-with-acm-private-ca-and-amazon-eks-2/)
