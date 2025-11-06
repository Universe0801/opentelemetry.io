---
Title: Helm Chart del Recolector de OpenTelemetry
linkTitle: Chart del Recolector
---

## Introducción

El [OpenTelemetry Collector](/docs/collector) es una herramienta importante para el monitoreo de Clusters de Kubernetes y todos los servicios que operan dentro. Para facilitar la instalación y gestión de un Deployment del recolector en Kubernetes la comunidad de Opentelemetry creó el [OpenTelemetry Collector Helm Chart](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-collector).
Este Helm Chart puede ser usado para instalar un recolector como un Deployment, Daemonset o Statefulset.

### Instalando el Chart

Para instalar el chart con el nombre de release `my-opentelemetry-collector`, ejecute los siguientes comandos:

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install my-opentelemetry-collector open-telemetry/opentelemetry-collector \
   --set image.repository="otel/opentelemetry-collector-k8s" \
   --set mode=<daemonset|deployment|statefulset>
```

### Configuración

El Chart del Recolector de Opentelemetry requiere que se configure el parámetro `mode`. `mode` puede ser `daemonset`, `deployment`, o `statefulset` dependiendo del tipo de implementación de Kubernetes que requiera tu caso de uso.

Una vez instalado, el Chart proporciona algunos componentes predeterminados para que pueda empezar a utilizarlos. Por defecto, la configuración del recolector tendrá el siguiente aspecto:

```yaml
exporters:
  # NOTA: Antes de la version v0.86.0 use `logging` en vez de `debug`.
  debug: {}
extensions:
  health_check: {}
processors:
  batch: {}
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
receivers:
  jaeger:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:14250
      thrift_compact:
        endpoint: ${env:MY_POD_IP}:6831
      thrift_http:
        endpoint: ${env:MY_POD_IP}:14268
  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: opentelemetry-collector
          scrape_interval: 10s
          static_configs:
            - targets:
                - ${env:MY_POD_IP}:8888
  zipkin:
    endpoint: ${env:MY_POD_IP}:9411
service:
  extensions:
    - health_check
  pipelines:
    logs:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
    metrics:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
        - prometheus
    traces:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
        - jaeger
        - zipkin
  telemetry:
    metrics:
      address: ${env:MY_POD_IP}:8888
```

El Chart tambien habilitará los puertos según los receptores predeterminados. La configuración predeterminada se puede eliminar estableciendo el valor en `null` en el archivo `values.yaml`. Los puertos tambien se pueden deshabilitar en el archivo `values.yaml`.

Puedes añadir o modificar cualquier parte de la configuración mediante la sección `config` en el archivo `values.yaml`. Al modificar un pipeline, tienes que listar todos los componentes que estan en el pipeline, incluyendo los componentes predeterminados.

Por ejemplo, para deshabilitar las metricas, los pipelines de logging y los receptores que no sean OTLP:

```yaml
config:
  receivers:
    jaeger: null
    prometheus: null
    zipkin: null
  service:
    pipelines:
      traces:
        receivers:
          - otlp
      metrics: null
      logs: null
ports:
  jaeger-compact:
    enabled: false
  jaeger-thrift:
    enabled: false
  jaeger-grpc:
    enabled: false
  zipkin:
    enabled: false
```

Todas las opciones de configuración (con comentarios) disponibles en el Chart se pueden ver en su [archivo values.yaml](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/main/charts/opentelemetry-collector/values.yaml).

### Preajustes

Muchos de los componentes importantes que utiliza el Recolector de Opentelemetry para monitorizar Kubernetes requieren una configuración especial en la implementación del propio Recolector. Para facilitar el uso de estos componentes, el Chart del Recolector de Opentelemetry incluye algunas configuraciones preestablecidas que, al activarse, gestionan la compleja configuración de estos componentes importantes.

Estas configuraciones preestablecidas deben utilizarse como punto de partida. Configuran una funcionalidad básica, pero completa, para los componentes relacionados. Si tu caso de uso requiere configuración adicional de estos componentes, se recomienda NO utilizar la configuración preestablecida y, en su lugar, configurar manualmente el componente y todo lo que requiere (volumenes, RBAC, etc.).

