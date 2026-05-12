# Kubernetes Interview Questions & Answers

> All interview questions are vetted and signed

---

**Q: How does Kubernetes decide which node to schedule a pod on?**

**A:** The Kubernetes scheduler follows a two-phase process:
1. **Filtering (Predicates)**: eliminates nodes that can't run the pod — checks resource requests (CPU/memory), node selectors, taints/tolerations, affinity/anti-affinity rules, volume topology constraints, and port availability.
2. **Scoring (Priorities)**: ranks the remaining nodes using weighted scoring functions — least-requested resources, balanced resource allocation, inter-pod affinity, image locality (prefer nodes that already have the image).

The node with the highest score wins. If scores are tied, one is chosen at random. Custom schedulers or scheduler extenders can augment this logic.

---

**Q: What happens internally when you run `kubectl apply`?**

**A:**
1. kubectl serializes the resource manifest to JSON and sends a `PATCH` (strategic merge patch) or `PUT` request to the Kubernetes API server.
2. The API server authenticates (mTLS/token) and authorizes (RBAC) the request.
3. Admission controllers run (Mutating webhooks first, then Validating webhooks).
4. The object is persisted to etcd.
5. The relevant controller (e.g., Deployment controller) watches etcd for changes via the informer/watch mechanism.
6. The controller reconciles the desired state: creates/updates ReplicaSets, which create Pods.
7. The scheduler assigns unscheduled pods to nodes.
8. The kubelet on the assigned node pulls the image and starts the container via containerd/runc.

---

**Q: How does Kubernetes service discovery work?**

**A:** Kubernetes uses **CoreDNS** (cluster DNS) for service discovery:
- Every Service gets a DNS A record: `<service>.<namespace>.svc.cluster.local`.
- Pods resolve this name via the cluster DNS server (configured in `/etc/resolv.conf` of each pod).
- For ClusterIP services, the kube-proxy (or eBPF/Cilium) programs iptables/ipvs rules to DNAT traffic to the Service's ClusterIP → load-balance to one of the backing pod IPs (Endpoints).
- Headless services (`clusterIP: None`) return the pod IPs directly in DNS, enabling stateful peer discovery (e.g., for StatefulSets).

---

**Q: What is the difference between readiness and liveness probes internally?**

**A:**

| | Readiness Probe | Liveness Probe |
|---|---|---|
| Purpose | Is the pod ready to receive traffic? | Is the pod alive/functioning? |
| Failure action | Pod removed from Service Endpoints (no traffic) | Container is killed and restarted (based on restartPolicy) |
| Affects scheduling | No | No |
| Use case | Startup completion, transient degradation | Deadlock, infinite loop, unrecoverable state |

Kubelet runs both probes on a configurable period (`periodSeconds`). Readiness is checked throughout the pod's lifetime; a readiness failure mid-run takes the pod out of the load balancer rotation without restarting it.

---

**Q: How does Horizontal Pod Autoscaler (HPA) make scaling decisions?**

**A:** HPA polls the Metrics Server (or custom metrics via Prometheus Adapter) every 15 seconds (default):
1. Fetches current metric values for all pods in the target (e.g., average CPU utilization).
2. Computes desired replicas: `desiredReplicas = ceil(currentReplicas * (currentMetric / desiredMetric))`.
3. Applies min/max replica constraints.
4. Updates the target's replica count if the new value differs and the change exceeds the stabilization window (to prevent thrashing).

For scale-up: default stabilization window is 0s (immediate). For scale-down: 300s (5 minutes) to avoid premature scale-down under brief traffic drops.

---

**Q: How does Kubernetes handle pod failures and self-healing?**

**A:**
- **ReplicaSet controller**: continuously reconciles the desired pod count. If a pod dies or is deleted, the controller creates a replacement immediately.
- **Liveness probe**: kubelet restarts containers that fail health checks based on `restartPolicy` (Always, OnFailure, Never).
- **Node failure**: if a node becomes NotReady, the node controller waits `pod-eviction-timeout` (default 5 min) then marks pods for eviction; the scheduler places them on healthy nodes.
- **CrashLoopBackOff**: kubelet applies exponential backoff (10s, 20s, 40s, … up to 5 min) between restarts to avoid tight crash loops.

