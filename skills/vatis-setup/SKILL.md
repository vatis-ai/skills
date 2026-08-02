---
name: vatis-setup
description: Use to WIRE a project up to Vatis — configure the MCP connection so this project's agent can plan with it. Trigger on "connect to Vatis", "set up Vatis here", "wire up the Vatis MCP", "configure Vatis in this repo". Once connected, use the vatis-plan skill to turn docs INTO a plan.
---

# Wiring a project to Vatis

Vatis is a promise-network planning MCP at `https://api.vatis.dev`. This gets you connected in **one
human click** — no token to paste, no REST, no `curl`, no id to find first.

⚠️ Vatis is dogfood-stage (deployed, not battle-tested). Enforcement is optional and OFF by default —
start planning-only. The guard fails **open**, and a node's write-set **cannot be amended** once seeded.

## The whole setup

Point `.mcp.json` at the bare `/mcp` bootstrap URL. It connects you **org-scoped** — from there you list
or create a project (nothing auto-provisions one, and no id is needed first):

```json
{ "mcpServers": { "vatis": { "type": "http", "url": "https://api.vatis.dev/mcp" } } }
```

On the first click Vatis hands the agent an org-scoped setup grant. Call `list_projects` to see the org's
projects, or `create_project({ name })` to make one — deliberately named for this codebase (a project is
one repo / one contention space). It returns a `prj_…` id. Then point `.mcp.json` at that project's URL
(`https://api.vatis.dev/mcp/project/<prj_id>`) and reconnect to plan and work it.

## Steps

**1. Write `.mcp.json`** at the repo root with the bare bootstrap URL above (create or merge). No token,
no headers — auth is OAuth, negotiated on first use.

**2. Reconnect.** Ask the human to reload MCP servers (or restart Claude Code). An OAuth flow opens; the
human clicks **one** magic-link email to confirm. That link lands on a real Vatis page (not JSON) and
logs the agent in.

**3. Pick or create a project.** Call `mcp__vatis__list_projects` to see the org's projects, or
`mcp__vatis__create_project({ name })` to make one — named for this codebase. It returns a `prj_…` id.

**4. Reconnect to the project, then plan it.** Point `.mcp.json` at
`https://api.vatis.dev/mcp/project/<prj_id>` and reload. You now hold a planner grant on that project:
`mcp__vatis__create_plan({ goal })` returns a `pl_…` id. Then invoke the **vatis-plan** skill to encode
the plan docs and `seed` it. Every plan-operating tool takes a `planId` arg — pass the id you created.

**5. (Optional, later) Enforcement.** `mcp__vatis__guard_setup({ planId })` returns hook files that arm
the walls at the point of action. Leave it off until trusted.

## The three roles — three different connections

The bare `/mcp` bootstrap above is how you *start* (org-scoped: `list_projects`/`create_project`). A real
project run has connection kinds, each a different token with different verbs. Pick the URL for the role:

| Role | `.mcp.json` URL | Gets |
|---|---|---|
| **Bootstrap (org setup)** | `https://api.vatis.dev/mcp` | `list_projects`, `create_project` — no id needed; reconnect to the project after |
| **Planner** (works a project solo) | `https://api.vatis.dev/mcp/project/<projectId>` | `create_plan` + authoring + execution; cannot adjudicate |
| **Executor** (a dispatched worktree working one plan) | `https://api.vatis.dev/mcp/plan/<planId>` | execution verbs; `planId` implicit |
| **Manager** (plans, reviews + approves, never codes) | `https://api.vatis.dev/mcp/manager/<projectId>/full` | the whole planning + approval surface, below |

One rule covers the table: **approve XOR execute.** A planner connection authors *and* executes and so
cannot adjudicate; a manager authors *and* adjudicates and so cannot execute. Authoring a plan is not
doing the work — the wall is that the party approving is not the party that did it.

The manager surface, in full:

<!-- vatis:verbs kind=manager -->
`feedback`, `create_project`, `create_plan`, `frontier`, `brief`, `seed`, `bet`, `unfold`,
`discover`, `revoke`, `adjudicate`, `absorb`, `acknowledge`
<!-- /vatis:verbs -->

`unfold` takes a `nodeId` there (a manager holds no lease), and `discover` is narrowed to a finding or an
artifact break the graph already proves. `guard_setup` is **not** on it — arming the hooks is a setup act
run once per repo from a planner connection.

**If you are one person running your own project, use the manager URL.** It plans, dispatches and
approves on one credential, so you never swap `.mcp.json` mid-run.

- **Manager `/full`** is the owner's opt-in to clear `human` gates too. ⚠️ **`/full` MUST be a path
  segment, never `?full`** — an MCP client authorizes against the path-only resource from
  protected-resource metadata, so a `?full` query string is stripped before consent and silently mints a
  plain manager. Drop `/full` for a plain (reviewer-only) manager. Only an **owner** may authorize a
  manager at all.
- **Upgrading a scope needs a FRESH auth.** Reconnecting reuses the cached token at its old scope ("Auth
  successful", no consent page). To go plain→full manager or executor→planner: **clear the vatis auth,
  then reconnect** so consent actually re-runs.
- To run a fleet (a manager dispatching executor sessions), read `frontier` each cycle and hand each
  ready node to an executor session — you coordinate and approve; executors claim and build. The
  **vatis-review** skill is the manager's discipline.

## Let the agents actually act — classifier allow-rules

Claude Code's auto-mode classifier auto-denies high-authority MCP calls and sensitive config edits, which
will silently stall a run. Add allow-rules once, in `.claude/settings.local.json`:

```json
{ "permissions": { "allow": [
  "mcp__vatis__adjudicate", "mcp__vatis__absorb", "mcp__vatis__acknowledge", "Edit(.mcp.json)"
] } }
```

An agent **cannot self-grant** — editing the permissions file is itself blocked (correctly). The **owner**
adds this once (or an agent writes it via a shell heredoc only after explicit owner authorization).

## Notes
- **The human clicks once. That is the only manual step, ever.** If anyone is reaching for `curl`, a
  pasted token, or an `auth.vatis.dev` URL, they're on the old broken path — stop and use the bare `/mcp`.
- The magic-link token expires in 15 minutes and works once. Open the newest email if you clicked twice.
