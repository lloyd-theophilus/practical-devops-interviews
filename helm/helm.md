# Helm Interview Questions & Answers

---

**Q: How do you perform zero-downtime upgrades for a stateful workload using Helm 3?**

**A:** Zero-downtime upgrades for stateful workloads (StatefulSets) require careful coordination:

1. **Use `RollingUpdate` strategy with `maxUnavailable: 0`** and `partition` in the StatefulSet update strategy to do canary-style rolling — only update pods above the partition index first.
2. **Pre-upgrade hooks** (`helm.sh/hook: pre-upgrade`) run Jobs to quiesce the application, flush queues, or take a snapshot before the upgrade starts.
3. **PodDisruptionBudgets**: ensure a PDB prevents Kubernetes from removing too many replicas simultaneously.
4. **Readiness probes**: gate traffic cutover on the new pod being truly ready (not just running).
5. **`--atomic` flag**: `helm upgrade --atomic --timeout 5m` automatically rolls back if any resource fails to become ready within the timeout.
6. For databases, use init containers or a migration Job (with `helm.sh/hook: pre-upgrade`) to run schema migrations before pods restart.

---

**Q: Explain the folder structure of a basic Helm chart. What commands do you use to deploy with Helm?**

**A:**
```
mychart/
├── Chart.yaml          # chart metadata (name, version, appVersion)
├── values.yaml         # default configuration values
├── charts/             # chart dependencies (sub-charts)
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # named templates / helper functions
│   ├── NOTES.txt       # post-install usage notes
│   └── tests/
│       └── test-connection.yaml
└── .helmignore         # files to exclude from packaging
```

**Key commands**:
```bash
helm install <release> ./mychart -f values.yaml           # first install
helm upgrade <release> ./mychart -f values.yaml           # upgrade existing
helm upgrade --install <release> ./mychart                 # idempotent
helm uninstall <release>                                   # delete release
helm list -A                                               # list all releases
helm rollback <release> <revision>                        # rollback
helm template ./mychart -f values.yaml                    # render locally
helm lint ./mychart                                        # validate chart
```

---

**Q: What is email signing and Helm chart signing? Which tools do you use to sign Helm charts?**

**A:** **Helm chart signing** ensures the integrity and authenticity of a chart package — consumers can verify the chart hasn't been tampered with and came from a trusted source.

**How it works**:
1. Chart is packaged: `helm package mychart/`
2. A `.prov` (provenance) file is generated alongside the `.tgz`: `helm package --sign --key 'My Key' --keyring ~/.gnupg/secring.gpg mychart/`
3. The provenance file contains a SHA256 digest of the chart archive and a PGP signature.
4. Consumers verify: `helm verify mychart-1.0.0.tgz` or `helm install --verify`.

**Tools**:
- **GnuPG (GPG)**: standard for key management and signing.
- **Cosign (Sigstore)**: keyless signing using OIDC tokens; increasingly adopted for OCI-based Helm charts.
- **Notation**: CNCF project for signing OCI artifacts including Helm charts stored in OCI registries.

---

**Q: What are values.yaml and how do you override them?**

**A:** `values.yaml` is the default configuration file for a Helm chart. Templates reference values via `{{ .Values.<key> }}`. It provides sensible defaults that can be overridden at deploy time.

**Override methods** (in order of increasing precedence):
1. Default `values.yaml` in the chart.
2. Parent chart's `values.yaml` (for sub-charts).
3. `-f custom-values.yaml` flag (can be specified multiple times).
4. `--set key=value` on the CLI (highest precedence).
5. `--set-string`, `--set-file`, `--set-json` for type-specific overrides.

```bash
helm upgrade myapp ./chart \
  -f values-prod.yaml \
  --set image.tag=v2.0.1 \
  --set replicaCount=5
```

---

**Q: How do you manage multiple environment deployments using Helm?**

**A:** Common patterns:
1. **Per-environment values files**: `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`. Deploy with `helm upgrade -f values-<env>.yaml`.
2. **Helmfile**: orchestrates multiple releases and environments; each environment has its own state file.
3. **ArgoCD + ApplicationSets**: GitOps approach where each environment maps to a Git path or branch; Helm values are per-environment overlays.
4. **Helm chart per environment**: separate releases in separate namespaces (`myapp-dev`, `myapp-prod`) using the same chart.

Separate sensitive overrides (credentials, endpoints) into files managed by Helm Secrets (backed by SOPS + AWS KMS or GCP KMS).

---

**Q: How do you debug a failed Helm release?**

**A:**
```bash
# See release status and failure reason
helm status <release>
helm history <release>

# Get rendered manifests that were applied
helm get manifest <release>
helm get values <release>

# Describe failed resources
kubectl describe pod -l app=<release> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Render templates locally without installing
helm template <release> ./chart -f values.yaml | kubectl apply --dry-run=client -f -

# Enable debug output
helm upgrade <release> ./chart --debug --dry-run

# Check hooks
kubectl get jobs -n <namespace>
kubectl logs job/<hook-job-name>
```

---

**Q: What is the difference between Helm Chart, Release, and Repository?**

**A:**

| Concept | Definition |
|---|---|
| **Chart** | The package itself — a directory (or `.tgz`) containing templates, `values.yaml`, and `Chart.yaml`. Analogous to a Docker image. |
| **Release** | A deployed instance of a chart in a Kubernetes cluster. Running `helm install` creates a release. Multiple releases of the same chart can coexist (e.g., `myapp-dev` and `myapp-prod`). Analogous to a running container. |
| **Repository** | A web server hosting an `index.yaml` that lists available charts and their download URLs. Examples: ArtifactHub, ChartMuseum, ECR (OCI). Added with `helm repo add`. |

A release tracks revision history (each `helm upgrade` increments the revision), enabling `helm rollback` to any previous revision. Release metadata is stored as Kubernetes Secrets in the release namespace.
