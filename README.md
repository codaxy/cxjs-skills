# CxJS Skills

Agent skills for building applications with [CxJS](https://cxjs.io) — for Claude Code, Cursor, Codex, and any other agent that reads `SKILL.md`.

| Skill | Use it for |
| --- | --- |
| [`cxjs`](skills/cxjs/SKILL.md) | Writing, reviewing and debugging CxJS code: typed models, store and bindings, controllers, widgets, forms, grids, charts, routing and theming |
| [`cxjs-diagrams`](skills/cxjs-diagrams/SKILL.md) | Node-and-edge diagrams with [`cx-diagrams`](https://github.com/codaxy/cx-diagrams): layout on a grid, shapes and lines, zoom and pan, selection, drag & drop, diagrams built from data |

Skills are plain folders (`SKILL.md` + `references/`). Agents load the short description up front and read the rest only when a task needs it.

## Installation

### Any agent

```bash
npx skills add codaxy/cxjs-skills
```

The [skills CLI](https://github.com/vercel-labs/skills) installs the skills for the agents you choose (Claude Code, Cursor, Codex, …). Add `-g` to install for your user instead of the current project, and run `npx skills update` to pull new versions.

### Claude Code plugin

```
/plugin marketplace add codaxy/cxjs-skills
/plugin install cxjs@cxjs
```

Run `/plugin marketplace update cxjs` to pull new versions.

### Manual

Copy the folder of the skill you want into the location your agent reads:

| Agent | Project | User |
| --- | --- | --- |
| Claude Code | `.claude/skills/<name>/` | `~/.claude/skills/<name>/` |
| Cursor | `.agents/skills/<name>/` or `.cursor/skills/<name>/` | `~/.cursor/skills/<name>/` |
| Codex | `.agents/skills/<name>/` | `~/.agents/skills/<name>/` |

Committing the skill into your project repository makes it available to everyone on the team, whichever agent they use.

The skills here cover CxJS itself. Where an application has its own conventions, the `cxjs` skill tells the agent to follow them.

## Contributing

Found the agent doing something wrong, or know a pattern worth teaching? See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
