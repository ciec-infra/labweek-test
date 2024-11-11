# Istio Rollout Plan

## Current situation

Our current (02/2022) ingress strategy is to based on AWS ALB:

* Clients resolve the `public DNS` record
   * This is round-robing between at least two regions
   * Via the `weight` in Route53 we have the ability to remove a whole region from the rotation
   * For testing, there are `protected DNS` records pointing to the ALB of a specific region
* A client connects to an ALB
   * The ALB is a reflection of the K8s `ingress` object, the ALB specifics are defined via [annotation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)
* The traffic is routed to a pod
   * During our deployments, we employ Argo to traffic shift gradually to new software versions
      * Before (and while) the traffic shift happens, all 'new' software version pods can be accessed via the `canary DNS` record - which routes the traffic on the ALB level only to new pods

```plantuml
@startuml
skinparam nodesep 10
skinparam ranksep 10

skinparam component {
  FontSize 13
  BackgroundColor<<Apache>> Red
  BorderColor<<Apache>> #0000FF
  FontName Courier
  BorderColor black
  BackgroundColor lightblue
  ArrowFontName Impact
  ArrowColor #FF6655
  ArrowFontColor #777777
}

left to right direction

' Azure
!define AzurePuml https://raw.githubusercontent.com/RicardoNiepel/Azure-PlantUML/release/2-1/dist
!include AzurePuml/AzureCommon.puml
!include AzurePuml/AzureSimplified.puml
!include AzurePuml/Containers/AzureContainerRegistry.puml
!include AzurePuml/DevOps/AzurePipelines.puml

' Kubernetes
!define KubernetesPuml https://raw.githubusercontent.com/dcasati/kubernetes-PlantUML/master/dist
!include KubernetesPuml/kubernetes_Common.puml
!include KubernetesPuml/kubernetes_Container.puml
!include KubernetesPuml/kubernetes_Context.puml
!include KubernetesPuml/kubernetes_Simplified.puml
!include KubernetesPuml/OSS/KubernetesApi.puml
!include KubernetesPuml/OSS/KubernetesPod.puml
!include KubernetesPuml/OSS/KubernetesRs.puml
!include KubernetesPuml/OSS/KubernetesCrd.puml
!include KubernetesPuml/OSS/KubernetesDeploy.puml
!include KubernetesPuml/OSS/KubernetesHpa.puml
!include KubernetesPuml/OSS/KubernetesDs.puml
!include KubernetesPuml/OSS/KubernetesIng.puml
!include KubernetesPuml/OSS/KubernetesSvc.puml

' AWS
!define AWSPuml https://raw.githubusercontent.com/awslabs/aws-icons-for-plantuml/v10.0/dist
!include AWSPuml/AWSCommon.puml
!include AWSPuml/Compute/all.puml
!include AWSPuml/Containers/all.puml
!include AWSPuml/NetworkingContentDelivery/all.puml
!include AWSPuml/ManagementGovernance/CloudWatch.puml

' Icon
!define ICONURL https://raw.githubusercontent.com/tupadr3/plantuml-icon-font-sprites/v2.1.0
!include ICONURL/common.puml
!include ICONURL/material/search.puml

title XVP ALB-Ingress Architecture

ElasticLoadBalancing(alb, "Application Load Balancer", "ALB")

Route53HostedZone(zone_global, "Public zone", "exp.xvp.na-1.xcal.tv") {
    Route53(r53_public_record, "Public Service record", "disco.exp.xvp.na-1.xcal.tv")
}
Route53HostedZone(zone_region, "Protected zone", "prod.exp.us-east-1.aws.xvp.xcal.tv") {
    Route53(r53_protected_record, "Protected Service record", "disco.prod.exp.us-east-1.aws.xvp.xcal.tv")
}

actor clients

Cluster_Boundary(cluster, "Kubernetes") {
    ElasticKubernetesService(eks, "EKS Cluster", "prod") {

        Namespace_Boundary(discons, "disco-api") {

            KubernetesIng(ingress, "alb", "")


            KubernetesSvc(svc, "xvp_stable", "")
            KubernetesSvc(canary_svc, "xvp_canary", "")

            KubernetesRs(rs,"Replication Set","")
            KubernetesPod(pod, "canary-release", "")


            KubernetesCrd(argo_rollout, "Rollouts", "Argo")

        }
    }
}

Rel_U(alb, r53_protected_record, "CName")
Rel_U(alb, r53_public_record, "Weighted Round Robin")

Rel_D(rs, pod, "scale")
Rel_R(svc, pod , " ")
Lay_R(svc, pod)
Lay_R(canary_svc, rs)

Rel(alb,ingress, " ")
Rel(ingress, svc, "regular traffic")
Rel(ingress, canary_svc, "canary traffic")

Rel_L(argo_rollout, alb, "Traffic shaping")
Rel_R(argo_rollout, ingress, "Traffic shaping")
Rel_R(clients, r53_public_record, " ")

@enduml
```

