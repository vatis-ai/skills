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

Point `.mcp.json` at `/mcp`, **naming this repo** so it gets its own dedicated project — one repo, one
project, never sharing a contention space with another codebase:

```json
{ "mcpServers": { "vatis": { "type": "http", "url": "https://api.vatis.dev/mcp?repo=<REMOTE>" } } }
```

`<REMOTE>` is this repo's git remote, URL-encoded. Compute and write it in one step:

```bash
REMOTE=$(git remote get-url origin 2>/dev/null)
# URL-encode REMOTE and set the .mcp.json url to  https://api.vatis.dev/mcp?repo=<encoded REMOTE>
```

Vatis **canonicalises** the remote server-side, so the `https://…` and `git@…` forms of the same repo
resolve to the **same** project — reconnecting always lands you in the same place. No remote at all
(throwaway/local work)? Use plain `https://api.vatis.dev/mcp` and you get a single generic personal
project instead.

You do **not** need a projectId first. On the first click Vatis makes (or finds) this repo's project and
hands the agent a grant scoped to it. The agent can then create and work plans.

## Steps

**1. Write `.mcp.json`** at the repo root with the URL above — `repo=` set to your URL-encoded git remote
(create or merge). No token, no headers — auth is OAuth, negotiated on first use.

**2. Reconnect.** Ask the human to reload MCP servers (or restart Claude Code). An OAuth flow opens; the
human clicks **one** magic-link email to confirm. That link lands on a real Vatis page (not JSON) and
logs the agent in — the consent screen names the repo, so they can see what they're approving.

**3. Verify.** If you can see `mcp__vatis__create_plan` in your tools, you're connected as a planner on
this repo's project. (If you don't see it, you connected as an executor — ask the human for planner.)

**4. Create a plan and plan it.** Call `mcp__vatis__create_plan({ goal })` → it returns a `pl_…` id.
Then invoke the **vatis-plan** skill to encode the project's plan docs into the network and `seed` it.
Every plan-operating tool on this connection takes a `planId` arg — pass the id you just created.

**5. (Optional, later) Enforcement.** `mcp__vatis__guard_setup({ planId })` returns hook files that arm
the walls at the point of action. Leave it off until trusted.

## The three roles — three different connections

The `/mcp?repo=` bootstrap above is how you *start*. A real project run has up to three connection kinds,
each a different token with different verbs. Pick the URL for the role:

| Role | `.mcp.json` URL | Gets |
|---|---|---|
| **Bootstrap / planner** | `https://api.vatis.dev/mcp?repo=<REMOTE>` | finds-or-makes this repo's project; `create_plan` + authoring if planner-role |
| **Executor** (a dispatched worktree working one plan) | `https://api.vatis.dev/mcp/plan/<planId>` | execution verbs; `planId` implicit |
| **Manager** (reviews + approves, never codes) | `https://api.vatis.dev/mcp/manager/<projectId>/full` | `frontier`, `brief`, `create_plan`, `adjudicate`, `absorb`, `acknowledge` |

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
  pasted token, or an `auth.vatis.dev` URL, they're on the old broken path — stop and use `/mcp?repo=`.
- The magic-link token expires in 15 minutes and works once. Open the newest email if you clicked twice.
