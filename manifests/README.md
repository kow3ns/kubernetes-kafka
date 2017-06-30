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
You need to configure the Kafka cluster to communicate with the zookeeper ensemble you created above. All of these
[manifests](https://github.com/kow3ns/kubernetes-zookeeper/tree/master/manifests) create a client service that the 
Kafka brokers can use to connect to a running server in the ZooKeeper ensemble. The name of the service is `zk-cs` and 
the CNAME is `zk-cs.default.cluster.local` and its port is `2181`. You should ensure that the 
`--override zookeeper.connect` is set to the correct service and port for your ZooKeeper ensemble as below.

```yaml 
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: kafka
spec:
...
      containers:
      - name: k8skafka
        imagePullPolicy: Always
        image: gcr.io/google_containers/kubernetes-kafka:1.0-10.2.1
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
        ports:
        - containerPort: 9093
          name: server
        command:
        - sh
        - -c
        - "exec kafka-server-start.sh /opt/kafka/config/server.properties --override broker.id=${HOSTNAME##*-} \
          --override listeners=PLAINTEXT://:9093 \
          --override zookeeper.connect=zk-cs.default.svc.cluster.local:2181 
```

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

### Scaling the Cluster
You can scale the cluster by updating the number of `spec.replicas` field of the StatefulSet. You can accomplish this 
with the `kubectl scale` command.

```shell 
kubectl scale sts kafka --replicas=4
```

scales the cluster to four brokers. 

```shell 
kubectl scale sts kafka --replicas=3
```

scales the cluster back down to 3 brokers.

**Note that you may need to manually rebalance the partitions in your topics using the `kafka-topics.sh` command**

### Updating the Cluster 
You can update any of portion of the `spec.template` in the StatefulSet, and the StatefulSet controller will perform 
a rolling update to apply the update to the Pods in the StatefulSet. The Pods will be destroyed and recreated, one at a 
time, in reverse ordinal order.

You can use `kubectl patch` to update fields in the `spec.template`, or you can update a manifest and use 
`kubectl apply` to apply your changes.

In one terminal watch the Pods in the Kafka cluster.

```shell
kubectl get po -lapp=kafka -w
NAME      READY     STATUS    RESTARTS   AGE
kafka-0   1/1       Running   2          1d
kafka-1   1/1       Running   0          1d
kafka-2   1/1       Running   2          1d
```

In another terminal update the cpu resource request.

```shell
kubectl patch sts kafka --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value":"250m"}]'
```
The StatefulSet will update each Pod to apply the modification. It will start with Pod with highest ordinal, `kafka-2`, 
terminate each Pod, recreate the Pod from the modified template, and wait for the Pod to be Running and Ready prior 
to advancing to the next Pod.

```shell
kubectl get po -lapp=kafka -w
NAME      READY     STATUS    RESTARTS   AGE
kafka-0   1/1       Running   2          1d
kafka-1   1/1       Running   0          1d
kafka-2   1/1       Running   2          1d
kafka-2   1/1       Terminating   0         14m
kafka-2   0/1       Terminating   0         14m
kafka-2   0/1       Terminating   0         14m
kafka-2   0/1       Terminating   0         14m
kafka-2   0/1       Pending   0         0s
kafka-2   0/1       Pending   0         0s
kafka-2   0/1       ContainerCreating   0         0s
kafka-2   0/1       Running   0         10s
kafka-2   1/1       Running   0         22s
kafka-1   1/1       Terminating   0         1d
kafka-1   0/1       Terminating   0         1d
kafka-1   0/1       Terminating   0         1d
kafka-1   0/1       Terminating   0         1d
kafka-1   0/1       Pending   0         0s
kafka-1   0/1       Pending   0         0s
kafka-1   0/1       ContainerCreating   0         0s
kafka-1   0/1       Running   0         5s
kafka-1   1/1       Running   0         8s
kafka-0   1/1       Terminating   2         1d
kafka-0   0/1       Terminating   2         1d
kafka-0   0/1       Terminating   2         1d
kafka-0   0/1       Terminating   2         1d
kafka-0   0/1       Terminating   2         1d
kafka-0   0/1       Pending   0         0s
kafka-0   0/1       Pending   0         0s
kafka-0   0/1       ContainerCreating   0         0s
kafka-0   0/1       Running   0         19s
kafka-0   1/1       Running   0         21s
```

## Components
Each manifest creates a Headless Service, `kafka-hs`, each creates a StatefulSet `kafka`, and all but `kafak_micro.yaml` 
create a PodDisruptionBudget.

### Headless Service
Each manifests creates a Headless Service, `kafka-hs`, that controls the domain of the SRV and A records of the brokers 
in the Kafka cluster. The service has a single port, `server`, that the brokers use to communicate. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-hs
  labels:
    app: kafka
spec:
  ports:
  - port: 9093
    name: server
  clusterIP: None
  selector:
    app: kafka
```

### StatefulSet
Each manifest creates a StatefulSet, `kafka`, that controls the brokers in the Kafka cluster. The StatefulSet launches 
`.spec.replicas` brokers using `Parallel` pod management. It uses an anti-affinity rule to spread the brokers across 
nodes, and it uses a affinity rule to attempt to collocate brokers with ZooKeeper servers. The 
`.spec.updateStrategy.type` is set to `RollingUpdate`. This causes the StatefulSet controller to update the Pods in the 
StatefulSet when changes are applied to the StatefulSet's `.spec.template`. Note that the StatefulSet's 
`.spec.template.containers[0].ports` needs to contain a container port that corresponds to the `server` port of the 
`kafka-hs` Headless Service, and this port must be configured as a listen port using the `-- override litenders` 
parameter in the `.spec.template.containers[0].command`.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-hs
  replicas: 3
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: kafka
    spec:
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
      terminationGracePeriodSeconds: 300
      containers:
      - name: k8skafka
        imagePullPolicy: Always
        image: gcr.io/google_containers/kubernetes-kafka:1.0-10.2.1
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
        ports:
        - containerPort: 9093
          name: server
        command:
        - sh
        - -c
        - "exec kafka-server-start.sh /opt/kafka/config/server.properties --override broker.id=${HOSTNAME##*-} \
          --override listeners=PLAINTEXT://:9093 \
          --override zookeeper.connect=zk-cs.default.svc.cluster.local:2181 \
          --override log.dir=/var/lib/kafka \
          --override auto.create.topics.enable=true \
          --override auto.leader.rebalance.enable=true \
          --override background.threads=10 \
          --override compression.type=producer \
          --override delete.topic.enable=false \
          --override leader.imbalance.check.interval.seconds=300 \
          --override leader.imbalance.per.broker.percentage=10 \
          --override log.flush.interval.messages=9223372036854775807 \
          --override log.flush.offset.checkpoint.interval.ms=60000 \
          --override log.flush.scheduler.interval.ms=9223372036854775807 \
          --override log.retention.bytes=-1 \
          --override log.retention.hours=168 \
          --override log.roll.hours=168 \
          --override log.roll.jitter.hours=0 \
          --override log.segment.bytes=1073741824 \
          --override log.segment.delete.delay.ms=60000 \
          --override message.max.bytes=1000012 \
          --override min.insync.replicas=1 \
          --override num.io.threads=8 \
          --override num.network.threads=3 \
          --override num.recovery.threads.per.data.dir=1 \
          --override num.replica.fetchers=1 \
          --override offset.metadata.max.bytes=4096 \
          --override offsets.commit.required.acks=-1 \
          --override offsets.commit.timeout.ms=5000 \
          --override offsets.load.buffer.size=5242880 \
          --override offsets.retention.check.interval.ms=600000 \
          --override offsets.retention.minutes=1440 \
          --override offsets.topic.compression.codec=0 \
          --override offsets.topic.num.partitions=50 \
          --override offsets.topic.replication.factor=3 \
          --override offsets.topic.segment.bytes=104857600 \
          --override queued.max.requests=500 \
          --override quota.consumer.default=9223372036854775807 \
          --override quota.producer.default=9223372036854775807 \
          --override replica.fetch.min.bytes=1 \
          --override replica.fetch.wait.max.ms=500 \
          --override replica.high.watermark.checkpoint.interval.ms=5000 \
          --override replica.lag.time.max.ms=10000 \
          --override replica.socket.receive.buffer.bytes=65536 \
          --override replica.socket.timeout.ms=30000 \
          --override request.timeout.ms=30000 \
          --override socket.receive.buffer.bytes=102400 \
          --override socket.request.max.bytes=104857600 \
          --override socket.send.buffer.bytes=102400 \
          --override unclean.leader.election.enable=true \
          --override zookeeper.session.timeout.ms=6000 \
          --override zookeeper.set.acl=false \
          --override broker.id.generation.enable=true \
          --override connections.max.idle.ms=600000 \
          --override controlled.shutdown.enable=true \
          --override controlled.shutdown.max.retries=3 \
          --override controlled.shutdown.retry.backoff.ms=5000 \
          --override controller.socket.timeout.ms=30000 \
          --override default.replication.factor=1 \
          --override fetch.purgatory.purge.interval.requests=1000 \
          --override group.max.session.timeout.ms=300000 \
          --override group.min.session.timeout.ms=6000 \
          --override inter.broker.protocol.version=0.10.2-IV0 \
          --override log.cleaner.backoff.ms=15000 \
          --override log.cleaner.dedupe.buffer.size=134217728 \
          --override log.cleaner.delete.retention.ms=86400000 \
          --override log.cleaner.enable=true \
          --override log.cleaner.io.buffer.load.factor=0.9 \
          --override log.cleaner.io.buffer.size=524288 \
          --override log.cleaner.io.max.bytes.per.second=1.7976931348623157E308 \
          --override log.cleaner.min.cleanable.ratio=0.5 \
          --override log.cleaner.min.compaction.lag.ms=0 \
          --override log.cleaner.threads=1 \
          --override log.cleanup.policy=delete \
          --override log.index.interval.bytes=4096 \
          --override log.index.size.max.bytes=10485760 \
          --override log.message.timestamp.difference.max.ms=9223372036854775807 \
          --override log.message.timestamp.type=CreateTime \
          --override log.preallocate=false \
          --override log.retention.check.interval.ms=300000 \
          --override max.connections.per.ip=2147483647 \
          --override num.partitions=1 \
          --override producer.purgatory.purge.interval.requests=1000 \
          --override replica.fetch.backoff.ms=1000 \
          --override replica.fetch.max.bytes=1048576 \
          --override replica.fetch.response.max.bytes=10485760 \
          --override reserved.broker.max.id=1000 "
        env:
        - name: KAFKA_HEAP_OPTS
          value : "-Xmx512M -Xms512M"
        - name: KAFKA_OPTS
          value: "-Dlogging.level=INFO"
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/kafka
```

### Pod Disruption Budget
Both the `kafka_mini.yaml` and `kafka.yaml` manifests have a PodDisruptionBudget specified that ensures that only one 
server will be taken down at a time during drains and evictions.
```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: kafka-pdb
spec:
  selector:
    matchLabels:
      app: kafka
  maxUnavailable: 1
```