---

**Q: What happens during a rolling deployment in Kubernetes?**

**A:** With `strategy: RollingUpdate` (default for Deployments):
1. A new ReplicaSet is created for the new pod template.
2. Kubernetes scales up the new RS by `maxSurge` pods (default 25% above desired).
3. As new pods pass readiness probes, old pods are terminated — limited by `maxUnavailable` (default 25% can be unavailable at any time).
4. The process continues until the new RS has the full desired replica count and the old RS is scaled to 0.
5. The old RS is retained (for rollback) but has 0 replicas.

`kubectl rollout status deployment/myapp` monitors progress; `kubectl rollout undo` triggers a rollback.

---

**Q: How would you implement fine-grained service discovery across 1000+ microservices using Envoy or Istio?**

**A:** With Istio:
- Istio's control plane (Istiod) aggregates service registry data from Kubernetes and distributes xDS (Endpoint Discovery Service, Cluster Discovery Service) updates to all Envoy sidecar proxies dynamically.
- Use `VirtualService` and `DestinationRule` for fine-grained routing (header-based, weight-based canary, circuit breaking, retries, timeouts).
- At 1000+ services, optimize for control plane performance: enable `PILOT_ENABLE_EDS_DEBOUNCE`, use `Sidecar` resources to scope each proxy's discovery to only the services it actually calls (reduces xDS payload from O(n²) to O(k)).
- Use Envoy's `STRICT_DNS` or `EDS` cluster types. At scale, prefer EDS with health checking at the proxy level.
- Separate the service mesh into smaller domains (waypoints in Ambient Mesh) to reduce control-plane fan-out.

---

**Q: Explain how you'd leverage eBPF + Cilium to enforce network security policies at runtime, and what the advantages are over traditional CNIs?**

**A:** Cilium uses eBPF programs loaded into the Linux kernel's network stack (TC hooks, XDP) instead of iptables:
- **Network policies**: Cilium implements `CiliumNetworkPolicy` with L7 awareness (HTTP method/path, Kafka topic, DNS FQDN) — not just IP/port like standard NetworkPolicy.
- **Runtime enforcement**: eBPF programs evaluate policy inline in the kernel, with no userspace roundtrip. Policy changes are hot-swapped without packet loss.
- **Identity-based security**: Cilium assigns a numeric identity to each pod based on labels; policies use identities (not IPs), so they remain valid even as pods are rescheduled.
- **Advantages over kube-proxy/iptables**: no iptables rule explosion at scale, O(1) lookup via eBPF maps vs O(n) iptables traversal, native support for L7 policies, lower latency, kernel bypass with XDP for certain traffic paths.
- **Hubble**: Cilium's observability layer provides per-flow L7 visibility and Prometheus metrics without a sidecar.

---

**Q: What happens when systemd units fail intermittently on EKS nodes? How do you detect and heal?**

**A:**
- **Detection**: deploy Node Problem Detector (NPD) as a DaemonSet; it watches systemd journal for known patterns (OOM, disk pressure, kernel panics) and reports them as node conditions or events.
  - Custom problem definitions in `/etc/node-problem-detector/config/`.
  - CloudWatch Logs Agent / Fluent Bit ships systemd logs to CloudWatch for alerting.
- **Healing**:
  - NPD + Cluster Autoscaler / AWS Node Termination Handler: cordon & drain the problematic node, then terminate it so the ASG replaces it.
  - For critical units (kubelet, containerd): configure `Restart=always` and `StartLimitIntervalSec=0` in the systemd unit override.
  - Use AWS Systems Manager Run Command for remediation scripts without SSH.

---

**Q: Walk through advanced kube-probe configurations to detect business logic failures, not just HTTP 200.**

