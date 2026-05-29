# Git Interview Questions & Answers

---

**Q: How does Git merge and rebase differ internally?**

**A:**
- **Merge**: creates a new merge commit that has two parents — the tip of the current branch and the tip of the target branch. The full history of both branches is preserved. The commit graph shows a non-linear history with merge nodes.
- **Rebase**: replays commits from the current branch on top of the target branch one by one. Each commit gets a new SHA (new parent pointer). The result is a linear history as if the work was always done on top of the target. Original commits are orphaned (garbage-collected eventually).

**Internally**: rebase uses `cherry-pick` logic — for each commit, it computes the diff (patch) and applies it to the new base, creating a new commit object.

---

**Q: How do you resolve merge conflicts?**

**A:**
1. `git merge <branch>` — Git marks conflicting files with `<<<<<<<`, `=======`, `>>>>>>>` markers.
2. Open the conflicting files and decide which changes to keep (or combine both).
3. Remove the conflict markers.
4. `git add <resolved_file>` to mark it resolved.
5. `git commit` to complete the merge.

Tools: `git mergetool` (opens vimdiff, IntelliJ, VS Code), `git diff --conflict` to review, `git checkout --ours`/`--theirs` to take one side wholesale.

For complex conflicts, `git log --merge` shows the commits that introduced the conflict.

---

**Q: Difference between git pull, git fetch, and git clone?**

**A:**

| Command | What it does |
|---|---|
| `git clone <url>` | Creates a full local copy of a remote repository (all branches, history, objects). One-time operation. |
| `git fetch` | Downloads new objects and refs from the remote into `FETCH_HEAD` and remote-tracking branches (e.g., `origin/main`) but does NOT update your working branch. Safe to run anytime. |
| `git pull` | `git fetch` + `git merge` (or `git rebase` with `--rebase` flag). Updates your working branch. Can introduce unexpected merge commits. |

**Best practice**: Use `git fetch` + review + `git merge`/`git rebase` for explicit control.

---

**Q: Use case of git stash.**

**A:** `git stash` temporarily shelves uncommitted changes (both staged and unstaged) so you can switch contexts without committing unfinished work.

**Common use cases**:
- Urgently need to switch to another branch to fix a bug without committing WIP code.
- Pull the latest changes when your working tree is dirty.
- Test how the code behaves without your local changes.

```bash
git stash push -m "WIP: feature-X"   # save with a message
git stash list                         # see all stashes
git stash pop                          # restore and drop latest stash
git stash apply stash@{2}              # apply a specific stash without dropping
git stash drop stash@{0}               # discard a stash
```

---

**Q: What is the .gitignore file and how does it work?**

**A:** `.gitignore` is a text file that tells Git which files and directories to exclude from tracking. Git checks `.gitignore` patterns before staging files.

- Patterns are matched relative to the `.gitignore` file location.
- `*` matches anything except `/`; `**` matches across directories.
- Prefix `!` to negate a pattern (un-ignore).
- Trailing `/` matches directories only.

```
node_modules/      # ignore the directory
*.log              # ignore all log files
!important.log     # but track this one
.env               # ignore environment file
dist/              # ignore build output
```

**Note**: `.gitignore` only works for untracked files. If a file is already tracked, use `git rm --cached <file>` to stop tracking it.

---

**Q: What's the difference between git rebase and git merge?**

**A:** (See internal mechanics in Q1 above.)

**When to use each**:
- **Merge**: when preserving the full context of a feature branch matters (e.g., merging to `main` in a team setting — keeps a clear history of when a branch was integrated).
- **Rebase**: when you want a clean, linear history (e.g., updating a feature branch with the latest `main` before opening a PR). Never rebase commits already pushed to a shared branch — rewriting public history causes problems for other contributors.

**Golden rule**: Rebase local, merge shared.

---

**Q: What branching strategy do you follow, and how do you handle merges to avoid breaking the release branch? If a bug appears in production, what's your approach to resolving it?**

**A:**

**Branching strategy (GitFlow variant)**:
- `main` — production-ready code; protected branch.
- `develop` — integration branch for feature work.
- `feature/<name>` — branched from `develop`; merged back via PR with at least one review + CI passing.
- `release/<version>` — cut from `develop` for stabilization; only bug fixes allowed; merged to both `main` and `develop` when ready.
- `hotfix/<name>` — branched from `main` for critical production bugs.

