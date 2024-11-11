# Performance Testing Cluster

Each service has a `perf` environment for performance testing.

The environment hosts each service with the `perf` profile enabled.
Each cluster also hosts a single wiremock server to stage data for the performance test.
You can then set up a `perf` environment configuration that points to the wiremock server.

There is a `perf` tab in each service's deployment pipeline that deploys the `perf` infra
and can run performance tests.

In the pipeline a deployment is triggered every morning at 2 am (EST) to deploy `main`.
`run-load-test-us` and `run-load-test-eu` run after this deployment automatically
(they may be trigger manually whenever as well).

### Custom image tests

Alternatively, the jobs `set-specific-version` can be used to leverage a pinned `image-ecr-ci` version
to deploy instead. `image-ecr-ci` includes builds from `dev-ci` so images from a `dev-ci` deployment
can be pushed to `perf` and tested there. 

*Note:* This will only deploy the image version without any infrastructure changes from the `dev-ci` branch
in question. The infrastructure will remain in sync from `main` - esp. as at 2 am `main` will be re-deployed
again.

## Setup
To set up a service with the performance test environment the service needs to add:

* bash scripts that run tests inside `service/my-api/ci`, see below
* SAT client `id` and `secret` in vault under `xvp/concourse-ci/xvp-sats-perf-us` and 
  `xvp/concourse-ci/xvp-sats-perf-eu`

## Customizing K8s Performance Test Task

You can customize the CPU, memory and timeout of the performance test tasks by adding a `perf:` block to a service's
pipeline's `config.yml`.

```yaml
    perf:
        timeout: 40m
        cpu: 2
        memory: 50Mi
```

### Details

There are 4 performance test jobs in the `perf` tab. Each performance test is actually run as a job
in k8s; not in concourse directly since concourse is unreliable performance-wise.

Each job will call an equivalent bash script in each service's `ci` directory to run performance tests:

| job name           | script that runs                       |
|--------------------|----------------------------------------|
| run-load-test-*    | service/my-api/ci/run-load-test.bash   |
| run-smoke-test-*   | service/my-api/ci/run-smoke-test.bash  |
| run-stress-test-*  | service/my-api/ci/run-stress-test.bash |
| run-other-test-*   | service/my-api/ci/run-other-test.bash  |

The bash script is provided the inputs to the concourse job (in the current directory):

* xvp.git
* xvp-qa-performance.git
* terraform

and some environment variables:

| environment variable        | description                                   | example value                                          |
|-----------------------------|-----------------------------------------------|--------------------------------------------------------|
| HOSTNAME                    | k8s address of service in performance cluster | https://linear-perf.perf.exp.us-east-1.aws.xvp.xcal.tv |
| WIREMOCK_ADDRESS            | address of wiremock pod in cluster            | http://wiremock:8888                                   |
| BEARER_TOKEN                | SAT for making requests with in Auth header   | base64 encoded token                                   |
| AWS_ACCESS_KEY_ID           | used for aws cli access                       |                                                        |
| AWS_SECRET_ACCESS_KEY       | used for aws cli access                       |                                                        |
| AWS_REGION                  |                                               | us-east-1                                              |

