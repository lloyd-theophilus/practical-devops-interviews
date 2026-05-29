# Jenkins Interview Questions & Answers

---

**Q: How to trigger a Jenkins job when code is pushed to GitHub?**

**A:**
1. **Webhook** (recommended): In GitHub repo → Settings → Webhooks → Add webhook. Set the Payload URL to `https://<jenkins-url>/github-webhook/`, content type `application/json`. In the Jenkins job, check "GitHub hook trigger for GITScm polling" under Build Triggers.
2. **Multibranch Pipeline**: Jenkins automatically creates jobs for each branch/PR and configures the webhook via the GitHub Branch Source plugin.
3. **SCM Polling** (fallback): `H/5 * * * *` polls GitHub every 5 minutes — less efficient and introduces latency.

Ensure the Jenkins server is reachable from GitHub's webhook IP ranges, or use a GitHub App integration for better security (HMAC validation, fine-grained token scopes).

---

**Q: How do you handle secrets in Jenkins pipelines?**

**A:**
- **Jenkins Credentials Store**: Store secrets as `Username/Password`, `Secret Text`, or `Secret File` credential types. Reference in pipeline:
  ```groovy
  withCredentials([string(credentialsId: 'my-token', variable: 'TOKEN')]) {
      sh 'curl -H "Authorization: Bearer $TOKEN" ...'
  }
  ```
- **HashiCorp Vault Plugin**: Fetch secrets dynamically from Vault at runtime; secrets are never stored in Jenkins.
- **AWS Secrets Manager**: Use IAM role attached to the Jenkins agent; fetch secrets via AWS CLI or SDK in pipeline steps.
- **Best practices**: Mask secrets in logs, avoid `echo $SECRET`, don't pass secrets as build parameters (they appear in the build history), use short-lived credentials.

---

**Q: Explain the CI/CD workflow you follow and the kind of pipeline you use. How do you define and invoke pipelines in Jenkins?**

**A:** I use **Declarative Pipelines** stored as `Jenkinsfile` in the application repository (pipeline-as-code). The Multibranch Pipeline project type scans the repo and automatically creates pipeline jobs per branch.

**Typical workflow**:
```groovy
pipeline {
  agent { label 'docker-agent' }
  stages {
    stage('Build') { steps { sh 'mvn package -DskipTests' } }
    stage('Test')  { steps { sh 'mvn test' } }
    stage('SonarQube') {
      steps {
        withSonarQubeEnv('SonarQube') { sh 'mvn sonar:sonar' }
      }
    }
    stage('Docker Build & Push') {
      steps {
        sh 'docker build -t myregistry/myapp:${GIT_COMMIT} .'
        sh 'docker push myregistry/myapp:${GIT_COMMIT}'
      }
    }
    stage('Deploy') { steps { sh 'helm upgrade --install myapp ./chart --set image.tag=${GIT_COMMIT}' } }
  }
  post {
    failure { slackSend channel: '#alerts', message: "Build failed: ${env.BUILD_URL}" }
  }
}
```

---

**Q: What are shared libraries in Jenkins, and how are they written and defined?**

**A:** Shared Libraries allow you to extract reusable pipeline logic into a separate Git repository that multiple `Jenkinsfile`s can import. This avoids copy-pasting pipeline code across repos.

**Structure**:
```
jenkins-shared-library/
├── vars/
│   ├── buildDocker.groovy      # global variables / helper functions
│   └── deployHelm.groovy
├── src/
│   └── org/example/
│       └── GitUtils.groovy     # Groovy classes
└── resources/
    └── scripts/setup.sh        # static resources
```

**Defining**: In Jenkins → Manage Jenkins → Configure System → Global Pipeline Libraries. Set the Git repo URL, default version (branch/tag), and a library name.

**Using in Jenkinsfile**:
```groovy
@Library('my-shared-lib@main') _
buildDocker image: 'myapp', tag: env.GIT_COMMIT
deployHelm releaseName: 'myapp', chart: './chart'
```

