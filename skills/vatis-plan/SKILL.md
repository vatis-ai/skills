---
name: vatis-plan
description: Use when turning this project's planning docs (MD files, a spec, a roadmap) into a Vatis plan — a typed promise network — and working it. Vatis is a planning substrate for coding agents, where a plan is a network of promises with typed reliances, not a work queue. Trigger on "plan this with Vatis", "seed the plan", "turn the spec into a promise network", or when a Vatis MCP server (mcp__vatis__*) is connected.
---

# Working a project with Vatis

Vatis models a plan as a **promise network**: each node makes a **promise** (a shape, an artifact, or a
finding) and edges are **reliances** on those promises. Its value is *when* you settle — publishing a
shape releases everyone who only needed the shape, long before the code exists — plus surgical
invalidation when something breaks. You interact through ~13 MCP tools (`mcp__vatis__*`).

⚠️ **Honest status (read this):** Vatis is deployed but has **not been driven live end-to-end**. Treat it
as dogfood, not production. Two facts that shape everything you do:
- **A write-set can only GROW, and only into free ground.** If a node under-declares, its holder can
  `widen` it to add uncontended files as it discovers them — but you can never remove a path or take one
  another live lease holds. So declare what you know and widen as you learn; don't try to predict the whole
  footprint up front.
- **The guard fails OPEN.** If Vatis is unreachable or misconfigured, writes proceed unrefused. Never
  assume the wall is protecting you without checking.

Start **planning-only with the guard OFF** (below). Get the structural value — the network, the frontier,
the bets — with zero enforcement risk. Arm the guard later, on a small slice, once it's trusted.

---

## 0. Prerequisites — connect, then create the plan yourself

You connect in **one click** and create plans yourself (see the `vatis-setup` skill for wiring):
- `.mcp.json` points at the bare `https://api.vatis.dev/mcp` — an org-scoped bootstrap. On first use,
  Claude Code opens an OAuth flow; the human clicks **one** magic-link to approve (a real page, not JSON).
  Then `list_projects`/`create_project({ name })` to pick or make this codebase's project, reconnect to
  `…/mcp/project/<prj_id>`, and you're a **planner** on it (you see `mcp__vatis__create_plan`).
- **Make the plan:** `mcp__vatis__create_plan({ goal })` → returns a `pl_…` id. This is your plan; you did
  not need a projectId, a pasted token, or any REST.
- ⚠️ **On this connection every plan-operating tool takes a `planId` argument** — pass the id you just
  created to `seed`, `frontier`, `claim`, etc. (A dispatched executor instead gets a plan-scoped
  connection, `.../mcp/plan/<planId>`, where `planId` is implicit — it works exactly one plan.)

---

## 1. Encode the planning docs into a promise network (the core skill)

Read every planning MD/spec file first. Then translate prose into nodes. This is the hard part and where
the value is — do it deliberately.

**A NODE is one unit of work = one agent = one context = one verification = one merge.** Size it to *what
must be held in one head at once*, not to a file count. A node needs:

| field | what to put | rule |
|---|---|---|
| `id` | short kebab id | unique; can never be re-used |
| `intent` | what this node is for, cold-startable | a stranger must be able to execute from this alone |
| `writes` | the files this node will **modify** | the *only* field that drives contention. It can **grow** later (the holder `widen`s it with uncontended files it discovers), but never shrink — so under-declaring is recoverable, over-declaring is not. When unsure, list the file. A trailing `/` makes an entry a **directory** covering everything beneath it (`apps/router/`) — but it serialises everyone who touches anything inside, so prefer listing files; use a directory only when the node genuinely rewrites the whole subtree. |
| `reads` | files it depends on but will **not** change | free, blocks nobody |
| `verification` | the command that must exit 0 to prove it done | **required** — null means it can never be fulfilled. Make it real (a test/build), not `node --check`. |
| `reliances` | what this node needs from others (see below) | the edges of the network |
| `adjudicator` | `"reviewer"` (a manager agent OR a person may clear it — "does the work meet the contract?") or `"human"` (a **person only**) for a DECISION rather than a task | Use `reviewer` for the bulk (conformance sign-offs). Use `human` only when it needs a real person: taste, an irreversible/costly call, or a fact only someone can vouch for (money spent, entitlement on) — a manager is refused those. Either way an executing agent can't clear it. |

