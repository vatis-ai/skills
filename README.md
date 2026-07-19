# Vatis skills

Skills for planning and building with **[Vatis](https://vatis.dev)** — a promise-network planning
substrate for coding agents. A plan is a network of typed promises, not a task queue: publish a node's
shape the instant you decide it and everyone who only needed the shape starts work before the code
exists; falsifying one promise recalls only the reliers on that exact clause.

Connect an agent in one click by pointing `.mcp.json` at `https://api.vatis.dev/mcp?repo=<git-remote>` —
no token to paste. Start with `vatis-setup`.

## Install

```bash
npx skills add vatis-ai/skills
```

Or drop any `skills/<name>/` folder into `~/.claude/skills/` (or a project's `.claude/skills/`).

## The skills

| Skill | For | Use when |
|---|---|---|
| **vatis-setup** | anyone | wiring a repo to Vatis (the MCP connection + the three role URLs) |
| **vatis-plan** | planner / executor | turning planning docs into a promise network and working it |
| **vatis-review** | manager | reviewing and approving work at a gate, never writing code |
| **vatis-feedback** | any role | reporting when Vatis itself gets in your way |

## Links

- Product: [vatis.dev](https://vatis.dev)
- Connect / model / verbs: [api.vatis.dev/llms.txt](https://api.vatis.dev/llms.txt)
