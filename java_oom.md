# Recovering from JVM OOM exceptions

In case of an OutOfMemory (OOM) exception in a service, the service's liveness health check may still pass but the service might not be in a state to respond satisfactorily or efficiently to requests. 

In other words, the service may be left in a state that is responsive enough for the health check but not responsive enough to service requests quickly enough.

In addition, it's sometimes difficult to know what caused the OOM exception and generating a heap dump would be helpful. 

Fortunately, Java provides switches that can be set a the startup of the JVM in the container running in Kubernetes and that can generate the heap dump and then restart the JVM and the service.

## How to Add the Switches

If you set following options 
`"-XX:+HeapDumpOnOutOfMemoryError", "-XX:HeapDumpPath=/home", "-XX:+ExitOnOutOfMemoryError"` 
in the `start` function of `entrypoint.sh` script referenced in the DockerFile for the microservice like this:

```dockerfile
ENTRYPOINT [ "/app/entrypoint.sh" ]
```
```shell
  start() {
     if [ -f /app/init.sh ]; then
       init
     fi
      java -XX:-OmitStackTraceInFastThrow -XX:+ShowCodeDetailsInExceptionMessages -Djdk.attach.allowAttachSelf=true \
           -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home -XX:+ExitOnOutOfMemoryError -XX:+AllowRedefinitionToAddDeleteMethods \
           org.springframework.boot.loader.JarLauncher "$@" &
      export JAVA_PID=$!
      echo "JAVA_PID: " $JAVA_PID
     }
```

it will generate a Java Heap Dump on an OutOfMemoryError to the `/home` directory of the container and then EKS will restart the container. Note that the signal handler will first try to do graceful shutdown before doing force termination.

## How to Retrieve the Heap Dump
With the durable heap dump storage solution in place, the heapdumps can be downloaded from respective environment's S3 bucket. The heapdumps are stored in S3 bucket in the following format:
`xvp-heaplogs/${EKS_NAME}/${AWS_REGION}/${XVP_ENV}/${HOSTNAME}/${TIMESTAMP}_heaplogs.hprof`.
Timestamp format: YYYYMMDDHHMMSS

Sample file path in S3 bucket:
`xvp-heaplogs/xvp-eks-dev-v76/us-west-2/xvp-dev/canary-release-5d4d94b94f-d578x/20240131001510_heaplogs.hprof`

heap dumps can be downloaded from the respective S3 buckets using `aws console` or `aws cli`.

### S3 bucket names for respective environments/regions:

|             US              |          EU                 |        AP        |     AWS Account (respective business region)      |
|:---------------------------:|:---------------------------:|:----------------:|:-------------------------------------------------:|
|       xvp-exp-dev-us        |       xvp-exp-dev-eu        |  xvp-exp-dev-ap  |                        DEV                        |
| xvp-exp-performance-test-us | xvp-exp-performance-test-eu |                  |                        DEV                        |
|       xvp-exp-stg-us        |       xvp-exp-stg-eu        |  xvp-exp-stg-ap  |                        DEV                        |
|       xvp-exp-prod-us       |       xvp-exp-prod-eu       | xvp-exp-prod-ap  |                       PROD                        |
