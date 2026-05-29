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

**Q: Explain how SonarQube is used in your CI/CD pipeline.**

**A:** SonarQube provides static code analysis — it scans source code for bugs, vulnerabilities, code smells, duplications, and test coverage gaps, then enforces a **Quality Gate** that can block a merge or deployment.

**Setup**:
1. Deploy SonarQube server (self-hosted or SonarCloud for SaaS).
2. Create a project in SonarQube, generate an authentication token.
3. Store the token as a Jenkins credential or GitHub Actions secret.
4. Add the SonarQube Scanner to the CI pipeline.

**Jenkins integration**:
```groovy
stage('SonarQube Analysis') {
  steps {
    withSonarQubeEnv('SonarQube') {   // references Jenkins > Configure System > SonarQube server
      sh '''
        sonar-scanner \
          -Dsonar.projectKey=myapp \
          -Dsonar.sources=src \
          -Dsonar.tests=tests \
          -Dsonar.coverage.exclusions=**/*test* \
          -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
      '''
    }
  }
}
stage('Quality Gate') {
  steps {
    timeout(time: 5, unit: 'MINUTES') {
      waitForQualityGate abortPipeline: true   // fails the build if gate not passed
    }
  }
}
```

**GitHub Actions integration**:
```yaml
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
  with:
    args: >
      -Dsonar.projectKey=myapp
      -Dsonar.sources=src
```

**Quality Gate**: SonarQube evaluates conditions on **new code** (the diff since the last analysis):
- New bugs: 0
- New vulnerabilities: 0
- New code coverage: ≥ 80%
- New code duplication: < 3%

If the gate fails, `waitForQualityGate abortPipeline: true` stops the pipeline — the code cannot merge or deploy.

**PR decoration**: SonarQube posts inline comments on GitHub/GitLab PRs showing exactly which lines introduced issues, so developers fix them before merge rather than after.

**Best practices**:
- Analyze on every PR, not just main — shift-left catches issues early.
- Use `sonar.pullrequest.key`, `sonar.pullrequest.branch`, `sonar.pullrequest.base` for PR-specific analysis.
- Exclude generated code, vendored dependencies, and test fixtures from analysis with `sonar.exclusions`.

---

