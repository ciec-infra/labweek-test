# XVP Infrastructure Triaging

A reference guide on triaging XVP infrastructure that will continue to grow as the team
learns and adapts its strategies.

Before you begin looking at the various commands below you will first need to make sure that
you have your local machine setup to work with AWS, kubectl, etc.

Please refer to the local setup document found [here](../local-setup.md).

## Helpful Kubectl Commands

* Get all pods currently deployed in the cluster:

  ```shell
  kubectl get pods --all-namespaces
  ```

* Use pod name to get logs for your service:

  ```shell
  kubectl logs $POD_NAME -n $NAMESPACE -c $CONTAINER
  ```

* tty (terminal) access into a pod similar to SSH:

  ```shell
  kubectl exec $POD_NAME -n $NAMESPACE --stdin --tty -- /bin/sh
  ```

  this will create an interactive terminal within the pod for you to run
  linux commands and install tools to help triage

* To copy a file from local machine to a pod running in AWS:

  ```shell
  kubectl cp /tmp/foo $NAMESPACE/$POD_NAME:/tmp/bar -c $CONTAINER
  ```

* To list all of the resources in a namespace regardless of type:

  ```shell
  kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n $NAMESPACE
  ```

* To tail the logs of a service:

  ```shell
  kubectl logs -n $NAMESPACE $(kubectl get po -n $NAMESPACE \
  | egrep -o $POD NAME[a-zA-Z0-9-]+) -f
  ```

* To find all the pods in a given namespace:

  ```shell
  kubectl get pod -n $NAMESPACE -o wide
  ```

* To find node resource utilization usage in a cluster:

  ```shell
  kubectl top node
  ```

* To find pod resource utilization usage:

  ```shell
  kubectl top pod -A
  ```

* Sometimes you may need to run a command a couple of times which can be very
manual and cumbersome, instead use the `watch` command which will display the
output in a given interval without running the kubectl command over and over again:

note: need to have `watch` installed via `brew install watch`

  ```shell
  watch $kubectl_command -n $INTERVAL
  ```

> ex: `watch kubectl top node -n 5`

The watch command above for example will run every 5 seconds until you exit out via ctrl + c