**A RELIANCE** is `{ on: "<other-node-id>", kind: "contract" | "artifact" | "finding", clauses?: [...] }`:
- **`contract`** — you need the *shape decided* (signature/schema/route). Released the moment that node
  *settles*, while its code is still vapour. Name the `clauses` you actually need (e.g. `["verify"]`) so a
  change elsewhere doesn't recall you needlessly; omit clauses only if you truly need the whole shape.
- **`artifact`** — you need it *built and proven*, not just shaped. Released when it's fulfilled.
- **`finding`** — you need to have *read what it learned* (a decision, a measurement). This is how a
  human decision blocks work: give the blocked node a `finding` reliance on the decision node.

**BETS** are separate (`mcp__vatis__bet`, planner-only): a load-bearing assumption nobody has verified —
`{ id, belief, falsifier, cost }`. No confidence number. Anything you're *gambling* on ("the API isn't
paginated", "the lib supports streaming") is a bet; declare it so a human sees what the plan rests on.
Anything downstream of an unresolved high-blast bet is **fiction** — put an unfolding there instead.

**THE HORIZON.** Don't invent nodes for work whose shape you can't know yet. Plan honestly to where you
stop knowing, and leave the rest for an `unfold` later.

**Encoding gotchas (verified against the code):**
- You **cannot supply contract clauses at seed** — the server forces `contract: {}`. Shapes are published
  later, per node, via `settle`. At seed you only *declare that a node owes a contract* (via others'
  `contract` reliances on it); the clauses come when its owner settles.
- Seed refuses a defective plan (all measured at 0 false-positives): `CYCLE` (a reliance loop),
  `EMPTY_WRITE_SET` (a work node that writes nothing — exempt only for decisions and re-planning nodes),
  `DANGLING_RELIANCE` (relies on an id not in the plan), `UNPUBLISHABLE_CLAUSE`. Fix these before seeding.

**Then seed it** (planner): `mcp__vatis__seed({ planId, goal, nodes: [...] })` — one call installs the
network into the plan you created. To add bets, call `mcp__vatis__bet({ planId, ... })` per bet after.
(On a plan-scoped executor connection, omit `planId` — it's implicit.)

**`seed` APPENDS — this is how you add the next milestone.** Plan honestly to your horizon, seed it, work
it; when it's done, `seed` again with the *next* stretch. New nodes may rely on already-finished ones
(`{ on: "<installed-node-id>", kind: "artifact" }` and the like) — reliances are validated against the
whole plan, not just this batch. Two rules: **use fresh ids** (re-seeding an existing id is refused — seed
only adds, it never overwrites installed work; to change an installed node use `amend_intent`/`widen`/an
`unfold`), and the milestone's own internal structure must be sound (no cycle, no dangling edge onto an id
in neither the batch nor the plan). This is the milestone-by-milestone rhythm: `create_plan` once, then a
`seed` per milestone.

Before seeding, show the human the node list and the bets and get a nod — a wrong write-set is expensive
to correct.

---

## 2. Guard — leave it OFF to start

Vatis enforces walls at the point of action via two Claude Code hooks. **For a first pass, do not install
them.** You get all the planning value (frontier, brief, settle, bets, surgical recall bookkeeping)
without the enforcement risk, and nothing can wrongly refuse a write.

When you *do* want enforcement (later, on a small slice): call `mcp__vatis__guard_setup` (planner-only),
which returns the exact files to write (`.vatis/guard.json`, three `.sh` hook scripts) and the
`.claude/settings.json` hooks to merge. It needs `jq`, `curl`, `git` on PATH. To soften it: set
`"shellWall": false` in `.vatis/guard.json` (quiets the shell-write refusal; recall still bites). To turn
it fully off: remove the hooks. There is no log-only mode — a refusal is a blocked write or nothing.

