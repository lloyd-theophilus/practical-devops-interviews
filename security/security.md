# Security Interview Questions & Answers

---

**Q: Are you aware of security scanning tools? How do you scan Docker images — both during build and at the registry level? Are you using any extensions or tools for image scanning?**

**A:**

**Security scanning tools I use**:
- **Trivy** (Aqua Security): fast, comprehensive — scans OS packages, language dependencies, Dockerfiles, Kubernetes manifests, and IaC for CVEs, misconfigurations, and secrets. My primary tool.
- **Grype** (Anchore): vulnerability scanner for container images and filesystems; integrates well with CI pipelines.
- **Snyk**: developer-friendly SCA + container scanning with fix advice; integrates with GitHub, GitLab, Jenkins.
- **OWASP Dependency-Check**: for SCA on Java/Node/Python application dependencies.
- **Checkov**: IaC scanning (Terraform, CloudFormation, Helm, Kubernetes manifests) for security misconfigurations.
- **Falco**: runtime security — detects anomalous behavior inside running containers.

**During build (CI gate)**:
```bash
# In Jenkinsfile / GitHub Actions
- name: Scan image with Trivy
  run: |
    trivy image \
      --exit-code 1 \
      --severity HIGH,CRITICAL \
      --ignore-unfixed \
      myapp:${IMAGE_TAG}
```
This blocks the pipeline if any unfixed HIGH or CRITICAL CVE is found.

**At the registry level**:
- **AWS ECR Enhanced Scanning**: integrates with Amazon Inspector v2; scans on push and continuously rescans as new CVEs are published. Configure in ECR → Scanning configuration → Enhanced scanning.
- **Harbor**: open-source registry with built-in Trivy/Clair scanning; can enforce policy (block pull of images with critical CVEs).
- **Docker Hub**: has built-in scanning for paid plans.
- **Artifact Registry (GCP)**: integrates with Container Analysis for vulnerability scanning.

**IDE / pre-commit extensions**:
- **Snyk VS Code extension**: highlights vulnerable dependencies and Dockerfile issues inline.
- **Trivy VS Code extension**: scans the Dockerfile and manifests in the editor.
- **Hadolint**: Dockerfile linter that enforces best practices (ADD vs COPY, `latest` tags, running as root).

**My pipeline approach** (defense in depth):
1. `hadolint Dockerfile` — lint during development.
2. `trivy image` — scan in CI before push; fail on HIGH/CRITICAL unfixed.
3. ECR Enhanced Scanning — continuous scanning after push; alert on newly discovered CVEs.
4. Falco DaemonSet — runtime anomaly detection in the cluster.
5. Periodic re-scan of all images in the registry (Trivy operator or ECR scheduled scans).