---

**Q: What kind of applications do you deploy using Jenkins pipelines, and what deployment tools do you use?**

**A:** Jenkins pipelines I've built deployed:
- **Microservices** (Java/Spring Boot, Node.js, Python/FastAPI) to Kubernetes via `helm upgrade --install`.
- **Frontend SPAs** (React, Vue) to S3 + CloudFront with cache invalidation.
- **Lambda functions** using AWS SAM CLI (`sam deploy`) or Serverless Framework.
- **Terraform infrastructure** using `terraform plan` + manual approval gate + `terraform apply`.

Deployment tools used: Helm, kubectl, AWS CLI, SAM CLI, ArgoCD (triggered via `argocd app sync`), Ansible for VM-based deployments.

---

**Q: If the Jenkins pipeline runs but the build doesn't happen, what possible issues could be causing it?**

**A:**
- **No agent/executor available**: all agents are busy or offline; check "Build Executor Status."
- **Agent label mismatch**: `agent { label 'docker' }` but no agent has that label configured.
- **Workspace locked**: another build holds a workspace lock.
- **SCM checkout failure**: invalid credentials, network issue reaching the Git server, or wrong branch specified.
- **Pipeline syntax error**: Declarative syntax error causes the pipeline to skip execution silently — check the pipeline editor or `jenkinsfile-runner` lint.
- **Plugin issue**: missing plugin or version incompatibility (e.g., Docker plugin not installed but pipeline calls `docker.build()`).
- **Docker not available on agent**: the agent lacks Docker daemon access; check socket mount or DinD setup.
- **Environment variable not set**: a required env var is missing, causing an early exit.

---

**Q: How do you use Jenkins shared libraries? Explain their typical structure and how they are integrated into your Jenkinsfiles.**

**A:** (See "What are shared libraries" answer above for structure.)

**Integration pattern**:
```groovy
// Jenkinsfile
@Library('platform-lib@v1.2.0') _   // pin to a tag for stability

pipeline {
  agent any
  stages {
    stage('Build & Push') {
      steps {
        script {
          buildAndPushImage(
            imageName: 'myapp',
            registry: 'my.registry.io',
            tag: env.GIT_COMMIT[0..7]
          )
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          helmDeploy(
            releaseName: 'myapp',
            namespace: 'production',
            valuesFile: 'values-prod.yaml'
          )
        }
      }
    }
  }
}
```

The shared library abstracts Docker credentials, registry login, and Helm commands behind well-named functions, so application teams only declare *what* to deploy, not *how*.

---

**Q: Why choose Declarative Pipeline over Scripted Pipeline?**

**A:**
- **Readability**: Structured syntax (`pipeline {}`, `stages {}`, `post {}`) is easier for teams to read and review.
- **Validation**: Jenkins validates Declarative syntax at parse time and provides a built-in linter.
- **Built-in features**: `post {}` conditions (always, success, failure, unstable), `options {}`, `parameters {}`, `triggers {}` are first-class citizens — no boilerplate.
- **Restart from stage**: Declarative supports resuming a failed pipeline from a specific stage.
- **Parallel stages**: clean `parallel {}` block syntax.
- **When to use Scripted**: complex dynamic stage generation, conditional logic not expressible in Declarative, or when advanced Groovy is needed. Even then, encapsulate the complexity in a Shared Library function called from a Declarative pipeline.

---

**Q: How do you integrate SonarQube into a Jenkins pipeline?**

**A:**
1. Install the SonarQube Scanner plugin in Jenkins.
2. Add SonarQube server in Jenkins → Manage Jenkins → Configure System → SonarQube servers (URL + authentication token).
3. In the pipeline:
```groovy
stage('SonarQube Analysis') {
  steps {
    withSonarQubeEnv('MySonarQube') {
      sh 'mvn sonar:sonar -Dsonar.projectKey=myapp'
      // or: sonar-scanner for non-Maven projects
    }
  }
}
stage('Quality Gate') {
  steps {
    timeout(time: 5, unit: 'MINUTES') {
      waitForQualityGate abortPipeline: true
    }
  }
}
```
The `waitForQualityGate` step polls SonarQube via webhook until the analysis result is available, then fails the pipeline if the quality gate is not passed.

