# Contributing

These skills get better when people report what agents get wrong. Contributions of every size are welcome.

## Reporting a problem

Open an issue with:

1. **The prompt** you gave the agent (and which agent and model).
2. **What it produced** — the relevant code.
3. **What it should have produced**, and why (a link to the docs, the CxJS source, or your project's convention).
4. The `cx` version from your `package.json`.

## Changing a skill

<!-- TODO: finalize once the first skill is written -->

- **Where things go**
  - `SKILL.md` — the rules and pitfalls an agent needs on almost every task. Keep it short.
  - `references/*.md` — detailed patterns for one area, loaded only when needed.
- **Every claim must be verifiable** against the [CxJS source](https://github.com/codaxy/cxjs/tree/master/packages/cx/src) or the [documentation](https://cxjs.io/docs). Link the source file when a rule is not obvious.
- **Code snippets must compile** against the current `cx` release.
- **Link documentation pages in their Markdown form** — `https://cxjs.io/docs/tables/grid.md`, not `https://cxjs.io/docs/tables/grid` — so agents fetch clean text instead of HTML.
- **Stay tool-neutral.** Do not name tools of a specific agent; say "read `node_modules/cx/src/...`", not "use the Read tool".
- **Framework, not application.** Conventions specific to one app belong in that app's project skill, not here.

## Pull requests

1. Fork and branch from `master`.
2. Make the change and verify the snippets you touched.
3. Describe what the agent did before and after, ideally with the prompt you tested.