See the [Example](#example) below on how to hook up your tests.

The original QA performance InfluxDB is available to the TSF perf cluster. This means to read the test
results you can use [grafana dashboards](https://dashboard.io.comcast.net/dashboards/f/zzd-RQ67k/xvp-exp-performance-test-new)
(if your test is outputting to InfluxDB, see example below on how to do this).

The job in concourse will also report the results in the job logs when finished.
For [example](https://ci.comcast.net/teams/xvp/pipelines/xvp-disco-api/jobs/run-load-test-us/builds/15).

The performance job also has a TTL of 120 seconds. This is for looking at logs or executing kubectl commands if debugging needs to happen.
(`kubectl logs jobs/...` , etc).

### External Pipelines

Even though there are built in jobs for running performance tests in the `perf` tab, each performance deployment
creates a new version in s3 so other concourse pipelines can use it to run their own setups. 

This means you may create external pipelines that use the perf cluster deployments. The pipeline you make
can use the s3 version as a trigger to run its own jobs. You can use the version as a trigger.

### WireMock

One WireMock instance per service is available for staging mock responses.
Each WireMock pod resides separate from the service's node group to not affect performance results.
Performance tests that need to mock upstream should clear the mappings before running and after.

#### Using WireMock in Your Test

1. Create or add to the service's `application-perf.yml` settings file to override an upstream service's endpoint
   to WireMock in k8s
   * Note the DNS name is `wiremock`. The full FQDN includes the K8s namespace: `wiremmock.xvp-$service-perf.svc.cluster.local`
      * By default, in K8s the DNS resolution happens 'within' the same K8s namespace, thus only `wiremock` can be used
      * This makes it locally testable, too: Just map `wiremock` to `localhost` on your developer machine
      * The port is `8888`
      * Wherever you can, use the Terraform output `wiremock_address` instead of hardcoding it!
   * This `perf` Spring profile can be used for other performance related tweaks as well,
     like tweaking a setting then seeing how it affects performance.
   * Example `src/main/resources/application-perf.yml` file:
     ```yaml
     metadata-service-discovery:
       services:
         MerlinEntity:
           root: http://wiremock:8888/entityDataService/data/
           endpoints:
             getProgramsByIds: Program/
     ```
2. Stage mappings from inside the k6 performance test
   * The common k6 WireMock library `common/k6-common/wiremock.js` can make common WireMock tasks 
     easier inside a k6 test
   
See the [WireMock Example](#wiremock-example) below.

#### Accessing WireMock from Your Machine

Go to the perf terraform plan output to find the AWS EKS command to set up your local kubectl
(https://ci.comcast.net/teams/xvp/pipelines/xvp-disco-api/jobs/terraform-plan-apply-perf-us-east-2/):

![img.png](img/kubctl-eks-wiremock.png)

Run the command from the job:

```shell
aws eks --region us-east-2 update-kubeconfig --name xvp-eks-shared-perf
```

If you need to debug WireMock problems, it may be helpful to access the WireMock instance in k8s from
your local machine:

```shell
kubectl port-forward -n xvp-$service-api-perf service/wiremock 8888:8888
```

After running that command you'll be able to use `localhost:8888` to access the WireMock instance in k8s.

#### Local Performance Test Development

You can develop the performance tests on your local machine for faster iteration.
First, you need to port-forward wiremock to your local machine. Second, just
set up the proper environment variables and run your script with k6:

```shell
aws eks --region us-east-2 update-kubeconfig --name xvp-eks-shared-perf
kubectl port-forward -n xvp-$service-api-perf service/wiremock 8888:8888 &
export WIREMOCK_ADDRESS=http://localhost:8888
export BEARER_TOKEN='<taken from vault>'
# hostname can be found inside terraform output in the perf pipeline, or you can point to a local instance
export HOSTNAME='https://<service-name>.perf.exp.us-east-2.aws.xvp.xcal.tv'
k6 run ./load_test.js
```

### Example

Here is an example on how to have the ` run-load-test-us` and `run-load-test-eu` jobs run a load test every
night for linear-api. Two files below are added to the XVP git repository:

#### `services/linear-api/ci/run-load-test.bash`

```shell
INFLUX_DB="http://${INFLUXDB_USERNAME}:${INFLUXDB_PASSWORD}@100.88.89.253:8086/xvpperformancetestmetrics-linear-us-east-1"
k6 run xvp-qa-performance.git/src/test/xvp-linear-api/load_test.js --out "influxdb=${INFLUX_DB}"
```

#### `services/linear-api/src/k6/bulk_chunks.js`

Environment variables (described in detail above) are accessible through `__ENV.*`:

```js
export default function () {
    let url = `${__ENV.HOSTNAME}/v1/partners/Comcast/stations/chunks`;
    let params = {
        headers: {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
            'Accept-Encoding': 'gzip, deflate',
            'Bulk-Listing-Cache-Enabled': 'true',
            'Authorization': `Bearer ${__ENV.SAT_TOKEN}`,
        },
    };
    const response = http.post(url, payload, params);
    check(response, {
        'status is 200': (r) => r.status === 200,
        'response is not empty': (r) => r.body && r.body.length !== 0
    });
    sleep(1)
}
```

### WireMock Example

#### `src/main/resources/application-perf.yml`

* Note the DNS name is just `wiremock`
* This `perf` Spring profile (application-perf.yml) can be used for other performance related tweaks as well,
   like tweaking a setting then seeing how it effects performance.
  
```yaml
metadata-service-discovery:
  services:
    MerlinEntity:
      root: http://wiremock:8888/entityDataService/data/
      endpoints:
        getProgramsByIds: Program/
```
  
#### example k6 performance test

 * Stages WireMock before the tests are run, then clears the mapping when the test is over
 * `wiremock.js` automatically uses the `WIREMOCK_ADDRESS` environment variable to connect to wiremock
 * You could use the environment variable yourself to do manual WireMock API calls if needed
```js
import {check, sleep} from 'k6';
import http from 'k6/http';
import {addMapping, setupWiremock, teardownWiremock} from "../../../../xvp.git/common/k6-common/wiremock.js";

// to load files from integration test or elsehwere...
// k6 requires files be opened in the init phase (see https://k6.io/docs/javascript-api/k6-data/sharedarray/)
const responseBodies = new SharedArray('ars', function () {
    // All heavy work (opening and processing big files for example) should be done inside here.
    // This way it will happen only once and the result will be shared between all VUs, saving time and memory.
    const f = JSON.parse(open('../integration-test/resources/SpringTest/ars.json'));
    return [f]; // f must be an array
});

export function setup() {
    setupWiremock();
    let [arsResponseBody] = responseBodies;
    addMapping({
        request: {
            headers: {
                Accept: {
                    equalTo: "text/plain"
                }
            },
            method: "GET",
            url: "/some/thing"
        },
        response: {
            body: arsResponseBody,
            headers: {
                "Content-Type": "text/plain"
            },
            status: 200
        }
    });
}

export function teardown(data) {
    teardownWiremock();
}

export default function () {
    // test goes here
}
```