**Protecting the release branch**:
- Branch protection rules: require PR, passing CI, no direct pushes.
- Mandatory code review + automated quality gates (tests, coverage, linting).
- Feature flags to decouple deploy from release.

**Production bug (hotfix) process**:
1. Branch `hotfix/issue-description` off `main` (or the tagged release commit).
2. Reproduce, fix, and write a regression test.
3. PR against `main` with expedited review.
4. After merge to `main`, immediately backport to `develop` (or `release` branch if active) with `git cherry-pick`.
5. Tag `main` with a patch version; trigger the CD pipeline to deploy.
6. Post-mortem to prevent recurrence.


**Q: Git merge succeeds but deployment starts failing. What will you investigate?**

**A:** A successful merge only means the code combined without conflicts — it doesn't guarantee the result works. The failure is somewhere between the merged code and the running environment.

**Investigation checklist**:

1. **Check the CI/CD pipeline logs**: what stage failed? Build, test, Docker push, Helm deploy, health check?

2. **Config/environment drift**: the merge may have introduced a new env var or config key that isn't set in the deployment environment. Check `diff` between the merged code's config requirements and what's in Secrets Manager / ConfigMaps.

3. **Database migrations not run**: the code expects a new column or table that doesn't exist yet. Check if a migration step was skipped in the pipeline.

4. **Image build vs deploy mismatch**: the CI built a new image but the Helm values or deployment manifest still references the old tag. Verify `image.tag` in the deployed manifest matches the newly built image.

5. **Dependency version conflict**: the merge brought in a `package-lock.json` or `go.sum` change that introduced an incompatible library version. Check the build logs for dependency resolution warnings.

6. **Test environment passed, prod fails**: env-specific config (different DB, smaller memory limit, different secrets). Compare `kubectl describe pod` between environments for missing env vars or failing probes.

7. **Git blame the breakage**: `git bisect` to find the exact commit that broke the deployment. `git log --oneline -20` to see what changed.

8. **Rollback immediately if impactful**: `kubectl rollout undo deployment/myapp` or trigger the previous CI pipeline run to restore the last working image while you investigate.

---

**Q: What is the difference between git fetch, git pull, and git rebase?**

**A:**

| Command | What it does | Safe? |
|---|---|---|
| `git fetch` | Downloads new commits/refs from the remote into `origin/<branch>` tracking branches. **Does not touch your working branch.** | Always safe |
| `git pull` | `git fetch` + `git merge` — updates your working branch by merging the remote changes. Can create merge commits. | Safe but messy history |
| `git pull --rebase` | `git fetch` + `git rebase` — replays your local commits on top of the fetched commits. Produces a linear history. | Safe for local branches; **never rebase shared/public branches** |

**When to use each**:
- `git fetch` + inspect + decide: the safest workflow — see what changed before acting.
- `git pull`: quick updates on team branches where merge commits are acceptable.
- `git rebase` (or `git pull --rebase`): keep feature branches up to date with `main` while maintaining a clean, linear history before opening a PR.

**Rebase internals**: rebase detaches your commits and replays them one-by-one on top of the new base. Each replayed commit gets a new SHA — this rewrites history, which is why rebasing pushed/shared commits causes problems for other contributors.

---

**Q: What happens internally when you run git revert?**

**A:** `git revert <commit-sha>` creates a **new commit** that undoes the changes introduced by the specified commit. It does **not** rewrite history — the original commit remains in the log.

**Internals**:
1. Git computes the inverse diff of the target commit: if the commit added a line, the revert removes it; if it removed a line, the revert adds it back.
2. This inverse diff is applied as a new commit to the current HEAD, with a message like `"Revert 'original commit message'"`.
3. The commit graph grows forward — the bad commit stays in history, but its effect is neutralized.

```bash
git revert abc1234          # reverts a single commit
git revert abc1234..def5678 # reverts a range of commits (creates one revert commit per commit)
git revert -n abc1234       # stages the revert but doesn't commit (--no-commit) — lets you combine
```

**vs `git reset`**:
- `git revert`: safe for shared/public branches — adds a new commit, doesn't rewrite history.
- `git reset --hard`: rewrites history by moving HEAD back — only safe for local, unpushed commits.

