# Export Metrics related to StressChaos Experiments

## Summary

Chaos Daemon exports the statistical metrics of the container, Chaos Controller Manager exports the relation metrics of StressChaos experiment and selected container. Helm Charts include Prometheus rules that join these metrics together to export experiment metrics of StressChaos.

## Motivation

Currently, there are deficiencies in the observability of StressChaos experiments. Users need to deploy observation tools themselves to observe pods and containers affected by StressChaos. Although there is already a Grafana plugin that adds the start and end times of StressChaos experiments to Grafana through Annotation, it is still difficult for users to determine which Pods and Containers will be affected by the experiment. These issues make it difficult for users to observe the experimental effects of StressChaos, and also make it hard to evaluate the steady-state.

This RFC aims to solve this problem by actively exporting StressChaos related metrics from Chaos Mesh. After implementing this RFC, users will be able to directly observe the effects of StressChaos in Prometheus through metrics.

## Detailed design

### Overview

The purpose of this feature is to export metrics related to StressChaos. To achieve this, we need to modify Chaos Daemon and Chaos Controller Manager to export some intermediate metrics, and add prometheus rules to Helm Charts to export the final metrics.

For end users, the basic outcome features are:

- Users can observe the effects of StressChaos experiments through Prometheus.
- Users can filter the metrics of specific experiments through Prometheus.

### Metrics

Overall, this design exports 3 types of metrics.

#### Statistical metrics

Statistical metrics are the metrics that describe the statistical information of the container. These metrics are exported by Chaos Daemon. We plan to export the following metrics:

| Metric Name                                             | Description                                                 |
| ------------------------------------------------------- | ----------------------------------------------------------- |
| `chaos_daemon_container_cpu_usage_seconds_total`        | Total CPU usage in seconds of the container.                |
| `chaos_daemon_container_memory_working_set_bytes`       | The amount of working set memory in bytes of the container. |
| `chaos_daemon_container_memory_available_bytes`         | The available memory in bytes of the container.             |
| `chaos_daemon_container_memory_usage_bytes`             | The memory usage in bytes of the container.                 |
| `chaos_daemon_container_memory_rss_bytes`               | The amount of RSS memory in bytes of the container.         |
| `chaos_daemon_container_memory_page_faults_total`       | Total number of page faults of the container.               |
| `chaos_daemon_container_memory_major_page_faults_total` | Total number of major page faults of the container.         |
| `chaos_daemon_container_memory_swap_available_bytes`    | The available swap in bytes of the container.               |
| `chaos_daemon_container_memory_swap_usage_bytes`        | The swap usage in bytes of the container.                   |

Statistical metrics are exported with the following labels:

| Label Name  | Description                     |
| ----------- | ------------------------------- |
| `namespace` | The namespace of the container. |
| `pod`       | The pod name of the container.  |
| `container` | The container name.             |

The export of statistical metrics is not directly related to the StressChaos experiment. Instead, Chaos Daemon exports these metrics for all Kubernetes managed containers on the node.

#### Relation metrics

Relation metrics are the metrics that describe the relation between the Chaos experiment and the container. These metrics are exported by Chaos Controller Manager. The proposed metric name is: `chaos_controller_manager_chaos_experiments_container_relation`. It is exported with the following labels:

| Label Name  | Description                                   |
| ----------- | --------------------------------------------- |
| `namespace` | The namespace of the container.               |
| `kind`      | The kind of the experiment.                   |
| `phase`     | The phase of the experiment.                  |
| `name`      | The name of the experiment.                   |
| `uid`       | The UID of the experiment.                    |
| `pod`       | The pod name of the selected container.       |
| `container` | The container name of the selected container. |

The relation metrics are exported for all Chaos experiments managed by Chaos Controller Manager if the phase of the experiment is not `Finished` or `Deleting`. This prevents exporting too many metrics by ignoring inactive experiments.

For each selected container in the experiment, Chaos Controller Manager exports a relation metric. The value of the metric is always fixed to `1` for the convenience of joining metrics.

#### Experiment metrics

Experiment metrics are the metrics that describe the effects of the StressChaos experiment. This is the final metrics that end users can observe. The proposed metric name is: `chaos_mesh:stress_chaos:<metric_name>`. For example:

- Statistical metrics: `chaos_daemon_container_cpu_usage_seconds_total`
- Experiment metrics: `chaos_mesh:stress_chaos:container_cpu_usage_seconds_total`

It is exported with the following labels:

