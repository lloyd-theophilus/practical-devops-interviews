# CI/CD Interview Questions & Answers

---

**Q: What happens internally in a CI/CD pipeline from commit → deploy?**

**A:** When a developer pushes a commit, a webhook triggers the CI server (Jenkins, GitHub Actions, GitLab CI, etc.). The pipeline then:
1. **Source checkout** — clones the repo at the commit SHA.
2. **Build** — compiles code, installs dependencies.
3. **Test** — runs unit, integration, and sometimes contract tests.
4. **Static analysis / SAST** — linting, security scanning (Trivy, Snyk, SonarQube).
5. **Artifact creation** — builds a Docker image or binary and tags it with the commit SHA or semantic version.
6. **Push to registry** — image pushed to ECR/GCR/Artifactory.
7. **Deploy** — updates manifests (Helm values, Kustomize overlay) and applies to the target environment (kubectl apply, Helm upgrade, ArgoCD sync).
8. **Post-deploy validation** — smoke tests, health checks, rollback trigger if probes fail.

---

**Q: How does a pipeline handle parallel jobs and dependencies?**

**A:** Most CI systems model the pipeline as a DAG (Directed Acyclic Graph). Jobs with no dependency on each other run in parallel across agents/runners. Dependencies are declared explicitly (e.g., `needs:` in GitHub Actions, `dependsOn:` in Azure Pipelines, `parallel {}` blocks in Jenkins Declarative). The scheduler waits for all upstream jobs to succeed before starting downstream jobs. Failed upstream jobs block dependents and can optionally trigger notifications or rollback stages.

---

**Q: What are the most common production mistakes in DevOps setups?**

**A:**
- Deploying without automated rollback or health-check gates.
- Storing secrets in environment variables or source control instead of a vault.
- Skipping canary/blue-green deploys on high-traffic services.
- Running CI as root or with overly permissive IAM roles.
- No artifact pinning — pulling `latest` tags can introduce unexpected changes.
- Missing post-deploy smoke tests so broken releases reach users.
- Neglecting pipeline-as-code — manual job configuration leads to configuration drift.

---

**Q: Explain a typical CI/CD pipeline you've worked on.**

**A:** A typical pipeline I've worked on:
1. **Trigger**: Push to a feature branch or PR.
2. **CI stage**: Checkout → `docker build` (multi-stage) → unit tests → SonarQube scan → image push to ECR tagged with `<branch>-<short-sha>`.
3. **Dev deploy**: Helm upgrade to the dev namespace; Argo Rollouts handles canary.
4. **Integration tests**: Run Postman/Newman tests against the dev endpoint.
5. **Promotion gate**: Manual approval or automated quality gate (code coverage > 80%, zero critical vulnerabilities).
6. **Staging deploy**: Same Helm chart with staging `values.yaml`; Selenium smoke tests run.
7. **Prod deploy**: Blue-green via ALB target group switching; CloudWatch alarms monitored for 10 minutes; auto-rollback if error rate > 1%.

---

**Q: Difference between Declarative and Scripted pipeline?**

**A:**

| | Declarative | Scripted |
|---|---|---|
| Syntax | Structured, opinionated (`pipeline {}` block) | Groovy DSL, free-form |
| Error handling | Built-in `post {}` sections | Manual `try/catch` |
| Learning curve | Lower | Higher |
| Flexibility | Less | Full Groovy power |
| Linting | Supported by Jenkins linter | Limited |

Declarative is preferred for most teams due to readability and built-in validation. Scripted is used when complex conditional logic or dynamic stage generation is needed.

---

**Q: How to pass parameters between stages?**

**A:** In Jenkins Declarative:
- Use `environment {}` block to set variables available to all stages.
- Write to a file and `stash`/`unstash` across stages.
- Use `script {}` blocks to set `env.MY_VAR` dynamically.

```groovy
stage('Build') {
  steps {
    script {
      env.IMAGE_TAG = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
    }
  }
}
stage('Deploy') {
  steps {
    sh "helm upgrade myapp ./chart --set image.tag=${env.IMAGE_TAG}"
  }
}
```

In GitHub Actions, use `outputs` at the job level and `needs.<job>.outputs.<name>` to consume them downstream.

---

**Q: What is the purpose of a webhook, and how is it used in a CI/CD pipeline?**

**A:** A webhook is an HTTP POST callback sent by a source system (GitHub, GitLab, Bitbucket) to a target URL when an event occurs (push, PR open, tag creation). In CI/CD:
1. The SCM sends a JSON payload to the CI server's webhook endpoint.
2. The CI server validates the payload (HMAC signature check using a shared secret).
3. It extracts the branch, commit SHA, and event type.
4. It triggers the appropriate pipeline job with those parameters.

Webhooks make pipelines event-driven and near-real-time, eliminating the need for polling.

---

**Q: Describe your typical deployment flow and CI/CD workflow. What stages do you define in your Jenkins pipeline, and how do you ensure full quality checks during deployment?**

**A:** My Jenkins Declarative pipeline stages:
1. **Checkout** — SCM checkout with shallow clone.
2. **Build & Unit Test** — parallel execution of build and test to save time.
3. **Code Quality** — SonarQube scan; pipeline fails if quality gate is not passed.
4. **Security Scan** — Trivy scans the Docker image for HIGH/CRITICAL CVEs.
5. **Publish** — Docker image pushed to ECR; Helm chart pushed to ChartMuseum.
6. **Deploy to Dev** — `helm upgrade --install` with dev values; wait for rollout.
7. **Integration Tests** — automated API tests using Newman.
8. **Deploy to Staging** — same flow; requires manual approval for prod.
9. **Deploy to Prod** — blue-green or canary; Datadog/CloudWatch monitors SLOs.
10. **Post-deploy** — smoke test job; if it fails, automated rollback via `helm rollback`.

Quality is enforced through gates at each stage: test coverage thresholds, SonarQube quality gates, and image scanning policies that block deployment on critical vulnerabilities.

---

**Q: Describe your experience with CI/CD pipelines.**

**A:** I have designed and maintained CI/CD pipelines using Jenkins (Declarative + Shared Libraries), GitHub Actions, and GitLab CI. Key experiences include:
- Migrating from freestyle Jenkins jobs to pipeline-as-code stored in the application repository.
- Building reusable Jenkins Shared Libraries that standardized build, scan, and deploy steps across 20+ microservices.
- Implementing GitOps with ArgoCD where the pipeline's only job is to push updated image tags to a Git-ops repo; ArgoCD handles reconciliation.
- Integrating security scanning (Trivy, OWASP Dependency-Check) as a mandatory gate.
- Setting up matrix builds in GitHub Actions for multi-architecture Docker images (amd64, arm64).
- Reducing average pipeline duration by 40% through Docker layer caching and parallel test execution.