---

## 3. Work the loop (executor)

1. `mcp__vatis__frontier` — what's runnable now, and why the rest isn't.
2. `mcp__vatis__claim(nodeId)` — take a lease; returns a `leaseId` (your permit for settle/attest/release).
3. `mcp__vatis__brief(nodeId)` — everything to execute it: goal, the settled contracts you rely on, your
   write-set, your verification, findings you inherit.
4. `mcp__vatis__settle(leaseId, { clause: "<the signature/schema/route you promise>" })` — publish your
   shape **the instant you've decided it**, before writing code. This releases everyone waiting on the
   shape. Settling late throws the whole point away.
5. Do the work. Run your `verification`.
6. `mcp__vatis__attest(leaseId, evidence)` — submit `{ verification, command, exitCode: 0, commit, files,
   summary }` + the material choices you made. The system adjudicates; you don't declare done. **Attest
   each slice as you finish it, before merge** — `attest` is retroactive-hostile: the cited `commit`'s
   files must sit inside your write-set, and once slices are merged together no single historical commit is
   cleanly ⊆ your write-set. Preflight `git show --name-only <commit>` ⊆ your write-set first.

**Never let your agent die holding a lease.** Before the session ends or is `/clear`ed, `attest` (done) or
`release` (unfinished) — the clean exit. A lease left held auto-reclaims after **30 min** of silence (or a
manager `revoke`s it sooner), but don't lean on that: it's a backstop, not a workflow.

**Leases are SESSION-BOUND — a `checkpoint` does not survive a `/clear`.** The `leaseId` lives in your
context and the guard binds the lease to your session id; a context clear loses both, so the *same* lease
cannot be resumed even by the same agent label. After a clear, `claim` the node again (the lapsed lease
reclaims cleanly). `checkpoint` records progress for the human — it is not a resumable-lease token.

