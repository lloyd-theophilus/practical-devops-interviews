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

**Q: How would you design auto-scaling for 50M+ concurrent viewers across multiple K8s clusters without over-provisioning?**

**A:** The core challenge is handling massive burst capacity while keeping idle cost low. A layered approach:

**Cluster topology**
- Federate clusters per region (us-east-1, eu-west-1, ap-south-1) with a global traffic manager (Route53 latency routing or Cloudflare).
- Each cluster has dedicated node groups per tier: edge/CDN proxy, stream-processing, transcoding, and API — each with independent HPA/KEDA targets.

**Scaling stack**
- **KEDA** over plain HPA: scale on real business metrics (active viewer sessions from Redis Streams, Kafka consumer lag, SQS queue depth) rather than CPU/memory.
- **Cluster Autoscaler + Karpenter**: Karpenter provisions right-sized nodes in ~45s; configure `consolidation: true` to bin-pack and reclaim idle nodes automatically.
- **Predictive scaling**: use historical patterns (match-day schedule, prime-time) to pre-provision a baseline via scheduled CronJobs or KEDA cron trigger; reactive HPA handles unexpected spikes on top.

**Avoiding over-provisioning**
- Set tight `resources.requests` based on p95 measured usage (VPA in recommendation mode), not guesses.
- Use Spot/Preemptible for stateless tiers (edge proxies, API pods) with Spot interruption handlers (AWS Node Termination Handler).
- Scale-down stabilization window tuned per tier: aggressive (30s) for bursty API pods, conservative (5 min) for stateful stream processors.

**Example KEDA ScaledObject**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: stream-processor
spec:
  scaleTargetRef:
    name: stream-processor
  minReplicaCount: 10
  maxReplicaCount: 5000
  triggers:
    - type: redis-streams
      metadata:
        address: redis-master:6379
        stream: viewer-events
        consumerGroup: processors
        lagCount: "50"        # scale up when each replica lags by >50 messages
```

---

**Q: During an IPL final, a new region needs to spin up instantly. How would you pre-warm nodes and scale workloads with zero cold-start impact?**

**A:** Zero cold-start at event time means all work must be done before the event starts.

**Node pre-warming**
- Use a scheduled CronJob (or Argo Workflows) that fires T-60 minutes before the event: scales the target node group’s ASG `minSize` to the projected baseline.
- Karpenter `Provisioner` with `ttlSecondsAfterEmpty: 600` — nodes stay warm for 10 minutes after they drain, so brief traffic dips don’t destroy capacity.

**Image pre-pull**
- DaemonSet `image-prepuller` runs on all nodes as soon as they join; pulls the 3-5 critical container images so the first pod start is instant (no registry pull latency).
```yaml
initContainers:
  - name: prepull
    image: gcr.io/google-containers/pause:3.9   # tiny image, just to pre-warm kubelet image cache
```
- Or use Karpenter node templates with `userData` to pre-pull via `containerd` before the node marks itself Ready.

**Workload pre-scaling**
- Scheduled HPA override via a Kubernetes `CronJob` that patches the HPA’s `minReplicas` before the event:
```bash
kubectl patch hpa stream-api -p ‘{"spec":{"minReplicas":200}}’ --as event-scaler
```
- Restore via a post-event CronJob.

**DNS and LB warm-up**
- Pre-register the new region’s NLB target groups with Route53 weighted routing at 0% weight; ramp to 100% over 5 minutes using a script, avoiding cold DNS TTL propagation hitting users.

**Validation gate**
- Synthetic traffic replay (k6 or Gatling) runs against the new region 30 minutes before go-live; automated rollback if p99 latency > threshold.

---

**Q: What’s your approach to multi-zone pod affinity/anti-affinity to ensure a node failure doesn’t impact regional streaming SLAs?**

**A:** The goal is to spread replicas across failure domains (zones, nodes) while keeping latency-sensitive pod pairs co-located.

**Spread across zones (anti-affinity)**
```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: stream-processor
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: stream-processor
```
`TopologySpreadConstraints` is preferred over `podAntiAffinity` at scale — it’s declarative about the allowed imbalance rather than hard "never co-locate" rules that can cause unschedulable pods.

**Co-locate latency-sensitive pairs (affinity)**
```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: redis-sidecar
          topologyKey: kubernetes.io/hostname
```

**PodDisruptionBudgets to protect SLAs**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: stream-processor-pdb
spec:
  minAvailable: "70%"   # at least 70% of replicas always up during drains/upgrades
  selector:
    matchLabels:
      app: stream-processor
```

**Zone-aware routing**: use Kubernetes `topologyKeys` in Service `spec.topologyKeys` (or Cilium topology-aware hints) to prefer same-zone endpoints, so a zone failure auto-drains traffic to surviving zones without routing cross-zone unnecessarily.

---

**Q: Describe K8s readiness/liveness probe configs to catch buffering/lag issues in stream-processing microservices before users notice.**

**A:** Standard HTTP 200 probes are insufficient for stream processors — a pod can be "up" but silently falling behind. Layer three probe types:

**Startup probe** — give slow JVM/Python services time to initialize without liveness killing them:
```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10    # up to 5 minutes before liveness takes over
```

**Liveness probe** — detect deadlocks and processing stalls via a custom business-logic check:
```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - |
        LAG=$(curl -sf http://localhost:9090/metrics | awk ‘/kafka_consumer_lag_sum/{print $2}’)
        PROC=$(curl -sf http://localhost:9090/metrics | awk ‘/frames_processed_rate_1m/{print $2}’)
        # Fail if lag > 10000 AND processing rate has dropped to near zero
        [ "$LAG" -lt 10000 ] || [ "$PROC" -gt 0.1 ]
  initialDelaySeconds: 60
  periodSeconds: 30
  failureThreshold: 3
  timeoutSeconds: 10
```

