---
name: your-skill-name
description: States what the skill does, then when to use it. Third person, specific, keyword-rich. Example - Audits a backend or API for security vulnerabilities and stability bugs, then fixes them on request. Use whenever the user asks to review, audit, secure, or production-proof a backend, mentions vulnerabilities or exposed keys, or is about to deploy.
license: MIT
metadata:
  version: 1.0.0
  author: nuu-maan
  homepage: https://qala.lol
---

# Your Skill Name

One or two lines on what this skill makes the agent do. No preamble about why the topic matters.

## Operating rules

Standing constraints that apply for the whole task. Put anything non-negotiable here — the agent
reads this section every time.

- Cite evidence for claims.
- Ask before doing anything destructive.
- Say when you could not verify something.

## Workflow

Copy this checklist into your reply and tick items as you go:

```
- [ ] Step 1: ...
- [ ] Step 2: ...
- [ ] Step 3: ...
```

### Step 1: ...

What to do. Be specific about which files to look at and what to look for.

### Step 2: ...

...

## Reference files

Load only what the task needs.

| Load this | When |
| :--- | :--- |
| [references/topic-a.md](references/topic-a.md) | The project has X |
| [references/topic-b.md](references/topic-b.md) | The user asked about Y |

## Output

Point at a template, or state the format inline if it is short.

See [templates/output.md](templates/output.md).
