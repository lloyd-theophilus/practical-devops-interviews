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