**Readiness probe** — pull out of rotation before users hit buffering:
```yaml
readinessProbe:
  httpGet:
    path: /health/ready    # checks: buffer fill %, upstream connectivity, output queue depth
    port: 8080
  periodSeconds: 5
  failureThreshold: 2      # fail fast: 10s to remove from LB
  successThreshold: 2      # require 2 passes to re-admit (avoid flapping)
  timeoutSeconds: 3
```

The `/health/ready` endpoint should internally check: Kafka consumer lag < threshold, output buffer fill < 80%, upstream segment store reachable. Expose these checks as individual Prometheus gauges for alerting too.

---

**Q: Playback failures spike for only 3% of users in the APAC region. CPU, memory, and pods look fine. Could you walk through your triage plan?**

**A:** 3% failure rate with healthy infrastructure metrics is a routing, CDN, or data-path issue, not a compute issue.

**Step 1 — Characterize the 3%**
- Segment the failures by: ISP, device type, client SDK version, specific stream resolution/bitrate, geographic sub-region (country within APAC).
- Pull error codes from the client telemetry pipeline — are they HTTP 4xx, 5xx, timeout, or client-side decode errors?

**Step 2 — Trace a failing request end-to-end**
```bash
# Check if specific pods are serving the failing users (sticky sessions, consistent hashing)
kubectl logs -l app=stream-edge -n apac --since=15m | grep "error\|5[0-9][0-9]"

# Check endpoint health for the APAC service
kubectl get endpoints stream-edge -n apac
```

**Step 3 — CDN and network path**
- Check CDN PoP error rates — the 3% may be routing to a specific edge node with a bad upstream connection.
- Test from APAC VPN: `curl -v --http2 https://stream.example.com/segment/001.ts` — check TLS handshake, TTFB, and response headers for cache HIT/MISS.

**Step 4 — Kubernetes-level checks**
- Are the failing users hitting a specific pod? Check access logs with pod IP correlation.
- Check for DNS resolution issues: `kubectl exec` into a pod in the APAC cluster and resolve upstream dependencies.
- Inspect `NetworkPolicy` — is a recently deployed policy accidentally blocking certain egress paths?

**Step 5 — Stateful path issues**
- If using consistent hashing for session affinity, check if a recently scaled event caused a rehash, routing some users to pods that don’t have their session state.
- Check Redis/session store latency from APAC pods: are some keys hitting a cross-region replica with high latency?

**Step 6 — Resolution and rollback gate**
- If a specific pod revision correlates with failures: `kubectl rollout undo deployment/stream-edge -n apac`.
- If CDN PoP: re-route traffic away from the failing PoP via DNS/CDN config.

---

**Q: Your pod keeps getting stuck in CrashLoopBackOff, but logs show no errors. How would you approach debugging and resolution?**

**A:** No-log CrashLoopBackOff is one of the trickiest cases. The container exits before it can write logs, or the crash is happening at a layer below the application.

**Step 1 — Check exit code**
```bash
kubectl get pod <pod> -o jsonpath=’{.status.containerStatuses[0].lastState.terminated.exitCode}’
# 137 = OOMKill (SIGKILL)
# 139 = Segfault (SIGSEGV)
# 1   = generic application error (but logs empty → app crashed before logger init)
# 126/127 = command not found / permission denied on entrypoint
```

**Step 2 — Override the entrypoint**
```bash
# Run the same image with a sleep so you can exec in and manually run the app
kubectl run debug-pod --image=<same-image> --restart=Never \
  --command -- sleep 3600
kubectl exec -it debug-pod -- /bin/sh
# Now manually run the application binary and see the actual error
```

**Step 3 — Init container and volume issues**
```bash
kubectl describe pod <pod>    # check init container status, volume mount events
# Common: init container wrote a file the main container tries to read but permissions are wrong
# Or: emptyDir mount path conflict with image filesystem
```

**Step 4 — Resource limits causing immediate OOM**
```bash
# Check if memory limit is set so low the process can’t even start
kubectl get pod <pod> -o jsonpath=’{.spec.containers[0].resources}’
# Check node-level OOM killer
kubectl get events --field-selector reason=OOMKilling
```

**Step 5 — Security context blocking execution**
```bash
# securityContext readOnlyRootFilesystem: true can cause crashes if the app writes temp files
# Check for AppArmor/seccomp profiles blocking syscalls
kubectl get pod <pod> -o yaml | grep -A10 securityContext
```

**Step 6 — Image entrypoint mismatch**
```bash
docker inspect <image> | jq ‘.[0].Config.Entrypoint, .[0].Config.Cmd’
# Compare with what’s in the pod spec
```

---

**Q: You have a StatefulSet deployed with persistent volumes, and one of the pods is not recreating properly after deletion. What could be the reasons, and how do you fix it without data loss?**

**A:**

**Diagnose first**
```bash
kubectl describe pod <statefulset-pod-name>    # what phase is it stuck in?
kubectl describe pvc <pvc-name>               # is the PVC bound?
kubectl get pv                                 # is the PV in Released/Failed state?
kubectl get events -n <ns> --sort-by=lastTimestamp
```

**Common causes and fixes**