**A:**
```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - |
        # Check queue depth — business logic health
        depth=$(curl -s http://localhost:8080/metrics | grep queue_depth | awk '{print $2}')
        [ "$depth" -lt 10000 ]
  initialDelaySeconds: 30
  periodSeconds: 20
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready    # custom endpoint that checks DB connectivity, cache, downstream services
    port: 8080
    httpHeaders:
      - name: X-Health-Check
        value: "true"
  periodSeconds: 5
  successThreshold: 1
  failureThreshold: 2

startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30    # allows 5 minutes for slow startup before liveness kicks in
  periodSeconds: 10
```

---

**Q: Kubernetes nodes are healthy. But `kubectl logs` is blank for critical pods. What's happening?**

**A:** Possible causes:
- **Application not writing to stdout/stderr**: it logs to a file inside the container. Use a sidecar or check the file with `kubectl exec`.
- **Log rotation wiped the buffer**: logs rotated before you checked; increase `--container-log-max-size` on kubelet.
- **CRI buffer flushed**: some runtimes buffer stdout; check if logs appear after a flush.
- **Pod restarted**: you're seeing the new container's logs (empty); use `--previous` to see the terminated container's logs.
- **Excessive log rate causing truncation**: kubelet truncates log files; tune `--container-log-max-files`.
- **Centralized logging pipeline failure**: if using Fluent Bit/Fluentd to ship logs, check if the log shipper DaemonSet pod on that node is healthy.

---

**Q: A production pod was OOMKilled, but you can't find logs. Walk through a forensic-level debug.**

**A:**
1. `kubectl describe pod <pod>` — look for `OOMKilled` in `Last State` and the exit code (137).
2. `kubectl logs <pod> --previous` — logs from the killed container instance.
3. If logs are empty: check if the application flushed logs before OOM. OOM kills the process abruptly — unflushed buffers are lost.
4. **Node-level forensics**:
   ```bash
   # On the node
   journalctl -u kubelet | grep -i oom
   dmesg | grep -i "oom\|killed"
   # Shows which process was killed and its memory stats at time of kill
   ```
5. Check Prometheus/Datadog for `container_memory_working_set_bytes` historical graph — did it ramp up steadily (leak) or spike suddenly?
6. Look at HPA — was the pod under-resourced for the actual load?
7. **Remediation**: increase memory limit, fix the memory leak, add a liveness probe that triggers restart before OOM, add memory-based HPA.

---

**Q: Design a multi-tenant EKS cluster with isolation across dev, QA, and prod, with no noisy neighbors.**

**A:**
- **Separate node groups per environment**: dev/QA on spot instances (separate ASGs), prod on on-demand. Use taints (`environment=prod:NoSchedule`) and tolerations in pod specs.
- **Namespace isolation**: one namespace per tenant/environment. RBAC limits each team to their namespace.
- **Network policies**: Cilium/Calico `NetworkPolicy` blocks cross-namespace traffic by default; explicit allow rules for shared services.
- **Resource quotas**: `ResourceQuota` per namespace limits total CPU/memory/pod count to prevent one team starving others.
- **LimitRanges**: enforce min/max per container to prevent unbounded resource requests.
- **Priority classes**: prod pods get a higher `PriorityClass`; when the cluster is under pressure, lower-priority pods (dev) are evicted first.
- **Pod Disruption Budgets**: protect prod services during node drain.
- **Separate EKS clusters for prod** (strongest isolation) vs. multi-namespace on a shared cluster (cost-efficient for non-prod).

---

**Q: What's your approach to managing 10+ Kustomize overlays without drift or duplication?**

