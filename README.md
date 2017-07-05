# Kuberentes Kafka
This project contains tools to facilitate the deployment of 
[Apache Kafka](https://kafka.apache.org/) on 
[Kubernetes](http://kubernetes.io/) using 
[StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/). 
It requires Kubernetes 1.7 or greater.netes.

## Limitations
1. Persistent Volumes must be used. emptyDirs will likely result in a loss of data.
1. Storage media I/O isolation is not generally possible at this time. Consider using Pod Anti-Affinity rules to place 
noisy neighbors on separate Nodes.

## Kafka Docker Image 
The [docker](docker) directory contains the [Makefile](docker/Makefile) for a [Docker image](docker/Dockerfile) that 
runs a Kafka broker.

## Manifests
The [manifests](manifests) directory contains server Kubernetes [manifests](manifests/README.md) that can be used for 
demonstration purposes or production deployments. If you primarily deploy manifests directly you can modify any of 
these to fit your use case.

## ZooKeeper
Kafka requires an installation of Apache Zookeeper for broker configuration storage and coordination. Manifests and 
helm charts for deploying an ensemble can be found [here](https://github.com/kow3ns/kubernetes-zookeeper). It is 
recommended to deploy a separate small, ZooKeeper ensemble for each Kafka cluster instead of using a large 
multi-tennant ensemble.

## Configuration and Administration
This section contains common configuration and administration items for running Kafka on Kubernetes.

### OS Image tuning
For production use, it is important to configure the base OS image to allow for a sufficient number of file 
descriptors for your workload. 

- For each broker, `(partition) * (partition_size/segment_size)` determines the number of files the Broker will have 
open at any give time. You must ensure that this will not result in the Broker process dying because it has exhausted 
its allowable number of file descriptors.

### Cluster Size
The size of the Kafka cluster, the number of brokers, is controlled by the `.spec.replicas` field of the StatefulSet. 
You should ensure that the size of the cluster supports your planned throughput and latency requirements for all topics.
If the size of the cluster gets too large, you should consider segregating your topics into multiple smaller clusters.

### CPU
Typical production Kafka broker deployments run on dual processor Xeon's with multiple hardware threads per core. 
However, CPU is unlikely to be your bottleneck. An 8 CPU deployment should be more than sufficient for good 
performance. You should start by simulating your workload with 2-4 CPUs and titrating up from there. The amount of 
CPU is controlled by the StatefulSet's `spec.template.containers[0].resources.cpus`.

### Memory
Kafka utilizes the OS page cache heavily to buffer data. To fully understand the interaction of Kafka and Linux 
containers you should read [the Kafka file system design](https://kafka.apache.org/documentation/#design_filesystem) 
and [memory cgroups documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt). 
In particular, its is important to understand the accounting and isolation offered for the page cache for a mem cgroup. 
If your primary concern is isolation and performance you should do the following.

1. Determine the number of seconds of data you want to buffer `t (time)`.
1. Determine the total write throughput of the deployment `tp (storage/time)`. 
1. `tp * t` gives the memory storage requirement that you should reserve. This 
should be set as the memory request for the container using the StatefulSet's `.spec.template.containers[0].resources.mem`. 
You must also account for the JVM heap and process memory. Adding an extra 2-4 Gib is generally adequate.

The JVM heap size of the Kafka brokers is controlled by `KAFKA_HEAP_OPTS` environment variable. "-Xms=2G -Xmx=2G" is 
sufficient for most deployments.

### Disk
Disk throughput is the most common bottleneck that users encounter with Kafka. Given that Persistent Volumes are backed 
by network attached storage, the throughput is, in most cases, capped on a per Node basis without respect to the 
number of Persistent Volumes that are attached to the Node. For instance, if you are deploying Kafka onto a GKE or GCP 
based Kubernetes cluster, and if you use the standard PD type, your maximum sustained per instance throughput is 
120 MB/s (Write) and 180 MB/s (Read). If you have multiple applications, each with a Persistent Volume mounted, these 
numbers represent the total achievable throughput. If you find that you have contention you should consider using 
Pod Anti-Affinity rules to ensure that noisy neighbors are not collocated on the same Node. You can control the amount 
of disk allocated by your provisioner using the `.spec.volume.volumeClaimTemplates[0].resources.requests.storage` field.

### Networking
The `kafka-hs` Headless Service must specify a `server` port that corresponds to the `server` port in the 
`spec.template.containers[0].ports` field of the `kafka` StatefulSet. The `--override listeners` port must also 
correspond 

### Spreading
The Kafka Pod in the StatefulSet's `PodTemplateSpec` contains a Pod Anti-Affinity
and a Pod Anti-Affinity rule.

```yaml
    affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values: 
                    - kafka
              topologyKey: "kubernetes.io/hostname"
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
             - weight: 1
               podAffinityTerm:
                 labelSelector:
                    matchExpressions:
                      - key: "app"
                        operator: In
                        values: 
                        - zk
                 topologyKey: "kubernetes.io/hostname"
```

The Pod Anti-Affinity rule ensures that two Kafka Broker's will never be launched on the same Node. This isn't strictly 
necessary, but it helps provide stronger availability guarantees in the presence of Node failure, and it helps 
alleviate [disk throughput](#disk) bottlenecks.
The Pod Affinity rule attempts to collocate Kafka and ZooKeeper on the same Nodes. You will likely have more Kafka 
brokers than ZooKeeper servers, but the Kubernetes scheduler will attempt to, where possible, collocate Kafka brokers 
and ZooKeeper servers while respecting the hard spreading enforced by the Pod Anti-Affinity rule. This optimization 
attempts to minimize the amount of network I/O between the ZooKeeper ensemble and the Kafka cluster. However, if 
disk contention becomes an issue, it is equally valid to express a Pod Anti-Affinity rule to ensure that ZooKeeper 
servers and Kafka brokers are not scheduled onto the same node.

### Log Retention
Kafka will periodically truncate or compact logs is a partition to reclaim disk space. You can configure the log 
retention using `--override log.retention.bytes` and `--override log.retention.hours` parameters passed to the 
StatefulSet's Pods' container's command. The default will retain logs for 168 hours and never purge messages due to 
log size. You should adjust this based on the desired retention and expected load for your cluster.

### Kafka Application Logging
Kafka's application logs are written to standard out so they can be captured by the default Kuberentes logging 
infrastructure (as is considered to be the best practice for containerized applications). The logging level can be 
controlled by the `KAFKA_OPTS` environment variable. Setting its value to  "-Dlogging.level=<level>", where level is 
one of `INFO`, `DEBUG`, `WARNING`, `ERROR`, or `FATAL` controls the logging threshold

### Readiness and Liveness
The Liveness of a broker is decided by whether or not the JVM process running the broker is still alive. Readiness is 
decided by a readiness check to determine if the application can accept requests.

```yaml 
readinessProbe:
  exec:
   command:
    - sh
    - -c
    - "/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server=localhost:9093"
```