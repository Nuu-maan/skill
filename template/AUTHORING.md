# Writing skills

The rules these skills follow, distilled from the
[Agent Skills spec](https://agentskills.io) and Anthropic's
[skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices).

## Contents

- How an agent actually loads a skill
- Frontmatter
- Writing the description
- Structuring the body
- Progressive disclosure
- Degrees of freedom
- Bundled scripts
- Anti-patterns
- Testing a skill
- Checklist

---

## How an agent actually loads a skill

Three levels, loaded at different times. Design around this or you will waste context.

| Level | When it loads | Cost | What it is |
| :--- | :--- | :--- | :--- |
| 1 — Metadata | At startup, always | ~100 tokens per skill | `name` + `description` from the frontmatter |
| 2 — Instructions | When the skill triggers | The whole body | `SKILL.md` markdown |
| 3 — Resources | When the agent chooses to read a file | Nothing until read | `references/*.md`, `scripts/*`, templates |

Consequences:

- **The description does all the triggering work.** If it is vague, the skill never fires and nothing
  else matters.
- **The body is a recurring cost.** In Claude Code it stays in context for the rest of the session.
  Keep it tight.
- **Bundled files are free until opened.** This is why a skill can ship 3,000 lines of reference
  material and still be cheap: a small audit reads two files, a big one reads eight.

Design the skill as a router, not a document.

---

## Frontmatter

Only these six fields are in the standard. Use only these if you want the skill to work in claude.ai
uploads, the Skills API, and non-Claude agents — those paths reject unknown keys with a hard error,
they do not ignore them.

| Field | Required | Rules |
| :--- | :--- | :--- |
| `name` | Yes | ≤64 chars, lowercase letters, numbers, hyphens only. No XML tags. Cannot contain "anthropic" or "claude". |
| `description` | Yes | Non-empty, ≤1024 chars, no XML tags. What it does **and** when to use it. |
| `license` | No | SPDX identifier. |
| `compatibility` | No | ≤500 chars. Environment requirements. |
| `metadata` | No | Free-form map. Version, author, homepage. |
| `allowed-tools` | No | Tools pre-approved for the turn that invokes the skill. |

Claude Code additionally accepts `when_to_use`, `argument-hint`, `arguments`,
`disable-model-invocation`, `user-invocable`, `disallowed-tools`, `model`, `effort`, `context`,
`agent`, `background`, `hooks`, `paths`, and `shell`. Useful, but they make the skill
Claude Code-only. Document them in the skill's README instead of putting them in `SKILL.md`.

In Claude Code, the **directory name** determines the command you type, not the `name` field. Keep
them the same to avoid confusion.

---

## Writing the description

This is the highest-leverage text in the whole skill. It goes into the system prompt and is what the
agent matches a request against, alongside every other installed skill.

**Structure:** what it does → when to use it → trigger phrases.

```yaml
description: Audits a backend or API for security vulnerabilities, stability bugs, and design flaws,
  then fixes them on request. Covers auth, IDOR, injection, secrets, transactions, and rate limiting
  across any language. Use whenever the user asks to review, audit, secure, or production-proof a
  backend; mentions vulnerabilities, exposed API keys, or leaked secrets; or is about to deploy.
```

**Rules:**

- **Third person, always.** "Audits a backend", not "I can audit your backend" or "You can use this
  to audit". Mixed point of view measurably hurts discovery.
- **Be slightly pushy.** Models under-trigger skills. "Use whenever…" beats "Can be used for…".
- **Include the words users will actually type.** If someone would say "is my API secure", those
  words belong in the description.
- **Name the technologies.** Framework and tool names are strong matching signals.
- **Say when *not* to use it** if the skill is easy to confuse with another one.

Bad: `description: Helps with backend stuff`

---

## Structuring the body

Keep `SKILL.md` **under 500 lines**. Aim well under.

**Assume the model is already competent.** Only add what it does not already know: your conventions,
your workflow, the specific things it tends to get wrong. Every paragraph should survive the question
"does this justify its token cost?"

```markdown
<!-- 50 tokens, good -->
## Extract PDF text
Use pdfplumber:
```python
with pdfplumber.open("f.pdf") as pdf: text = pdf.pages[0].extract_text()
```

<!-- 150 tokens for the same information, bad -->
## Extract PDF text
PDF (Portable Document Format) files are a common file format that contains text,
images, and other content. To extract text you'll need a library. There are many...
```

**Use workflows with checklists** for multi-step tasks. Giving the agent a checklist to copy into its
reply and tick off measurably reduces skipped steps.

**Use feedback loops** where quality matters: do the thing → validate → fix → repeat → only proceed
when it passes.

**Be consistent with terminology.** Pick "endpoint" or "route" and never mix them. Inconsistent
vocabulary makes instructions harder to follow.

**No time-sensitive statements.** "As of March 2025" becomes wrong. If you must record deprecated
behavior, put it in a collapsed "old patterns" section.

---

## Progressive disclosure

The pattern: `SKILL.md` is a table of contents; the depth lives in files it points to.

```
my-skill/
├── SKILL.md              # router + workflow, always loaded when triggered
├── references/           # loaded selectively
│   ├── topic-a.md
│   └── topic-b.md
├── scripts/              # executed, never loaded into context
│   └── validate.py
└── templates/
    └── output.md
```

**Route with a table**, so the agent can decide what to read without reading it:

```markdown
| Load this | When |
| :--- | :--- |
| [references/auth.md](references/auth.md) | The codebase has login or sessions |
| [references/data.md](references/data.md) | The codebase has a database |
```

**Keep references one level deep.** `SKILL.md` → `reference.md` is fine.
`SKILL.md` → `advanced.md` → `details.md` is not: agents partial-read files reached through a chain
(often with `head -100`) and silently miss the rest.

**Add a table of contents to any reference file over ~100 lines**, so a partial read still reveals
the full scope.

**Name files for their content.** `form-validation-rules.md`, not `doc2.md`.

---

## Degrees of freedom

Match specificity to how fragile the task is.

| Freedom | Use when | Form |
| :--- | :--- | :--- |
| High | Many valid approaches, context decides | Prose guidance: "Analyze structure, check for edge cases, suggest improvements" |
| Medium | A preferred pattern exists, variation is fine | Pseudocode or a parameterized template |
| Low | Fragile, order-dependent, or destructive | An exact command: "Run exactly `python scripts/migrate.py --verify --backup`. Do not add flags." |

The test: is this a narrow bridge with cliffs, or an open field? Give guardrails on the bridge and
direction in the field.

---

## Bundled scripts

Scripts are executed via bash and their **code never enters context** — only their output. That makes
them cheaper and more reliable than asking the agent to generate equivalent code.

- **State the intent explicitly.** "Run `analyze.py` to extract fields" (execute) versus "See
  `analyze.py` for the algorithm" (read). Ambiguity costs a wasted file read.
- **Solve, do not defer.** Handle the error inside the script rather than failing and leaving the
  agent to figure it out.
- **No magic numbers.** `TIMEOUT = 47` needs a comment saying why, or it should not be 47.
- **List dependencies.** Do not assume a package is installed. Note that Skills API containers have
  no network access and cannot install anything at runtime.
- **Forward slashes only** in every path, on every platform. `scripts/helper.py`, never
  `scripts\helper.py`.

---

## Anti-patterns

- **Vague description.** The single most common reason a skill never fires.
- **Offering five options.** "You could use pypdf, or pdfplumber, or PyMuPDF, or…" — give one default
  and one escape hatch.
- **Explaining what the model already knows.** What HTTP is, what a database does, why security
  matters.
- **A wall of prose.** Tables, checklists, and short sections are followed more reliably than
  paragraphs.
- **Nested references.** See above — they get partially read.
- **Windows paths.** Break on Unix.
- **Reserved words in `name`.** "claude" or "anthropic" anywhere in it is rejected.
- **One skill trying to do everything.** If the description needs three "or use when" clauses for
  unrelated tasks, it is two skills.

---

## Testing a skill

Triggering is not the same as working. Test both, separately.

1. **Does it fire?** Write 5–10 prompts a real user would send. Run each in a **fresh session** — your
   authoring conversation contains context that masks gaps in the written instructions. Count how
   many trigger the skill. Also write prompts that should *not* trigger it and confirm they do not.
2. **Is the output better?** Run the same prompts with the skill disabled and compare. If the output
   is the same, the skill is paying context rent for nothing.
3. **Watch how it navigates.** Which files does it read, in what order, and which does it never open?
   A file that is never read is either unnecessary or badly signposted from `SKILL.md`. A file read
   on every single run probably belongs in `SKILL.md`.
4. **Test on more than one model.** What a large model infers from a hint, a smaller one needs spelled
   out.

In Claude Code, the `skill-creator` plugin automates the comparison:
`/plugin marketplace add anthropics/claude-plugins-official` then
`/plugin install skill-creator@claude-plugins-official`. It runs each test case in an isolated
subagent, grades against assertions, and benchmarks with-skill versus without.

The productive loop is: build the skill with one agent session, test it with a fresh one, bring the
failures back to the first.

---

## Checklist

Before publishing:

- [ ] `name` is lowercase-hyphens, ≤64 chars, matches the directory name, contains no reserved word
- [ ] `description` says what **and** when, in third person, ≤1024 chars, with trigger keywords
- [ ] Only spec frontmatter fields, if the skill should work outside Claude Code
- [ ] `SKILL.md` body under 500 lines
- [ ] Detail lives in reference files, one level deep from `SKILL.md`
- [ ] Reference files over ~100 lines have a table of contents
- [ ] All paths use forward slashes
- [ ] No time-sensitive statements
- [ ] Terminology is consistent throughout
- [ ] Examples are concrete code, not descriptions of code
- [ ] Scripts handle their own errors and document their dependencies
- [ ] Tested in a fresh session against at least three realistic prompts
- [ ] Compared against a run with the skill disabled
- [ ] A `README.md` explains what it does and how to install it
