# Kafka client rack awareness

When Kafka brokers are spread across availability zones, you can reduce cross-AZ traffic and latency by enabling **rack awareness** on both brokers and clients.

## Broker prerequisites

Your Kafka cluster must advertise a **broker rack** per broker and use a replica selector that allows consumers to prefer nearby replicas. For **Strimzi**, that typically means:

- `spec.kafka.rack.topologyKey` set to your node topology label (commonly `topology.kubernetes.io/zone`).
- Broker config `replica.selector.class.name: org.apache.kafka.common.replica.RackAwareReplicaSelector`.

Without this, setting `client.rack` on clients has little or no effect.

## Chart configuration

Enable the global flag in your values:

```yaml
global:
  kafkaClientRackAwareness:
    enabled: true
    # optional overrides:
    # envName: KAFKA_CLIENT_RACK
    # fieldPath: metadata.labels['topology.kubernetes.io/zone']
```

When `enabled` is `true`, the chart:

1. Injects **`KAFKA_CLIENT_RACK`** into pod env via the **downward API** (`valueFrom.fieldRef`), using `fieldPath` you configure.
2. **Sentry PoC hotfix** (on by default via `sentryKafkaConfigPyClientRackHotfix`): Sentry’s `get_kafka_cluster_options` only permits keys listed in `SUPPORTED_KAFKA_CONFIGURATION` in `sentry/utils/kafka_config.py`, which does **not** currently include `client.rack`. Until [getsentry/sentry](https://github.com/getsentry/sentry) adds it, the chart runs an **initContainer** on every workload using the Sentry image: it copies `/usr/src/sentry/src/sentry/utils/kafka_config.py` from the image into an `emptyDir`, inserts `"client.rack"` into the tuple, and the main container **mounts that file over the original path**. Set `sentryKafkaConfigPyClientRackHotfix: false` once upstream supports `client.rack`, then remove this behavior from the chart.
3. Merges **`client.rack`** into:
   - **Sentry** — `DEFAULT_KAFKA_OPTIONS["common"]` in generated `sentry.conf.py` (only if the env var is non-empty after `strip()`).
   - **Snuba** — `BROKER_CONFIG` in generated `settings.py` (same non-empty guard).
   - **Relay** — `processing.kafka_config` as `client.rack: "${KAFKA_CLIENT_RACK}"`.

The hotfix (item 2) plus `client.rack` in `sentry.conf.py` (item 3) are what make Sentry Python accept `client.rack` at runtime.

**Vroom** and **uptime-checker** are **not** given this env or Kafka client rack settings; those components do not support configuring `client.rack` through this chart.

## Pod topology / `fieldPath`

The default `fieldPath` assumes the pod has the label `topology.kubernetes.io/zone`. That is **not** set on pods by default on every cluster. You may need:

- A version / configuration of Kubernetes that propagates topology labels to pods, or
- An admission controller / mutating webhook that copies the node’s zone onto the pod, or
- A different `fieldPath` (for example an annotation your platform sets).

If `KAFKA_CLIENT_RACK` is empty, Sentry and Snuba **omit** `client.rack` so librdkafka is not configured with an empty rack id.

## Verification ideas

1. **Env**: In a Sentry or Snuba consumer pod, `KAFKA_CLIENT_RACK` should match the pod’s AZ (or your chosen rack id).
2. **Brokers**: Strimzi/Kafka should show each broker’s `broker.rack` aligned with node zones.
3. **End-to-end**: Exact “always same AZ” behavior depends on replication factor, ISR, leader placement, and consumer protocol; use Kafka metrics / logs if you need proof of follower reads.

For more background on Strimzi rack configuration, see the [Strimzi documentation](https://strimzi.io/docs/operators/latest/configuring.html).
