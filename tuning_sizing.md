# Tuning and Sizing Guidance for Microservices
# Problem statement
Because XVP micro-service components are built around Reactor and Netty, the role of infrastructure and how that infrastructure is used in responding to incoming requests is critical to the tuning effort.

At the same time, because microservices are deployed into EKS, the memory and cpu footprint of individual pods is highly configurable. The way pods are deployed onto nodes has numerous options as well. One could deploy numerous pods with low resource footprint on large or small nodes. It is difficult to know what the appropriate configurations should be, how they will perform and what to do when they don't.

# Caveat
This is a Work In Progress and will be updated. It's based on a relatively small amount of tuning for a small number of relatively simple mocked endpoints.

Some settings recommended here (especially client configuration settings) might be different for the load scenarios of a particular service or endpoint.

The recommendations are based on using a mocked, representative test as the basis for a projection or estimation of what should work in the real world. 


# Some basic starting principles and constraints
* The number of "Comcast" ip addresses is constrained - we cannot scale by provisioning a large number of nodes for each service, because we are constrained by the number of Comcast IP addresses that can be assigned to those nodes.
* It makes sense to *slightly* over-provision each pod at the start and then optimize further. One answer to the wide array of configuration options might be configure every pod endpoint for 1000 requests per second and then forget about it. However, that is both wasteful and not helpful to any effort to tune the service later. Over-provisioning slightly allows you to target a number of requests per second for each endpoint and tune its configuration without the disruption of not having enough resources.
* When deriving a node-pod topology, we want to ensure high availability by deploying enough nodes so that at least one is placed in each Availability Zone (AZ) and at least one pod is deployed in that AZ.

## Netty memory model

While XVP is a 'regular' Spring/JVM application and should be treated as such during sizing, 
the usage of Netty introduces an additional facet to that exercise:

*Netty uses _Direct Memory_ for it's operation*

### Background

Netty uses direct memory to 

* avoid the impact of GC on its operation
* avoid an additional copy operation on an OS level between the network stack and the application stack itself.

Both of those improve the performance of Netty:

### Impact

Usually, a JVM application is sized mainly via and for its Heap memory.
JVM off-heap memory must be considered but to a lesser extent as massive amounts of threads are not used in XVP (thanks to Reactor)
and the JVM itself has become quite good and keeping its off-heap memory usage reasonable.

Though with Netty, we have no 3 sizing areas of memory:

* `Xmx`/`Xms` for the heap size (and their derived parameters for the various generations etc.)
* `MaxDirectMemorySize` to size the direct memory
  * Note: `MaxDirectMemorySize` has no percentage-of-memory version unlike `Xmx`/`Xms`
  * Netty can use the JVM's direct memory or allocate its own native memory. To keep things reasonably complex
    it is recommended for XVP to use the JVM's direct memory via `-Dio.netty.maxDirectMemory=0`. Otherwise there
    would be 4 areas to size.
  * It is especially important to note that by default, the direct memory is set to the maximum heap size.
    This means the total memory consumption of the JVM can be *twice* of `-Xmx`!
* Various parameter that indirectly size the off-heap size 
  * Code caching, String handling & Thread/stack sizes

### Conclusion

To properly size the JVM memory load tests should be used. 
* The metric `jvm_memory_used_bytes{area="heap"}` provides insights into the heap usage in conjunction with 
  the GC performance via `sum by(pod_name) (rate(jvm_gc_pause_seconds_sum{}[$__interval])`. By comparing values of 
  "used" memory and "max" memory (`jvm_memory_max_bytes` metric and `-Xmx` JVM setting), we can see if heap usage is 
  within the desired range. As a guide line we try to keep heap memory usage within 60-80 percent of the max value.
* The metric `jvm_memory_used_bytes{area!="heap"` provides insights into the off-heap usage.
  * The various parts of the off-heap memory are reported individually
* The Netty direct memory consumption is available via `reactor_netty_bytebuf_allocator_used_direct_memory{}`

All three parts make up the vast majority of the used memory of a given XVP container.
The overall memory consumed by the whole pod is reported as `container_memory_working_set_bytes{}`.

Also, the `limit` (maximum memory available per pod) is reported via `container_spec_memory_limit_bytes`.
This limit is configured per service via `max_memory` and is *strictly enforced* by EKS.
If a container uses more memory than this limit, EKS does kill the pod immediately. Such a pod will be restarted and its
`restart` counter will be increased but log the event:

