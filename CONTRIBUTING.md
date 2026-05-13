# Contributing to Practical DevOps Interviews

Thank you for helping grow this resource. This guide covers everything you need to know before submitting a contribution.

---

## What We Accept

- New interview questions with answers for existing topics
- New topic folders with an initial question and answer set (minimum 5 Q&A pairs)
- Corrections to existing questions or answers (unclear wording, factual errors, outdated information)
- Improved structure or formatting within a topic file

We do **not** accept:
- Shallow questions or answers that rely on single-word or trivial definitions ("What is Docker?" without depth)
- Questions that duplicate existing entries
- Promotional content, links to paid courses, or affiliate links

---

## Question & Answer Quality Bar

Every question must meet these criteria before it will be merged:

1. **Tests understanding, not recall.** Prefer "How does X work internally?" or "What happens when you do Y?" over "What is the definition of Z?"
2. **Vetted.** The question reflects something that is genuinely asked in real interviews, not hypothetical trivia.
3. **Unambiguous.** The question has a clear, answerable scope — not so broad that it could mean anything.
4. **Standalone.** Each question makes sense without needing the context of the questions around it.

Every answer must meet these criteria:

1. **Accurate and current.** Reflects real-world behavior of the tool or system as it exists today.
2. **Explains the "why".** Don't just state what — explain how or why it works that way.
3. **Practical.** Where applicable, include commands, config snippets, or concrete examples.
4. **Concise.** Cover what matters; avoid padding. Bullet points and tables are preferred over long prose.

---

## Format

Each entry follows this structure:

```markdown
**Q: How does X work internally?**

**A:** Your answer here. Use bullet points, code blocks, or tables as needed.

---
```

Code blocks should specify the language for syntax highlighting:

````markdown
```bash
docker run -p 8080:80 nginx
```
````

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

- Questions and answers go inside the relevant `<topic>/<topic>.md` file.
- Follow the `**Q:**` / `**A:**` format shown above.
- Separate each Q&A block with a `---` horizontal rule.
- If you are adding a new topic, create a new folder and a single `.md` file inside it. Follow the naming pattern of existing folders (`lowercase`, spaces allowed for multi-word topics like `system design/`).

### 4. Commit your changes

Write a clear, concise commit message:

```bash
# Adding questions with answers
git commit -m "feat(kubernetes): add scheduler and affinity questions"

# Fixing a question or answer
git commit -m "fix(docker): clarify layer caching answer"

# New topic
git commit -m "feat: add linux topic with initial question set"
```

### 5. Open a pull request

Push your branch and open a PR against `main`. In the PR description:
- List the questions you are adding or changing
- Briefly explain where the questions come from (personal interview experience, public job boards, etc.)

---

## Folder and File Conventions

| Convention | Example |
|---|---|
| Folder name | `kubernetes/`, `system design/` |
| File name | `k8s.md`, `docker.md`, `cicd.md` |
| Question format | `**Q: Question text here?**` |
| Answer format | `**A:** Answer text here.` |
| Separator | `---` between each Q&A block |

---

## Code of Conduct

Be respectful. Contributions that include discriminatory language, harassment, or personal attacks will be closed without review.

---

## Questions?

Open an issue and tag it `question` or `discussion`.