* To find number of pods running on a node in json format:

  ```shell
  kubectl get po -o json --all-namespaces | \
     jq '.items | group_by(.spec.nodeName) | map({"nodeName": .[0].spec.nodeName, "count": length}) | sort_by(.count)'```
  ```

* To find memory limits for a namespace:

  ```shell
  kubectl get pods -n $NAMESPACE -o=custom-columns='NAME:spec.containers[*].name,MEMREQ:spec.containers[*].resources.requests.memory,
  MEMLIM:spec.containers[*].resources.limits.memory,CPUREQ:spec.containers[*].resources.requests.cpu,CPULIM:spec.containers[*].resources.limits.cpu'
  ```

### Scaling

The Horizontal Pod Autoscaler (HPA) is responsible for scaling the number of
pods in a replica set, deployment, and more based on a set of metrics that it
looks at. More info [here](shared-infra.md#Autoscaling).

This can also be used to do a rolling restart of pods.

To find out what deployment type your service is using run:

```shell
kubectl get hpa -A
```

which outputs

```shell
kubectl get hpa -A
NAMESPACE                   NAME                   REFERENCE                         TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
xvp-disco-api-dev           hpa                    Rollout/canary-release            6%/50%    3         5         3          110d
xvp-ess-api-dev             hpa                    Rollout/canary-release            1%/50%    6         36        6          110d
xvp-linear-api-dev          hpa                    Rollout/canary-release            6%/50%    3         5         3          110d
xvp-mindata-api-dev         hpa                    Rollout/canary-release            5%/50%    3         5         3          99d
xvp-parental-api-dev        hpa                    Rollout/canary-release            0%/50%    6         36        6          107d
xvp-playback-api-dev        hpa                    Rollout/canary-release            14%/50%   2         4         2          110d
xvp-rights-api-dev          hpa                    Rollout/canary-release            5%/50%    3         5         3          108d
xvp-saved-api-dev           hpa                    Rollout/canary-release            2%/50%    6         36        6          70d
xvp-session-api-devloadqa   hpa                    Rollout/canary-release            2%/50%    1         1         1          9d
xvp-transactional-api-dev   hpa                    Rollout/canary-release            8%/50%    3         5         3          110d
xvp-xifa-api-dev            hpa                    Rollout/canary-release            0%/50%    6         36        6          90d
```

If you look at the `REFERENCE` column you'll see reference to `Rollout/canary-release`:
Technically speaking, the HPA scales the (Argo Canary) rollout - which in turn scales the replication set.
Practically speaking, the HPA gets updated by Keda (from the `scaledObject` CRD) - so any manual edit of the HPA object
gets immediately overwritten.

(In K8s installations without Argo, HPA would scale the replication set directly. This is __not__ the case for XVP
services).

#### Edit autoscaling

To alter the boundaries of the autoscaling via HPA/Keda you
must alter the `scaledObject`:

```shell
$ kubectl get scaledObject
NAME                    SCALETARGETKIND                SCALETARGETNAME   MIN   MAX   TRIGGERS   AGE
scaled-canary-release   argoproj.io/v1alpha1.Rollout   canary-release    1     4     cpu        10d
```

The `scaled-canary-release`-object can be either `kubectl edit`ed or patched to just alter the min/max:

```shell
kubectl patch scaledObject -n $NAMESPACE scaled-canary-release --patch '{"spec":{"minReplicaCount":$MIN_NUMBER,"maxReplicaCount":$MAX_NUMBER}}' --type=merge
```

> ex: `kubectl patch scaledObject -n xvp-disco-api-dev scaled-canary-release --patch
> '{"spec":{"minReplicaCount":5,"maxReplicaCount":7}}' --type=merge`
> will tell HPA to operate Disco prod with 5 to 7 pods based on the existing scaling metric (ie. CPU)

_Note:_ The `--type=merge` is needed as the `scaledObject` is a CR. Also be careful because there is [JSON patch](https://tools.ietf.org/html/rfc6902)
and [JSON merge patch](https://tools.ietf.org/html/rfc7386) in case more complex `patch` fragments are used!

Also keep in mind that you cannot scale down to zero with this appraoch! The boundaries must be at least `min=0` & `max=1` -
this is true for all K8s clusters. To actually scale down to zero see the next chapter.
The practical impact/background is that with 0 pods, HPA never scale up again as the 
[underlying algorithm](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details)
will always return 0.

#### Scale to zero

Starting with Keda 2.12, XVP can be scaled to `0` via HPA/Keda in a given region _and disable HPA completely_:

It is required that the rollout is stable and healthy and no canary deployment is happening:
![](./img/healthy-deployment.png)

To scale to zero, run the `kubectl` command to add the annotation `autoscaling.keda.sh/paused-replicas="0"`":

```shell
kubectl -n $NAMESPACE annotate scaledObject scaled-canary-release autoscaling.keda.sh/paused-replicas="0"
```

This will terminate all pods & also completely remove the HPA object itself. This is because the HPA object
requires a `minReplica` of `>0`.

To bring the pods back & let autoscaling via Keda/HPA take control again, remove the annotation again via `kubectl`:

```shell
kubectl -n $NAMESPACE annotate scaledObject scaled-canary-release autoscaling.keda.sh/paused-replicas-
```

This will re-create the HPA object and start `desiredCount` pods (same as during a regular deployment) again until
metrics are available and HPA scales up/down.

#### Edit cron autoscaling

If only the cron-based scale-out should be updated, alter the `scaledObject`:

It is important to note that Keda will always use the maximum pod based on all triggers: 
Ie. If

* `cron` scale-out demand 15 pods and
* CPU-metrics request 17 pods then
* a total of 17 pods will be scaled to

```shell
kubectl patch scaledObject -n $NAMESPACE  scaled-canary-release --type='json' -p='[{"op": "replace", "path": "/spec/triggers/5/metadata/desiredReplicas", "value":"$MAX_NUMBER"}]'
```

#### Scale via Replication Set (In case of a failure)

There might be a situation that an old canary deployment is still around and now
there is not enough compute/memory resources available to start the new canary
pods, leaving Argo in a degraded state:

```shell
⟳ canary-release                            Rollout     ✖ Degraded    34d
├──# revision:4
│  └──⧉ canary-release-6d449cfc45           ReplicaSet  • ScaledDown  14h  canary,delay:passed
├──# revision:3
│  └──⧉ canary-release-f5fc599d9            ReplicaSet  ✔ Healthy     15h
│     ├──□ canary-release-f5fc599d9-s5kgb   Pod         ✔ Running     15h  ready:1/1
│     ├──□ canary-release-f5fc599d9-wfq56   Pod         ✔ Running     15h  ready:1/1
│     └──□ canary-release-f5fc599d9-8db9b   Pod         ✔ Running     15h  ready:1/1
├──# revision:2
│  └──⧉ canary-release-6b5c5d9957           ReplicaSet  ✔ Healthy     32d  stable
│     └──□ canary-release-6b5c5d9957-lzg94  Pod         ✔ Running     15h  ready:1/1
└──# revision:1
   └──⧉ canary-release-5c84c5754c           ReplicaSet  • ScaledDown  34d