```
$ kubectl describe pod -n $NAMESPACE $POD_NAME

Containers:
  $service-api:
    State:          Running
      Started:      Fri, 28 Jan 2022 05:09:10 -0500
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Thu, 27 Jan 2022 10:04:30 -0500
      Finished:     Fri, 28 Jan 2022 05:09:09 -0500
    Ready:          True
    Restart Count:  1
```

In conclusion, each service must properly set the JVM parameters:

* `"-Dio.netty.maxDirectMemory=0`

and adjust their

* `-Xmx`, `-Xms`, and `-XX:MaxDirectMemorySize=1g` in `ci/Dockerfile`
* `min_memory` / `max_memory` in the respective `.tfvars`

### References

Good explanation:

* https://www.programmersought.com/article/9322400832/

Recommendations:

* https://stackoverflow.com/questions/42651707/how-to-find-a-root-cause-of-the-following-netty-error-io-netty-util-internal-ou
* https://stackoverflow.com/questions/67068818/oom-killed-jvm-with-320-x-16mb-netty-directbytebuffer-objects/67069998#67069998

NMT:

* https://www.baeldung.com/native-memory-tracking-in-jvm
* https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html
* https://stackoverflow.com/questions/62635023/native-memory-consumed-by-jvm-vs-java-process-total-memory-usage

# Step 1: Estimate Load Profile
Before any discussion of "tuning" or "sizing," an estimate of actual load is needed. While this estimate may be very difficult to derive, a baseline is needed so that when monitoring shows that a particular endpoint is receiving more or less traffic than first estimated, the configuration can be sized and tuned appropriately. Even if service owners don't really know what the load will be, they should start with some assumption, compare everything that follows to that assumption and adjust accordingly. In addition, because some tuning happens at the client level, it's important to understand what requests go through which client so that client configuration settings can be sized appropriately.

So the first step is to build out a table like this:

| Inbound Endpoint  | Normal RPS load | Peak RPS load  | Upstream Client class|
|-------------------|-----------------|----------------|----------------------|
| Endpoint 1        | 800             |2000            | Client1             |
| Endpoint 1        |                 |                | Client2             |
| Endpoint 1        |                 |                | Client3             |
| Endpoint 2        | 10,000          |20,000          | Client3             |
| Endpoint 3        | 100             |250             | Client1             |
| Endpoint 3        |                 |                | Client2             |
| Endpoint 3        |                 |                | Client3             |
| TOTALS            | 10,900          |22,250          |                     |

In addition to the total projected throughput, we can also see that :

* Client 1 is used for a normal load of 900rps and peak load of 2,250.

* Client 2 is used for a normal load of 900rps and peak load of 2,250.

* Client 3 is used for a normal load of 10,900rps and a peak load of 22,250.

# Step 2: Adjust for long or short upstream calls
The load testing reference models have been built using a single upstream call with a built-in 500ms delay. If any of the above upstream calls take longer, more resources (CPU and memory) may be required, because there will be more queuing and "back pressure."  To make this adjustment, multiply the number of requests per second proporationally. If the call takes 2 seconds - multiply times 4. 500ms * 4 = 2 seconds. Remember, this is for estimate purposes.

If a call is normally less than 500ms, do not adjust. It's better to over-provision slightly.

# Step 3: Evaluate Regions and Availability Zones
For XVP, we want to follow the best practices of making our services highly available. For any service, this means that we have the goal of deploying at least one instance of the service in each availability zone of each region in which it is deployed so that in the case of an outage at the AZ or regional level, the other regions and availability zones can pick up load and scale up.

Given multiple nodes to deploy, EKS will attempt to distribute them across availability zones. Production regions currently have the 3 availability zones each.

Given the load above and the principle of high availability, we can derive the minimum number of requests per AZ and per region. If we have 2 regions, each with 4 Availability Zones, this means we can divide the normal and peak RPS by 8 to derive these minimums - 1362 RPS and 2781 RPS when all zones are available. However, the maximum should be calculated based on a whole region going down (at least) - which would put the peak at 5562 RPS. Note the degree of availability is impacted by how much load can be packed into a single region or AZ. If a region goes down and then another AZ goes down within the remaining region of this example, the remaining region might not be able to handle the peak load. Ideally, there are three regions in a given continent grouping, however.

# Step 4: Pick a starting configuration
Given these constraints, the next step is to pick a starting configuration. 

Pod configuration profiles are rated at 3 different loads - 1000 rps, 600 rps and 300 rps with the following CPU and memory limits;