- `mcp__vatis__heartbeat(leaseId)` — proof you're still alive on a node during a long *silent* step (a slow
  build, a big test run, a deep think). Every ordinary verb (`settle`/`widen`/`discover`/`amend_intent`)
  already renews your lease, so while you're making progress you never need this; reach for it only when
  real work happens with no Vatis call to show for it, and beat every few minutes. If it comes back refused,
  your node was reclaimed while you were away — stop, and `claim` again (or ask `frontier` what's yours).

When reality diverges:
- `mcp__vatis__discover(...)` — report a fact; optionally break a contract clause / artifact / bet you've
  *falsified* (with real evidence — a failing run of the declared verification).
- `mcp__vatis__checkpoint(leaseId, summary)` — pause part-way for a human to review, **keeping your lease**.
  Use this — NOT `split` — when a person wants to look before you continue: your node isn't mis-sized, you
  just want eyes on it. Keeps the node yours and in progress, lands in the human's inbox, blocks nothing.
- `mcp__vatis__amend_intent(leaseId, intent)` — correct your node's stale description (a planning decision
  changed what it's for). Touches only the wording, never the write-set.
- `mcp__vatis__widen(leaseId, paths)` — add files you've discovered you must write to your own write-set.
  Additive and safe: refused (`WIDEN_CONTENDED`) if another live lease holds a path, and you still cite
  every added file at `attest`. This is the answer to "I need to edit a file that's in my `reads`, not my
  `writes`" — widen to it (if it's free) rather than re-plan. Add only files you're really about to write.
- `mcp__vatis__split(leaseId, why)` — "this doesn't fit in one context." Drops your lease, raises a
  re-planning node for a planner. Only for genuine mis-sizing — for a review checkpoint use `checkpoint`.
- `mcp__vatis__release(leaseId)` — give up a lease unfinished; the node's state is untouched.
- **Recalled?** If a discovery moves your ground, your next write is refused with `RECALLED`. Stop. Do not
  work around it. Claim the repair node the discovery raised, or a node it invalidated — that's the door.

Also in `brief`: `readOwners` — nodes that own a file you `read` and have decided something about it
(settled a clause / reported a finding) that reaches you through no reliance edge. Go look before you build.

Planner-only repair: `unfold` (install replacements + `retire` dead nodes), `bet`, `revoke`, `seed`. To
**decompose a node that already settled a contract**, or to **re-point reliers off a node you're
replacing**, use `unfold(..., succeed:[{from, to}])` instead of `retire`: the heir `to` inherits the
contract, every reliance on `from` moves to it, and `from` retires — all at once. `retire` alone is refused
while anyone still relies on the node (`RELIES_ON_RETIRED`), and `succeed` is the only thing in the system
that repoints a reliance. `frontier.answerable` flags a human gate an unheard finding already answers.

Human-only (not yours): adjudicating a `human` decision node, and absorbing/acknowledging findings —
those happen on a separate human surface a signed-in person reaches, which your token cannot.

**Need to write a file that isn't in your write-set?** If it's free (no other live lease holds it),
`widen` to it — that's the normal case, including a spec in your `reads` your verification proved wrong.
Only when the file is genuinely CONTENDED, or the work needs a whole **new node**, does it leave your
hands: an executor cannot add a node (that's a planner's `unfold`, which needs a re-planning node you don't
hold — do NOT `split` to manufacture one). For those, **`discover` it as a finding** with evidence, and
attest what you own now; `brief`'s `readOwners` carries your finding to whoever reads that file next. A
human `acknowledge`s or `absorb`s it — but note what `absorb` does and doesn't do: it raises a **decision
node for a person**, not a re-planning node. It is how a question reaches a human, not how a planner gets
authority to install nodes. For that, see "where re-planning nodes come from" in §5.

---

## 4. When you're stuck

`frontier` and `brief` explain themselves — they tell you *why* a node is or isn't runnable. The first
question when something looks stuck is usually "why is X *not* on the frontier?" — read the frontier's
derivation; it says. If a wall refuses you, the refusal text names the legal move. Follow it; don't route
around it.

**The one deadlock a plan can grow on its own: `UNPUBLISHABLE_CLAUSE`.** Node X contract-relies on Y for a
clause Y never published — a typo, or a clause that actually belongs to a third node. This is *legal at
seed* (the defect's guard is "Y is fulfilled"), and becomes a wedge the moment Y attests, having settled
everything it ever meant to promise. Symptoms: `frontier` says X is waiting on a clause Y "NEVER
PUBLISHED", Y can't be re-claimed (`FULFILLED`), X can't be claimed (`UNMET_RELIANCE`, so `split` is out),
and `seed` refuses every later milestone for a defect that batch didn't introduce.

Vatis now catches this at the attestation that creates it: `attest` returns `stranded` + `replan`, and the
repair node is already on the frontier. **On a plan that predates that** — or if you meet one in the wild —
the repair is the same, done by hand:

1. `discover({ finding: "<why X can never run>", breaks: { kind: "artifact", node: X } })` — **with no
   `evidence`**. The *only* thing that mints a re-planning node is a `discover` whose fallout is
   non-empty, and an artifact break needs no citation when the graph already proves the node is dead.
   Break **X**, not Y.
2. `claim` the returned `unfold_f_…`, then `unfold(nodes: [X-v2 with the reliance pointed at the clause's
   real owner], succeed: [{ from: X, to: X-v2 }])`.

⚠️ **Do not go hunting for a failing run to cite.** The obvious move — run X's declared `verification`
and cite the red — works only if that standard can actually go red, and on a wedged node it usually
**passes**: nothing was ever built for it to fail on (`pnpm typecheck` over an empty scaffold exits 0).
Citing a pass is refused (`EVIDENCE_NOT_A_FAILURE`), X can't be claimed so `split` is out, and that is
every door shut on a node that is provably dead. That dead end is precisely why the no-citation
exemption exists. Omit `evidence`.

`succeed` is not optional: `retire` alone is refused (`RELIES_ON_RETIRED`) because everything behind X is
still pointing at it. `succeed` repoints them onto the heir first, then retires the corpse — and a retired
node is skipped by the structural walls entirely, so the plan is seedable again.

## 5. Working within the walls — the gotchas that trip agents

These are the walls agents most often fight. Each is deliberate; the move is to work *with* it, not around.

- **Declare the verification that actually PROVES the node.** A node's `verification` is the command that,
  run green, means it's done — and the command a break must show FAILING later. If correctness is
  type-level (a schema, a signature, a nullability), declare a typecheck (`pnpm typecheck` / `tsc
  --noEmit`), **not** `pnpm test` — a passing test over a type-only defect proves nothing, and you won't
  be able to `discover`-break it with evidence when it's wrong. Declare the standard that can actually go
  red.
- **You can't silently fix a settled clause or an installed edge — and that's the point.** A settled
  contract is a promise others built against; rewriting it under them is refused (`CONTRACT_CHANGE`). The
  sanctioned correction is to `discover`-break the wrong clause/artifact with a failing run of *its own*
  declared verification (or `split` your own node) — that withdraws it and lets it be re-settled/re-planned
  cleanly, recalling only the reliers on that exact clause. **A dependency you were mis-wired onto and
  can't satisfy** is the `UNPUBLISHABLE_CLAUSE` case in §4: break the wedged node's artifact (no citation
  needed — the graph is the proof) to raise a re-planning node, then a planner `unfold`s a replacement
  with `succeed:`. There is no verb that re-points a reliance in place, and `absorb` will not give you
  one. Don't try to re-settle around the wall.
- **Two artifact breaks need no citation, and neither is a loophole.** Your OWN node under your own lease
  (that's `split`), and a node the GRAPH already proves can never run — relying on a clause that can never
  be published, on a retired node, or on one not in the plan. In the second case the server re-derives the
  claim itself, so there is nothing for you to cite. Everything else still needs a failing run of the
  standard that node declared.
- **Where re-planning nodes come from — the question that has exactly one answer.** Only `discover` with a
  break whose fallout is non-empty mints one (`split` and `checkpoint` are `discover` underneath; the
  system also raises one at `attest` when an attestation strands a relier). `absorb` mints the *other*
  kind — a decision node with `adjudicator: "human"`, which `unfold` refuses (`NOT_YOURS_TO_ADJUDICATE`)
  because what it promises is a judgement, cleared by a person. **Nothing else mints either**, and you
  cannot author `unfolds` at `seed`. If you're hunting for a route to a re-planning node, it is the
  `discover` break or nothing — and if you find yourself reaching for `absorb`, you are about to raise a
  bar only a human can clear and then be refused at it.
- **`attest` returns `stranded` — read it.** Normally empty. When it isn't, your attestation just made
  someone else's reliance unmeetable (they named a clause you never published, and you can't be claimed
  again to publish it). You did nothing wrong and there's nothing for you to fix; the system has already
  raised the repair node named in `replan`. Hand it to a planner, or claim it if you are one.
- **Owed work that isn't a node must be a FINDING, not a buried clause.** If you close a slice but leave
  real work undone that nobody owns yet, don't record it only inside a contract clause — it will vanish
  from `frontier`. `discover` it as a finding so it surfaces in the manager's unheard inbox and can be
  raised into work.
- **External preconditions are yours to check — the frontier can't.** `frontier` tracks node-to-node
  reliances only; it knows nothing about the world. So a node can read READY while being physically
  unrunnable (a namespace that doesn't exist, an entitlement not enabled, an endpoint never built). State
  any such precondition in the node's `intent`, and CHECK it before you `claim` — a green frontier is not
  a promise the world is ready.