| Cause | Symptom | Fix |
|---|---|---|
| PVC stuck in `Terminating` | Pod can’t start because PVC exists but is terminating | Remove `kubernetes.io/pvc-protection` finalizer: `kubectl patch pvc <pvc> -p ‘{"metadata":{"finalizers":[]}}’ --type=merge` |
| PV in `Released` state | PVC bound but PV reclaim policy released it from old binding | Patch PV `claimRef` to `null` to make it `Available` again, then re-bind |
| AZ mismatch | Pod scheduled in zone A, EBS volume in zone B | Add `topologySpreadConstraints` or `nodeAffinity` to pin StatefulSet pods to the correct zone |
| Node cordoned | Pod can’t be scheduled | `kubectl uncordon <node>` or let the scheduler place it on another node if volumes are portable (EFS/NFS) |
| Finalizer blocking pod deletion | Pod stuck in `Terminating` | `kubectl patch pod <pod> -p ‘{"metadata":{"finalizers":[]}}’ --type=merge` — only after confirming safe |

**Preventing data loss**
- **Never** delete a PVC while data is in use. Always confirm the pod is stopped first.
- Take an EBS snapshot before any PV manipulation: `aws ec2 create-snapshot --volume-id <vol-id>`.
- Use `VolumeSnapshotClass` and `VolumeSnapshot` CRDs for application-consistent snapshots.
- Set `persistentVolumeReclaimPolicy: Retain` on critical PVs so accidental PVC deletion doesn’t destroy the underlying volume.

---

**Q: Your cluster autoscaler is not scaling up even though pods are in Pending state. What would you investigate?**

**A:**

**Step 1 — Check autoscaler logs**
```bash
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100 | grep -i "scale up\|cannot\|pending\|error"
```

**Step 2 — Why are pods Pending?**
```bash
kubectl describe pod <pending-pod>    # look at the "Events" section
# "0/5 nodes available: 3 Insufficient cpu, 2 node(s) had taint..."
```

**Common reasons CA doesn’t scale up**

| Reason | Details |
|---|---|
| Pod has `priorityClass` too low | CA won’t scale for pods below the configured priority threshold |
| All pods have local storage (`emptyDir`, `hostPath`) | CA considers these unschedulable even on new nodes; it won’t scale |
| Pod requests exceed the largest instance type in any node group | CA can’t satisfy the request even with new nodes |
| `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation on all pods of a node group | CA won’t scale that group |
| Node group at `maxSize` | Check AWS console: ASG max count reached |
| ASG scaling cooldown | AWS-side cooldown preventing the ASG from adding nodes |
| Pod has `nodeSelector` or affinity targeting a tainted/non-existent node group | No node group matches the pod’s scheduling constraints |
| Expander config wrong | `--expander=least-waste` may pick a group incorrectly; try `random` to confirm |

**Step 3 — Verify node group configuration**
```bash
# Check if node groups have available capacity
kubectl get nodes -L node.kubernetes.io/instance-type
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> \
  --query ‘AutoScalingGroups[0].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity}’
```

**Step 4 — Fix**
- If at max size: increase ASG `maxSize` in Terraform/console.
- If affinity mismatch: adjust pod `nodeSelector`/tolerations or add a new node group.
- If local storage: replace `emptyDir` with a shared volume (EFS) for pods that need to be re-schedulable.

---

**Q: One of your microservices has to connect to an external database via a VPN inside the cluster. How would you architect this in Kubernetes with HA and security in mind?**

**A:**

**Architecture options**

**Option 1 — AWS-native (preferred for EKS)**
- Deploy the external database behind AWS Transit Gateway or Site-to-Site VPN.
- EKS worker nodes reside in a VPC with a VPN attachment; pods inherit node-level routing.
- No Kubernetes-specific VPN pod needed; all traffic is routed at the VPC level.

**Option 2 — In-cluster VPN sidecar / DaemonSet**
```yaml
# VPN client as a sidecar — not recommended for HA (single point of failure per pod)
# Better: VPN DaemonSet on dedicated nodes with taint + toleration
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: vpn-gateway
spec:
  selector:
    matchLabels:
      app: vpn-gateway
  template:
    spec:
      tolerations:
        - key: vpn-node
          effect: NoSchedule
      containers:
        - name: vpn
          image: vpn-client:1.0
          securityContext:
            capabilities:
              add: ["NET_ADMIN"]
          volumeMounts:
            - name: vpn-config
              mountPath: /etc/vpn
              readOnly: true
      volumes:
        - name: vpn-config
          secret:
            secretName: vpn-credentials
```

**HA design**
- Run VPN gateways across 3 AZs (one DaemonSet pod per VPN node, one node per AZ).
- Use a Kubernetes `Service` (ClusterIP) in front of the VPN pods; application pods route database traffic to this service, which load-balances across the 3 VPN endpoints.
- Configure BGP or static routes so each VPN pod can reach the external subnet.

**Security**
- Store VPN credentials in AWS Secrets Manager, mounted via Secrets Store CSI Driver — never as Kubernetes Secrets in etcd.
- `NetworkPolicy` restricts database egress to only the specific pods that need it (label-based).
- mTLS between microservice and the VPN gateway endpoint.
- Audit: log all connections via the VPN gateway using Cilium Hubble or Envoy access logs.

---

**Q: You’re running a multi-tenant platform on a single EKS cluster. How do you isolate workloads and ensure security, quotas, and observability for each tenant?**

**A:**

**Namespace-per-tenant as the isolation boundary**

```
cluster/
  namespace: tenant-alpha
  namespace: tenant-beta
  namespace: tenant-gamma
  namespace: shared-infra      # ingress, observability stack
```

**Network isolation**
```yaml
# Default deny all cross-namespace traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: tenant-alpha
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Allow egress to DNS only (add per-service allow rules as needed)
spec:
  egress:
    - ports:
        - port: 53
          protocol: UDP