## Migration phase

During the migration phase, a hybrid routing model exists:

* The basic Istio elements 'exist' but are unused by a given service:
   * The wildcard cert protecting the traffic of the NLB
   * A centralized NLB
   * The Isio K8s objects

When a service is ready, it can opt in to *additionally* use the new Ingress path (in one or more regions via the existing `tfvars`) on top of the existing ALB path

* The existing clients will continue to use the `public DNS` record and use the ALB
* Additionally, Istio-specific `protected DNS` records are available to test traffic flow through the new Ingress path
* Canary rollouts are applied to the ALB & Ingress path to allow for testing
* It is up to the service to add the Istio/NLB path as additional entry of the `public DNS` record
   * This allows (via the `weight`) to gradually enable traffic via the new Istio/NLB Ingerss

```plantuml
@startuml
skinparam nodesep 10
skinparam ranksep 10

skinparam component {
  FontSize 13
  BackgroundColor<<Apache>> Red
  BorderColor<<Apache>> #0000FF
  FontName Courier
  BorderColor black
  BackgroundColor lightblue
  ArrowFontName Impact
  ArrowColor #FF6655
  ArrowFontColor #777777
}

left to right direction

' Azure
!define AzurePuml https://raw.githubusercontent.com/RicardoNiepel/Azure-PlantUML/release/2-1/dist
!include AzurePuml/AzureCommon.puml
!include AzurePuml/AzureSimplified.puml
!include AzurePuml/Containers/AzureContainerRegistry.puml
!include AzurePuml/DevOps/AzurePipelines.puml

' Kubernetes
!define KubernetesPuml https://raw.githubusercontent.com/dcasati/kubernetes-PlantUML/master/dist
!include KubernetesPuml/kubernetes_Common.puml
!include KubernetesPuml/kubernetes_Container.puml
!include KubernetesPuml/kubernetes_Context.puml
!include KubernetesPuml/kubernetes_Simplified.puml
!include KubernetesPuml/OSS/KubernetesApi.puml
!include KubernetesPuml/OSS/KubernetesPod.puml
!include KubernetesPuml/OSS/KubernetesRs.puml
!include KubernetesPuml/OSS/KubernetesCrd.puml
!include KubernetesPuml/OSS/KubernetesDeploy.puml
!include KubernetesPuml/OSS/KubernetesHpa.puml
!include KubernetesPuml/OSS/KubernetesDs.puml
!include KubernetesPuml/OSS/KubernetesIng.puml
!include KubernetesPuml/OSS/KubernetesSvc.puml

' AWS
!define AWSPuml https://raw.githubusercontent.com/awslabs/aws-icons-for-plantuml/v10.0/dist
!include AWSPuml/AWSCommon.puml
!include AWSPuml/Compute/all.puml
!include AWSPuml/Containers/all.puml
!include AWSPuml/NetworkingContentDelivery/all.puml
!include AWSPuml/ManagementGovernance/CloudWatch.puml

' Icon
!define ICONURL https://raw.githubusercontent.com/tupadr3/plantuml-icon-font-sprites/v2.1.0
!include ICONURL/common.puml
!include ICONURL/material/search.puml

title XVP Istio-Hybrid Ingress Architecture

ElasticLoadBalancing(alb, "service Application Load Balancer", "ALB")
ElasticLoadBalancing(nlb, "shared Network Load Balancer", "NLB")

actor clients

Route53HostedZone(zone_global, "Public zone", "exp.xvp.na-1.xcal.tv") {
    Route53(r53_public_record, "Public Service record", "disco.exp.xvp.na-1.xcal.tv")
}
Route53HostedZone(zone_region, "Protected zone", "prod.exp.us-east-1.aws.xvp.xcal.tv") {
    Route53(r53_protected_record, "Protected Service record", "disco.prod.exp.us-east-1.aws.xvp.xcal.tv")
    Route53(r53_protected_record_istio, "Protected Istio record", "disco-nlb.prod.exp.us-east-1.aws.xvp.xcal.tv")

}


Cluster_Boundary(cluster, "Kubernetes") {
    ElasticKubernetesService(eks, "EKS Cluster", "prod") {

        Namespace_Boundary(discons, "disco-api") {

            KubernetesIng(ingress, "alb", "")
            KubernetesIng(ingressIstio, "istio", "")


            KubernetesSvc(svc, "xvp_stable", "")
            KubernetesSvc(canary_svc, "xvp_canary", "")

            KubernetesRs(rs,"Replication Set","")
            KubernetesPod(pod, "n canary-release", "")


            KubernetesCrd(argo_rollout, "Rollouts", "Argo")

        }
    }
}

Rel_U(alb, r53_protected_record, "CName")
Rel_U(alb, r53_public_record, "Weighted Round Robin")
Rel_U(nlb, r53_protected_record_istio, "CName")
Rel_U(nlb, r53_public_record, "0-Weighted Round Robin")
note left of nlb
  No traffic flows through the NLB by default.
  Regions via Istio can be added via R53
end note
note right of discons
  Istio objects not shown
end note

Rel_D(rs, pod, "scale")
Rel_R(svc, pod , " ")
Rel_R(canary_svc, pod , " ")
Lay_R(canary_svc, rs)
Lay_D(alb, nlb)

Rel(alb,ingress, " ")
Rel(nlb,ingressIstio, " ")

Rel(ingress, svc, "regular traffic")
Rel(ingress, canary_svc, "canary traffic")
Rel(ingressIstio, svc, "regular traffic")
Rel(ingressIstio, canary_svc, "canary traffic")

Rel_L(argo_rollout, alb, "Traffic shaping")
Rel_L(argo_rollout, nlb, "Traffic shaping")
Rel_R(argo_rollout, ingress, "Traffic shaping")
Rel_R(argo_rollout, ingressIstio, "Traffic shaping")

Rel_R(clients, r53_public_record, " ")
Rel_R(tester, r53_protected_record_istio, " ")

@enduml
```