---

**Q: How do you build → containerize → push → deploy using Jenkins?**

**A:**
```groovy
pipeline {
  agent { label 'docker' }
  environment {
    REGISTRY = 'my-account.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE    = "${REGISTRY}/myapp"
    TAG      = "${GIT_COMMIT[0..7]}"
  }
  stages {
    stage('Build')       { steps { sh 'mvn package -DskipTests' } }
    stage('Containerize') {
      steps {
        sh "docker build -t ${IMAGE}:${TAG} ."
      }
    }
    stage('Push') {
      steps {
        withAWS(credentials: 'aws-ecr-creds', region: 'us-east-1') {
          sh "aws ecr get-login-password | docker login --username AWS --password-stdin ${REGISTRY}"
          sh "docker push ${IMAGE}:${TAG}"
        }
      }
    }
    stage('Deploy') {
      steps {
        sh "helm upgrade --install myapp ./chart --set image.tag=${TAG} -f values-prod.yaml --wait"
      }
    }
  }
  post {
    failure { sh "helm rollback myapp || true" }
  }
}
```

**Q: Jenkins build succeeds but deployment fails. What will you verify?**

**A:**

1. **Helm/kubectl error in pipeline output**: read the error message — is it a connection error (kubeconfig), auth error (RBAC), resource conflict (existing resource not managed by Helm), or a timeout waiting for rollout?

2. **kubeconfig / cluster access**:
   ```bash
   # On the Jenkins agent
   kubectl cluster-info    # can the agent reach the cluster?
   kubectl auth can-i create deployment -n production   # RBAC check
   ```
   If the agent uses an IAM role (EKS), verify the role is in the `aws-auth` ConfigMap or EKS access entries.

3. **Image pull failure**: the new image was pushed to ECR, but the Kubernetes node can't pull it. Check: ECR repo policy, node IAM role has `ecr:GetAuthorizationToken` and `ecr:BatchGetImage`, image tag exists in ECR.

4. **Helm release in a failed state**: a previous broken deploy left the Helm release in `failed` state — `helm upgrade` may not proceed.
   ```bash
   helm status myapp -n production
   helm history myapp -n production
   # Fix: helm rollback myapp 1 -n production  (roll back to last good revision)
   ```

5. **Resource quota exceeded**: the new deployment requests more CPU/memory than the namespace quota allows. Check: `kubectl describe namespace production | grep -A5 "Resource Quotas"`.

6. **Readiness probe failure**: Helm's `--wait` flag waits for pods to become Ready. If the new pods fail readiness probes, Helm times out and returns an error even though the pods are running. Check: `kubectl describe pod <new-pod>`.

7. **ConfigMap or Secret missing**: the new deployment references a ConfigMap/Secret that doesn't exist in the target namespace yet.

---

**Q: Deployment succeeds but users still see the old application version. Why?**

**A:** A successful deployment means the new pods are running and healthy — but the user still reaching old code means something is serving cached or stale content.

**Most common causes**:

1. **CDN / CloudFront cache**: the CDN is serving the previous version from edge caches. Fix:
   ```bash
   aws cloudfront create-invalidation \
     --distribution-id E1234ABCD \
     --paths "/*"
   ```
   Add cache invalidation as a step in your pipeline after deployment.

2. **Browser cache**: the user's browser has cached the old HTML/JS. Force a new cache-busting URL by versioning static assets (`app.v2.js`) or setting `Cache-Control: no-cache` on HTML responses.

3. **Kubernetes service still routing to old pods**: the rolling update didn't complete — some old pods are still running.
   ```bash
   kubectl rollout status deployment/myapp -n production
   kubectl get pods -n production -l app=myapp  # check all pods show the new image
   ```