```

**Resource quotas and limits**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-alpha-quota
  namespace: tenant-alpha
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    count/pods: "100"
    count/services: "20"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-alpha-limits
  namespace: tenant-alpha
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 8Gi
```

**RBAC — tenant teams only see their namespace**
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tenant-alpha-admin
  namespace: tenant-alpha
subjects:
  - kind: Group
    name: tenant-alpha-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

**Admission control — Kyverno policies**
- Require resource requests/limits on all containers.
- Disallow privileged containers and host network.
- Enforce image registry allowlist per namespace.

**Observability per tenant**
- Prometheus with `namespace` label on all metrics; Grafana dashboards filtered by namespace.
- Loki log streams labeled by namespace; per-tenant Grafana folders with access control.
- AWS Cost Allocation Tags on node groups mapped to tenant namespaces.

---

**Q: You notice the kubelet is constantly restarting on a particular node. What steps would you take to isolate the issue and ensure node stability?**

**A:**

**Step 1 — Immediately cordon the node** (prevent new pods from scheduling while you investigate)
```bash
kubectl cordon <node-name>
```

**Step 2 — Check kubelet logs on the node**
```bash
# SSH to the node or use SSM Session Manager
journalctl -u kubelet -n 200 --no-pager
journalctl -u kubelet --since "30 minutes ago" | grep -E "error|fatal|panic|certificate"
```

**Common causes and remediation**

| Symptom in logs | Cause | Fix |
|---|---|---|
| `x509: certificate has expired` | kubelet client cert expired | Rotate certs: `kubeadm certs renew` or delete and re-approve CSR |
| `failed to get node info` | API server unreachable from node | Check security groups, VPC routing, API server health |
| `PLEG is not healthy` | Pod lifecycle event generator falling behind; usually disk/CPU pressure | Free disk space, investigate I/O wait: `iostat -x 1` |
| `eviction manager: attempting to reclaim` followed by OOM | Node running out of memory | Add memory, set resource limits on noisy pods, check for memory leaks |
| `transport: Error while dialing dial tcp: connection refused` | containerd not running | `systemctl restart containerd && systemctl restart kubelet` |
| Kubelet restart loop with no clear error | Kubelet config corruption | Check `/var/lib/kubelet/config.yaml` and `/etc/kubernetes/kubelet.conf` |

**Step 3 — Check system-level health**
```bash
dmesg | tail -50                    # kernel messages, OOM events
df -h && df -i                      # disk space and inode exhaustion
free -m                             # memory
systemctl status containerd         # container runtime
ps aux | grep kubelet               # is kubelet actually running between restarts?
```

**Step 4 — Resolve and uncordon**
After fixing the root cause, drain remaining pods if needed (`kubectl drain --ignore-daemonsets --delete-emptydir-data <node>`), apply the fix, verify kubelet is stable for 5+ minutes, then uncordon.

---

**Q: A critical pod in production gets evicted due to node pressure. How would you prevent this from happening again, and how do QoS classes play a role?**

**A:**

**QoS classes determine eviction order**

Kubernetes assigns QoS classes based on resource spec:

| QoS Class | Criteria | Eviction Priority |
|---|---|---|
| `Guaranteed` | `limits` == `requests` for all containers (CPU + memory) | Last to be evicted |
| `Burstable` | At least one container has `requests` < `limits` | Evicted after BestEffort |
| `BestEffort` | No `requests` or `limits` set | First to be evicted |

**Immediate fix — make critical pods Guaranteed**
```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "2Gi"    # must equal requests for Guaranteed class
    cpu: "1000m"
```

**PriorityClass — prevent eviction in favor of lower-priority pods**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-production
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
---
# In pod spec:
priorityClassName: critical-production
```
High-priority pods preempt lower-priority ones during scheduling; eviction manager also respects priority when choosing victims.

**PodDisruptionBudget — prevent voluntary evictions**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: critical-service
```

**Node-level prevention**
- Set kubelet `--eviction-hard` and `--eviction-soft` thresholds conservatively; use Cluster Autoscaler to add nodes before hard eviction thresholds are hit.
- Reserve resources for system daemons: `--system-reserved=cpu=500m,memory=1Gi` and `--kube-reserved=cpu=500m,memory=1Gi` so application pods aren’t competing with the OS.
- Alert on node memory pressure before eviction: `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.15`.

---

**Q: You need to deploy a service that requires TCP and UDP on the same port. How would you configure this in Kubernetes using Services and Ingress?**

**A:** Kubernetes Services support mixed protocols, but with important caveats depending on the Service type.

**Service configuration with mixed protocols**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server
spec:
  selector:
    app: game-server
  ports:
    - name: game-tcp
      protocol: TCP
      port: 7777
      targetPort: 7777
    - name: game-udp
      protocol: UDP
      port: 7777
      targetPort: 7777
  type: LoadBalancer
```

**Caveats**
- **LoadBalancer type**: prior to Kubernetes 1.26, mixing TCP and UDP on the same port in a `LoadBalancer` service was not supported on most cloud providers. From 1.24+, the feature gate `MixedProtocolLBService` is GA.
- **AWS NLB** (via AWS Load Balancer Controller): supports TCP and UDP on the same port for NLB. Annotate with `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"`.
- **Ingress**: Ingress controllers are HTTP/HTTPS only (L7). TCP/UDP cannot be routed through a standard Ingress resource. Use NGINX Ingress’s `tcp-services` and `udp-services` ConfigMaps instead:

