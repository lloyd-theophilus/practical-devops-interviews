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

**Q: Users report intermittent failures but dashboards look healthy. What will you do?**

**A:** "Dashboards look healthy" usually means your aggregate metrics are fine but something affecting a subset of users or requests is being averaged away. This is a signal-to-noise problem.

**Investigation approach**:

1. **Disaggregate your metrics**: break down error rate, latency, and success rate by: pod, node, AZ, region, user segment, API endpoint, and client version. Aggregates hide outliers.
   ```promql
   # Per-pod error rate instead of cluster-wide
   rate(http_requests_total{status=~"5.."}[5m]) by (pod)
   ```

2. **Check percentiles, not averages**: p99/p99.9 latency can be terrible while p50 looks fine. Add histogram quantile panels to your dashboards.
   ```promql
   histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
   ```

3. **Correlate with traces**: use trace sampling to find the specific request paths that are failing — Jaeger/Tempo can filter by error status. A 1% failure rate won't show in dashboards but will appear in traces.

4. **Check logs at DEBUG level**: the application may be swallowing exceptions silently. Search logs for `warn`, `exception`, or stack traces during the reported window.

5. **Synthetic monitoring**: run a synthetic probe (Blackbox Exporter, Datadog Synthetics) that mimics real user flows — it catches failures that don't show in infrastructure metrics.

6. **Client-side telemetry**: browser/mobile RUM (Real User Monitoring) captures failures that never reach your servers (DNS failures, CDN edge errors, client-side JS errors). If you don't have RUM, this is a blind spot.

7. **Check downstream dependencies**: third-party APIs, payment gateways, external auth providers — their failures may not show in your infrastructure dashboards.

---

**Q: Application logs suddenly stop generating. Where will you start?**

**A:**

**Step 1 — Is the application still running?**
```bash
kubectl get pods -n <namespace>             # are pods Running?
kubectl logs <pod> --tail=10               # any recent output at all?
kubectl describe pod <pod>                 # check for OOMKilled, eviction
```

**Step 2 — Is the log shipper healthy?** (Fluent Bit, Fluentd, Logstash DaemonSet)
```bash
kubectl get pods -n logging -l app=fluent-bit
kubectl logs -n logging <fluent-bit-pod> --tail=50 | grep -E 'error|drop|backpressure'
```
Fluent Bit backpressure: if the downstream (Elasticsearch, Loki, CloudWatch) is slow, Fluent Bit may pause ingestion. Check `fluentbit_output_dropped_records_total`.

**Step 3 — Is the logging backend receiving data?**
```bash
# CloudWatch: check if log group has recent events
aws logs describe-log-streams \
  --log-group-name /app/production \
  --order-by LastEventTime --descending --limit 5

# Loki/Elasticsearch: query for recent logs
curl "http://loki:3100/loki/api/v1/query?query={app=\"myapp\"}&limit=5"
```

**Step 4 — Check if the app changed its logging configuration**
- Was logging level changed to a higher threshold (e.g., from INFO to FATAL)?
- Was a structured logging library updated and the output format changed (breaking the log parser)?
- Was stdout/stderr redirected to a file inside the container (Fluent Bit tails `/var/log/containers/`, not in-container files)?

**Step 5 — Disk full on the node**
```bash
# On the node
df -h /var/lib/docker    # container log storage
du -sh /var/log/containers/*   # find large log files
```
If disk is full, the container runtime can't write log files → logs stop.

**Step 6 — Log rotation issue**
Kubelet rotates container logs. If `--container-log-max-size` is too small, logs may rotate faster than Fluent Bit can read them → log loss.

---

**Q: Monitoring dashboards suddenly show no data. What will you check?**

**A:**

**Quick triage (5 minutes)**:
```bash
# 1. Is Prometheus scraping data?
curl http://prometheus:9090/api/v1/query?query=up
# If this returns results, Prometheus is running and has data

# 2. Is the data source reachable from Grafana?
# Grafana → Configuration → Data Sources → Test

# 3. Check when data stopped
curl "http://prometheus:9090/api/v1/query?query=max(timestamp(up{}))"
# Shows the latest timestamp Prometheus has data for
```

**Common causes and fixes**:

