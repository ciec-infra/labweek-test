# Running k6 Operator Tests

## Setup
The k6 operator needs to be installed to the cluster where you want to test. If it's not, contact the platform team.

### Context
Ensure that you are pointed to the correct config/context for the region and environment in which you wish to deploy your tests. For instance,

`aws eks --region us-east-1': aws eks --region us-east-1 update-kubeconfig --name xvp-eks-shared-dev`

## Code of conduct
This is a shared resource and it can potentially create a lot of kubernetes resources. You MUST clean these resources up. It's the responsibility of each team to clean up after themselves - see the section below on cleaning up for how to do this. If the clean up does not happen, the platform team will remove the operator from the environment.

## Deploy a test
Create a k6 test as you normally would. The test is going to be saved to the kubernetes environment as a configmap. Once you have written a test you save it to kubernetes as follows:

`kubectl create configmap <config_map> --from-file <test_file_name>`

Here's an example:

`kubectl create configmap simple-test --from-file createSessionWiremock.js

## Testing your wiremock endpoints via the test
Wiremock will be set up with a kubernetes service. Run the following command to see the services available on the cluster:

`kubectl get services -A`

Find the service that corresponds to your wiremock setup. Here is an example for session's wiremock setup:

```
xvp-session-api-devloadqa                       wiremock-devloadqa                                     NodePort       10.100.220.7     <none>        8080:32666/TCP                               27d
```

You need the name of the service, its namespace and its port to construct a url that will hit it directly. For the above example the url would be:

`http://wiremock-devloadqa.xvp-session-api-devloadqa.svc.cluster.local:8080`

or

`http://<service_name>.<namespace>.svc.cluster.local:8080`

Note that the service is not addressable via https - only http.

## Running a test

Tests are run by deploying a custom resource definition that will be recognized by the k6 operator running in the environment. The custom resource definition identifies the configmap previously deploy and instructs the operator to deploy kubernetes jobs to run the test from the configmap.

This operator will deploy these jobs as a couple of pods - and you can retrieve the results of the job by tailing the logs of one of the pods. Note you can also run parallel tests - meaning that it will deploy multiple jobs simultaneously.

### Create the Custom Resource Definition (CRD)

The custom resource definition looks like this:

```
apiVersion: k6.io/v1alpha1
kind: K6
metadata:
  name: <name for the CRD>
spec:
  parallelism: 1
  script:
    configMap:
      name: <name of the configmap>
      file: <original filename used to create the configmap>
```

Provide the 3 fields identified in the CRD above and deploy it to kubernetes as follows:

` kubectl apply -f <filename> `   

### Finding the pods

Once the CRD is deployed, the k6 operator deploys a couple of jobs as pods:

Use `kubectl get pods` to see these pods. When the jobs are completed the status will show as COMPLETED. Note that they are in the default namespace. Here's an example:

```
default                        k6-simple-test-1-tbr27                                            0/1     Completed           0          26m
default                        k6-simple-test-starter-tdx7n                                      0/1     Completed           0          26m
```

### Obtaining Results

At any point while the test is running or when it's completed, tail the logs to see the results. For instance,

`kubectl logs -f k6-simple-test-1-tbr27 `

will show the normal k6 output:

```

          /\      |‾‾| /‾‾/   /‾‾/   
     /\  /  \     |  |/  /   /  /    
    /  \/    \    |     (   /   ‾‾\  
   /          \   |  |\  \ |  (‾)  | 
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: /test/createSessionWiremock.js
     output: -

  scenarios: (100.00%) 1 scenario, 2000 max VUs, 35s max duration (incl. graceful stop):
           * contacts: 600.00 iterations/s for 5s (maxVUs: 800-2000, gracefulStop: 30s)


     ✓ status was 200

     checks.........................: 100.00% ✓ 3001       ✗ 0    
     data_received..................: 22 MB   4.0 MB/s
     data_sent......................: 23 MB   4.2 MB/s
     http_req_blocked...............: avg=201.15µs min=1.7µs    med=2.66µs   max=24.52ms  p(90)=357.24µs p(95)=380.96µs
     http_req_connecting............: avg=163.82µs min=0s       med=0s       max=24.46ms  p(90)=311.5µs  p(95)=332.61µs
     http_req_duration..............: avg=501.02ms min=500.7ms  med=500.83ms max=537.03ms p(90)=501.02ms p(95)=501.39ms
       { expected_response:true }...: avg=501.02ms min=500.7ms  med=500.83ms max=537.03ms p(90)=501.02ms p(95)=501.39ms
     http_req_failed................: 0.00%   ✓ 0          ✗ 3001 
     http_req_receiving.............: avg=65.74µs  min=24.29µs  med=60.41µs  max=1.59ms   p(90)=86.85µs  p(95)=95.96µs 
     http_req_sending...............: avg=45.72µs  min=21.95µs  med=36.63µs  max=2.05ms   p(90)=60.49µs  p(95)=71.65µs 
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s      
     http_req_waiting...............: avg=500.91ms min=500.61ms med=500.72ms max=536.82ms p(90)=500.88ms p(95)=501.19ms
     http_reqs......................: 3001    545.306827/s
     iteration_duration.............: avg=501.51ms min=500.93ms med=501.08ms max=537.27ms p(90)=501.75ms p(95)=502.29ms
     iterations.....................: 3001    545.306827/s
     vus............................: 800     min=0        max=800
     vus_max........................: 800     min=800      max=800

```

# CLEAN UP!!!!!!!

To clean up after yourself, you must delete the CRD and delete the configmap. Deleting the CRD will also delete the jobs. 

## Delete the CRD

`kubectl delete -f <filename>`

Then run `kubectl get pods` to ensure the pods are deleted.

## Delete the configmap

`kubectl delete configmap <configmap name>`