4. **Sticky sessions**: the load balancer or Ingress has session affinity enabled — some users are pinned to specific (old) pods. Disable stickiness or wait for sessions to expire.

5. **Multiple replicas, deployment partially rolled out**: with `maxUnavailable: 25%`, some old pods are still serving traffic. Wait for `rollout status` to show "successfully rolled out".

6. **Wrong namespace deployed to**: the pipeline deployed to `staging` instead of `production`. Verify: `kubectl get deployment myapp -n production -o jsonpath='{.spec.template.spec.containers[0].image}'`.

7. **DNS TTL caching**: if a DNS change was also involved, clients cache the old DNS record. Check TTL and wait.

---

**Q: How do you configure automatic Jenkins triggers from GitHub pushes?**

**A:** The recommended approach is a **GitHub webhook** pointing to Jenkins:

**Step 1 — Configure Jenkins**:
- Install the **GitHub plugin** and **GitHub Branch Source plugin**.
- In the Jenkins job (Freestyle or Pipeline): Build Triggers → check "GitHub hook trigger for GITScm polling".
- For Multibranch Pipelines: the webhook is configured automatically once the GitHub org/repo is scanned.

**Step 2 — Configure the GitHub webhook**:
1. Go to the GitHub repo → Settings → Webhooks → Add webhook.
2. **Payload URL**: `https://<jenkins-url>/github-webhook/`
3. **Content type**: `application/json`
4. **Secret**: set an HMAC secret; configure it in Jenkins Credentials for validation.
5. **Events**: "Just the push event" (or add "Pull requests" for PR builds).

**Step 3 — Network access**:
- Jenkins must be reachable from GitHub's IP ranges. Check [GitHub's meta API](https://api.github.com/meta) for the `hooks` CIDR list.
- If Jenkins is behind a firewall, use **GitHub App** authentication with a self-hosted runner, or expose Jenkins via a reverse proxy/ngrok for testing.

**GitHub App approach (preferred for organisations)**:
- Register a GitHub App, install it on the repo/org.
- The Jenkins GitHub App plugin handles authentication with short-lived tokens and validates webhook signatures automatically.
- More secure than PATs: scoped permissions, automatic rotation, audit trail per app.

**Verify it's working**:
```bash
# Push a commit and check Recent Deliveries in GitHub webhook settings
# Or trigger a test delivery from the webhook settings page
```

---

**Q: Explain the deployment strategies used in your projects.**

**A:**

| Strategy | How it works | Downtime | Rollback speed | Use case |
|---|---|---|---|---|
| **Recreate** | Stop all old pods, start all new pods | Yes (brief) | Slow (redeploy old) | Dev environments, stateful apps that can't run two versions |
| **Rolling Update** | Replace pods gradually (default in Kubernetes) | Zero (if probes are correct) | Fast (`kubectl rollout undo`) | Standard stateless services |
| **Blue/Green** | Deploy new version alongside old; switch traffic all at once | Zero | Instant (flip selector/DNS) | High-traffic production, critical releases |
| **Canary** | Route a small % of traffic (e.g., 5%) to the new version; expand gradually | Zero | Fast (shift 0% to new) | Risk reduction for major changes |
| **Shadow** | New version receives a copy of real traffic but responses are discarded | Zero | N/A (no live traffic) | Pre-production validation, ML model testing |

**How I use them**:

- **Kubernetes Deployments**: rolling update is the default. I set `maxUnavailable: 0` and `maxSurge: 1` for zero-downtime with one extra pod during the rollout.

- **Blue/Green via Argo Rollouts**:
  ```yaml
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false   # require manual promotion
  ```

- **Canary via Argo Rollouts + Istio**:
  ```yaml
  strategy:
    canary:
      steps:
        - setWeight: 10    # 10% to canary
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 100
  ```

- **For Lambda**: I use **weighted aliases** — point the `prod` alias at 90% old version, 10% new version, then shift to 100% once metrics confirm stability.

---

