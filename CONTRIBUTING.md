# Contributing to Practical DevOps Interviews

Thank you for helping grow this resource. This guide covers everything you need to know before submitting a contribution.

---

## What We Accept

- New interview questions for existing topics
- New topic folders with an initial question set (minimum 5 questions)
- Corrections to existing questions (unclear wording, factual errors)
- Improved structure or formatting within a topic file

We do **not** accept:
- Questions answered with a single-word or trivial definition ("What is Docker?" without depth)
- Questions that duplicate existing entries
- Promotional content, links to paid courses, or affiliate links

---

## Question Quality Bar

Every question must meet these criteria before it will be merged:

1. **Tests understanding, not recall.** Prefer "How does X work internally?" or "What happens when you do Y?" over "What is the definition of Z?"
2. **Vetted.** The question reflects something that is genuinely asked in real interviews, not hypothetical trivia.
3. **Unambiguous.** The question has a clear, answerable scope — not so broad that it could mean anything.
4. **Standalone.** Each question makes sense without needing the context of the questions around it.

---

## How to Contribute

### 1. Fork and clone

```bash
git clone https://github.com/<your-username>/practical-devops-interviews.git
cd practical-devops-interviews
```

### 2. Create a branch

Use a short, descriptive name:

```bash
git checkout -b add/kubernetes-scheduler-questions
git checkout -b fix/docker-layer-caching-wording
git checkout -b new-topic/linux
```

### 3. Make your changes

- Questions go inside the relevant `<topic>/<topic>.md` file, one question per line, as a Markdown list item:
  ```markdown
  - How does the Linux OOM killer decide which process to terminate?
  ```
- If you are adding a new topic, create a new folder and a single `.md` file inside it. Follow the naming pattern of existing folders (`lowercase`, no spaces).
- Do not add answers inline. This repo is intentionally question-only to keep the focus on preparation and avoid answer drift.

### 4. Commit your changes

Write a clear, concise commit message:

```bash
# Adding questions
git commit -m "feat(kubernetes): add scheduler and affinity questions"

# Fixing a question
git commit -m "fix(docker): clarify layer caching question wording"

# New topic
git commit -m "feat: add linux topic with initial question set"
```

### 5. Open a pull request

Push your branch and open a PR against `main`. In the PR description:
- List the questions or changes you are adding
- Briefly explain where the questions come from (personal interview experience, public job boards, etc.)

---

## Folder and File Conventions

| Convention | Example |
|---|---|
| Folder name | `kubernetes/`, `system design/` |
| File name | `k8s.md`, `docker.md`, `cicd.md` |
| Question format | Markdown list item (`- Question text here`) |
| No trailing punctuation on questions | Preferred but not a blocker |

---

## Code of Conduct

Be respectful. Contributions that include discriminatory language, harassment, or personal attacks will be closed without review.

---

## Questions?

Open an issue and tag it `question` or `discussion`.
