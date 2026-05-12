# Observability Interview Questions & Answers

---

**Q: How do logs, metrics, and traces work together in observability?**

**A:** The three pillars of observability serve complementary roles:

- **Metrics**: numeric measurements aggregated over time (counters, gauges, histograms). Efficient for alerting and dashboards. Example: request rate, CPU utilization, error rate. Tools: Prometheus, CloudWatch, Datadog.
- **Logs**: timestamped, structured or unstructured text records of discrete events. Rich in context for debugging a specific error. Tools: Loki, Elasticsearch (ELK), CloudWatch Logs.
- **Traces**: end-to-end records of a request as it flows through multiple services. Each trace is composed of spans; each span records the operation, duration, and parent-child relationships. Tools: Jaeger, Zipkin, AWS X-Ray, Tempo.

**How they work together**:
1. A Prometheus alert fires on high error rate (metric).
2. You drill into Grafana → link to Loki logs for that time window → find a specific error log with a trace ID.
3. You open Tempo with that trace ID to see which microservice in the chain is slow or failing.

OpenTelemetry provides a unified SDK/collector to instrument all three signals and ship them to any backend.

---

**Q: You've been asked to move from centralized logging to a service-mesh-based observability model. What are your tradeoffs?**

**A:**

**Centralized logging (current)**:
- Simple mental model: all logs flow to one place (ELK, Loki).
- Application teams own what they log.
- Easy to query across services.
- High cardinality logs can be expensive to store.

**Service-mesh-based observability (Istio/Linkerd)**:
- **Pros**: automatic L7 metrics (request rate, error rate, latency) per service pair without code changes; automatic mTLS; distributed tracing headers injected by Envoy sidecar; consistent golden signals for all services.
- **Cons**: sidecar overhead (CPU/memory per pod); control plane complexity; traces only cover inter-service calls (not internal app behavior); application-level business logic still needs custom logging; observability blind spots for non-HTTP protocols.

**My recommendation**: complement, not replace. Use the service mesh for infrastructure-level telemetry (L7 metrics, traces, mTLS audit) and keep centralized logging for application-level events and business logic. Use the OpenTelemetry Collector as the unifying agent.

---

**Q: How do you set up custom metrics for Kubernetes pods?**

**A:**
1. **Instrument the application**: expose a `/metrics` endpoint in Prometheus format (using the Prometheus client library for your language).
2. **Configure Prometheus scraping**: add a `ServiceMonitor` (if using the Prometheus Operator) or add a scrape config in `prometheus.yml`:
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: myapp
     namespace: monitoring
   spec:
     selector:
       matchLabels:
         app: myapp
     endpoints:
       - port: metrics
         interval: 30s
   ```
3. **HPA on custom metrics**: deploy the Prometheus Adapter; configure it to expose your metric via the `custom.metrics.k8s.io` API:
   ```yaml
   rules:
     - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
       resources:
         overrides:
           namespace: {resource: "namespace"}
           pod: {resource: "pod"}
       metricsQuery: 'rate(http_requests_total[2m])'
   ```
4. Create an HPA referencing the custom metric.

---

**Q: How do you troubleshoot missing data points in Grafana dashboards?**

**A:**
1. **Check time range and step**: ensure the dashboard time range covers when data should exist; Grafana's auto step may be too coarse for short intervals.
2. **Query inspector**: click "Query inspector" in Grafana to see the raw query sent to Prometheus and the response — check for empty results or errors.
3. **Prometheus targets**: `http://<prometheus>:9090/targets` — is the scrape target `UP`? Check `last scrape error`.
4. **Metric existence**: `http://<prometheus>:9090/graph` — query the metric directly. Use `up{job="myapp"}` to verify scraping.
5. **Label mismatch**: the dashboard query may filter on labels that changed (e.g., `namespace`, `pod` labels renamed after a deploy).
6. **Retention period**: if querying data older than Prometheus retention (default 15 days), it's gone. Use Thanos or Cortex for long-term storage.
7. **Recording rules**: if using recording rules, check if the rule evaluation interval aligns with the data you expect.
8. **Data source configuration**: verify the Grafana data source URL, credentials, and TLS settings are correct.

---

**Q: How do you create alerts for high CPU or memory usage?**

**A:** In Prometheus + Alertmanager:

