# Shard Autoscaling

* Owners:
  * [Arthur Sens](https://github.com/ArthurSens)
  * [Nicolas Takashi](https://github.com/nicolastakashi)
* Related Tickets:
  * [#4927](https://github.com/prometheus-operator/prometheus-operator/issues/4727)
  * [#4946](https://github.com/prometheus-operator/prometheus-operator/issues/4946)
  * [#4967](https://github.com/prometheus-operator/prometheus-operator/issues/4967)
* Other docs:
  * n/a

This document aims to solve many small problems to unblock the usage of Horizontal Pod Autoscalers with Prometheus sharding mechanism.

# Why

Managing Prometheus instances with highly variable workloads can be quite challenging. When the number of scraped targets can scale up and down between a few items to thousands over a short period of time (e.g. within a day), operators are forced to opt for the safest scenario, ending up with over-provisioned instances. Additionally, the WAL replay often becomes a problem after a restart as it takes a long time to be replayed into memory. Without available replicas, this leads to unecessary downtime.

To address these issues, this document outlines a solution to enhance the Prometheus Operator by enabling usage of Horizontal Pod Autoscalers(HPAs) to scale shard up and down. This improvement aims to simplify the process of scaling Prometheus instances, making it more accessible and efficient for operators.

# Pitfalls of the current solution

Prometheus can scale vertically pretty well, but instances that grow up to hundreds of GiB/TiB in WAL size start to get troublesome:

* Eventual cardinality explosions have a big blast radius if a single Prometheus instance is responsible for scraping the majority of monitored applications.
* Recovering from a crash takes several minutes due to the WAL replay.

Meanwhile, another strategy would be to use Horizontal Pod Autoscalers (HPAs) with Prometheus statefulsets by increasing the number of replicas. However, arguably it makes things more complicated:

* Replicas with the same scrape configuration will not shard the scraping load into different instances. Instead, we get several Prometheus instances generating duplicated metrics.

# Goals

* Enable scaling of the Prometheus shards up and down via Horizontal Pod Autoscaler objects.

# Non-Goals

* Implement the autoscaling logic into the Prometheus operator.
* Automatic balancing of targets to ensure homogeneous load across all Prometheus pods.

# Audience

* Users looking for a shard autoscaling mechanism with a good cost-reliability ratio.

# How

Today, there are a few strategies to measure load on top of Prometheus instances.

1. Perhaps you can use [Kubernetes Metrics API](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#metrics-api) to measure load in terms of CPU and/or Memory usage.

2. Another strategy might be to measure the amount of samples ingested per second

- Prometheus Agents: `rate(prometheus_agent_samples_appended_total[5m])`
- Prometheus Servers: `rate(prometheus_tsdb_head_samples_appended_total[5m])`

3. Or even measuring samples queried per second for Prometheus instances under heavy query load.

- Prometheus Servers: `rate(prometheus_engine_query_samples_total[5m])`

Modern Custom Horizontal Pod Autoscalers (HPAs), such as Keda are already capable of [scaling CRDs](https://keda.sh/docs/2.11/concepts/scaling-deployments/#scaling-of-custom-resources) based on different parameters, such as [CPU, Memory](https://keda.sh/docs/2.11/scalers/cpu/) or even [custom Prometheus metrics](https://keda.sh/docs/2.11/scalers/prometheus/).

## Scale Status subresource

When working with CRDs, common HPAs (e.g. Keda) depends on a Status subresource called [Scale](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#scale-subresource). Unfortunately, Prometheus-Operator resources still don't implement this field. There was an attempt to implement this in https://github.com/prometheus-operator/prometheus-operator/pull/4735, but was reverted because the PR was scaling replicas instead of shards. Although scaling replicas can help with high availability, it is expensive and hard to manage since it duplicates scrapes while not reducing the load on top of Prometheus because all replicas still scrape the same targets. Sharding serves as a better autoscaling strategy since the load is spread into all instances instead of getting duplicated.

With only this change, Prometheus Agents can already be horizontally scaled without problems, but for Prometheus Servers it gets a little more complicated.

## Graceful scale-down of Prometheus Servers

Prometheus Servers don't behave like the usual stateless applications. A few cases require Prometheus pods to still be available even after a scale-down event is triggered:

* Thanos proxying queries back to the sidecars to retrieve data that still wasn't uploaded to object storage.
* Prometheus servers with long retention periods still hold data that users might want to query.

If we simply delete Prometheus pods during scale-down, data loss will be a common problem, so a more complex scale-down strategy needs to be used here.

We propose that Prometheus-Operator introduces a new field to the Prometheus Server CRDs:

```yaml
apiVersion: monitoring.coreos.com
kind: Prometheus
metadata:
  name: example
spec:
  shardRetentionPolicy:
    # Default: Delete
    whenScaled: Retain|Delete 
    # if not specified, the operator picks up the retention period if it's defined,
    # otherwise (e.g. size-based retention) the statefulset lives forever.
    retain:
      retentionPeriod: 3d
```

Inspired by the [StatefulSet API](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#persistentvolumeclaim-retention), when setting `.sharRetentionPolicy.whenScaled` is set to `Delete` the operator would simply delete the underlying StatefulSet on the spot, not caring about the data stored in that particular shard. Although not the safest approach, it is exactly the current behavior, and changing it might surprise a lot of users.

If `.shardRetentionPolicy.whenScaled` is set to `Retain` Prometheus-Operator will keep the shard running for a customizable period of time. By default the Operator will keep the Shard for the whole retention period, or forever if no retention period is set. Alternatively, the user can choose a customized retention period.

If retention needs to be done, the deletion logic is controlled by an annotation present on the StatefulSet:

```yaml
operator.prometheus.io/deletion-timestamp: X
```

On scale downs, the configuration of all shards would be re-arranged to make sure the Prometheus that was scaled down doesn't scrape any metrics anymore, but it would still be available for queries.

When the current time finally surpasses the one in the annotation, the Operator will finally trigger the deletion of the statefulset associated with the shard.

# Scale down Alternatives

## Dump & Backfill

During scale-down, the Prometheus-Operator could read TSDB Blocks from the Prometheus instance being deleted and backfill it into another instance.

***Advantages:***
* Prometheus can be shut down immediately without data loss.

***Disadvantages:***
* Loading TSDB Blocks into memory is expensive, requiring Prometheus-Operator to run with big Memory/CPU requests.
* Complex and hard to coordinate workflow. (Require restarts to reload TSDB blocks, hard to indentify possible corruptions)

## Snapshot & Upload on shutdown

During scale-down, Prometheus-Operator could send an HTTP request to [Prometheus' Snapshot endpoint](https://prometheus.io/docs/prometheus/latest/querying/api/#snapshot). Thanos sidecar could be extended to watch the snapshot directory and automatically upload snapshots to Object storage.

There is a related issue open already: https://github.com/thanos-io/thanos/issues/6263

***Advantages:***
* Prometheus can be shut down immediately without data loss.

***Disadvantages:***
* Prometheus running without Thanos sidecars won't benefit from this strategy.

# Action Plan

1. Re-implement [Pull Request #4735](https://github.com/prometheus-operator/prometheus-operator/pull/4735), but this time focusing on Shards and making sure we add the [selectorpath to the scale subresource](https://book.kubebuilder.io/reference/generating-crd.html#scale). Enabling horizontal autoscaling for Prometheus Agents.

2. Implement the graceful shutdown strategy of Prometheus servers.
