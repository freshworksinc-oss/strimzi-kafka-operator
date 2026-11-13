# Deployment: Strimzi operator 1.3.1 + Kafka 4.3.2 (FIPS images)

| Component | Image |
|---|---|
| Operator, Topic/User operator, kafka-init | `642367526092.dkr.ecr.us-east-1.amazonaws.com/central2.0/k8s_operator:1.3.1-fips.6` |
| Kafka broker/controller, exporter | `642367526092.dkr.ecr.us-east-1.amazonaws.com/central2.0/kafka_broker:1.3.1-kafka-4.3.2-fips.6` |

Target: dev cluster `dev-central-eks-us-east-1-642367526092` (arm64 nodes). Do not apply to staging.

## Layout

- `operator/values.yaml`: values for the in-repo Helm chart `packaging/helm-charts/helm3/strimzi-kafka-operator`.
- `kafka/storageclass.yaml`: the `ebs-sc-kafka` StorageClass (cluster-scoped, EBS gp3, encrypted).
- `kafka/configmaps.yaml`: Kafka logging (log4j2) and JMX metrics ConfigMaps.
- `kafka/kafka-cluster.yaml`: `Kafka` and two `KafkaNodePool`s (3 controllers and 3 brokers), API `kafka.strimzi.io/v1`.

## Deploy

```bash
kubectl config use-context dev-central-eks-us-east-1-642367526092

# 1. Operator (CRDs are installed by the chart's crds/ directory)
helm upgrade --install strimzi-kafka-operator ../packaging/helm-charts/helm3/strimzi-kafka-operator \
  -n strimzi-system --create-namespace -f operator/values.yaml

# 2. Kafka
kubectl create namespace strimzi-kraft-test --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f kafka/storageclass.yaml
kubectl apply -f kafka/configmaps.yaml
kubectl apply -f kafka/kafka-cluster.yaml
kubectl -n strimzi-kraft-test wait kafka/kafkadev --for=condition=Ready --timeout=900s
```

Preview the operator manifests with `helm template strimzi-kafka-operator ../packaging/helm-charts/helm3/strimzi-kafka-operator -n strimzi-system -f operator/values.yaml`.

## Differences from the 0.45.2 / Kafka 3.9.2 deployment

- **KRaft only.** Strimzi 1.x and Kafka 4.x have no ZooKeeper, so there is no `zookeeper:` block and a dedicated
  `controller` node pool is required. Brokers use the `broker` pool.
- **API version `kafka.strimzi.io/v1`.** The `strimzi.io/node-pools` and `strimzi.io/kraft` annotations are gone.
  `spec.kafka.metadataVersion` is `4.3-IV0` (replaces `inter.broker.protocol.version`).
- **Logging** uses log4j2 (`log4j2.properties` key) instead of log4j 1.
- **Not upgradable in place from 0.45.2 / 3.9.2.** Moving from ZooKeeper to KRaft requires the migration path
  (Strimzi 0.45 -> 0.46+ with KRaft migration, then Kafka 4). Use this for a new cluster.
- **Listener authentication.** The existing clusters use the custom `sasl_scram_and_plain` listener type and the
  `FWSS_*` init container. Those patches are not in `release-1.3.x`, so this deployment uses upstream
  `scram-sha-512`. Clients that use SASL/PLAIN will not work until the patch is ported.
- **Image tags.** `image.tag` is set explicitly, so the chart uses that tag for every Kafka version entry; the
  Kafka CR also pins the image explicitly.
- Namespaces, storage classes (`ebs-sc-kafka`), and sizing follow the dev values. No node selectors or node affinity are set (test cluster). Adjust
  as needed. The sample metrics config is minimal; replace it with the existing `kafka-metrics` ConfigMap if you
  want identical dashboards.