```yaml
# prometheus-rules.yaml
groups:
  - name: resource-usage
    rules:
      - alert: HighCPUUsage
        expr: |
          (
            sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)
            /
            sum(container_spec_cpu_quota{container!=""} / container_spec_cpu_period{container!=""}) by (pod, namespace)
          ) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} CPU > 85%"
          description: "Pod {{ $labels.pod }} in {{ $labels.namespace }} has been using more than 85% CPU for 5 minutes."

      - alert: HighMemoryUsage
        expr: |
          (
            container_memory_working_set_bytes{container!=""}
            /
            container_spec_memory_limit_bytes{container!=""}
          ) > 0.90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} memory > 90%"
```

Alertmanager routes alerts to Slack, PagerDuty, or email based on severity labels. In Grafana, use "Alert rules" (unified alerting) to create the same alerts with a GUI and route through contact points.

---

**Q: How would you monitor HPA scaling decisions in real-time and detect if the metrics server is lagging?**

**A:**

**Monitoring HPA decisions in real-time**:
```bash
# Watch HPA status continuously
kubectl get hpa -n <namespace> -w

# See the full decision context (current vs desired replicas, metric values)
kubectl describe hpa <name> -n <namespace>
# Key fields: "Conditions", "Metrics", "Events"

# Kubernetes events for HPA actions (scale up/down decisions)
kubectl get events -n <namespace> \
  --field-selector reason=SuccessfulRescale \
  --sort-by=’.lastTimestamp’
```

**Prometheus metrics for HPA observability**:
```promql
# Current vs desired replicas
kube_horizontalpodautoscaler_status_current_replicas
kube_horizontalpodautoscaler_status_desired_replicas

# Last scale time (detect if HPA is stuck and not scaling)
time() - kube_horizontalpodautoscaler_status_last_scale_time > 3600
# Alert: HPA hasn’t scaled in 1 hour despite high load

# HPA condition — is it able to scale?
kube_horizontalpodautoscaler_status_condition{condition="ScalingActive", status="false"}
# Alert if this is 1 (scaling disabled) for > 5 minutes
```

**Detecting Metrics Server lag**:
```bash
# Direct API query — if this is slow, Metrics Server is lagging
time kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods

# Check Metrics Server pod health and logs
kubectl logs -n kube-system deployment/metrics-server --tail=50

# Check API service availability
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
# Look for: Available: True and no latency in the conditions
```

```promql
# Metrics Server scrape latency (if using kube-state-metrics + Prometheus)
apiserver_request_duration_seconds{resource="pods", subresource="proxy", verb="GET"}

# Alert: Metrics Server pod not ready
kube_pod_container_status_ready{namespace="kube-system", container="metrics-server"} == 0
```

**Synthetic lag detection**: deploy a test pod that constantly writes a known CPU load; alert if HPA doesn’t respond to a crossing of the threshold within 2× the expected controller sync period (default 15s for HPA).

---

**Q: An internal metrics exporter started leaking PII to an external Prometheus endpoint. What’s your containment and audit recovery plan?**

**A:**

**Immediate containment (first 15 minutes)**:
1. **Isolate the exporter**: apply a NetworkPolicy to block all egress from the exporter pod except to the internal Prometheus scraper.
   ```bash
   # Emergency: delete the exporter deployment to stop active leakage
   kubectl delete deployment <exporter-name> -n <namespace>
   # OR block external egress immediately with a NetworkPolicy
   ```
2. **Identify the external endpoint**: check the exporter’s config/env vars and VPC Flow Logs to confirm what external IP/endpoint was receiving data.
3. **Notify**: trigger incident response — notify the security team, DPO (Data Protection Officer), and on-call engineering lead. If the external endpoint is outside the org, this may be a reportable data breach (GDPR Article 33: 72-hour notification window).

**Scope the blast radius**:
```bash
# What data was exported? Pull the exporter’s metric output
kubectl exec -it <exporter-pod> -- curl http://localhost:<metrics-port>/metrics | grep -E ‘name|email|ssn|account|card’

# Check Prometheus TSDB for what was scraped
# In Prometheus UI: search for metric names that contain PII field names
# How long has this been running?
kubectl get deployment <exporter> -o jsonpath=’{.metadata.creationTimestamp}’

# VPC Flow Logs: how much data was sent to the external endpoint
# Query CloudWatch Logs Insights filtered by destination IP
```