```yaml
# ConfigMap for NGINX Ingress TCP forwarding
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  "7777": "default/game-server:7777"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: udp-services
  namespace: ingress-nginx
data:
  "7777": "default/game-server:7777"
```

Then pass `--tcp-services-configmap` and `--udp-services-configmap` args to the NGINX Ingress controller.

---

**Q: An application upgrade caused downtime even though you had rolling updates configured. What advanced strategies would you apply to ensure zero-downtime deployments next time?**

**A:** Rolling update configuration alone is insufficient — there are several other failure modes.

**Root-cause checklist for rolling update downtime**

| Failure mode | Fix |
|---|---|
| `maxUnavailable` too high — pods removed before new ones are ready | Set `maxUnavailable: 0` |
| Readiness probe not tuned — new pods marked Ready before they can actually serve | Tighten readiness probe, add `successThreshold: 2` |
| Termination grace period too short — in-flight requests killed | Increase `terminationGracePeriodSeconds`, add preStop lifecycle hook |
| Old pods removed before connections drained | Add `preStop: sleep 5` to give load balancer time to de-register |
| Database schema migration broke old pods still serving traffic | Blue-green or expand/contract migration pattern |

**Corrected Deployment spec**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0       # never remove a pod until a new one is fully ready
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: app
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]   # allow LB to drain
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 2   # must pass twice before receiving traffic
```

**Advanced strategies**

- **Argo Rollouts with analysis**: automated canary that promotes only if error rate and p99 latency metrics pass.
- **Blue-green**: instant cutover via Service selector patch; instant rollback.
- **Feature flags**: deploy code without activating it; activate via flag. Decouple deploy from release.
- **Progressive delivery**: route 5% → 25% → 100% traffic with automated analysis gates using Flagger or Argo Rollouts.

---

**Q: You need to create a Kubernetes operator to automate complex application lifecycle events. How do you design the CRD and controller loop logic?**

**A:**

**CRD design principles**
- Model your CRD around the application’s lifecycle state machine, not infrastructure resources.
- Use `spec` for desired state (user-declared intent) and `status` for observed state (what the controller reports).
- Keep the API backwards-compatible: use `+kubebuilder:storageversion` and conversion webhooks for version migrations.

**Example CRD for a streaming pipeline**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: streampipelines.streaming.example.com
spec:
  group: streaming.example.com
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [source, sink, replicas]
              properties:
                source:
                  type: string
                sink:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: [Pending, Running, Degraded, Failed]
                readyReplicas:
                  type: integer
                lastTransitionTime:
                  type: string
  scope: Namespaced
  names:
    plural: streampipelines
    singular: streampipeline
    kind: StreamPipeline
```

**Controller reconcile loop logic**
```go
func (r *StreamPipelineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the CR
    pipeline := &streamingv1alpha1.StreamPipeline{}
    if err := r.Get(ctx, req.NamespacedName, pipeline); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Compute desired state
    desiredDeployment := r.buildDeployment(pipeline)

    // 3. Create or update child resources (Deployment, Service, ConfigMap)
    existing := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: pipeline.Name, Namespace: pipeline.Namespace}, existing)
    if errors.IsNotFound(err) {
        return ctrl.Result{}, r.Create(ctx, desiredDeployment)
    }
    existing.Spec = desiredDeployment.Spec
    if err := r.Update(ctx, existing); err != nil {
        return ctrl.Result{}, err
    }

    // 4. Update status based on observed state
    pipeline.Status.ReadyReplicas = existing.Status.ReadyReplicas
    pipeline.Status.Phase = r.computePhase(pipeline, existing)
    r.Status().Update(ctx, pipeline)

    // 5. Re-queue if not converged
    if pipeline.Status.Phase != "Running" {
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    return ctrl.Result{}, nil
}
```

**Key design rules**
- **Idempotent**: every reconcile must be safe to re-run; use `CreateOrUpdate`.
- **Ownership references**: set `controllerutil.SetControllerReference` on all child resources so garbage collection works automatically.
- **Status conditions**: use standard `metav1.Condition` types for rich status reporting.
- **Finalizers**: add a finalizer before creating external resources (e.g., S3 buckets, cloud subscriptions); remove it only after cleanup is confirmed.
- **Rate limiting**: use controller-runtime’s default work queue with exponential backoff; avoid tight reconcile loops.

Build with `kubebuilder` or `operator-sdk` to generate scaffolding, RBAC manifests, and CRD validation schemas.

---

**Q: Multiple nodes are showing high disk IO usage due to container logs. What Kubernetes features or practices can you apply to avoid this scenario?**

**A:**

**Root cause**: containers writing high-volume logs to stdout/stderr, which the container runtime writes to disk at `/var/log/pods/` and `/var/log/containers/`.

**Kubelet-level log rotation**
Configure kubelet to limit log file size and count:
```bash
# In kubelet config (/etc/kubernetes/kubelet.conf or kubelet-config ConfigMap on EKS)
containerLogMaxSize: "10Mi"      # rotate log file at 10 MB
containerLogMaxFiles: 5          # keep at most 5 rotated files per container
```
This caps disk usage at 50 MB per container instead of unbounded growth.

**Reduce log verbosity at the application level**
- Set log level to `WARN` or `ERROR` in production via environment variable; use structured logging (JSON) to make logs parseable without verbosity.
- Sample high-frequency debug logs: log 1 in 1000 repetitive events.

**Async log shipping — offload disk pressure**
- Deploy Fluent Bit as a DaemonSet to tail log files and ship to CloudWatch Logs / Loki / Elasticsearch in near real-time. Once shipped, logs can be rotated aggressively.
- Use Fluent Bit’s `tail` input plugin with `DB` (SQLite position tracking) so no log lines are lost during a restart.