**A:**
- **Base + overlay hierarchy**: `base/` contains the canonical manifest set; each `overlays/<env>/` only declares patches. No resource duplication.
- **Components**: extract reusable patches (e.g., HPA config, resource limits) into `kustomize components` and include them in overlays that need them.
- **Schema validation**: run `kustomize build overlays/prod | kubeval` or `kubeconform` in CI on every change.
- **Drift detection**: ArgoCD `--self-heal` + `OutOfSync` alerts detect when cluster state diverges from Git.
- **DRY patches**: use `patchesStrategicMerge` or `patchesJson6902` for targeted changes; avoid copy-pasting full resource files into overlays.
- **Testing**: `kustomize build` all overlays in CI; diff output against the previous build to catch unintended changes.

---

**Q: Walk through your strategy to detect & mitigate pod-to-pod lateral movement inside a cluster.**

**A:**
- **Default-deny NetworkPolicy**: apply a baseline policy that blocks all ingress/egress in every namespace, then explicitly allow only required flows.
- **Cilium L7 policies**: restrict not just ports but HTTP paths and methods between services.
- **Runtime threat detection**: Falco DaemonSet detects anomalous syscalls (unexpected shell spawns, sensitive file reads, network connections from unexpected processes) and generates alerts.
- **mTLS via Istio/Cilium**: mutual TLS for all pod-to-pod communication authenticates workload identity; a compromised pod can't impersonate another service.
- **RBAC + IRSA**: pods have minimal IAM permissions; a compromised pod can't escalate to AWS.
- **Image signing**: Cosign + Kyverno admission policy — only signed images from trusted registries run in the cluster.
- **Audit logs**: EKS API server audit logs to CloudWatch; alert on unexpected `exec` into pods or new `ClusterRoleBinding` creation.

---

**Q: Difference between Deployment and StatefulSet?**

**A:**

| | Deployment | StatefulSet |
|---|---|---|
| Pod identity | Random names (pod-abc123) | Stable, ordinal names (pod-0, pod-1) |
| Storage | Shared or ephemeral | Each pod gets its own PVC (VolumeClaimTemplates) |
| Scaling order | Parallel | Sequential (pod-0 before pod-1) |
| Use case | Stateless apps (web servers, APIs) | Stateful apps (databases, Kafka, ZooKeeper) |
| DNS | Single service endpoint | Stable DNS per pod: `pod-0.svc.namespace.svc.cluster.local` |

---

**Q: What is a DaemonSet used for?**

**A:** A DaemonSet ensures exactly one pod runs on every (or a subset of) node(s). Common use cases:
- **Log collection**: Fluent Bit, Fluentd to collect container logs from node `/var/log`.
- **Monitoring**: Prometheus Node Exporter, Datadog agent.
- **Networking**: CNI plugins (Calico, Cilium), kube-proxy.
- **Security**: Falco (runtime security), vulnerability scanners.
- **Storage**: CSI node plugins, Ceph agents.

When nodes are added, the DaemonSet controller automatically schedules pods on them.

---

**Q: How does a Service in Kubernetes work?**

**A:** A Service is an abstraction that provides a stable network endpoint to a dynamic set of pods selected by a label selector.

- **ClusterIP** (default): virtual IP reachable only within the cluster; kube-proxy programs iptables/ipvs rules to DNAT to pod IPs.
- **NodePort**: exposes the service on a static port (30000–32767) on every node.
- **LoadBalancer**: provisions a cloud load balancer (ALB, NLB) in front of NodePorts.
- **ExternalName**: CNAME alias to an external DNS name.
- **Headless** (`clusterIP: None`): DNS returns pod IPs directly; no VIP or load balancing — used by StatefulSets.

---

**Q: What is a ConfigMap vs Secret?**

**A:** Both store configuration data as key-value pairs, but:

| | ConfigMap | Secret |
|---|---|---|
| Data type | Plain text | Base64-encoded (not encrypted at rest by default) |
| Use case | Non-sensitive config (env vars, config files) | Passwords, tokens, TLS certificates |
| RBAC | Broader access common | Should be tightly restricted |
| Encryption | Not encrypted | Encrypted at rest with EncryptionConfiguration + KMS provider |

