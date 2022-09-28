## Building a Collector Trace Pipeline
In this lab you'll learn how to :
* build a Trace pipeline
* use the processor K8sattrributes
* use the processor spanMetrics
* use the otlp http exporter

### Step 1: Export Traces to Logs

1. Look at  the OpenTelemtryCollector template

In the Bastion host, go to o the folder : `exercice/01_collector/trace`
   ```bash
   (bastion)$ cd ~/ACE_OTEL_SCRIPT
   (bastion)$ cd exercice/01_collector/trace`
   (bastion)$ cat openTelemetry-manifest.yaml
   ```
This collector is currently receiving traces and exporting it directly the logging exporter.

2. Deploy the OpenTelemetryCollector 
   ```bash
   (bastion)$ kubectl apply -f openTelemetry-manifest.yaml
   ```
   This will deploy OpenTelemtry Collector in a daemon set mode.
   
3. Look at the produced logs 
   
   ```bash
   kubectl get pods 
   ```
   you should have one pod running with our collector :
   ![Pod 01](../../assets/images/pod_01.png)

   Copy the pod name of the collector , and display the logs :

   ```bash
   kubectl get logs <Collector pod name>
   ```

### Step 2. Update the current Trace Pipeline 

1. Edit the OpenTelemetryCollector object
   In the Bastion host, edit the file  : `exercice/01_collector/metrics/openTelemetry-manifest.yaml
   Change the Trace pipeline to process span , where each task will always :
      - start with the memory_limiter
      - end with the back processor 
   ```bash
   (bastion)$ vi openTelemetry-manifest.yaml
   ```
   ![Pod 01](../../assets/images/processor_flow.png)
   
2. Add k8s attributes 
   Adding k8s attributes to the generated traces , means using the processor `k8sattributes`.
   `k8sattributes` will interact with the k8s Api to collect details related to the span.
   To interact with the Api , you need to use a Service Account having the right cluster role.
   The following Service Account has already been deployed and has all the right privileges: `otelcontribcol`
   Add the following extra attributes :
     - k8s.pod.name
     - k8s.pod.uid
     - k8s.deployment.name
     - k8s.namespace.name
     - k8s.node.name
     - k8s.pod.start_time
   Here is the link to documentation of the [k8sattributes processor](https://pkg.go.dev/github.com/open-telemetry/opentelemetry-collector-contrib/processor/k8sattributesprocessor).
 
   Edit the `openTelemetry-manifest.yaml`, by adding the right defintion of processor 
   
   ```bash
   (bastion)$ vi openTelemetry-manifest.yaml
   ```

1. Change the Processor flow
   
   Change the Processor flow in the trace pipeline to add the `k8sattributes` after the `memory_limiter`
   Apply the change made on the collector :
   
    ```bash
      (bastion)$ kubectl apply -f openTelemetry-manifest.yaml
    ```
   
1. Look a the logs of the collector to see the updated format of the spans
   Get the new Pod name of the colellector and look a the logs :
   ```bash
   (bastion)$ kubectl logs  <Collector pod name>
   ```

### Step 3. Export the generated spans to Dynatrace

1. Update the current trace pipeline
   In the OpenTelemtry Collector pipeline , edit  `openTelemetry-manifest.yaml` to export the spans to :
      -The logs of the collector
      - Dynatrace OpenTelemtry Trace ingest API

   ```bash
   (bastion)$ vi openTelemetry-manifest.yaml
   ```
   
   expected flow :
   ![exporter 01](../../assets/images/exporter_flow.png)
   
1. Look at he generated traces in Dynatrace
   Open your Dynatrace tenant :
   > 1. Navigate to `Distributed traces` via Dynatrace Menu 
   > 2. Click on ingested Traces
   > 3. Click on the Trace named HTTP GET or HTTP Post

1. Add all span attributes not stored by Dynatrace
   Look at each generated span, add all span attributes detected but not stored by dynatrace.
   ![spanattribute 01](../../assets/images/span_attribute.png)
   
1. Look at the Service Screen
   In your Dynatrace tenant: 
   > 1. Navigate to `Services` via Dynatrace Menu 
   > 2. Click on the `checkoutservice`

### Step 4. Add the spanMetrics processor

1. Add the fake  `otlp/spanmetrics` receiver and exporter
   Edit the and add the fake receiver and exporter:
   
    ```yaml
   otlp/spanmetrics:
      endpoint: "localhost:<random port>"
      tls:
      insecure: true
    ```
   Note: the port of the receiver and the exporter needs to be different.
  ```bash
   (bastion)$ vi openTelemetry-manifest.yaml
   ```
1. Add a specific  `metrics/spanmetrics` pipeline
   Similar to the trace pipeline , we need to add a "fake" pipeline for ``metrics/spanmetrics`
   ```yaml
   metrics/spanmetrics:
      receivers: [otlp/spanmetrics]
      exporters: [otlp/spanmetrics]
   ```
   
2. Add the processor `spanmetrics`
   The settings of the `spanmetrics` processor limits by defining the name of the current metric exporter.
   THe current pipeline has only metric exporter defined : `prometheus`
   ```yaml
   spanmetrics:
      metrics_exporter: prometheus
   ```
3. Update the trace pipeline 
   Add the `spanmetrics` processor in the trace processor flow:

   ```yaml
   traces:
      receivers: [otlp]
      processors: [memory_limiter,k8sattributes,spanmetrics,batch]
      exporters: [otlphttp]
   ```

4. Add a metric pipeline
   To be able to look at the produced metrics , let's add a metric pipeline
   
   ```yaml
   metrics:
      receivers: [otlp]
      processors: [memory_limiter,k8sattributes,batch]
      exporters: [prometheus]
   ```
   
5. Apply the changes 

   ```bash
    (bastion)$ kubectl apply -f openTelemetry-manifest.yaml
   ```
   
6. Look at the prometheus metrics produced by the Collector
   Get the new Pod name of the Collector
   
   ```bash
    (bastion)$ kubectl get pods
   ```

   Expose the port 9090 locally on the bastion host :
   
   ```bash
   (bastion)$ kubectl port-forward <collector pod name> 9090:9090
   ```

   Open another terminal and connecto the the bastion host.
   Look at the metrics by send http request to `http://localhost:9090/metrics`
   
   ```bash
   (bastion)$ curl http://localhost:9090/metrics
   ```
   