**Ephemeral node IO limits via cgroups**
- Set `io.weight` via `runtimeClass` or node-level configuration to prevent a single container from monopolizing disk IO.

**Separate log volume**
- Mount a dedicated EBS volume (`/var/log`) on nodes with high-throughput IO needs (e.g., `gp3` with provisioned IOPS) to separate log IO from the root volume where the container filesystem lives.

**Sidecar log shipping (for bursty services)**
```yaml
containers:
  - name: app
    volumeMounts:
      - name: logs
        mountPath: /app/logs
  - name: log-shipper
    image: fluent/fluent-bit:2.1
    volumeMounts:
      - name: logs
        mountPath: /app/logs
volumes:
  - name: logs
    emptyDir: {}
```
The sidecar reads from a shared `emptyDir` volume, so logs don’t hit the node’s container log directory at all.

---

**Q: Your Kubernetes cluster’s etcd performance is degrading. What are the root causes and how do you ensure etcd high availability and tuning?**

**A:**

**Diagnosing degraded etcd performance**
```bash
# Check etcd health and leader election
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check latency metrics
etcdctl endpoint status --write-out=table

# Check disk latency (critical for etcd — must be <10ms for wal fsync)
etcdctl check perf
```

**Common root causes**

| Cause | Symptoms | Fix |
|---|---|---|
| Slow disk (HDD or overloaded EBS) | High `wal_fsync_duration_seconds` p99 | Move etcd to dedicated NVMe SSD or provisioned IOPS EBS (`io2`). Target <1ms fsync. |
| Large etcd database size | Slow range queries, high memory | Compact and defrag: `etcdctl compact $(etcdctl endpoint status --write-out=json \| jq ‘.[0].Status.header.revision’)` then `etcdctl defrag` |
| Too many Kubernetes objects (secrets, configmaps, events) | DB size growth | Set `--event-ttl=1h` on API server; prune old Helm releases, completed jobs. |
| Network latency between etcd members | Frequent leader re-elections | etcd members should be in the same region, <5ms RTT between peers. Use dedicated instance network interfaces. |
| noisy neighbors on same node | CPU/IO contention | Run etcd on dedicated nodes with taints; use `ionice -c 1` (real-time IO class) for the etcd process. |
| Large number of watcher connections | High CPU on leader | Limit API server watch connections; use `--max-request-bytes` to cap large list operations. |

**High availability configuration**
- Always run an **odd number** of etcd members (3 or 5) for quorum (n/2+1). 3 members = tolerate 1 failure; 5 members = tolerate 2 failures.
- Distribute members across 3 AZs; never co-locate two members on the same physical host.
- Enable etcd learner mode for adding new members without disrupting quorum.

**Ongoing maintenance**
```bash
# Automated daily compaction via cron (or enable auto-compaction in etcd config)
# etcd.yaml:
auto-compaction-retention: "1h"
auto-compaction-mode: "periodic"

# Scheduled defrag during maintenance window
etcdctl defrag --endpoints=https://etcd1:2379,https://etcd2:2379,https://etcd3:2379
```

**Backup**
```bash
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d%H%M).db
# Ship to S3 immediately; test restore monthly
```

---

**Q: You’re running 1,000+ stateful services across multi-region Kubernetes clusters. Walk us through your approach to maintaining consistency and quorum during a regional failover.**

**A:**

**Architecture foundation**
At 1,000+ stateful services, you need a clear data-sovereignty model before failover can work reliably:
- **Active-active** (e.g., Cassandra, CockroachDB, YugabyteDB): all regions accept writes; conflict resolution via vector clocks or last-write-wins. Failover is transparent — traffic reroutes, no data catch-up needed.
- **Active-passive** (e.g., PostgreSQL with streaming replication, Redis Sentinel): one primary region accepts writes; standby promotes on failure. Failover involves promotion and possible replication lag.

**Failover procedure for active-passive stateful services**

1. **Detect failure**: route health checks via Route53 health checks + CloudWatch alarms. Automated failover triggers when primary region health check fails for >60s (avoids split-brain on transient network blips).

2. **Verify replication lag before promotion**:
```bash
# PostgreSQL example — check standby lag
psql -h standby-host -c "SELECT now() - pg_last_xact_replay_timestamp() AS lag;"
# Only promote if lag < 5 seconds; otherwise wait or accept data loss window
```

3. **Promote standby**:
```bash
# For each StatefulSet: promote the standby and update the Kubernetes Service selector or DNS
kubectl patch service postgres-primary -n tenant-alpha \
  -p ‘{"spec":{"selector":{"role":"standby-promoted"}}}’
```

4. **Update global traffic routing**:
- Update Route53 weighted/failover records to point to the new primary region.
- Use ExternalDNS with TTL ≤60s on critical services to minimize propagation delay.

**Quorum maintenance during split-brain**
- Configure all stateful clusters with **odd-numbered replicas across an odd number of regions** (3 regions, 1 primary + 2 standby = majority quorum logic).
- Use `PodDisruptionBudgets` with `minAvailable: quorum_size` so Kubernetes drain operations don’t break quorum inadvertently.
- For distributed systems (Kafka, ZooKeeper, etcd): never let a maintenance event reduce the cluster below `(n/2)+1` members. Use Kafka’s `min.insync.replicas` and `acks=all` to prevent writes to a degraded cluster.