Secrets are not inherently more secure than ConfigMaps unless etcd encryption is configured and RBAC restricts access. Use AWS Secrets Manager / HashiCorp Vault CSI driver for production secrets management.

---

**Q: What are taints and tolerations?**

**A:**
- **Taint**: applied to a node to repel pods that don't explicitly tolerate it. Syntax: `kubectl taint nodes node1 key=value:effect`.
  - Effects: `NoSchedule` (don't place new pods), `PreferNoSchedule` (avoid if possible), `NoExecute` (evict existing pods).
- **Toleration**: applied to a pod spec to allow it to be scheduled on a tainted node.

Use case: reserve GPU nodes for ML workloads (taint the node, only ML pods have the toleration); isolate prod/dev node groups.

---

**Q: How do liveness and readiness probes work?**

**A:** (See detailed answer in the readiness/liveness probe question above.)

Summary:
- **Liveness**: kubelet kills and restarts the container if the probe fails `failureThreshold` times consecutively.
- **Readiness**: kubelet removes the pod from the Service endpoint slice if the probe fails; restores it when the probe passes again.
- Probe types: `httpGet`, `tcpSocket`, `exec` (shell command), `grpc`.

---

**Q: How to troubleshoot a pod stuck in CrashLoopBackOff?**

**A:**
```bash
# Step 1: Check recent events
kubectl describe pod <pod-name> -n <namespace>

# Step 2: View logs from the crashed container
kubectl logs <pod-name> -n <namespace> --previous

# Step 3: Get the exit code
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Step 4: Override entrypoint to debug
kubectl run debug --image=<same-image> --command -- sleep 3600
kubectl exec -it debug -- /bin/sh
```

Common root causes: application error at startup (bad config, missing env var, wrong DB connection string), OOMKilled (exit 137), permission issues on mounted files, readiness probe too aggressive causing restart cycle.

---

**Q: How do you create and manage Kubernetes clusters (using tools like Terraform), and what are the master and worker nodes?**

**A:**
- **EKS via Terraform**: use the `terraform-aws-modules/eks/aws` module; define node groups (managed or self-managed), instance types, scaling config.
- **Master nodes (Control Plane)**: run etcd, API server, scheduler, controller manager. On EKS, AWS manages these; you don't access them directly.
- **Worker nodes**: run kubelet, kube-proxy, container runtime (containerd). Your workload pods run here. Managed via node groups (EC2 ASGs).

```hcl
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "my-cluster"
  cluster_version = "1.29"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    general = {
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 10
      desired_size   = 3
    }
  }
}
```

---

**Q: What are common Kubernetes errors you've faced (like CrashLoopBackOff, ImagePullError), and how did you resolve them?**

**A:**
- **CrashLoopBackOff**: container exits immediately after start. Fix: check `kubectl logs --previous`, fix application startup error or missing config.
- **ImagePullBackOff / ErrImagePull**: can't pull image. Fix: verify image name/tag, check registry credentials (`kubectl get secret`), ensure ECR auth token isn't expired, check network access from node to registry.
- **OOMKilled**: container exceeded memory limit. Fix: increase `resources.limits.memory`, fix memory leak.
- **Pending (Unschedulable)**: no node can satisfy constraints. Fix: check `kubectl describe pod` for "Insufficient cpu/memory" — scale the cluster or reduce requests.
- **CreateContainerConfigError**: bad ConfigMap/Secret reference. Fix: verify the ConfigMap/Secret exists in the same namespace.
- **NodeNotReady**: kubelet or network plugin issue on the node. Fix: SSH to node, check `systemctl status kubelet`, `journalctl -u kubelet`.

---

**Q: What is the command to access a pod and how can you define or create a Kubernetes class or object?**

**A:**
```bash
# Access a pod's shell
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# Apply a manifest (create/update)
kubectl apply -f resource.yaml

# Create imperatively
kubectl create deployment myapp --image=nginx --replicas=3
kubectl expose deployment myapp --port=80 --type=LoadBalancer
```

A Kubernetes object is defined in YAML with `apiVersion`, `kind`, `metadata`, and `spec` fields. Apply with `kubectl apply -f`.

---

**Q: How do you handle authentication for EKS clusters and store secrets securely?**

**A:**
- **Authentication**: EKS uses IAM for authentication. The AWS IAM Authenticator maps IAM roles/users to Kubernetes RBAC via the `aws-auth` ConfigMap (or the newer access entries API in EKS 1.29+).
  ```bash
  aws eks update-kubeconfig --name my-cluster --region us-east-1 --role-arn arn:aws:iam::123:role/EKSAdmin
  ```
- **Pod-level auth**: IRSA (IAM Roles for Service Accounts) — annotate a ServiceAccount with an IAM role ARN; pods using that SA get temporary credentials via OIDC.
- **Secrets**: use AWS Secrets Manager + Secrets Store CSI Driver to mount secrets as files/env vars; secrets never stored in etcd.

---

**Q: How do you expose a Kubernetes application to external traffic?**

**A:**
1. **LoadBalancer Service**: creates a cloud LB (NLB/ALB). Simple but one LB per service = costly.
2. **Ingress + Ingress Controller** (AWS Load Balancer Controller, NGINX): single ALB routes to multiple services by hostname/path. Most common approach.
3. **NodePort**: exposes on a port on every node — typically used with an external LB in front.
4. **ExternalDNS**: automatically creates Route53 records for Services/Ingresses.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

---

**Q: How would you implement blue-green deployment in Kubernetes?**

**A:**
- Deploy both `blue` (current) and `green` (new) Deployments simultaneously.
- The Service selector points to `version: blue`.
- After validating green, update the Service selector to `version: green`:
  ```bash
  kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'
  ```
- Traffic instantly shifts to green. If issues arise, revert the selector to blue.
- Automate with Argo Rollouts (`BlueGreen` strategy) for automated analysis and promotion.

---

**Q: How do you implement network policies to restrict pod-to-pod communication?**

**A:**
```yaml
# Default deny all ingress in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
# Allow only frontend to call backend on port 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

**Q: A critical production Kubernetes cluster is experiencing multiple issues: Pods stuck in ImagePullBackOff, some pods being evicted, and users reporting 503 errors. Walk through your troubleshooting and prevention.**

**A:**
**Triage in parallel**:

1. **ImagePullBackOff**: `kubectl describe pod <pod>` → check events for "unauthorized" or "not found". Fix: refresh ECR auth token, check `imagePullSecrets`, verify image tag exists.
2. **Pod eviction**: `kubectl get events | grep Evicted`. Check node resource pressure: `kubectl describe node` → `Conditions` section. If `MemoryPressure` or `DiskPressure`, nodes are evicting low-priority pods. Fix: add nodes, clean disk (`docker system prune`), increase memory on nodes, review resource requests.
3. **503 errors**: `kubectl get endpoints myapp` — are pod IPs listed? If no endpoints, readiness probes are failing. Check `kubectl describe pod` for probe failures. Also check Ingress controller pod health.

**Prevention**: resource quotas per namespace, PodDisruptionBudgets, cluster autoscaler with appropriate scale-out policies, image tag pinning (no `latest`), pre-pull critical images as DaemonSet init, regular node disk cleanup via DaemonSet job.

---

**Q: Your Pod is restarting frequently. How do you identify the root cause?**

**A:**
```bash
kubectl describe pod <pod>          # check Last State exit code, reason
kubectl logs <pod> --previous       # logs from previous instance
kubectl get events -n <ns>          # cluster events for the pod

# Exit codes:
# 0  = clean exit (restartPolicy issue)
# 1  = application error
# 137 = OOMKilled (SIGKILL)
# 143 = SIGTERM (graceful shutdown, but why?)
```

Check: liveness probe configuration (too aggressive `failureThreshold`), memory limits (OOMKilled), application startup crash (bad config), node disk pressure triggering eviction.

---

**Q: A Deployment is stuck in "progressing" state — how do you troubleshoot?**

**A:**
```bash
kubectl rollout status deployment/myapp     # shows current status
kubectl describe deployment myapp           # check Conditions
kubectl get rs -l app=myapp                 # new ReplicaSet pod count
kubectl describe pod <new-pod>              # why isn't it ready?
```

Causes: readiness probe failing (new pods never become Ready so rollout never completes), image pull error, resource quota exceeded (can't schedule new pods), `progressDeadlineSeconds` too short.

Fix or rollback: `kubectl rollout undo deployment/myapp`.

---

**Q: How do you debug a CrashLoopBackOff issue?**

**A:** See the detailed troubleshooting answer above. Key steps:
1. `kubectl logs <pod> --previous` — get the last output before crash.
2. `kubectl describe pod` — check exit code and events.
3. Override entrypoint to get a shell and manually run the application to see the error.
4. Check environment variables and mounted secrets/configmaps are present and correct.
5. Verify resource limits aren't causing OOM.

---

**Q: How do you check which Pods are consuming high memory or CPU?**

**A:**
```bash
# Requires Metrics Server
kubectl top pods -n <namespace> --sort-by=memory
kubectl top pods -A --sort-by=cpu

# Node-level
kubectl top nodes

# Detailed per-container metrics
kubectl top pods --containers -n <namespace>
```

For historical trends, use Prometheus + Grafana (`container_memory_working_set_bytes`, `container_cpu_usage_seconds_total`).

---

**Q: Node is in NotReady state — steps to investigate?**

**A:**
```bash
kubectl describe node <node-name>    # check Conditions (MemoryPressure, DiskPressure, NetworkUnavailable)
kubectl get events --field-selector involvedObject.name=<node-name>

# SSH to the node
systemctl status kubelet
journalctl -u kubelet -n 100
systemctl status containerd

# Check disk
df -h
# Check network
ping <api-server-ip>
```

Common causes: kubelet crashed, disk full, OOM on node, network partition, NTP drift causing cert issues, CNI plugin failure.

---

**Q: Your application is not accessible through service — what will you check?**

**A:**
```bash
# 1. Is the service defined correctly?
kubectl get svc myapp -n ns
kubectl describe svc myapp -n ns    # check selector labels

# 2. Are there healthy endpoints?
kubectl get endpoints myapp -n ns   # should list pod IPs

# 3. Do pod labels match service selector?
kubectl get pods -n ns --show-labels

# 4. Is the pod actually ready?
kubectl get pods -n ns

# 5. Test from inside the cluster
kubectl run test --image=busybox --rm -it -- wget -qO- http://myapp.ns.svc.cluster.local

# 6. Check NetworkPolicy blocking
kubectl get networkpolicies -n ns
```

---

**Q: How do you handle failed daemonset pods?**

**A:**
- `kubectl describe ds <daemonset>` — check update strategy and tolerations.
- `kubectl get pods -l app=<ds-label> -A -o wide` — see which nodes have failing pods.
- `kubectl logs <ds-pod> -n <ns>` and `kubectl describe pod <ds-pod>` — get the failure reason.
- Common causes: node taint not tolerated by the DaemonSet, host path permission issue, port conflict on the node, resource constraints.
- If the DaemonSet update is broken: `kubectl rollout undo daemonset/<name>`.

---

**Q: Persistent Volume not attaching — what's your troubleshooting approach?**

**A:**
```bash
kubectl describe pvc <pvc-name> -n <ns>    # check status and events
kubectl describe pv <pv-name>              # check reclaim policy, access mode
kubectl get volumeattachment               # check if attachment object exists
kubectl describe pod <pod> -n <ns>        # events for volume mount errors
```

Common causes: AZ mismatch (EBS volume in us-east-1a, pod scheduled in us-east-1b — fix: use topology constraints or EFS for multi-AZ), `accessMode` mismatch (EBS is `ReadWriteOnce` — can't be mounted by two pods on different nodes), CSI driver pod unhealthy, PVC not bound (no matching PV or StorageClass issue).

---

**Q: How do you perform rolling updates and rollbacks safely?**

**A:**
```bash
# Update image
kubectl set image deployment/myapp container=myregistry/myapp:v2.0.0 -n ns
# OR edit the manifest and kubectl apply

# Monitor rollout
kubectl rollout status deployment/myapp -n ns --timeout=5m

# Rollback
kubectl rollout undo deployment/myapp -n ns
kubectl rollout undo deployment/myapp --to-revision=3   # specific revision

# View history
kubectl rollout history deployment/myapp -n ns
```

Safety mechanisms: always set readiness probes, use `maxUnavailable: 0` for zero-downtime, set `progressDeadlineSeconds`, use `--atomic` in Helm for automatic rollback.

---

**Q: How do you perform health checks for Pods and Nodes in live environments?**

**A:**
```bash
# Pod health
kubectl get pods -A                             # overview
kubectl top pods -A --sort-by=memory           # resource usage
kubectl describe pod <pod>                      # events, probe status

# Node health
kubectl get nodes                               # Ready/NotReady
kubectl top nodes                               # CPU/memory usage
kubectl describe node <node>                   # conditions, capacity, allocated resources

# Cluster health dashboard
# Prometheus + Grafana: kubernetes-mixin dashboards
# Lens IDE for visual cluster management
```

In production: Prometheus alerts for `KubeNodeNotReady`, `KubePodCrashLooping`, `KubeDeploymentReplicasMismatch`, and `KubePersistentVolumeFillingUp`.

- How would you design auto-scaling for 50M+ concurrent viewers across multiple K8s clusters without over-provisioning?
- During an IPL final, a new region needs to spin up instantly. How would you pre-warm nodes & scale workloads with zero cold-start impact?
- What’s your approach to multi-zone pod affinity/anti-affinity to ensure a node failure doesn’t impact regional streaming SLAs?
- Describe K8s readiness/liveness probe configs to catch buffering/lag issues in stream-processing microservices before users notice.
- Playback failures spike for only 3% of users in the APAC region. CPU, memory, and pods look fine. Could you walk through your triage plan?
- Your pod keeps getting stuck in CrashLoopBackOff, but logs show no errors. How would you approach debugging and resolution?
- You have a StatefulSet deployed with persistent volumes, and one of the pods is not recreating properly after deletion. What could be the reasons, and how do you fix it without data loss?
- ⁠Your cluster autoscaler is not scaling up even though pods are in Pending state. What would you investigate?
-  ⁠One of your microservices has to connect to an external database via a VPN inside the cluster. How would you architect this in Kubernetes with HA and security in mind?
- You're running a multi-tenant platform on a single EKS cluster. How do you isolate workloads and ensure security, quotas, and observability for each tenant?
- You notice the kubelet is constantly restarting on a particular node. What steps would you take to isolate the issue and ensure node stability?
- A critical pod in production gets evicted due to node pressure. How would you prevent this from happening again, and how do QoS classes play a role?
- You need to deploy a service that requires TCP and UDP on the same port. How would you configure this in Kubernetes using Services and Ingress?
- ⁠An application upgrade caused downtime even though you had rolling updates configured. What advanced strategies would you apply to ensure zero-downtime deployments next time?
- ou need to create a Kubernetes operator to automate complex application lifecycle events. How do you design the CRD and controller loop logic?
- ⁠Multiple nodes are showing high disk IO usage due to container logs. What Kubernetes features or practices can you apply to avoid this scenario?
- ⁠Your Kubernetes cluster's etcd performance is degrading. What are the root causes and how do you ensure etcd high availability and tuning?
- ⁠You’re running 1,000+ stateful services across multi-region Kubernetes clusters. Walk us through your approach to maintaining consistency and quorum during a regional failover.
- A Canary deployment in EKS passes all probes but causes intermittent trade submission failures. You have <5 minutes to rollback, your RCA path?




