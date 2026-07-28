# AGENTS.md

This repo contains one Agent Skill: **`loop-engineering`**, defined in [`SKILL.md`](SKILL.md).

## When to use it

Read `SKILL.md` and follow it when the user asks you to build, code, implement, set up,
install, fix, debug, deploy, migrate, upgrade, roll out, refactor, automate, plan out, tackle
or figure out a technical task — especially when it spans multiple systems, is risky or hard to
reverse, or could fail silently and needs proof it actually worked.

Otherwise, work normally. Don't preload it.

## What it does

Two phases, and the split matters more than the six step names.

**Phase 1 — PLAN is a gate.** Nothing runs until every unknown is resolved: the concrete goal,
dependencies, access, environment, the exact commands that will prove the work correct, the
rollback, and approval for any irreversible action. Questions are asked once, batched, up front.

**Phase 2 — execute unattended.** `ACT → OBSERVE → VERIFY → fix → VERIFY` runs to completion
without check-ins. The agent builds it, runs it, tests it, finds its own bugs and fixes them.

## Non-negotiable, whatever agent you are

1. **Never invent evidence.** Only quote output you actually received. If you couldn't run
   something, say so and stop — don't substitute reasoning for observation, and never format an
   expected result as if it ran. Redact secrets before pasting.
2. **Test your own work.** Run the suite, curl the endpoint, check the exit code. Never ask the
   user to verify, screenshot, or paste output back to you. If the verification method agreed in
   PLAN turns out not to work, that's a blocker — report it, don't downgrade to "should be
   working now."
3. **Don't check in mid-flight.** A failing test is the next iteration, not a reason to stop.
   Keep looping while each attempt tests a genuinely different theory. Stop only at a real dead
   end: no new hypothesis, the same failure three times running, or you need something only the
   user can supply.
4. **Irreversible actions need prior approval.** `delete`, `drain`, `terraform apply`,
   migrations, force-push, decommission. These get named and approved during PLAN, then run
   freely. Anything irreversible that wasn't pre-approved halts immediately — the original
   request is not approval.

`references/full-initiative-template.md` is the expanded form for larger multi-system work.

## Portability

The skill names no vendor-specific tools — it's process and prose, so it runs as-is on any
agent. Two places it assumes a capability you may not have:

- **A task list** (Tier 2/3 work packages). If you don't have one, keep a plain checklist in
  your replies and restate it as items close.
- **Sub-agents** for running independent work packages concurrently. If you don't have them,
  run the packages yourself in dependency order. Nothing in the loop requires parallelism.

If you lack something the skill assumes, say so once and do the parts you can. Don't skip
VERIFY — that's the part the whole skill exists for.
