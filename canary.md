# Canary deployments

## Background

We use [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) for (canary) deployments.

It was selected because

* as it works with AWS ALB and Istio on the Ingress side
* Provides Analytics via Prometheus and REST (Blackbird)
* Is un-opinionated and works with K8s base primitives like `Deployment`s and `Service`s

Other candidates:

* [Flagger](https://flagger.app/)
  * Brings its own Prometheus (XVP has Lynceus already)
  * Uses AWS ServiceMesh which is fairly opaque
  * Clashes with Comcast Governance (role names in Helm charts)
  * No AWS ALB support (not a blocker but requires therefore Istio from the start)

## Interaction with Kubernetes & AWS

![EKS Ingress](./img/EKS_ingress.svg)

* Argo CP uses the `Rollout` CRD to determine how to alters the K8s Ingress object
* ALB CP uses K8s Ingress object annotations (created by Argo) to update the actual AWS ALB resource via the AWS API
  * The canary `Host` header is used to direct traffic on the AWS ALB level to the canary K8s service

## Manual usage

### List all rollouts

```shell
kubectl argo rollouts list rollouts -A
```

### Complete a `pause` step

Pause step in the `canary-release` rollout can be 'completed' by [`promote`](https://argoproj.github.io/argo-rollouts/generated/kubectl-argo-rollouts/kubectl-argo-rollouts_promote/):

```shell
kubectl argo rollouts promote canary-release -n xvp-$SERVICE-api-$ENV
```

### Pause a progressive rollout

Pause the rollout at its current step is possible with this command: This allows for furhter investigations:

````shell
kubectl argo rollouts pause -n xvp-$SERVICE-api-$ENV canary-release
````

### Roll back a progressive rollout

Rolling back a release while the rollout is still happening can be done via the [`abort`](https://argoproj.github.io/argo-rollouts/generated/kubectl-argo-rollouts/kubectl-argo-rollouts_abort/) command.
It removes the new pods and shift traffic back to the old (still existing) pods. It will leave the rollout in `degraded` state.
If the rollout timed already out ("Error: The rollout is in a degraded state with message: ProgressDeadlineExceeded: ReplicaSet "canary-release-ddbf66977" has timed out progressing.") the rollout is already in a degraded state - read on!
Aka. this will cause the traffic to be back at the old pods but leaves the new version pending (ie. for `retry` or further investigation)

```shell
kubectl argo rollouts abort canary-release -n xvp-$SERVICE-api-$ENV
```

To get the rollout in a `healthy`, `stable` state again, issue

```shell
kubectl argo rollouts undo canary-release -n xvp-$SERVICE-api-$ENV
```

This will revert to the previopus state - and leaves the rollout `stable` & `healthy`.
If multiple canary deployments have been tried in the meantime, add `--to-revision=N`

#### Example

```text
NAME                                        KIND        STATUS         AGE    INFO
⟳ canary-release                            Rollout     ✖ Degraded     2d23h
├──# revision:3
│  └──⧉ canary-release-ddbf66977            ReplicaSet  ◌ Progressing  33m    canary,delay:27s
│     ├──□ canary-release-ddbf66977-h4tcb   Pod         ✔ Running      33m    ready:1/1
│     ├──□ canary-release-ddbf66977-mn62f   Pod         ✔ Running      33m    ready:1/1
│     ├──□ canary-release-ddbf66977-nl92h   Pod         ◌ Pending      33m    ready:0/1
│     ├──□ canary-release-ddbf66977-rgf8g   Pod         ✔ Running      33m    ready:1/1
│     ├──□ canary-release-ddbf66977-t2z8f   Pod         ✔ Running      33m    ready:1/1
│     └──□ canary-release-ddbf66977-tsmd4   Pod         ✔ Running      33m    ready:1/1
├──# revision:2
│  └──⧉ canary-release-68c54bd8f7           ReplicaSet  • ScaledDown   24h    delay:passed
└──# revision:1
   └──⧉ canary-release-786964c75f           ReplicaSet  ✔ Healthy      2d23h  stable
      └──□ canary-release-786964c75f-qwbxv  Pod         ✔ Running      2d23h  ready:1/1
```

The rollout is in `degraded` state. Canary #3 was not successful and also #2 was not successful.
In this state, use `undo --to-revision=1` to get back to a healthy, stable state.
There will be a new revision 4 that uses RS `canary-release-786964c75f` - so the existing pods are kept and traffic
is not interrupted.

## Emergency deployment

Emergency deployments are possible, please see [emergency deployment](../../Support/Playbooks/README.md#emergency-deployment).

## Automated rollout

A canary rollout executes the following steps:

1. Preparation
  1. Concourse builds new container image
  2. Concourse runs `terraform apply`
    * This updates the K8s `deployment` with the new image location
    * This does *not* change the pods yet. The ReplicationSet is set to `0` for the `deployment` (Because Argo controls this)
  3. Argo automatically detects the changes to the deployment
    * The first step of the Argo rollout plan is `- pause: {}` to ensure nothing is executed right away `infra/aws-eks-tsf/kubectl-argo.tf:24`
    * Canary endpoints are available to address newly created pods via protected DNS name `$service-canary.$protectedZone` ie. `disco-canary.dev.exp.us-west-2.aws.xvp.xcal.tv`.
      There is no public DNS record for canary deployments
2. Rollout promotion
  1. Concourse runs `kubectl argo rollouts promote canary-release` to [promote](https://argoproj.github.io/argo-rollouts/generated/kubectl-argo-rollouts/kubectl-argo-rollouts_promote/) (go to the next step aka. finish the `pause` step) the rollout plan
  2. Argo goes through the `rollout_plan`
    * The Terraform variable `rollout_plan` allows each service to have their own rollout plan.  
      The default plan is to just traffic shift to 100% onto the new pods.
    * Concourse will abort the rollout if it takes longer than 6h or the rollout execution fails itself - this is to ensure that no
      stuck rollout keeps pending forever. A regular `kubectl argo rollouts abort canary-release` is issued.
    * Each service's rollout plan is prefixed with two steps:
      1. Scale out a single canary pod
      2. `pause`
        * This is to have an actual pod target for QA/regression tests via the canary endpoint
          and to actually pause the execution. Otherwise, the `terraform apply` in the pipeline
          would already perform the rollout. But we want this to be a dedicated second job in
          the pipeline (i.e. to inject a QA job in between).
        * Each service can define a `rollout_plan` via the Terraform variables which can be global,
          per environment, per region, or per env-region following the common loading pattern
          of Terraform variables.
        * If additional `pause` steps *without a timeout* are added to a rollout, those must
          be completed via the [dashboard](#dashboard) or (`kubectl`)(#manual-usage) **on every
          deployment.**
        * Example: The following rollout plan will
          1. Create a canary pod (in the background, done for all rollouts)
            * This step waits indefinitely for a `promotion` - which is done by Concourse executing the `promote-canary-to` job
              2. Shift 25% of the traffic to the new pods
              3. Wait 3 minutes. Supported units are `s`, `m`, and `h`.
              4. Shift 50% of the traffic to the new pods
              5. Waits *indefinitely*
                * This step requires a manual `promote`!
              6. Shift 100% of the traffic to the new pods and concludes the rollout
                * The old pods are now removed (in the background, done for all rollouts) as per the Terraform variable `canary_rollout_plan`:
                  ```hcl
                  [
                  { setWeight = 25 }
                  { pause = { duration = "3m"}}
                  { setWeight = 50 }
                  { pause = {} }
                  { setWeight = 100 }
                  ]
                  ```
                  ![Rollout start](./img/argo-rollout-start.png)
3. Rollout finishing
  1. Argo rolls out new pods as needed and keeps the existing ones around
  2. After all pods are healthy, it starts to tear down the 'old' pods
     ![Rollout tear down](./img/argo-rollout-teardown.png)
4. End state
  * The old pods are gone and only new pods take traffic
    ![Rollout finished](./img/argo-rollout-finished.png)

## Dashboard

The [dashboard](https://argoproj.github.io/argo-rollouts/dashboard/) exposes the Argo rollouts **of the current namespace** via a web-ui.

Thus, first set `kubectl`'s namespace:

```shell
kubectl config set-context --current --namespace=xvp-$SERVICE-api-$ENV
```

before running

```shell
kubectl argo rollouts dashboard
```

Afterwards the UI can be reached via [`http://localhost:3100`](http://localhost:3100):
![Argo Dashboard overview](img/argo-dashboard-overview.png)

And individual deployments can be observed
![Argo Dashboard detail](img/argo-dashboard-detail.png)

## Rolling out upstream services in a canary fashion

See [adding upstream services](./istio/upstream-services.md)
