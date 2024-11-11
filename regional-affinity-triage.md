# Regional Affinity FAQ & Triage  

If you are looking to learn more about regional affinity and the design for the [XVP services](../platform/k8s/istio/regionalaffinity.md). This doc will serve show 
some help commands for Triage to validate everything is working as intended and a quick FAQ. Incase of emergency and need to traffic shift [please see](traffic_shift.md)

## Helpful Commands 

The first command will show you all the host and there availability to the service using istioctl this can be done by the pod level or at the ingress gateway. 
If a service gets ejected it generally met the Outlier Detecion criteria which will be found in the services TFVARS file. Or this can be done via the dashboard below.

```bash
istioctl proxy-config endpoint -n <NAMESPACE> <POD>
ENDPOINT                                                STATUS      OUTLIER CHECK     CLUSTER
127.0.0.1:15000                                         HEALTHY     OK                prometheus_stats
127.0.0.1:15020                                         HEALTHY     OK                agent
198.18.32.174:4317                                      HEALTHY     OK                outbound|4317||jaeger-collector.jaeger.svc.cluster.local
198.18.32.174:4318                                      HEALTHY     OK                outbound|4318||jaeger-collector.jaeger.svc.cluster.local
198.18.32.174:14269                                     HEALTHY     OK                outbound|14269||jaeger-collector.jaeger.svc.cluster.local
198.18.32.224:7000                                      HEALTHY     OK                outbound|7000||argo-cd-applicationset-controller.argocd.svc.cluster.local
198.18.32.57:4317                                       HEALTHY     OK                outbound|4317||istio-ingress-gateway.istio-ingress.svc.cluster.local
198.18.32.57:4318                                       HEALTHY     OK                outbound|4318||istio-ingress-gateway.istio-ingress.svc.cluster.local
198.18.32.57:8080                                       HEALTHY     OK                outbound|443||istio-ingress-gateway.istio-ingress.svc.cluster.local
198.18.32.57:9471                                       HEALTHY     OK                outbound|9471||istio-ingress-gateway.istio-ingress.svc.cluster.local
198.18.32.57:44301                                      HEALTHY     OK                outbound|44301||istio-ingress-gateway.istio-ingress.svc.cluster.local
198.18.32.88:6379                                       HEALTHY     OK                outbound|6379||argo-cd-redis.argocd.svc.cluster.local
198.18.33.39:8080                                       HEALTHY     OK                outbound|8080||xvp-ping-api-plat-dev-canary.xvp-ping-api-plat-dev
```

Another Command that can give you an idea if something is wrong is look into the istio side car proxy which is a reverse proxy to the application and contain logs 
for all of the request coming into the application. This will also tell you any errors that may happen from envoy. 

```bash
kubectl logs <POD> -n <NAMESPACE> -c istio-proxy -f
[2023-10-01T20:38:04.447Z] "GET /actuator/health/app?clientId=GSLB-HealthCheck HTTP/1.1" 503 - direct_response - "-" 0 28 0 - "15.177.2.42" "Amazon-Route53-Health-Check-Service (ref 810a14f3-3962-45fa-9d8c-f8b0d4cea7e3; report http://amzn.to/1vsZADi)" "1b04088f-ca5c-4eff-88d7-cde3c71160f4" "ping.dev.exp.us-east-2.aws.xvp.xcal.tv" "-" - - 198.18.87.91:8080 15.177.2.42:0 outbound_.8080_.stable_.xvp-ping-api-dev-rollout.xvp-ping-api-dev.svc.cluster.local direct-response
```

## Metrics  

Currently we implmented a [regional affinity dashboard](https://dashboard.io.comcast.net/d/8uJTab6Vk/regional-affinity) which all services can feel
free to use or could even add similar panels to there service dashboards.  

We added mutiple differnet envoy stats into promethus for each pod such as the amount of retries, Active outliers, and Many more. 
You will find possible stats on this page [here](https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cluster_stats).

A couple key metrics that will be helpful during triage, first is [retry panel](https://dashboard.io.comcast.net/d/8uJTab6Vk/regional-affinity?orgId=409&var-environment=stg&var-region=us-east-1&viewPanel=2) 
which will show you show you how many times the envoy side car proxy is retrying request which generally means the service you are calling may be getting ejected. 
The next panel shows if which services in regions have been ejected from the mesh [active outlier panel](https://dashboard.io.comcast.net/d/8uJTab6Vk/regional-affinity?orgId=409&var-environment=stg&var-region=us-east-1&viewPanel=13)
which will tell you if your service has actually been ejected by istio, you will also be able to mointor and see if it remains ejected or is coming back into service. 
The Last panel to look at is actually two we will be mointoring how many request by service go [across cluster](https://dashboard.io.comcast.net/d/8uJTab6Vk/regional-affinity?orgId=409&var-environment=stg&var-region=eu-central-1&var-service=parental-api&from=now-30m&to=now&viewPanel=20) and how many go 
[intra cluster](https://dashboard.io.comcast.net/d/8uJTab6Vk/regional-affinity?orgId=409&var-environment=stg&var-region=eu-central-1&var-service=parental-api&from=now-30m&to=now&viewPanel=19).
Until further notice we expect it to be roughly even distribution of about 30% intra and 60% extrenal in us and 50/50 in EU. 

## FAQ 

Why is my service throwing 503 with a body of "no healthy upstreams" ? 

* This error is typically thrown from the ingress gateway because the service has been ejected for meeting outlier detection 
and there are no healthy pods to respond to your request. In this scenario internal service calls will stop coming into the region, 
and route  53 will fail over to a different region. 

How long is my service ejected for ? 

* If your service is ejected it will be out of the pool in a given region for [base ejection time is set to](https://github.com/comcast-xvp/xvp/blob/main/infra/aws-eks-tsf/kubectl-istio.tf#L172) Number of times it has been ejected to consecutivly up to 5 minutes is the default max ejection time. 
