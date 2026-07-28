# loop-engineering

An Agent Skill that runs infrastructure and coding work through a structured execution loop
instead of diving straight in — so nothing gets marked "done" without evidence.

```
PLAN → ACT → OBSERVE → VERIFY → REFLECT → REPORT
```

If VERIFY fails, it loops back to ACT. "Should work now" is not verification; command output,
logs, a diff, or an API response is.

Built for Kubernetes, RHEL, Azure, Kubespray, CI/CD, migrations and incident fixes — the kind
of work that fails silently and needs proof it actually worked. It auto-scales: a one-off fix
gets a fast inline loop, multi-system work gets broken into work packages with explicit success
criteria.

## Two phases

The split matters more than the six step names.

**PLAN is a gate.** Nothing runs until every unknown is resolved. The agent restates the goal
concretely, then asks — in one batch, not dribbled out — about environment, dependencies,
access, conventions and constraints. Critically, it works out *how it will test the result*
before building anything, using only commands it can run itself. It captures a baseline, states
a rollback, and names any irreversible action it needs approval for.

**Then it executes unattended.** `ACT → OBSERVE → VERIFY → fix → VERIFY` runs to completion
with no check-ins. It builds it, starts it, tests it, finds its own bugs, fixes them, tests
again. A failing test is the next iteration, not a reason to hand the work back.

The trade is deliberate: front-load all the interaction, then get out of the way.

## Guardrails

- **No invented evidence.** Only real output counts. If the agent can't run something it says
  so and stops, rather than producing something plausible. Secrets redacted before pasting.
- **It tests its own work.** Never "please run this and paste the output", never "screenshot
  the dashboard". If a verification method would need you, that's caught in PLAN and replaced
  with a headless browser, a direct endpoint check, or a written test.
- **Irreversible actions are pre-approved, not ad-hoc.** `delete`, `drain`, `terraform apply`,
  migrations, force-push — named during PLAN with what they affect, approved once, then run
  freely. Anything irreversible that *wasn't* pre-approved halts immediately. The original
  request is never treated as approval.
- **Persistence with a thrash guard.** It keeps looping while each attempt tests a genuinely
  different theory. It stops at a real dead end: no new hypothesis, the same failure signature
  three times running, or it needs something only you can supply.

Plus explicit report shapes for *blocked* and *failed* — not just *done*.

## Install

**Claude Code** — clone into your skills directory:

```bash
git clone https://github.com/<you>/loop-engineering.git ~/.claude/skills/loop-engineering
```

**Any other agent** — clone anywhere, then create the file your tool reads in *your* project
and point it at this repo. The repo ships only `AGENTS.md`; the paths below are files you
create, not files you'll find here.

| Tool | Create this in your project | How |
|---|---|---|
| Codex, Zed, Aider, Jules, Cline | `AGENTS.md` | copy this repo's `AGENTS.md` |
| Antigravity | `AGENTS.md` or `.agents/rules/loop-engineering.md` | copy as-is |
| Gemini CLI | `GEMINI.md` or `AGENTS.md` | copy as-is |
| Copilot / VS Code | `.github/copilot-instructions.md` or `AGENTS.md` | copy as-is |
| Windsurf / Devin | `.windsurf/rules/` or `.devin/rules/loop-engineering.md` | copy as-is |
| Cursor | `.cursor/rules/loop-engineering.mdc` | copy, then **add the frontmatter below** |

Cursor won't activate an `.mdc` rule without YAML frontmatter. Prepend this before the copied
content:

```yaml
---
description: Structured PLAN/ACT/OBSERVE/VERIFY/REFLECT/REPORT loop for infrastructure and coding tasks
alwaysApply: false
---
```

`alwaysApply: false` plus a description makes it agent-requested — Cursor pulls it in when the
task matches instead of loading it on every message.

Wherever it lands, keep `SKILL.md` and `references/` reachable from that file, since `AGENTS.md`
points at them by relative path. Simplest is to clone the whole repo into the project (or as a
submodule) and copy `AGENTS.md` to the root.

[AGENTS.md](https://agents.md) is the Agentic AI Foundation standard, so most tools read it
without extra setup.

## Files

```
├── SKILL.md                              the skill
├── references/full-initiative-template.md template for multi-system work
├── AGENTS.md                             portability notes for non-Claude agents
├── README.md
└── LICENSE
```

## License

MIT — see [`LICENSE`](LICENSE).
