# bento-helm-chart
<p align="center" style="text-align: center">
    <img src="./assets/box.png" width="30%"><br/>
    Bento Helm Chart. <br/>
</p>

Bento is a high performance and resilient stream processor, able to connect various sources and sinks in a range of brokering patterns and perform hydration, enrichments, transformations and filters on payloads.

This Helm Chart deploys a single Bento instance in either streams mode or standalone.

## Installation
```bash
helm repo add bento-helm https://warpstreamlabs.github.io/bento-helm-chart
helm repo update
helm install bento bento-helm/bento
# OR
helm upgrade --install bento oci://ghcr.io/warpstreamlabs/bento-helm/bento --version 0.1.0
```

## Configuration

The config parameter should contain the configuration as it would be parsed by the Bento binary.
```yaml
# values.yaml
config:
  input:
    label: "no_config_in"
    generate:
      mapping: root = "This Bento instance is unconfigured!"
      interval: 1m
  output:
    label: "no_config_out"
    stdout:
      codec: lines
```

The full list of [available configuration for the Helm Chart can be found in the `values.yaml` file](./charts/bento/values.yaml). You should refer to the [upstream Bento documentation](https://warpstreamlabs.github.io/bento/docs/configuration/about) for the configuration of your pipeline.

## External Configuration

An application chart can own the Bento configuration ConfigMap and pass its name, key, and mount path to this chart. Set `configMap.checksum` to a value that changes with the external ConfigMap content so the Deployment rolls out when the configuration changes. When `configMap.name` is set, this chart does not create its default configuration ConfigMap.

```yaml
configMap:
  name: my-bento-config
  key: pipeline.yaml
  mountPath: /etc/bento/pipeline.yaml
  checksum: pipeline-v3

args:
  - -c
  - /etc/bento/pipeline.yaml
```

The referenced ConfigMap must exist in the release namespace. Keep the checksum synchronized with the content revision; the chart does not read external ConfigMap data.

## Streams mode

When running Bento in [streams mode](https://warpstreamlabs.github.io/bento/docs/guides/streams_mode/about), all individual stream configuration files should be combined and placed in a single Kubernetes `ConfigMap`, like so:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bento-streams
data:
  hello.yaml: |
    input:
      generate:
        mapping: root = "woof"
        interval: 5s
        count: 0
    output:
      stdout:
        codec: lines
  aaaaa.yaml: |
    input:
      generate:
        mapping: root = "meow"
        interval: 2s
        count: 0
    output:
      stdout:
        codec: lines
```

Then you can simply reference your `ConfigMap` and enable streams mode in your `values.yaml` file.
```yaml
# values.yaml
streams:
  enabled: true
  streamsConfigMap: "bento-streams"
```

The chart can manage stream files as a ConfigMap. Each key is a filename and each value is a Bento stream configuration. Changes to these files update the Deployment checksum and trigger a rollout.

```yaml
# values.yaml
streams:
  enabled: true
  files:
    hello.yaml: |
      input:
        generate:
          mapping: root = "woof"
          interval: 5s
          count: 0
      output:
        stdout:
          codec: lines
```

For an externally managed ConfigMap, set `streams.streamsConfigMap` instead. Do not set both `streams.files` and `streams.streamsConfigMap`; when using an external ConfigMap, manage its rollout separately.

### Global Configuration

When deploying Bento in streams mode, you may want to configure global tracing, logging and http configuration which is shared across all of your pipelines.

This can be done by specifying configuration under the `metrics`, `logger` and `tracing` configuration sections in your `values.yaml` file. These all use their respective upstream Bento configuration syntax.

```yaml
metrics:
  prometheus: {}

tracing:
  openTelemetry:
    http: []
    grpc: []
    tags: {}

logger:
  level: INFO
  static_fields:
    '@service': bento
```