**Audit and recovery**:
1. **Identify PII exposure window**: from when the exporter was deployed to when it was killed.
2. **Determine what was leaked**: specific metric labels (e.g., `user_id`, `email`, `ip_address` as label values in Prometheus metrics — a common mistake where PII ends up in high-cardinality labels).
3. **Notify affected users** if PII was confirmed exfiltrated (per GDPR/CCPA requirements).
4. **Prometheus internal cleanup**: if PII landed in internal Prometheus TSDB, use the Admin API to delete the series:
   ```bash
   curl -X POST http://prometheus:9090/api/v1/admin/tsdb/delete_series \
     -d ‘match[]={__name__=~"user_request.*"}’
   curl -X POST http://prometheus:9090/api/v1/admin/tsdb/clean_tombstones
   ```
5. **Root cause fix**: the exporter was labeling metrics with PII values. Fix: strip PII from metric labels at the instrumentation layer; use opaque IDs (hashed/tokenized) as label values instead.
6. **Prevention**: add a `metric-label-policy` OPA policy that scans new exporters for known PII field names; network policy as default-deny for all monitoring pods’ egress.

---

**Q: Grafana shows stale dashboards across clusters after OTel collector upgrade. Walk us through your incident triage chain.**

**A:**

**Step 1 — Confirm and scope** (first 5 minutes):
```bash
# Is this all dashboards or specific data sources?
# Check if Prometheus/Loki/Tempo are receiving data at all
curl http://prometheus:9090/api/v1/query?query=up   # should return current timestamp results
curl http://loki:3100/loki/api/v1/labels            # should return recent labels

# Check last ingestion time
curl "http://prometheus:9090/api/v1/query?query=max(timestamp(up{}))"
# If this is stale (more than a few minutes behind), ingestion has stopped
```

**Step 2 — OTel Collector health**:
```bash
# Collector pods running?
kubectl get pods -n observability -l app=otel-collector

# Collector logs — pipeline errors?
kubectl logs -n observability deployment/otel-collector --tail=100 | \
  grep -E ‘error|dropped|failed|refused|backpressure’

# Collector metrics endpoint (exposes its own pipeline metrics)
kubectl exec -n observability <otel-pod> -- curl http://localhost:8888/metrics | \
  grep -E ‘otelcol_receiver_refused|otelcol_exporter_send_failed|otelcol_processor_dropped’
```

**Step 3 — Diagnose the pipeline break**:

Common causes after an OTel collector upgrade:

| Symptom | Cause | Fix |
|---|---|---|
| `exporter_send_failed` spiking | Prometheus remote_write endpoint changed in new config | Check exporter config in the new collector ConfigMap |
| `receiver_refused_metric_points` | Breaking change in receiver config schema (new version requires different field names) | Read the OTel Collector changelog; update ConfigMap |
| Collector in CrashLoopBackOff | Config validation failure on new version | `kubectl describe pod` for exit code; check config syntax against new version schema |
| Data arrives but Grafana shows old | Grafana data source URL hardcoded to old collector endpoint | Update Grafana data source to new endpoint |
| Partial data (some clusters stale) | Collectors on some clusters not yet upgraded (rolling update lag) | Check DaemonSet rollout status across all clusters |

```bash
# Check ConfigMap diff between old and new version
kubectl get configmap otel-collector-config -n observability -o yaml

# Validate config against new binary
otelcol validate --config=collector-config.yaml
```

**Step 4 — Restore service**:
- **Fastest path**: roll back to the previous collector image:
  ```bash
  kubectl rollout undo deployment/otel-collector -n observability
  # Verify data flow resumes within 1-2 scrape intervals
  ```
- **If rollback isn’t viable**: apply the config fix forward (update ConfigMap to match new version’s schema), then restart pods: `kubectl rollout restart deployment/otel-collector -n observability`.

**Step 5 — Post-incident**:
- Add a CI test that validates the OTel config file against the target version before deployment: `otelcol validate --config`.
- Add a synthetic monitor: a test metric generated by a known source; alert if Grafana doesn’t show it within 2 minutes.
- Stage OTel upgrades: upgrade one cluster, validate for 30 minutes, then roll out to remaining clusters.