| Cause | Check | Fix |
|---|---|---|
| Prometheus pod crashed/restarted | `kubectl get pods -n monitoring` | Restart, check OOM / disk full |
| Grafana data source URL changed | Grafana → Data Sources → Test | Update URL to correct Prometheus endpoint |
| Metrics Server down (for k8s metrics) | `kubectl get apiservice v1beta1.metrics.k8s.io` | Restart Metrics Server |
| Node Exporter / app exporter down | `kubectl get pods -n monitoring -l app=node-exporter` | Check DaemonSet pod health |
| Time range mismatch | Grafana time picker | Ensure time range includes when data exists |
| Prometheus retention exceeded | `--storage.tsdb.retention.time` flag | Increase retention or query Thanos/Cortex |
| Network policy blocking scrape | `kubectl get networkpolicies -n <app-ns>` | Allow port 9090 from Prometheus namespace |
| Wrong Grafana dashboard variables | Check variable dropdowns | Verify `$namespace`, `$pod` variables resolve |

**After a recent OTel collector upgrade specifically**: check if the collector pipeline is broken (see the OTel upgrade answer above in this file).

---

**Q: How do you set up monitoring tools like CloudWatch and Grafana?**

**A:**

**CloudWatch setup (AWS)**:
```bash
# 1. Enable detailed monitoring on EC2 (1-minute intervals)
aws ec2 monitor-instances --instance-ids i-xxxxx

# 2. Install CloudWatch Agent on EC2 for custom metrics and logs
sudo yum install amazon-cloudwatch-agent
# Configure with wizard:
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# 3. Create a CloudWatch alarm
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:alerts-topic \
  --dimensions Name=InstanceId,Value=i-xxxxx

# 4. For containers: Container Insights (EKS/ECS)
aws eks update-addon --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability
```

**Grafana setup (with Prometheus)**:
```bash
# Install via Helm (kube-prometheus-stack — includes Prometheus + Grafana + Alertmanager)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=changeme \
  --set prometheus.prometheusSpec.retention=30d

# Access Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
# Login: admin / changeme

# Import pre-built dashboards
# Grafana → Dashboards → Import → use dashboard ID from grafana.com
# 315 = Kubernetes cluster dashboard
# 1860 = Node Exporter Full
```

**For production Grafana**:
- Use persistent storage for Grafana SQLite/PostgreSQL (dashboards and alerts survive restarts).
- Enable SSO (OIDC/SAML) for authentication.
- Use Grafana Alerting with Alertmanager or PagerDuty as the contact point.

---

**Q: How do Prometheus and Grafana work together?**

**A:**

**Architecture**:
```
Applications → expose /metrics endpoint (Prometheus format)
                    ↓
Prometheus → scrapes /metrics every 15–30s → stores time-series in TSDB
                    ↓
Grafana → queries Prometheus via PromQL (HTTP API) → renders charts/dashboards
                    ↓
Alertmanager → receives alerts from Prometheus → routes to Slack/PagerDuty/email
```

**How it works end to end**:

1. **Instrumentation**: your application exposes metrics at `GET /metrics`:
   ```
   # HELP http_requests_total Total HTTP requests
   # TYPE http_requests_total counter
   http_requests_total{method="GET",status="200"} 1234
   ```

2. **Prometheus scraping**: Prometheus reads the `scrape_configs` in `prometheus.yml` (or ServiceMonitor CRDs with the Prometheus Operator), hits each target's `/metrics` endpoint, and stores the data as time-series in its TSDB.

3. **PromQL queries**: Grafana sends PromQL queries to Prometheus's HTTP API (`/api/v1/query_range`):
   ```promql
   rate(http_requests_total{status="200"}[5m])
   ```
   Prometheus evaluates the query against its TSDB and returns JSON results.

4. **Grafana visualization**: Grafana renders the results as graphs, tables, heatmaps, etc. Dashboards are JSON definitions that encode which PromQL queries to run and how to display them.

5. **Alerting**: Prometheus evaluates alerting rules on a configurable interval. When a condition is met for the `for` duration, it fires an alert to Alertmanager, which deduplicates, groups, and routes the alert to the right channel.

**Key integration points**:
- Grafana data source: `http://prometheus:9090` (or the ClusterIP service name inside Kubernetes).
- Prometheus Operator: manages Prometheus config via CRDs (`ServiceMonitor`, `PrometheusRule`, `AlertmanagerConfig`) — no manual config file editing needed.

---