**Runbook and automation**
- Encode the failover runbook as an Argo Workflow or Tekton Pipeline so it’s executable, versioned, and auditable.
- Chaos test the failover path quarterly: terminate all pods in one region and verify the automated failover completes within the RTO target (e.g., <5 minutes).

---

**Q: A Canary deployment in EKS passes all probes but causes intermittent trade submission failures. You have <5 minutes to rollback, your RCA path?**

**A:** With <5 minutes, execute rollback and RCA in parallel — don’t wait for root cause before reverting.

**Immediate rollback (T+0 to T+2 minutes)**
```bash
# If using Argo Rollouts canary:
kubectl argo rollouts abort <rollout-name> -n trading
kubectl argo rollouts undo <rollout-name> -n trading

# If using standard Deployment with weighted Service routing:
kubectl patch service trade-api -p ‘{"spec":{"selector":{"version":"stable"}}}’ -n trading

# If using Helm:
helm rollback trade-api -n trading    # rolls to previous release
```
Confirm rollback: `kubectl rollout status` + watch error rate in Grafana drop within 30s.

**Parallel RCA (while rollback runs)**

1. **Capture canary pod logs before they’re replaced**:
```bash
kubectl logs -l version=canary -n trading --since=10m > /tmp/canary-logs.txt
kubectl logs -l version=canary -n trading --previous >> /tmp/canary-logs.txt
```

2. **Diff the canary vs stable pod spec** — look for: different env vars, new feature flags, changed ConfigMap values, new sidecar, different resource limits.

3. **Check distributed traces** (Jaeger/X-Ray): filter traces for `version=canary` pods where `http.status_code=5xx` or where trade submission spans show errors. The trace will show exactly which downstream call failed.

4. **Check the specific failure type**: "intermittent" + "trade submission" suggests:
   - Race condition or connection pool exhaustion introduced in the new code
   - Changed timeout values causing downstream calls to time out under load
   - New database query pattern causing deadlocks
   - A dependency (external exchange API, risk engine) returning different responses for the new code path

5. **Correlate with metrics**:
```promql
# Error rate by pod version
rate(http_requests_total{namespace="trading",status=~"5..",version="canary"}[1m])
  /
rate(http_requests_total{namespace="trading",version="canary"}[1m])
```

**Post-rollback RCA (full investigation)**
- Reproduce in a staging environment with production traffic replay.
- Focus on the intermittent nature: likely a concurrency bug, connection leak, or external dependency behavior difference.

---

**Q: EKS worker nodes become NotReady. What is your first action?**

**A:** Triage without causing further disruption.

```bash
# Step 1 — Scope the blast radius: how many nodes, which AZ, which node group?
kubectl get nodes -o wide | grep NotReady

# Step 2 — Check node conditions
kubectl describe node <not-ready-node> | grep -A20 Conditions

# Step 3 — Check recent events
kubectl get events --sort-by=lastTimestamp | grep <node-name>
```

**Simultaneously check AWS side**:
```bash
aws ec2 describe-instance-status --instance-ids <instance-id>
# Check: system status checks, instance status checks, scheduled events
```

**Common causes and immediate actions**

| Cause | Signal | Immediate action |
|---|---|---|
| kubelet crashed | `NotReady` + no events, node otherwise healthy | SSM into node: `systemctl restart kubelet` |
| Disk pressure | `DiskPressure=True` in conditions | Evict low-priority pods; run `docker system prune` on node |
| Network plugin failure | `NetworkUnavailable=True` | Restart CNI DaemonSet pod on the node |
| EC2 instance hardware failure | AWS instance status check failed | Terminate instance; ASG will replace it |
| Node group AMI/bootstrap issue | Multiple new nodes all NotReady simultaneously | Roll back node group launch template to previous AMI |

**Do not immediately drain or terminate** before you know the cause — if it’s a systemic issue (bad AMI rollout, CNI bug), terminating will just replace the node with the same broken version.

---

**Q: Pods are healthy but application requests fail. What will you investigate?**

**A:** Healthy pods + failing requests means the problem is in the network path between the client and the pod.

```bash
# 1. Verify Service endpoints are populated
kubectl get endpoints <service-name> -n <ns>
# Empty endpoints = pod labels don’t match Service selector

# 2. Verify pod labels match the Service selector
kubectl get pods -n <ns> --show-labels
kubectl get svc <service-name> -n <ns> -o yaml | grep selector -A5

# 3. Test connectivity from within the cluster
kubectl run nettest --image=busybox --rm -it -- \
  wget -qO- --timeout=5 http://<service-name>.<namespace>.svc.cluster.local

# 4. Test direct pod-to-pod (bypass Service)
kubectl exec -it <client-pod> -- curl http://<pod-ip>:<port>/health

# 5. Check Ingress/ALB configuration
kubectl describe ingress <ingress-name> -n <ns>
# Check ALB target group health in AWS console

# 6. Check NetworkPolicy blocking the traffic path
kubectl get networkpolicies -n <ns>

# 7. Check for application-level issues not visible in pod health
kubectl logs <pod> -n <ns> --tail=50
```

**Additional checks if above are clean**:
- **DNS resolution**: `kubectl exec` into a pod and `nslookup <service-name>` — confirm CoreDNS is resolving correctly.
- **Service port mismatch**: Service `targetPort` must match the container’s `containerPort`.
- **TLS/mTLS issues**: if Istio is installed, check `DestinationRule` and `PeerAuthentication` for misconfigured mTLS modes.
- **IAM or auth failures**: requests may be reaching the pod but failing authorization (check application logs for 401/403).

---

**Q: Pod-to-pod communication suddenly fails. What will you verify?**

**A:** Sudden failure (was working, now broken) points to a recent change: new NetworkPolicy, CNI issue, or node network configuration change.

