---
name: vatis-feedback
description: Use when you are working with Vatis (any role — executor, planner, or manager) and VATIS ITSELF gets in your way — a refusal you couldn't understand, a verb that behaved surprisingly, a docs/skill gap, or a capability Vatis is missing. Report it with mcp__vatis__feedback so the product improves. Trigger when you hit a Vatis wall you can't work out, when the skill said X but the system did Y, or when you think "Vatis should be able to… but there's no verb for it".
---

# Reporting feedback on Vatis

You are using Vatis to plan and build. When **Vatis itself — not your plan** — gets in your way, file it
with `mcp__vatis__feedback({ body })`. It lands where the people who maintain Vatis can read it. This is
the loop that makes the tool better; a wall you hit silently is a wall the next agent hits too.

## Feedback vs `discover` — get this right

- **`discover`** = a fact about your **plan's work** (a falsified contract, a measurement, a learning). It
  is filed per-plan into the world and can break promises. **Not** for product complaints.
- **`feedback`** = a fact about the **product**. The tool confused you, surprised you, or fell short.

## When to file

- A refusal whose reason you could **not** work out from the message.
- A verb that did something other than what its skill/description said.
- You wanted to do something reasonable and there was **no verb for it**.
- A skill documented X but the system did Y.
- Anything that cost you time and would cost the next agent the same time.

## How to file well — concrete beats polite

Name the verb / wall / skill, what you **expected**, and what you **got**. Good reports:

- *"seed refused `adjudicator: reviewer` even though vatis-plan documents that tier — had to author the gate as `human` instead."*
- *"no legal way to add a milestone to a finished plan: `unfold` needs a re-planning lease a fulfilled plan can't produce, and `seed` rejected reliances onto the installed nodes."*
- *"`frontier` on a 35-node plan returned 190k characters and blew my context — I had to `jq` it down."*

Weak report: *"the tool is confusing."* — nobody can act on that.

## Rules

- It is **free and always available**, in every role. You never need permission to file.
- Don't stop your work to file — note it, send one `feedback` call, move on.
- Keep plan work in `discover`; keep product feedback here. Mixing them loses both signals.