| Label Name  | Description                                   |
| ----------- | --------------------------------------------- |
| `namespace` | The namespace of the container.               |
| `kind`      | The kind of the experiment.                   |
| `phase`     | The phase of the experiment.                  |
| `name`      | The name of the experiment.                   |
| `uid`       | The UID of the experiment.                    |
| `pod`       | The pod name of the selected container.       |
| `container` | The container name of the selected container. |

The experiment metrics are exported by joining the statistical metrics and relation metrics. The join is done by Prometheus rules in Helm Charts. Thus, the value of the experiment metrics is the same as the statistical metrics, but with additional labels of the experiment.

### Joining Metrics with PromQL

The purpose of joining metrics is to add the experiment information labels to the statistical metrics. During the join, we first match the statistical metrics and relation metrics by the `namespace`, `pod`, and `container` labels. Then, we add the experiment information labels to the statistical metrics.

The join is done by PromQL specified in the PrometheusRule. The principle is PromQL’s `group_right` operator and vector matching. This operator does a one-to-many join between two metrics. For example, for statistical metrics `chaos_daemon_container_cpu_usage_seconds_total`, we use the following PromQL to join the relation metrics:

```promql
chaos_daemon_container_cpu_usage_seconds_total
	# Doing multiplication, which has no effect because the value is always 1.0, but is used to trigger the joining query
	*
	# Match based on the namespace, pod and container labels
	on(namespace, pod, container)
  # Join the relation metrics
	group_right
chaos_controller_manager_chaos_experiments_container_relation
```

### About Implementation

#### Chaos Daemon

We use CRI (Container Runtime Interface) to obtain the statistical metrics of the container. We will connect to the CRI socket to obtain the metrics. A new cli parameter `--cri-socket-path` will be added to specify the CRI socket. The default value is picked with the respect of `--runtime` and `--runtime-socket-path`. We will export the metrics to the current probe endpoint.

#### Chaos Controller Manager

We will export the metrics to the current probe endpoint.

#### Helm Charts

We will modify the current Prometheus ConfigMap to add the Prometheus rules.

We will add new values to control the proposed feature:

| Value Name                             | Description                                         | Default Value |
| -------------------------------------- | --------------------------------------------------- | ------------- |
| `prometheus.experimentMetrics.enabled` | Enable the feature of exporting experiment metrics. | `false`       |
| `chaosDaemon.criSocketPath`            | The path of the CRI socket.                         | -             |

## Drawbacks

We need to associate the statistical metrics of the container with the experimental information of StressChaos through prometheus rules. This makes part of this design dependent on Prometheus. For other metrics processing tools, users need to implement this association themselves.

Due to the limitations of PromQL, this design can only export one label combination for a container metric. For example, this design will not be able to export metrics like this:

```
# For this metrics, given a container, there's two label combinations:
#   1. type="working_set"
#   2. type="rss"
# Since PromQL cannot do many-to-many label joining, these metrics cannot be exported.
chaos_mesh:stress_chaos:memory_usage_bytes{container="test", type="working_set", ...}  1024
chaos_mesh:stress_chaos:memory_usage_bytes{container="test", type="rss", ...}          1024
```

Instead, we need to export two seperate metrics like this:

```
chaos_mesh:stress_chaos:memory_working_set_usage_bytes{container="test", ...}  1024
chaos_mesh:stress_chaos:memory_rss_usage_bytes{container="test", ...}          1024
```

## Alternatives

### Alternative #1: Export the experiment metrics by Chaos Daemon

We can leave the Chaos Controller Manager completely unmodified. Instead, we allow the Chaos Daemon to obtain the experiment records of StressChaos and let the Chaos Daemon directly add experiments information labels into metrics.

Pros:

- All problem metioned in the drawbacks section can be solved.

Cons:

- Chaos Daemon is not aware of the Experiment details at present. Therefore, Chaos Daemon needs to request the Kubernetes API Server to obtain the details of the experiment, which introduces additional complexity.
- Any metrics probing on any worker nodes' Chaos Daemon may result in several requests to the Kubernetes API Server, though some requests can be cached, it still may cause a performance issue.

### Alternative #2: Export the experiment metrics by Chaos Controller Manager

We can export the final metrics in Chaos Controller Manager. Specifically, when obtaining metrics from Chaos Controller Manager, it retrieves statistical metrics from all Chaos Daemons. Then, it adds the experiment information labels to the metrics and exports them as the final experiment metrics.

Pros:

- All problem metioned in the drawbacks section can be solved.

Cons:

- Each probe of the metrics will result in several requests to all Chaos Daemons, which may cause a performance issue.
- Chaos Controller Manager need to implement the logic of parsing, filtering, modifying, and exporting metrics, which may introduce additional complexity.