```bash
# 1. Identify if it’s all pods or specific pods/namespaces
kubectl exec -it <pod-a> -n ns-a -- curl http://<pod-b-ip>:8080
kubectl exec -it <pod-a> -n ns-a -- curl http://<pod-b-ip>:8080 -v

# 2. Check for recent NetworkPolicy changes
kubectl get networkpolicies -A
kubectl describe networkpolicy <policy> -n <ns>
# Was a new policy deployed? Check git history.

# 3. Check CNI pod health (Cilium/Calico/VPC CNI)
kubectl get pods -n kube-system | grep -E "cilium|calico|aws-node"
kubectl logs -n kube-system <cni-pod> --tail=50

# 4. Check if the issue is AZ-specific (cross-AZ routing issue)
kubectl get pod <pod-a> <pod-b> -o wide   # check which nodes/AZs they’re on

# 5. For AWS VPC CNI: check ENI and IP assignment
kubectl describe node <node> | grep -A20 "Allocated resources"
kubectl logs -n kube-system aws-node-<suffix> | grep -i error
```

**Cilium-specific diagnosis**
```bash
# Check Cilium endpoint status
kubectl exec -n kube-system <cilium-pod> -- cilium endpoint list
kubectl exec -n kube-system <cilium-pod> -- cilium policy trace \
  --src-k8s-pod ns/pod-a --dst-k8s-pod ns/pod-b --dport 8080
```

**Common causes**:
- New `NetworkPolicy` with a typo in `podSelector` that accidentally blocks traffic.
- CNI DaemonSet pod crash on a specific node — pods on that node lose networking.
- AWS VPC CNI IP exhaustion: subnet ran out of IPs, new pods get no ENI assignment.
- Node iptables rules corrupted — restart kube-proxy or CNI agent.

---

**Q: HPA scales pods but latency still increases. What could be wrong?**

**A:** HPA adding pods but latency still rising is a classic "scaling the wrong thing" problem.

**Diagnose where latency is actually incurred**
```bash
# Check if new pods are actually receiving traffic
kubectl get endpoints <service> -n <ns>    # are new pod IPs listed?
kubectl top pods -n <ns>                   # are new pods actually loaded?
```

**Common causes**

| Cause | Description | Fix |
|---|---|---|
| **Downstream bottleneck** | A shared dependency (DB, cache, external API) is saturated; adding more pods just creates more load on the bottleneck | Scale the downstream service; add caching; implement circuit breaker |
| **New pods not ready** | HPA added pods but they’re in readiness probe failure — endpoints not populated, so existing pods still take all traffic | Fix readiness probe; check startup time vs probe `initialDelaySeconds` |
| **Connection pool exhaustion** | More pods → more connections to DB; DB max connections hit | Use PgBouncer/RDS Proxy; increase connection pool size |
| **HPA scaling on wrong metric** | Scaling on CPU but the bottleneck is memory, goroutine count, or I/O wait | Add custom metrics via Prometheus Adapter; scale on the correct leading indicator |
| **Slow bin-packing** | New pods scheduled on already-busy nodes due to poor anti-affinity | Add `topologySpreadConstraints` to spread pods across nodes |
| **JVM/GC warm-up** | New Java pods taking 2-3 minutes to reach full throughput during JIT compilation | Implement startup probe + warm-up endpoint; pre-warm via `startupProbe` and traffic shadowing |
| **Service mesh sidecar not scaled** | Envoy proxy is CPU-constrained and throttling connections | Set proper resource limits on the Istio sidecar container |

**Investigation query**
```promql
# Is the latency increase in this service or downstream?
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="myapp"}[1m]))
# vs
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="downstream-db"}[1m]))
```

---

**Q: Why do you use EKS instead of ECS?**

**A:** The choice depends on context, but for complex production workloads, EKS offers significant advantages:

**Where EKS wins over ECS**

| Dimension | EKS | ECS |
|---|---|---|
| **Ecosystem** | Vast CNCF ecosystem: Helm, Argo, Istio, Kyverno, Flux, Cilium, Prometheus, Grafana | AWS-native tooling only; limited third-party integrations |
| **Portability** | Standard Kubernetes API; workloads can run on GKE, AKS, or on-prem with minimal changes | AWS-proprietary; migration to another cloud requires rewrite |
| **Advanced scheduling** | Affinity, anti-affinity, taints/tolerations, topology spread, priority classes, custom schedulers | Basic placement constraints |
| **Custom resources** | CRDs + operators for any stateful application lifecycle automation | No equivalent extensibility |
| **Multi-tenancy** | Namespaces, RBAC, NetworkPolicy, ResourceQuota — fine-grained isolation | Task definitions are flat; namespace-level isolation requires separate clusters |
| **GitOps** | ArgoCD/Flux work natively with Kubernetes manifests | Requires custom integration |
| **Service mesh** | Istio, Linkerd, Cilium Mesh — L7 observability, mTLS, traffic shaping | AWS App Mesh (less feature-rich, being deprecated) |

**Where ECS is the right choice**
- Simple containerized workloads with no need for custom scheduling or ecosystem tools.
- Teams without Kubernetes expertise; ECS is operationally simpler.
- AWS Fargate workloads where you want zero node management and the application is straightforward.
- Very small teams where the Kubernetes operational overhead isn’t justified.

**In practice**: EKS is the default choice for platforms with 10+ microservices, multi-team organizations, or any workload that benefits from the CNCF ecosystem. ECS is preferable for small, AWS-only teams running simple workloads where Kubernetes complexity is a net negative.



