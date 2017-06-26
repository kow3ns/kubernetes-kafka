# Kubernetes Kafka Manifests
Three different manifests are provided as templates based on different uses cases for a Kafka cluster.

1. [kafka.yaml](kafka.yaml) provides a manifest that is close to production readiness.
    - It provides 5 servers with a disruption budget of 1 planned disruption. This cluster will tolerate 1 planned and 
    1 unplanned failure.
    - Each server will consume 12 GiB of memory, 2 Gib of which will be dedicated to the ZooKeeper JVM heap.
    - Each server will consume 4 CPUs.
    - Each server will consume 1 Persistent Volume with 250 GiB of storage.
    - You can tune the parameters as nessecary to suite the needs of your deployment.
    - **The total footprint is 5 Nodes, 20 CPUs, 60 GiB memory, 1250 GiB disk**
1. [kafka_mini.yaml](kafka_mini.yaml) provides a manifest that is suitable for a demos, testing, or development 
use cases where a single Kafka broker is not desirable. 
    - It provides 3 servers with a disruption budget of 1 planned disruption. This ensemble will not tolerate any 
    concurrent unplanned failures during a planned disruption.
    - Each server will consume 1 GiB of memory, 512 MiB of which will be dedicated to the ZooKeeper 
   JVM heap.
    - Each server will consume 0.5 CPUs.
    - Each server will consume 1 Persistent Volume with 10 GiB of storage.
    - You can, again, tune this manifest to your specific needs.
    - **The total footprint is 3 Nodes, 1.5 CPUs, 3 GiB memory, 30 GiB disk**
1. [kafka_micro.yaml](kafka_micro.yaml) provides a manifest that is suitable for demos, testing, or development 
    use cases where a single Kafka broker will suffice. 
    - It provides 1 server with no disruption budget.
    - The server will consume 1 GiB of memory, 512 Mib of which will be dedicated to the Kafka JVM heap.
    - The server will consume 0.5 CPUs.
    - The server will consume 1 Persistent Volume with 10 GiB of storage.
    - **The total footprint is 1 Node, 0.5 CPUs, 1 GiB memory, 10 GiB disk**
    
## ZooKeeper 
Each of these manifests depends on a working ZooKeeper ensemble. Manifests for such an ensemble are available 
[here](https://github.com/kow3ns/kubernetes-zookeeper/tree/master/manifests).
You must create a ZooKeeper ensemble prior to launching your kafka cluster.

## Usage 
For this example we will use the small 3 server ensemble from the 
[zookeeper_mini.yaml](https://github.com/kow3ns/kubernetes-zookeeper/tree/master/manifests/zookeeper_mini.yaml) manifest, 
an the small 3 broker Kafka cluster from the [kafka_mini.yaml](kafka_mini.yaml) manifest.

### Creating the ZooKeeper Ensemble
Use `kubectl apply` to create the ZooKeeper ensemble contained in the manifest.

```shell 
kubectl create -f zookeeper_mini.yaml 
service "zk-hs" created
service "zk-cs" created
poddisruptionbudget "zk-pdb" created
statefulset "zk" created
```

Watch for all of the Pods in the StatefulSet to become Running and Ready.

```shell 
kubectl get po -lapp=zk -w
NAME      READY     STATUS              RESTARTS   AGE
zk-0      0/1       ContainerCreating   0          6s
zk-1      0/1       ContainerCreating   0          6s
zk-2      0/1       ContainerCreating   0          6s
zk-2      0/1       Running   0         11s
zk-1      0/1       Running   0         11s
zk-0      0/1       Running   0         19s
zk-1      1/1       Running   0         20s
zk-2      1/1       Running   0         20s
zk-0      1/1       Running   0         30s
```

### Creating the Kafka Cluster
You can create the Kafka cluster by using `kubectl apply` in the same way you did [above](#creating-the-zookeeper-ensemble).

```shell
kubectl create -f kafka_mini.yaml
service "kafka-hs" created
poddisruptionbudget "kafka-pdb" created
statefulset "kafka" created
```

Wait for all of the Pods to become Running and Ready.

```shell
kubectl get po -lapp=kafka -w
NAME      READY     STATUS    RESTARTS   AGE
kafka-0   1/1       Running   0          31s
kafka-1   1/1       Running   0          31s
kafka-2   1/1       Running   0          31s
```

### Testing the Cluster
First you will need to create a topic. You can use `kubectl run` to execute the `kafka-topics.sh` script.

```shell 
kubectl run -ti --image=gcr.io/google_containers/kubernetes-kafka:1.0-10.2.1 createtopic --restart=Never --rm -- kafka-topics.sh --create \
--topic test \
--zookeeper zk-cs.default.svc.cluster.local:2181 \
--partitions 1 \
--replication-factor 3
```

Now use `kubectl run` to execute the `kafka-console-consumer.sh` command and listen for messages.

```shell 
kubectl run -ti --image=gcr.io/google_containers/kubernetes-kafka:1.0-10.2.1 consume --restart=Never --rm -- kafka-console-consumer.sh --topic test --bootstrap-server kafka-0.kafka-hs.default.svc.cluster.local:9093
```

In another terminal, run the `kafka-console-producer.sh` command.

```shell 
kubectl run -ti --image=gcr.io/google_containers/kubernetes-kafka:1.0-10.2.1 produce --restart=Never --rm \
 -- kafka-console-producer.sh --topic test --broker-list kafka-0.kafka-hs.default.svc.cluster.local:9093,kafka-1.kafka-hs.default.svc.cluster.local:9093,kafka-2.kafka-hs.default.svc.cluster.local:9093 done;
```

When you type text into the second terminal. You will see it appear in the first.

