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
