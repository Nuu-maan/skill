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

## Install

```bash
npx skills add https://qala.lol/nuu-maan/backend-hardening
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

See [template/](template) for a starter `SKILL.md` and the authoring rules these skills follow.

## Security

Skills are instructions an agent will follow, and they can include scripts an agent will run. Treat
installing one like installing a dependency: read the `SKILL.md` and anything under `scripts/` before
you use it. Nothing in this repository executes code you have not seen, and no skill here fetches
remote content at runtime.

## License

MIT — see [LICENSE](LICENSE).