```

Here `canary-release-f5fc599d9` takes up all the resources that prevent `canary-release-6d449cfc45`
from being started, thus the `promote` can't happen (as the `canary` is not running).

To solve this,

1. Scale down the unused/previous Canary (not Stable!) replication set `canary-release-f5fc599d9`
   via `kubectl scale --replicas=0 rs/canary-release-f5fc599d9`
2. Re-start the canary rollout via `kubectl argo rollouts retry rollout canary-release`
    * This will trigger `canary-release-6d449cfc45` again to be scaled up
3. Run a regular `promote` (or `full-promote`)

See als ['Having too many deployments'](#having-too-many-deployments) below.



### Cycle pods

Re-running Terraform does not change anything in Kubernetes if the version/image version hasn't changed.

A single, unhealthy pod can be deleted via `kubectl delete pods -n $NAMESPACE $REFERENCE_NAME` and HPA will take care
of restarting pods to ensure the correct amount of pods is available.

To perform a rolling restart of all pods use

```shell
kubectl argo rollouts restart canary-release -n $NAMESPACE
```

> ex: `kubectl argo rollouts restart canary-release -n xvp-disco-api-dev`
> will restart all Disco dev pods one at a time

More helpful commands can be found on the [Kubernetes Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

## Known issues

### Missing software on microservice pod

The image containing the microservices are built with a very small footprint.
Sometimes it is helpful to install some test tools during triaging.

#### Background

Due to the TSF configuration, a simple `apk update` will timeout: Only HTTPS egress traffic is allowed on the pods.

#### Solution

Within the pod, run

```shell
echo "
https://dl-cdn.alpinelinux.org/alpine/v3.12/main
https://dl-cdn.alpinelinux.org/alpine/v3.12/community" >  /etc/apk/repositories
```

afterwards use `apk` as usual.

### Pod fails with `CreateContainerConfigError`

1. Use `kubectl get pods --all-namespaces` to check on the pods
2. Get details via `kubectl describe pod $POD_NAME -n $WORKSPACE_NAME`
3. Check the `events` at the bottom of the page

### Changes to `config-map`s

Config maps are often used to configure pods & daemon-sets.
But terraform has no concept of their relation to a deployment. Therefore, a
change to a `config-map` doesn't trigger a redeployment.

Workaround: Add a label with a hash (of the Terraform `config-map` representation)
to the `spec.template`, which changes when the `config-map` changes.
As any update to a DaemonSet's `.spec.template` will trigger an (`RollingUpdate`)
update, this will trigger a redeployment via Terraform.

### Kubectl fails

> error: You must be logged in to the server (Unauthorized)

If `kubectl` fails with the error above, please re-run `aadawscli`
to ensure you have a valid AWS CLI session token.

> error: You are not authorized to perform "Some operations!"

`aadawscli` and `aws eks update-kubeconfig` goes hand in hand... Make sure
you select the appropriate account and update the kubeconfig with appropriate region

> ex : You might be trying to login to EU dev account, but the config still has your old env regions ( `us-west-2` or something).
> error: unknown object type *v1beta1.CronJob

This happens when the local `kubectl` version doesn't align with the server EKS version.
Downgarde your `kubectl` via downloading an older version:

```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.16.0/bin/darwin/amd64/kubectl
```

Context: The issue affects EKS installations before 1.22. With the [upgarde](https://ccp.sys.comcast.net/browse/XPINF-197)
this should not happen anymore. [Source](https://stackoverflow.com/questions/68902269/kubernetes-create-job-from-cronjob-not-working)

### Manually deleted deployments

As with every Terraform-managed resource: Do not delete (or alter) Kubernetes
resources (for example: Deployments) manually.

The Terraform-Kubernetes provider is even less lenient than the AWS one: A
resource that was manually deleted and is planed for deletion by Terraform will
cause an error during `apply`!

In that situation you must recreate the resource manually (ie. create a
[simple deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment) -
only the name & namespace must match) to the Terraform state back in sync

### Having 'too many' deployments

Terraform may fail with

```shell
Error: Waiting for rollout to finish: 2 replicas wanted; 0 replicas Ready

