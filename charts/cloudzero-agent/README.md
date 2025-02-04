# cloudzero-agent

[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.1-4baaaa.svg)](CODE-OF-CONDUCT.md)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
![GitHub release](https://img.shields.io/github/release/cloudzero/template-cloudzero-open-source.svg)

A chart for using Prometheus in agent mode to send cluster metrics to the CloudZero platform.

## Prerequisites

- Kubernetes 1.23+
- Helm 3+
- A CloudZero API key

## Installation

### Get Helm Repository Info

```console
helm repo add cloudzero https://cloudzero.github.io/cloudzero-charts
helm repo update
```

_See [`helm repo`](https://helm.sh/docs/helm/helm_repo/) for command documentation._

### Install Helm Chart

The chart can be installed directly with Helm or any other common Kubernetes deployment tools.

If installing with Helm directly, the following command will install the chart:
```console
helm install <RELEASE_NAME> cloudzero/cloudzero-agent \
    --set existingSecretName=<NAME_OF_SECRET> \
    --set clusterName=<CLUSTER_NAME> \
    --set cloudAccountId=<CLOUD_ACCOUNT_ID> \
    --set region=<REGION>
```

### Secret Management

The chart requires a CloudZero API key in order to send metric data to the CloudZero platform. Admins can retrieve API keys [here](https://app.cloudzero.com/organization/api-keys).

The Deployment running Prometheus ingests the API key via a Secret; this Secret can be supplied as an existing secret (default), or created by the chart. Both methods will require the API key retrieved from the CloudZero platform.

If using a Secret external to this chart for the API key, ensure the Secret is created in the same namespace as the chart and that the Secret data follows the format:

```yaml
data:
  value: <API_KEY>
```

For example, the Secret could be created with:
```bash
kubectl create secret -n example-namespace generic example-secret-name --from-literal=value=<example-api-key-value>
```
The Secret can then be used by the agent by giving `example-secret-name` as the Secret name for the `existingSecretName` argument.

### Metric Exporters

This chart relies on metrics from the [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) and [node-exporter](https://github.com/prometheus/node_exporter) projects as chart dependencies.

By default, these subcharts are disabled to allow the agent to scrape metrics from existing instances of `kube-state-metrics` and `node-exporter`. If you have an existing deployment, you need to configure the cloudzero-agent to use the existing service endpoint addresses. You can set these addresses in the `values.yaml` file as follows by defining the relative `serviceEndpoint`:

```yaml
validator:
  serviceEndpoints:
     kubeStateMetrics: <kube-state-metrics>.<example-namespace>.svc.cluster.local:8080
     prometheusNodeExporter: <node-exporter>.<example-namespace>.svc.cluster.local:9100
```

> **Note:** Replace `<example-namespace>` and the service names with the ones used in your deployments.

Alternatively, if you do not have an existing kube-state-metrics and node-exporter, you can deploy them automatically by enabling the following settings. In this case you do not need to set the `validator.serviceEndpoints.*` values:

```yaml
kube-state-metrics:
  enabled: true
prometheus-node-exporter:
  enabled: true
```

This will deploy the necessary resources for metric scraping.

#### Passing values to subcharts

Values can be be passed to the metrics exporter subcharts by adding values according to the values specification for [kube-state-metrics](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-state-metrics/values.yaml) and [prometheus-node-exporter](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-node-exporter/values.yaml) in the `kube-state-metrics` and `prometheus-node-exporter` sections respectively.

A common addition may be to pull the container images from custom image registries/repositories:

```yaml
kube-state-metrics:
  enabled: true
  image:
    registry: my-custom-registry.io
    repository: my-custom-kube-state-metrics/kube-state-metrics

prometheus-node-exporter:
  enabled: true
  image:
    registry: my-custom-registry.io
    repository: my-custom-prometheus/node-exporter
```

### Custom Scrape Configs
If the chart is running **_without_** the `kube-state-metrics` and `prometheus-node-exporter` exporters enabled (meaning, those two exporters are deployed from some other source outside of this chart), then the scrape configs used by the underlying Prometheus agent may need to be adjusted.

As an example, the out-of-the-box scrape config in this chart attempts to find the `kube-state-metrics` and `node-exporter` exporters via an annotation on k8s Services deployed by the KSM ande node-exporter subcharts. If those subcharts were instead deployed without any annotations, and were only available via Services with the addresses `my-kube-state-metrics-service.default.svc.cluster.local:8080` and `my-node-exporter.default.svc.cluster.local:9100`, we could add the following:

custom-scrape-config.yaml
```yaml
prometheusConfig:
  scrapeJobs:
    kubeStateMetrics: # this disables the default kube-state-metrics scrape job, which will be replaced by an entry in additionalScrapeJobs
      enabled: false
    additionalScrapeJobs:
    - job_name: custom-kube-state-metrics
      honor_labels: true
      honor_timestamps: true
      scrape_interval: 1m
      scrape_timeout: 10s
      metrics_path: /metrics
      static_configs:
        - targets:
          - 'my-kube-state-metrics-service.default.svc.cluster.local:8080'
          - 'my-node-exporter.default.svc.cluster.local:9100'
      relabel_configs:
      - separator: ;
        regex: __meta_kubernetes_service_label_(.+)
        replacement: $1
        action: labelmap
      - source_labels: [__meta_kubernetes_namespace]
        separator: ;
        regex: (.*)
        target_label: namespace
        replacement: $1
        action: replace
      - source_labels: [__meta_kubernetes_service_name]
        separator: ;
        regex: (.*)
        target_label: service
        replacement: $1
        action: replace
      - source_labels: [__meta_kubernetes_pod_node_name]
        separator: ;
        regex: (.*)
        target_label: node
        replacement: $1
        action: replace
      kubernetes_sd_configs:
        - role: endpoints
          kubeconfig_file: ""
          follow_redirects: true
          enable_http2: true

```

This file can then be included with the helm install with the `-f custom-scrape-config.yaml` flag.

### Exporting Pod Labels
Pod labels can be exported as metrics for use in the CloudZero platform by using the `metric-labels-allowlist` CLI argument to the `kube-state-metrics` container. This is disabled by default due to the increase in cardinality that exporting all pod labels introduces.

To export **all** pod labels, set the following in the helm chart:
```yaml
kube-state-metrics:
  extraArgs:
    - --metric-labels-allowlist=pods=[*]
```
A subset of relevant pod labels can be included; as an example, exporting only the pod labels named `foo` and `bar` could be achieved with the following:
```yaml
kube-state-metrics:
  extraArgs:
    - --metric-labels-allowlist=pods=[foo,bar]
```
This is preferable to including all labels with `*` because the performance impact is lessened. Regular expression matching is not currently supported. See the `kube-state-metrics` [documentation](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/developer/cli-arguments.md) for more details.

## Values

**Manditory Values**
| Key               | Type   | Default               | Description                                                                                                             |
|-------------------|--------|-----------------------|-------------------------------------------------------------------------------------------------------------------------|
| cloudAccountId    | string | `nil`                 | Account ID in AWS or Subscription ID in Azure of the account the cluster is running in.                                 |
| clusterName       | string | `nil`                 | Name of the cluster. Required to be RFC 1123 compliant.                                                                 |
| host              | string | `"api.cloudzero.com"` | CloudZero host to send metrics to.                                                                                      |
| apiKey            | string | `nil`                 | The CloudZero API key to use to export metrics. Only used if `existingSecretName` is not set.                           |
| existingSecretName| string | `nil`                 | The name of the secret that contains the CloudZero API key. Required if not providing the API key via the apiKey value. |
| region            | string | `nil`                 | Region the cluster is running in. For example: `us-east-1`, `eastus`, etc. For more information, see: [AWS](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/global-infrastructure.html), [Azure](https://azuretracks.com/2021/04/current-azure-region-names-reference/). |
<!---
For later when we add support: [GCP](https://cloud.google.com/compute/docs/regions-zones), [Alibaba](https://www.alibabacloud.com/help/en/cloud-migration-guide-for-beginners/latest/regions-and-zones)
-->


## Requirements

| Repository                                         | Name                     | Version |
|----------------------------------------------------|--------------------------|---------|
| https://prometheus-community.github.io/helm-charts | kube-state-metrics       | 5.15.*  |
| https://prometheus-community.github.io/helm-charts | prometheus-node-exporter | 4.24.*  |


## Useful References

* [Deployment Validation Guide](./docs/deploy-validation.md)
