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

This skill lives in the `loop-engineering/` folder of the
[AI-skills repository](https://github.com/harshakondreddy/AI-skills).

### Claude Code

Download or clone AI-skills, then copy the entire `loop-engineering`
folder into `~/.claude/skills/loop-engineering/`.

Keep `SKILL.md`, `AGENTS.md`, and `references/` together.

### Other Agents

Read [SKILL.md](SKILL.md) and follow your agent's supported method for
loading skills or project instructions.

[AGENTS.md](AGENTS.md) provides additional guidance. If you incorporate
its instructions into an existing project instruction file, merge them
rather than overwriting existing instructions, and update relative paths
to point to this skill's `SKILL.md` and `references/` folder.

Support varies by tool and version; this repository does not establish
compatibility with every agent.