on ../aws-eks-tsf/k8s/k8s-config_api.tf line 29, in resource "kubernetes_deployment" "config_api":

29: resource "kubernetes_deployment" "config_api" {


Error: Waiting for rollout to finish: 2 replicas wanted; 0 replicas Ready
on ../aws-eks-tsf/k8s/k8s-microservice.tf line 21, in resource "kubernetes_deployment" "_":

21: resource "kubernetes_deployment" "_" {
```

this happens if Terraform doesn't detect the deployment (=pods) to start in its timeout window (usually 10 minutes).
This might be caused by the fact by the autoscaler being absent/not scaling up or a persisting deployment issue.

It is amplified by the fact that Terraform will always deploy the new deployment before it removes the old deployments.
This is great under normal circumstances to ensure constant service (as there is always a pod to handle request) but
in times when there are not enough resources to handle pods this just creates a larger amount of pods that can't start.

#### Triage

Check that the issue exists. Run `kubectl get pods -A`:

```shell
NAMESPACE           NAME                                                              READY   STATUS    RESTARTS   AGE
..
xvp-disco-api-dev   xvp-disco-api-dev-0-0-331-c46b3cac35992abf8bdafcf7c17a26abk76sn   2/2     Running   1          16h
xvp-disco-api-dev   xvp-disco-api-dev-0-0-334-c46b3cac35992abf8bdafcf7c17a26abshkzb   0/2     Pending   0          58m
xvp-disco-api-dev   xvp-disco-api-dev-0-0-336-7ba19ce14fc5bbd010045b70ea96c8d92sfs7   0/2     Pending   0          29m
xvp-disco-api-dev   xvp-disco-api-dev-0-0-336-7ba19ce14fc5bbd010045b70ea96c8d95zqjl   0/2     Pending   0          29m
xvp                 config-api-0-0-331-7e7938848737c47932e1410ea0bb2ca4-685c44d28nm   1/1     Running   0          16h
xvp                 config-api-0-0-334-7e7938848737c47932e1410ea0bb2ca4-685c4468dq8   0/1     Pending   0          58m
xvp                 config-api-0-0-336-7e7938848737c47932e1410ea0bb2ca4-685c442xprg   0/1     Pending   0          29m
xvp                 config-api-0-0-336-7e7938848737c47932e1410ea0bb2ca4-685c44jd9sl   0/1     Pending   0          29m
```

This shows an old deployment being active (`331`) but newer ones that are all in `Pending` state.
While the new version (`336`) isn't started, it will not remove the old deployments.

#### RCA

Investigate why the pods are not starting. Use `kubectl describe pods -n xvp-disco-api-dev xvp-disco-api-dev-0-0-336-7ba19ce14fc5bbd010045b70ea96c8d95zqjl`.\

##### Insufficient worker nodes

Insufficient worker nodes are shown by pods that can't be started because of this event message:

```shell
...
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  2m (x78 over 117m)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory
```

This can be double-checked by investigating the available nodes `kubectl get nodes`:

```shell
NAME                                           STATUS   ROLES    AGE   VERSION
ip-96-102-239-155.us-west-2.compute.internal   Ready    <none>   25h   v1.19.6-eks-49a6c0
ip-96-102-239-174.us-west-2.compute.internal   Ready    <none>   25h   v1.19.6-eks-49a6c0
```

Use the output of `kubectl describe nodes ip-96-102-239-155.us-west-2.compute.internal`
(and the other nodes) to determin what is consuming all the resources.
Ie. adopt the ASG of the node group via the Terraform variables `desire_count`/`min_count`/`max_count`.

#### Solution

If the underlying issue is resolved, all the pending deployments must be removed - by Terraform.
Do ___NOT___ delete the deployments manually via `kubectl`.

The easiest way to do so is to scale down the deployments to have zero pods - but leave the deployment around itself to
cleanup by Terraform.

Get the deployments (pending & running) via `kubectl get deployments -A`:

```shell
NAMESPACE           NAME                                                         READY   UP-TO-DATE   AVAILABLE   AGE
kube-system         aws-load-balancer-controller                                 1/1     1            1           6d4h
kube-system         coredns                                                      2/2     2            2           6d4h
xvp-disco-api-dev   xvp-disco-api-dev-0-0-337-7ba19ce14fc5bbd010045b70ea96c8d9   2/2     2            0           9h
xvp-disco-api-dev   xvp-disco-api-dev-0-0-339-7ba19ce14fc5bbd010045b70ea96c8d9   0/2     0            0           7h1m
xvp-disco-api-dev   xvp-disco-api-dev-0-0-340-7ba19ce14fc5bbd010045b70ea96c8d9   0/2     0            0           6h5m
xvp-disco-api-dev   xvp-disco-api-dev-0-0-341-7ba19ce14fc5bbd010045b70ea96c8d9   0/2     0            0           5h32m
xvp-disco-api-dev   xvp-disco-api-dev-0-0-342-7ba19ce14fc5bbd010045b70ea96c8d9   0/2     0            0           3h32m
shared              config-api-0-0-337-7e7938848737c47932e1410ea0bb2ca4          2/2     0            0           9h
shared              config-api-0-0-339-7e7938848737c47932e1410ea0bb2ca4          0/2     0            0           7h1m
shared              config-api-0-0-340-7e7938848737c47932e1410ea0bb2ca4          0/2     0            0           6h5m
shared              config-api-0-0-341-7e7938848737c47932e1410ea0bb2ca4          0/2     0            0           5h32m
shared              config-api-0-0-342-7e7938848737c47932e1410ea0bb2ca4          0/2     0            0           3h32m
```

And scale all (or all but the currently running one) down to zero via\
`kubectl scale  --replicas=0 -n $NAMESPACE deployemnt/$name`.
Use this bash script, replace the `grep` part to mass-edit it. This will remove
_all_ running pods in your current region!

```shell
kubectl get deployments -A \
  | grep disco-api \
  | while read ns n x; do kubectl scale  --replicas=0 -n $ns deployment/$n; done
```

Afterwards run Terraform (ie through a regular `build-image`) and let it cleanup.

### `Error: ImagePullBackOff`

Service images stored in AWS ECR for all environments and regions. This error can happen if the image version
referenced is not in ECR. You may need to upload the image to ECR manually. More information can be found in
[Elastic Container Repository](ecr.md#Elastic Container Repository) document

### `Error: CrashLoopBackOff`

A pod can run into this error when it is attempting to be in a `running` state but fails due to an underlying issue such
as the pod being unable to pull an image or a service-side startup issue.

Run `kubectl get pods -A` to view the status of all the pods in the EKS cluster

```shell
NAMESPACE           NAME                                                              READY   STATUS             RESTARTS   AGE
xvp-disco-api-dev   xvp-disco-api-dev-0-0-442-93a159d4047dbfdb9b0167efff333eb87wq65   1/2     CrashLoopBackOff   17         62m
```

In a `CrashLoopBackOff` scenario the pod will continue to restart itself until either it becomes stable or it is removed
from the cluster.

For investigating `CrashLoopBackOff` issues that are pod or Kubernetes based, the `kubctl` command does not give much
information beyond the status of the pod, so to dig further you can run `kubectl describe pod $podName -n $nameSpace`
and navigate to the `Events` section that will provide information on what happened to the pod in question.

However, if there's startup failures in Splunk or ELK that indicate the cause is a service-side startup issue (e.g. a
bug that aborts startup if any DB host is down, and there happens to be one down at the time of deployment), then the
proper solution is to simply rollback by repinning the previous version and triggering a new build with the ⊕ button.
Do not bother with `kubectl abort` or `kubectl undo`.

### Autoscaling / HPA

#### Triage ASG

Check the ASG via the AWS console. Use the `tags` of the AGS to find the matching group for your service.

#### Triage resources

* Check the deployment's `Limit` & `Resource`
* Use `kubectl top pod -A` to get current pod metrics

#### Triage HPA

To manually check on the autoscaling / HPA behavior you can check with the `hpa` resource:
`kubectl describe  hpa -n xvp-session-api-dev`:

```shell
Name:                                                  hpa
Namespace:                                             xvp-session-api-dev
...
Reference:                                             Deployment/xvp-session-api-devscale-0-0-6-7b8d46a7592651380986baf90bd6f408
Metrics:                                               ( current / target )
resource cpu on pods  (as a percentage of request):  24% (208m) / 50%
Min replicas:                                          1
Max replicas:                                          3
Deployment pods:                                       1 current / 1 desired
Conditions:
Type            Status  Reason              Message
  ----            ------  ------              -------
AbleToScale     True    ReadyForNewScale    recommended size matches current size
ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
Type     Reason             Age                   From                       Message
  ----     ------             ----                  ----                       -------
Normal   SuccessfulRescale  45m                   horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```

Check for the `Conditions` & the `events` to identify what happened.

##### HPA Metrics Issue

If a service is unable to scale, whether through organic traffic or load testing, it is usually a good sign to look at where the smoke may be, HPA. It may be that HPA is unable to scale a service pod if it cannot calcuate *how* or *when* to scale up/down.

Run `kubectl describe hpa -n $serviceNamespace` to inspect information on your service HPA setup/

```shell
...
Metrics:                                               ( current / target )
  "s1-prometheus" (target average value):              0 / 80
  "s2-prometheus" (target average value):              0 / 50
  "s3-prometheus" (target average value):              0 / 1k
  "s4-prometheus" (target average value):              0 / 10k
  resource cpu on pods  (as a percentage of request):  <unknown> / 50%
...
Conditions:
  Type            Status  Reason                   Message
  ----            ------  ------                   -------
  AbleToScale     True    SucceededGetScale        the HPA controller was able to get the target's current scale
  ScalingActive   False   FailedGetResourceMetric  the HPA was unable to compute the replica count: failed to get cpu utilization: missing request for cpu in container istio-proxy of Pod canary-release-5988f7c65d-gxk7w
  ScalingLimited  False   DesiredWithinRange       the desired count is within the acceptable range
Events:
  Type     Reason                   Age                    From                       Message
  ----     ------                   ----                   ----                       -------
  Warning  FailedGetResourceMetric  2m37s (x254 over 67m)  horizontal-pod-autoscaler  failed to get cpu utilization: missing request for cpu in container istio-proxy of Pod canary-release-5988f7c65d-gxk7w
```

Note this line:

`resource cpu on pods  (as a percentage of request):  <unknown> / 50%`

The `unknown` will be a key signal that HPA is unable to calculate as it cannot determine the CPU of a container on a pod.

This is further supported when you look at the `Events` of the HPA:

`failed to get cpu utilization: missing request for cpu in container istio-proxy of Pod`

which tells us that the istio-proxy pod, in this instance, does not have a request set for CPU.

All pods need to have both CPU and memory defined for a given requests and/or limits. You can learn more from the [offical Kubernetes docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).

You can reach out to the XVP Infra team and your service team to determine if any changes were introduced that could have impaired HPA for your service.

### AWS resources behave strange

"Regular" AWS triaging starts with a review of the security groups in case of any misbehaving component.
For AWS EKS & K8s in general, the first step is to review the `label`s & `selector`s of the involved components:

For example if the LB & nodes can't communicate, the labels & selectors of both must be checked. Chances are that
they not align and although it _looks_ like a firewall (Security Group) issue, the actual problem is that the
labels don't align with the selects and therefore the 'resource is not found' (in the example above the instance by the
LB).

### Vector

The Vector STDOUT contains
> WARN vector::topology::config::vars: unknown env var in config: “KEY_PASS_ENV_VAR”

on startup. This is a red hering and doesn't hurt.

#### ELK Logs are missing

Vector will continuously ship all XVP service logs to ELK. If ELK logs are missing or dropped, it could be either a Vector or ELK issue.

In each EKS cluster, there is a Vector pod per node, inspect one of the pods to look at its logs.

1. Authenticate to AWS
2. Select the appropriate AWS account
3. Authenticate to the EKS cluster
4. Run the following command

`kubectl logs $vectorPodName -n vector`

Example error logs:

```json
2024-06-17T13:42:20.861243Z ERROR sink{component_kind="sink" component_id=daas component_type=elasticsearch component_name=daas}:request{request_id=1355640}: vector_common::internal_event::service: Internal log [Service call failed. No retries or retries exhausted.] has been suppressed 29 times.
```

From the logs, you can see where the issue lies. In an instance where ELK is throwing rate limit, authentication errors. etc you can reach out in Slack in the [#observability-logging-support](https://comcast.enterprise.slack.com/archives/CAM4CRM41) to reach out to ELK team for support.

If there is an issue with Vector, it will affect all Vector pods in a cluster and be a cluster-wide impacting event. In such instances, please reach out to the XVP Infra team to assist with triaging.

### Dev-ci

Services in a dev-ci environment may sometimes be abandoned or the concourse pipeline is having difficulties at trying
to destroy the service and clean up the resources. In such a scenario a user can manually delete the resources however
note:

While deletions should always be executed by destroying the Kubernetes terraform state, it's possible
that something could be seriously wrong, and the state is corrupted. In such situations, Kubernetes resources
associated with a namespace can be removed by removing the namespace.

To learn more on how to delete a dev-ci service, please reach out to the XVP platform team in Slack. As this can be a
manual and sometimes dangerous process it is best to seek guidance from them.

This command or any deletion commands should be used as a ___last resort___. The one exception is that you may
wish to delete a pod so that a new pod is scheduled and the application is effectively "restarted."

But wait there's more! After deleting the namespace there will be leftover resources that will need to be cleaned up,
look for the following resources that you should remove as well:

* IAM roles
* ALB
* Certificates
* Route 53 records

searching by your branch name should reveal any resources in case they are there

### Istio

Istio sits behind an AWS Network Load Balancer (NLB) that the public Route53 endpoints points to. In the event of an
unhealthy region, Route53's health checks will fail and detect it as a signal to perform automatic failover to another
region that is reporting healthy.

However, there may be situations where a human operator needs to intervene to perform an emergency
traffic shift. One can do so by following the information in the [traffic shift docs.](../traffic_shift.md)

#### Kiali

The [Kiali UI](https://kiali.io/) is available per region at `https://kiali.$ENV.exp.$REGION.aws.xvp.xcal.tv/kiali`

#### Envoy

Istio heavily uses Envoy - which is the actual proxy deployed (usually as sidecars).
Triaging any Istio configuration issue usually means checking the Envoy configuration.

##### Checking Envoy

Envoy is deployed on the Ingress Gateway (IGW) as also on the pods.

First get the IGW pods via

```shell
kubectl get pods -n istio-ingress
```

Then use `kubectl logs -n istio-ingress $POD_NAME` from the previous output to see any errors from misconfigured Envoy
filters. Under normal operations, the IGW logs should only show access log-style messages!

The [admin interface](https://www.envoyproxy.io/docs/envoy/latest/operations/admin) can be found on port `15000`. Use
`kubectl port-forward` and your local browser via [`http://localhost:15000`](http://localhost:15000)
after running:

```shell
kubectl port-forward -n istio-ingress pods/$POD_NAME 15000:15000
```

Check the [`/config-dump`](http://localhost:15000/config_dump) output - it includes _a lot_ of
information including secrets. Be careful with sharing it!
To narrow down the content to review, only check the `dynamic_listeners` section - this is the part
that is "Istio". Other parts of the config are bootstrap (core) Envoy itself.
The output can be filtered down via [`/config_dump?resource=dynamic_listeners`](http://localhost:15000/config_dump?resource=dynamic_listeners)

* Check that the filters exists that you would expect to be there?
* Check that filters don't have error messages
  * Configuration issues may appear in the IGW log itself
    * Example:

      ```json
      {
        "last_update_attempt": "2022-06-30T20:45:19.724Z",
        "details": "Proto constraint validation failed (HttpConnectionManagerValidationError.StatPrefix: value length must be at least 1 characters): "
      }
      ```

  * Syntactical errors may (ie missing configuration parts due to misplaced whitespaces) be only visible in the
    `config-dump`
  * The `/config-dump` is the "active configuration", so after all `MERGE`s and `PATCH`s have been applied

#### High latency with intra-mesh service calls

See the `xvp-infra-core` docs [HERE](https://github.com/comcast-xvp/xvp-infra-core/blob/main/docs/troubleshoot.md#increased-latency-between-services)

### Distroless Containers

Some third party images are distributed as [distroless containers](https://github.com/GoogleContainerTools/distroless)
having just the application code and its runtime dependencies, and being devoid of all the usual trappings of a linux
distribution (shells, package managers etc). While this is desireable from a security standpoint it can complicate
debugging or exploring the pods running these containers.  A distroless image will prevent such tactics as obtaining
terminal access to the pod as decribed [here](#Helpful Kubectl Commands). This is because the image doesn't provide
any shells.

Kubernetes provides the concept of [ephemeral containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
which may be helpful here, but use of such containers is explicitly disallowed in our kubernetes clusters.
There is however an effective workaround.

Assuming your debugging or exploring needs pertain to how the third party image interacts with other kubernetes objects
we are managing via terraform such as secrets or configmaps and deployment concerns such as volumes and volume mounts,
and not anything specific to the third party application code itself, the following techique can prove quite useful.
We are going to run a modified deployment to schedule a pod that interacts with other kubernetes objects and applies
the same deployment instructions as the pod of interest, but replaces the distroless image with one that allows us to
poke around. Here's how:

1. We find the deployment that scheduled the pod running the distroless image by doing

   ```shell
   kubectl -n $NAMESPACE get deployments
   ```

2. Having identified the deployment we can get its yaml representation by doing

   ```shell
   kubectl -n $NAMESPACE get deployment $DEPLOYMENT -o yaml > $YOURFILE.yaml
   ```

3. The specifics of this next step will vary depending upon your particular circumstance, but we'll be editing the yaml
   file you exported in the previous step.  At a minimum you will want to:
   * Change the name of the deployment so the scheduler can recognize it as a new deployment and schedule a new pod,
     the one we'll use for our debugging or exploration.
   * Change the image specified in the deployment to something that has the trappings of a linux distribution and will
     allow you to gain terminal access to the pod such as `ubuntu:latest`.
   * This new container isn't going to have anything specific to do so it will immediately move to a `Completed` status
     unless we instruct it to hang around. Do so by adding these lines in the container's args configuration:

   ```shell
   - args:
     - sleep
     - 1d
   ```

   There may well be additional changes you'll require based on your specific circumstance.

4. Having modified the deployment yaml, we now wish to apply it to the cluster so that the scheduler will establish the
   desired pod for us.

   ```shell
   kubectl apply -f $YOURFILE.yaml
   ```

5. Now we verify our new pod is running:

   ```shell
   kubectl -n $NAMESPACE get pods
   ```

6. Having identified the pod we can obtain terminal access and go to town, debug and explore to our heart's content:

   ``` shell
   kubectl -n $NAMESPACE exec --tty --stdin $PODNAME -- /bin/bash (or other appropriate shell)
   ```

7. When your need for this pod is complete, be sure to clean up by deleting the new deployment you created.

   ```shell
   kubectl -n $NAMESPACE delete deployment $DEPLOYMENT
   ```

## failed to acquire lock on pool dev-ci 

This error happens when concourse worker node fails and the lock can not be removed. 
**The first thing you want to verify that no other dev-ci jobs are running on that PR with that specific number to ensure we will not break things more.** 
This error can happen for 20 to 30 minutes & is expected if mutiple jobs are running on a single PR. We would not want Destroy & Apply to be run at the same time.

Once you confirmed we need to take action. 

1. We want to [find our given dev-ci folder](https://github.comcast.com/xvp/xvp-dev-ci-locks) 
2. Then we want to navigate to the claimed folder you should see your PR number 
3. If you do create a PR to remove the file with your PR number just inculde a brief reason what happened and why you are removing it. 