## Final situation

When all services have migrated to the new Ingress path via Istio/NLB and all traffic is shifted away from the ALB

* The K8s ALB Ingress objects are removed
* The 'old' `protected DNS` records are used for Istio
   * The Istio `protected DNS` records are removed
   * AWS ALB itself will disappear 

This will have the benefit that we are no longer limited by the amount of ALBs in a region - which is a limiting factor for `dev-ci` deployments today.

```plantuml
@startuml
skinparam nodesep 10
skinparam ranksep 10

skinparam component {
  FontSize 13
  BackgroundColor<<Apache>> Red
  BorderColor<<Apache>> #0000FF
  FontName Courier
  BorderColor black
  BackgroundColor lightblue
  ArrowFontName Impact
  ArrowColor #FF6655
  ArrowFontColor #777777
}

left to right direction

' Azure
!define AzurePuml https://raw.githubusercontent.com/RicardoNiepel/Azure-PlantUML/release/2-1/dist
!include AzurePuml/AzureCommon.puml
!include AzurePuml/AzureSimplified.puml
!include AzurePuml/Containers/AzureContainerRegistry.puml
!include AzurePuml/DevOps/AzurePipelines.puml

' Kubernetes
!define KubernetesPuml https://raw.githubusercontent.com/dcasati/kubernetes-PlantUML/master/dist
!include KubernetesPuml/kubernetes_Common.puml
!include KubernetesPuml/kubernetes_Container.puml
!include KubernetesPuml/kubernetes_Context.puml
!include KubernetesPuml/kubernetes_Simplified.puml
!include KubernetesPuml/OSS/KubernetesApi.puml
!include KubernetesPuml/OSS/KubernetesPod.puml
!include KubernetesPuml/OSS/KubernetesRs.puml
!include KubernetesPuml/OSS/KubernetesCrd.puml
!include KubernetesPuml/OSS/KubernetesDeploy.puml
!include KubernetesPuml/OSS/KubernetesHpa.puml
!include KubernetesPuml/OSS/KubernetesDs.puml
!include KubernetesPuml/OSS/KubernetesIng.puml
!include KubernetesPuml/OSS/KubernetesSvc.puml

' AWS
!define AWSPuml https://raw.githubusercontent.com/awslabs/aws-icons-for-plantuml/v10.0/dist
!include AWSPuml/AWSCommon.puml
!include AWSPuml/Compute/all.puml
!include AWSPuml/Containers/all.puml
!include AWSPuml/NetworkingContentDelivery/all.puml
!include AWSPuml/ManagementGovernance/CloudWatch.puml

' Icon
!define ICONURL https://raw.githubusercontent.com/tupadr3/plantuml-icon-font-sprites/v2.1.0
!include ICONURL/common.puml
!include ICONURL/material/search.puml

title XVP Istio-Ingress Architecture

ElasticLoadBalancing(nlb, "shared Network Load Balancer", "NLB")


Route53HostedZone(zone_global, "Public zone", "exp.xvp.na-1.xcal.tv") {
    Route53(r53_public_record, "Public Service record", "disco.exp.xvp.na-1.xcal.tv")
}
Route53HostedZone(zone_region, "Protected zone", "prod.exp.us-east-1.aws.xvp.xcal.tv") {
    Route53(r53_protected_record, "Protected Service record", "disco.prod.exp.us-east-1.aws.xvp.xcal.tv")
}

actor clients


Cluster_Boundary(cluster, "Kubernetes") {
    ElasticKubernetesService(eks, "EKS Cluster", "prod") {

        Namespace_Boundary(discons, "disco-api") {

            KubernetesIng(ingressIstio, "istio", "")


            KubernetesSvc(svc, "xvp_stable", "")
            KubernetesSvc(canary_svc, "xvp_canary", "")

            KubernetesRs(rs,"Replication Set","")
            KubernetesPod(pod, "n canary-release", "")


            KubernetesCrd(argo_rollout, "Rollouts", "Argo")

        }
    }
}

Rel_U(nlb, r53_protected_record, "CName")
Rel_U(nlb, r53_public_record, "Weighted Round Robin")
note left of r53_protected_record
  Rename back to get rid of -nlb suffix
end note


Rel_D(rs, pod, "scale")
Rel_R(svc, pod , " ")
Rel_R(canary_svc, pod , " ")
Lay_R(canary_svc, rs)

Rel(nlb,ingressIstio, " ")

Rel(ingressIstio, svc, "regular traffic")
Rel(ingressIstio, canary_svc, "canary traffic")

Rel_L(argo_rollout, nlb, "Traffic shaping")
Rel_R(argo_rollout, ingressIstio, "Traffic shaping")

Rel_R(clients, r53_public_record, " ")

@enduml
```