| Requests per second (rps)  | CPU limit | Memory Limit  | Tested p95 performance*|
|-------------------|-----------------|------------------|------------------------|
| 1000              | 3000m           | 3000Mi           | 557.34ms
| 600               | 1800m           | 1800Mi           | 558.31ms
| 300               | 900m            | 900Mi            | 570.18ms      

*based on 500ms delay from upstream call

To support the minimum requirement, each AZ would need 2 1000 rps pods or 3 600 rps pods or 5 300 rps pods.

To support the maximum requirement, each AZ would need 6 1000 rps pods or 10 600 rps pods or 19 300 rps pods.

Note that these rps configurations are somewhat arbitrary, and one could derive a 500 rps (or some other fraction of 1000) just by adjusting the CPU and memory limits proportionally.  However, similar adjustments would also need to be made to other settings including JVM settings and server configuration settings.

To pick an initial pod config, it makes sense to think about how you are likely to scale - if load shifts in smaller increments, it will be more effective to use smaller pods. Whereas if there are bursts, it might be more efficient to deploy large pods so that more traffic is handled with each increment.

# Step 5: Pick a node instance type and size
In addition to the pods, the Vector daemonset and other "utility" code will be run on each node. Consequently, it's important to leave about 20% of a node for these utility functions. Vector as a daemonset (see below) will require at least 1 cpu and 2 GB of memory (to be updated).

