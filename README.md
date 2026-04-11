# agent-skills

Personal collection of agent skills by [Lionel Ndong](https://github.com/Lionelndong), installable via the [skills.sh](https://github.com/vercel-labs/skills) CLI.

Each skill is a focused instruction file that teaches AI coding agents (Claude Code, Cursor, Copilot, Codex, etc.) how to do one thing without tripping over the common footguns.

## Install

Install a specific skill into your current project:

```bash
npx skills add Lionelndong/agent-skills --skill strapi-expert
```

List available skills in this repo:

```bash
npx skills add Lionelndong/agent-skills --list
```

## Skills

| Skill | What it teaches | skills.sh |
|---|---|---|
| [`strapi-expert`](skills/strapi-expert/SKILL.md) | Create, update, draft, publish, and attach media to Strapi v5 Cloud content via the REST API — without silently publishing drafts or failing uploads. | [listing](https://skills.sh/lionelndong/agent-skills/strapi-expert) |

## Repo layout

```
agent-skills/
├── README.md
├── LICENSE
└── skills/
    └── strapi-expert/
        ├── SKILL.md            # main instructions
        └── references/         # deep-dive docs the skill points to
            └── errors.md
```

## License

MIT — see [LICENSE](LICENSE).