**When to use revert**: rolling back a bad commit that was already pushed to `main` or a shared branch. It's the standard safe rollback mechanism.

---

**Q: How is GitHub Actions integrated with AWS in your project?**

**A:** GitHub Actions authenticates to AWS using **OIDC (OpenID Connect) federation** — no long-lived AWS credentials stored in GitHub secrets.

**Setup**:
1. Create an IAM OIDC provider in AWS for GitHub Actions:
   ```bash
   aws iam create-open-id-connect-provider \
     --url https://token.actions.githubusercontent.com \
     --client-id-list sts.amazonaws.com \
     --thumbprint-list <github-thumbprint>
   ```

2. Create an IAM role with a trust policy that allows the OIDC provider:
   ```json
   {
     "Effect": "Allow",
     "Principal": { "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com" },
     "Action": "sts:AssumeRoleWithWebIdentity",
     "Condition": {
       "StringEquals": {
         "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
         "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
       }
     }
   }
   ```

3. In the workflow:
   ```yaml
   permissions:
     id-token: write
     contents: read

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4

         - name: Configure AWS credentials (OIDC)
           uses: aws-actions/configure-aws-credentials@v4
           with:
             role-to-assume: arn:aws:iam::123456789:role/GithubActionsDeployRole
             aws-region: us-east-1

         - name: Login to ECR
           run: aws ecr get-login-password | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

         - name: Build and push
           run: |
             docker build -t 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ github.sha }} .
             docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ github.sha }}

         - name: Deploy to EKS
           run: |
             aws eks update-kubeconfig --name my-cluster --region us-east-1
             helm upgrade --install myapp ./chart --set image.tag=${{ github.sha }}
   ```

---

**Q: How do you trigger workflows manually in GitHub Actions?**

**A:** Use the `workflow_dispatch` event trigger, which adds a "Run workflow" button in the GitHub Actions UI.

```yaml
# .github/workflows/deploy.yml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]
      image_tag:
        description: 'Docker image tag to deploy'
        required: false
        default: 'latest'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy ${{ inputs.image_tag }} to ${{ inputs.environment }}
        run: |
          echo "Deploying ${{ inputs.image_tag }} to ${{ inputs.environment }}"
          helm upgrade --install myapp ./chart \
            --set image.tag=${{ inputs.image_tag }} \
            -f values-${{ inputs.environment }}.yaml
```

**Trigger via CLI**:
```bash
gh workflow run deploy.yml \
  --field environment=staging \
  --field image_tag=v1.2.3
```

**Trigger via REST API**:
```bash
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/myorg/myrepo/actions/workflows/deploy.yml/dispatches \
  -d '{"ref":"main","inputs":{"environment":"staging","image_tag":"v1.2.3"}}'
```

Use cases: manual production deploys requiring human approval, re-deploying a specific version, triggering one-off maintenance jobs.

---

**Q: Explain ENTRYPOINT vs CMD in Docker with a real-time use case.**

**A:**

- **ENTRYPOINT**: the executable that always runs. It defines the container's purpose. Only overridable with `--entrypoint` at `docker run`.
- **CMD**: default arguments passed to ENTRYPOINT (or the default command if no ENTRYPOINT is set). Overridable by arguments at `docker run`.

**Real-time use case — database backup tool**:
```dockerfile
FROM python:3.12-alpine
WORKDIR /app
COPY backup.py .
RUN pip install boto3 psycopg2-binary

ENTRYPOINT ["python", "backup.py"]   # always run the backup script
CMD ["--mode", "full"]               # default: full backup
```

```bash
# Full backup (uses CMD default)
docker run mybackup

# Incremental backup (overrides CMD)
docker run mybackup --mode incremental

# Point-in-time restore (overrides CMD with different args)
docker run mybackup --mode restore --timestamp 2024-01-15T02:00:00Z
```

The container is **a backup tool** (ENTRYPOINT) that accepts different modes (CMD overrides). You can't accidentally run a shell in it without explicitly using `--entrypoint /bin/sh`.

**Contrast — no ENTRYPOINT**:
```dockerfile
CMD ["python", "backup.py", "--mode", "full"]
```
Now `docker run mybackup /bin/sh` silently replaces the whole CMD with `/bin/sh` — the container becomes a shell. ENTRYPOINT prevents this accidental misuse.

---