Node instance types that are supported can be found [here](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#supported-instance-types). In picking a size, it's important to think about the pod configuration and it's maximum resource footprint and how that footprint will scale under load.

Nodes are scaled by an autoscaling group - in the case above, the number of pods per AZ might just fit within a single instance type - for instance, a single `c5.9xlarge` could probably handle the maximum load. If however, those peak times are not frequent, then you might be paying for a lot of excess resource when you do not really need it.

# Step 6: Configure pod min/max, node min/max, desired count and resource requests and limits
For each region, given the above constraints calculate the number of minimum and maximum pods and minimum and maximum nodes.

The following variables need to be configured for each region;

* min pod count
* max pod count
* min node count
* max node count 
* desired count (for nodes)
* max cpu
* min cpu - 2/3 of max cpu
* max memory
* min memory - 1/3 of max memory


# Step 7: Configure the JVM settings in the Docker file
In the Docker file for the service found in the `ci` directory, there is a a line where the JVM is created in the Docker container. It looks something like this typically:

`ENTRYPOINT ["java", "-jar", "/app/app.jar"]`

The following JVM settings should be added:

* `-Xmx`: this should be expressed as .8 times the max memory value defined in the deployment descriptor for the service. For instance, if the memory is 3GB, then this parameter could look like this. Note that it is expressed in megabytes not gigabytes here:
  `"-Xmx2400m"`
* `-Xms`: should be equal to `-Xmx` and look like this: `"-Xms2400m"`

The `ENTRYPOINT` line above would now be transformed to the following, if the maximum memory is 3GB:

`ENTRYPOINT ["java", "-Xms2400m", "-Xmx2400m",  "-jar", "/app/app.jar"]`

# Step 8: Configure the Netty web server

The Netty web server and the client use a threading model based on something called an 'event loop.' You can read more about Netty's event loop model in Spring Netty [here](https://dzone.com/articles/spring-webflux-eventloop-vs-thread-per-request-mod) and [here](https://programmer.help/blogs/learn-more-about-how-netty-works.html). It is not a thread pool, does not use the "thread-per-request" model and should not be tuned as such - the number of threads should be very small. 

In load testing, adding a designated "selector" or "boss" thread slightly improved the performance under load. For the 1000 rps scenario, 1 selector thread is configured and 4 worker threads are configured. 

When configuring this proportionally for other request-per-second configurations, keep the selector thread and multiple 4 times the fraction of 1000 for which you are configuring. (500 rps would use 2 worker threads.)

In addition an idle timeout of 150 seconds and a read timeout of 120 seconds should be configured. Here is an example of a bean that configures the Netty server:

```
    @Override
    public void customize(@NotNull NettyReactiveWebServerFactory factory) {
        factory.addServerCustomizers(builder -> builder.runOn(LoopResources.create("netty-", 1,4, false))
                .idleTimeout(Duration.ofSeconds(150))
                .doOnConnection(conn -> conn
                        .addHandlerLast(new ReadTimeoutHandler(120, TimeUnit.SECONDS))));
    }
```

(Values should use configuration and not be hard-coded.)

# Step 9: Configure the clients

While the webflux client also uses an event loop structure and LoopResources can also be configured for the client, it is generally not necessary. However, there are other parameters that make a significant difference in the way the client performs and the way the service as a whole uses memory and CPU.

The configuration below is what should generally be used. The hard-coded numbers should be used, but all hard-coded values should be converted to Spring configuration. (It is hard-coded here for documentation purposes.) However, the number of `maxConnections` used in the ConnectionProvider should be scaled according to the request per second count. It should be 5 times the number of requests per second *for this client*. `maxConnections` represents the maximum number of connections for each remote host. This number works in conjunction with `pendingAcquireMaxCount` to enable the client to adapt to back pressure when requests have to be queued.

Connections should also be managed as last-in-first-out (`lifo`) rather than the default, which is `fifo`.

```
    @Bean
    public WebClientCustomizer webClientCustomizer() {

        ConnectionProvider connectionProvider = ConnectionProvider.builder("SessionConnectionProvider")
                .pendingAcquireMaxCount(-1)
                .maxConnections(5000)
                .lifo()
                .evictInBackground(Duration.ofSeconds(30))
                .pendingAcquireTimeout(Duration.ofMillis(1000))
                .metrics(false)
                .build();

        // create reactor netty HTTP client
        HttpClient httpClient = HttpClient.create(connectionProvider)
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 60000)
                .responseTimeout(Duration.ofSeconds(20))
                .doOnConnected(conn -> conn
                        .addHandlerLast(new ReadTimeoutHandler(10, TimeUnit.SECONDS))
                        .addHandlerLast(new WriteTimeoutHandler(5, TimeUnit.SECONDS)));

        return builder -> builder
                .defaultHeader(HttpHeaders.USER_AGENT, "xvp-session-api")
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .filter(new WebClientLoggingFilter(log, LogLevel.WARN))
                .filter(new RFC7807ErrorFilter());
    }
```

# Additional service-level tuning considerations
As a result of tuning efforts, some additional service-level changes were identified. 

## Changes to the logging framework
At the time of this writing, services use `log4j` with an `http` appender.  In load testing, we found that there are a series of changes that will make logging more efficient.

### Use a socket appender and a socket source in Vector
Socket logging was tested in initial load testing and showed significant improvement. 

#### Vector
To use a `socket` appender, services must deploy Vector with the `source_socket` branch of [observability-clients/xvp](https://github.com/comcast-observability-tenants/xvp/tree/source_socket/vector/xvp-eks). The CI deployment script points to the observability clients branch. The Vector source is configured to listen on port `9900` for incoming log events. It uses an in-memory buffer (not disk) as well. 

When Vector is configured as a sidecar using this branch, the resource settings for its deployment in [infra/aws-eks-tsf/k8s-microservice.tf](https://github.com/comcast-xvp/xvp/blob/main/infra/aws-eks-tsf/k8s-microservice.tf) should be configured like this:

```
resources {
    requests = {
      cpu    = "500m"
      memory = "64Mi"
    }
    limits = {
      memory = "2048Mi"
      cpu    = "1000m"
    }
  }
```

#### log4j
##### Config changes
Changes to the log4j configuration in the service should include using the socket appender pointed at port `9900` and asynchronous logging. Here is an example that was tested:

```
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" monitorInterval="30">
    <Appenders>
                <Socket name="socket" host="0.0.0.0" port="9900">
                    <EcsLayout serviceName="xvp-session-api"/>
                </Socket>
    </Appenders>
    <Loggers>
        <AsyncLogger name="com.comcast" level="info" />
        <AsyncRoot level="info">
            <AppenderRef ref="socket" />
        </AsyncRoot>
    </Loggers>
</Configuration>
```
##### Code changes
In addition, some settings related to the asynchronous ring logging were found to manage logging buffers more efficiently. These changes were made in a static block (which might already exist) in the boot class for the service and include the following:

```
 static {
        System.setProperty("java.util.logging.manager", "org.apache.logging.log4j.jul.LogManager");
        System.setProperty("log4j2.contextSelector", "org.apache.logging.log4j.core.async.AsyncLoggerContextSelector");
        System.setProperty("log4j2.asyncLoggerRingBufferSize", Long.toString(1024L));
        System.setProperty("log4j2.asyncQueueFullPolicy", "Discard");
        System.setProperty("log4j2.discardThreshold", "WARN"); 
  }
```
### Migration of Vector to a node daemonset
The platform team is working on a change that will migrate Vector from a sidecar to a daemonset running on each node (a single pod for the whole node). This means that if multiple services are running on the same node, they will use a single instance of Vector and Vector will listen on a single port on that node. The change will require some slight configuration changes to the log4j configuration file. Stay tuned for updates.

