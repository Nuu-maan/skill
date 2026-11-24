# skills

Agent Skills by [@Nuu-maan](https://github.com/Nuu-maan). Also on [qala.lol](https://qala.lol).

Each skill is a folder with a `SKILL.md`, following the open
[Agent Skills](https://agentskills.io) standard. They work in Claude Code, claude.ai, the Claude API,
Cursor, Codex, GitHub Copilot / VS Code, Gemini CLI, OpenCode, Goose, and any other agent that
supports the format.

## Skills

| Skill | What it does |
| :--- | :--- |
| [backend-hardening](skills/backend-hardening) | Audits a backend or API for security vulnerabilities, stability bugs, and design flaws, explains each one in plain language, then fixes them on request. Any language or framework. |
| [database-design](skills/database-design) | Reviews a database for schema, indexing, query, and migration-safety problems and fixes them on request — or designs a new schema from scratch. Postgres, MySQL, SQLite, MongoDB. |
| [not-a-demo](skills/not-a-demo) | Makes AI-built web apps look designed and behave like shipped software: a written design direction, every loading, empty and error state, real UI copy, purposeful motion, server-side auth and data checks, and speed from browser to database. |

## Install

```bash
npx skills add https://qala.lol/numan/backend-hardening
npx skills add https://qala.lol/numan/database-design
npx skills add https://qala.lol/numan/not-a-demo
```

Or install the whole collection in Claude Code as a plugin:

```
/plugin marketplace add Nuu-maan/skill
/plugin install qala@qala
```

Skills then appear as `/qala:<skill-name>`.

Every skill here uses only the six standard frontmatter fields — `name`, `description`, `license`,
`compatibility`, `metadata`, `allowed-tools` — so it also uploads to claude.ai and the Skills API
without modification. Those paths reject non-standard keys with a hard error.

## Writing your own

See [template/](template) for a starter file and the authoring rules these skills follow.

Copy [template/skill-template.md](template/skill-template.md) to
`skills/<your-skill-name>/SKILL.md` and fill it in. It is deliberately **not** named `SKILL.md` —
skill registries and agent runtimes glob the repository for that exact filename, and a template
sitting at that path gets picked up and published as a real skill.

## Security

Skills are instructions an agent will follow, and they can include scripts an agent will run. Treat
installing one like installing a dependency: read the `SKILL.md` and anything under `scripts/` before
you use it. Nothing in this repository executes code you have not seen, and no skill here fetches
remote content at runtime.

## License

MIT — see [LICENSE](LICENSE).
