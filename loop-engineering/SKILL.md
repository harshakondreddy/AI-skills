---
name: loop-engineering
description: >
  Runs development and infrastructure work — building, coding, debugging, deploying,
  installing, migrating, refactoring, incident fixes across Kubernetes, RHEL, Azure,
  Kubespray, CI/CD — through a PLAN, ACT, OBSERVE, VERIFY, REFLECT, REPORT loop. Front-loads
  every unknown into an interrogation phase: goal, scope, dependencies, environment, how the
  work will be tested, and which irreversible actions need pre-approval. Once PLAN is agreed
  it executes unattended — building, running, testing and fixing its own bugs in a loop
  without step-by-step check-ins — and never marks anything done without real evidence. Use
  whenever the user asks to build, code, implement, set up, install, fix, debug, deploy,
  migrate, upgrade, roll out, refactor, automate, plan out, tackle, work through or figure
  out a technical task, or says "loop engineering".
---

# Loop Engineering

A loop for getting technical work actually finished: interrogate first, then run unattended,
and prove every claim with real evidence.

**The contract:**

- **All the questions come up front.** PLAN is where you interrogate. Ask everything you need —
  it's cheap now and expensive later.
- **Then you run it yourself.** Build it, start it, test it, read the output, fix what broke,
  test again. Don't hand work back to the user that you could have done.
- **You don't check in mid-flight.** No "shall I continue?", no "please run this and paste the
  output", no "can you screenshot it?". If you'd need to ask during execution, you should have
  asked during PLAN.
- **You don't claim anything you didn't observe.** Ever.

## The loop

1. **PLAN** — interrogate until there are no unknowns left. Nothing runs until this closes.
2. **ACT** — do the work.
3. **OBSERVE** — capture the real result.
4. **VERIFY** — test it yourself against the done-criteria.
5. **REFLECT** — what changed, what's open, what's now risky.
6. **REPORT** — Done, Failed, or Blocked.

Steps 2–5 repeat autonomously until done or genuinely stuck. Step 1 happens once, properly.

---

# Phase 1 — PLAN (the gate)

**Nothing executes until PLAN closes.** The purpose of this phase is to make the rest of the
work runnable without the user in the room. Every question you don't ask here becomes an
interruption later, or a wrong assumption baked into the result.

Scale the depth to the task. A one-line fix or a read-only investigation needs almost none of
this. Anything that builds, installs, deploys, touches multiple files, or changes shared state
gets the full pass.

## 1.1 Nail the goal

Don't accept the request at face value. Restate it as a concrete, checkable outcome and get
confirmation.

- What does *done* actually look like? Not "add auth" — "a user can register, log in, and hit
  `/api/me` with a valid session cookie; invalid tokens get 401."
- What's explicitly **out** of scope for this pass?
- Is there a deadline, or constraints on approach (must use X, must not touch Y)?

If the request is vague, say so and propose a specific interpretation for the user to correct.
A wrong guess corrected in one line beats an hour building the wrong thing.

## 1.2 Find what's missing — then ask, in one batch

Work through what the task actually needs and identify every gap. Ask them **all at once**, as
a numbered list. Don't dribble questions out one at a time.

Cover, as applicable:

- **Environment** — dev / local / staging / prod. Never guess this.
- **Existing state** — is this greenfield or existing code? Where does it live? What's the
  current behaviour?
- **Dependencies** — languages, runtimes, versions, package managers, system libs. What's
  already installed vs what you'll need to install. Version constraints that matter.
- **Access and credentials** — API keys, registry auth, cluster context, DB connection, cloud
  subscription. If you need it and don't have it, that's a PLAN-stopper, not a mid-run surprise.
- **Config and secrets** — env vars, config files, what values they should hold.
- **Conventions** — existing patterns, style, test framework, build tooling, target platform.
- **Data** — fixtures, sample input, an account to test with.

State the assumptions you're making on anything you *didn't* ask about, so they can be
corrected cheaply.

## 1.3 Decide how you will verify it — before you build it

This is the step that makes unattended execution possible. **Do not proceed until you know
exactly how you'll prove the work is correct, using only what you can run yourself.**

Work out and state:

- The exact command(s) that will demonstrate success — `curl` against the running service, the
  test suite invocation, `kubectl get`, a build command, a script you'll write for the purpose.
- How you'll run the thing under test — dev server on a port, container, background process —
  and how you'll shut it down afterwards.
- What output constitutes pass, concretely: HTTP 200 with a specific field, exit code 0, "12
  passed", `3/3 Ready`.

**Never plan a verification step that requires the user to do something.** "Ask the user to
open the browser and confirm", "have them screenshot the dashboard", "the user can then test
it" — all invalid. If a UI genuinely must be exercised, use a headless browser, hit the
endpoint directly, assert on the rendered HTML, or write a test.

If no verification exists yet, building one is part of the task.

### Probe before you commit to a method

**Don't assume a capability — check for it.** The failure this prevents: planning "I'll load it
in Chrome and click through it", then discovering in Phase 2 that you have a shell and a
filesystem and no browser, and handing the work back.

Run the checks. They're cheap and read-only:

```
which node npm npx python3 pip docker git curl jq
npx playwright --version    # or: python3 -c "import playwright"
docker info                 # daemon actually running, not just installed
curl -sI https://registry.npmjs.org  # is there network / can you install at all
```

Also check what's wired into *this* session — browser-automation tools, connected
integrations, sandbox network policy. A tool that exists on the machine but isn't reachable
from where you're executing doesn't count.

`references/verification-playbook.md` maps common project types — browser extension, web UI,
CLI, HTTP service, library, container, IaC, scheduled job — to how each is self-tested and what
it needs installed. Read it when the project type isn't obviously testable.

### When the capability is missing — the ladder

Work down. Stop at the first rung that gives you a **real** check.

1. **Install it.** Project-local installs (`npm i -D`, `pip install` in a venv, `npx`) are
   reversible — do it without asking. System-level installs (`apt`, `brew`, `sudo`, anything
   touching the machine outside the project) get named in PLAN for approval.
2. **Substitute a different real check.** No browser? Assert on the rendered HTML string, unit
   test the logic directly, validate the manifest against its schema, exercise the API the UI
   would call. Lower fidelity is fine. Fabricated is not.
3. **Narrow the definition of done.** Say plainly what you *can* prove and what you can't:
   "I can verify the extension builds, the manifest is valid, and the content-script logic
   passes unit tests. I cannot verify it renders correctly in a real Chrome window."
4. **Declare it blocked — in PLAN, not after.** If nothing above yields a real check, say so
   before building and agree what happens instead.

State any capability gap in the PLAN summary. The user deciding "that's fine, ship it" is a
valid outcome. The user finding out afterwards is not.

### Screenshots and artifacts

Take your own screenshots. **Never ask the user to capture, open, or look at something for
you.** Playwright does it in one line:

```js
await page.screenshot({ path: '.loop/artifacts/login-after-fix.png', fullPage: true });
```

Write run artifacts — screenshots, logs, test output, before/after configs — to
`.loop/artifacts/` in the project. Add `.loop/` to `.gitignore`. Name files so they're
self-explanatory later (`checkout-500-error.png`, not `screenshot1.png`), and reference the
paths in REPORT so they can be opened without hunting.

If you can see images, look at your own screenshots when a visual bug is suspected — layout
breakage, invisible text, overlapping elements, wrong theme. Assertions catch "the element
exists"; only looking catches "it's rendered behind the header." If you can't see images, say
so and rely on assertions.

A screenshot is a record, not proof. The evidence in REPORT is still the assertion that passed.

## 1.4 Scope: what you're allowed to touch

Establish reach before executing, so nothing is discovered mid-run.

**Local project** — the working directory. Free rein within it.

**Other repos and paths on disk** — name them in PLAN with their paths. Reading outside the
working directory is fine once declared. Writing to another repo is a separate approval — say
which one and why.

**Remote hosts over SSH** — name the hosts in PLAN. Read-only diagnostics run unattended:
`systemctl status`, `journalctl`, `kubectl get`, `df`, `ps`, config reads, log tails. Anything
that **writes** to a remote host — editing a file, restarting a service, installing a package,
applying a manifest — is named in PLAN and approved before execution.

**Live and remote URLs** — `GET`/`HEAD` against a declared URL runs freely. Any `POST`, `PUT`,
`PATCH` or `DELETE` against a live endpoint is treated as irreversible: name it in PLAN, get
approval. Never point a load test at production without explicit sign-off.

Discovering mid-run that you need a host, repo or endpoint you didn't declare means **stop and
ask** — same rule as an unapproved irreversible action. It's a PLAN miss; note it in REFLECT.

## 1.5 Baseline and rollback

Before anything mutating:

- **Baseline** — capture current state so you can prove a delta and get back. Working git tree
  committed or stashed, `kubectl get -o yaml > /tmp/before.yaml`, current config saved,
  the failing output recorded.
- **Rollback** — one line on how you undo it. If you can't state one, stop and reconsider.

## 1.6 Pre-approve the irreversible actions

Classify what the task will require:

| Class | Examples | Rule |
|---|---|---|
| **Read-only** | `get`, `describe`, `logs`, `--dry-run`, `plan`, `diff`, test runs | Run freely. |
| **Reversible** | write a file, edit config you've backed up, restart a service, scale a deployment, install a package, commit locally | Run freely during execution. |
| **Irreversible** | `delete`, `drain`, `terraform apply`/`destroy`, DB migration or drop, force-push, decommission, cert rotation, deleting a volume, anything in prod that can't be trivially undone | **Name it in PLAN and get explicit approval.** |

List every irreversible action you expect to need, with what it affects. Once approved, those
run unattended — no further check-ins.

**Anything irreversible that was not pre-approved halts immediately**, even mid-execution. The
original request is not approval. Discovering you need one you didn't anticipate means stopping
and asking — that's a PLAN miss, and worth noting in REFLECT.

Prefer the reversible path where one exists: cordon before drain, scale to zero before delete,
`--dry-run=server` before apply, snapshot before migrate, branch before rewriting history.

## 1.7 Close the gate

Before running anything, state in a few lines: the goal, the tier, the verification method,
the rollback, and the irreversible actions you're asking approval for. Then wait for the
go-ahead — this is the **last** time you pause unless something genuinely blocks.

---

# Phase 2 — Execute (unattended)

Once PLAN closes, run ACT → OBSERVE → VERIFY → (fix) → VERIFY until it's done. **The user is
not part of this loop.**

**Do the work yourself.** Install the dependency. Write the file. Start the server. Run the
test. Read the log. Find the bug. Fix it. Run it again. A failing test is not a reason to stop
and report — it's the next iteration.

## Rule 1 — Never invent evidence

The rule everything else rests on. Breaking it makes the whole loop theatre.

- **Only quote output you actually received.** Never write example, expected, or remembered
  output and present it as a result. No "it would show something like…" formatted as if it ran.
- **If you couldn't run it, say so and stop.** No access, missing credential, command not
  installed, no network. State it plainly. Do not substitute reasoning for observation.
- **Mark inference as inference.** If you're concluding rather than observing, the word is
  "inferred", not "verified".
- **Redact before pasting.** Real output carries secrets — tokens, kubeconfig data, connection
  strings, `Authorization:` headers, `.data` in a Secret, SAS URLs, passwords in
  `ansible-playbook -vvv`, `.env` contents. Replace with `<redacted>`.

If you cannot produce real evidence, the task is **blocked**, not done.

## Rule 2 — Test it yourself, every time

VERIFY means *you* ran something and read the result. Run the suite. Curl the endpoint. Check
the exit code. Diff against the baseline.

Never ask the user to verify on your behalf. If the verification method you agreed in PLAN
turns out not to work, that's a blocker — say so, don't silently downgrade to "should be
working now."

Where a baseline exists, show the delta: "was 0/3 Ready, now 3/3" beats "pods are healthy."

## Rule 3 — Keep going until done or genuinely stuck

When VERIFY fails, that's information, not a stopping point. Form a new hypothesis and go
again.

**Keep looping while each attempt tests a genuinely different theory.** Persistence is the
point — most real work takes several iterations.

**Stop and report only at a true dead end:**

- You have no new hypothesis left to test.
- The same failure signature has repeated 3 times running — you're thrashing, not iterating.
- You need something only the user can supply: a credential, an access grant, a product
  decision, an irreversible action that wasn't pre-approved.
- Continuing would require a fundamentally different approach than the one agreed in PLAN.

Repeating the same fix hoping for a different outcome is not persistence. Each cycle must
change something you can name.

**Distinguish two failure modes:**

- **Verified failed** — the check ran, criteria weren't met. Iterate.
- **Could not verify** — the check itself didn't run: no access, timeout, tool missing. This is
  neither pass nor fail. Don't retry blindly; treat it as a blocker.

**If ACT partially applied** — some of it landed, then failed — say so explicitly and list
what's now in an in-between state. Either roll back to baseline or clearly state what's left
half-done. Before re-running anything, check it's idempotent: re-applying a manifest is safe,
re-running a migration or an append is not.

---

# Tiers

Size the task first; state which tier and why in one line.

**Tier 1 — Quick Loop** (default)
One system, one change, reversible, approach is clear. A bug fix, a config change, a small
feature, a broken pod. → Minimal PLAN: goal, how you'll verify, rollback. Then execute.

**Tier 2 — Standard** (multi-step, 2+ systems/components, or a stated project/migration/rollout)
→ Full PLAN interrogation. Break into 2–5 **work packages**. Note which are independent and
which depend on another. Track with whatever task-list mechanism you have, or a checklist.

**Tier 3 — Full Initiative** (large, multi-system, production risk, hard to reverse, spans days)
→ Full PLAN plus `references/full-initiative-template.md`: execution waves, dependency diagram,
validation gates, risk controls, release readiness.

Default down a tier when ambiguous. **Escalate mid-task if** the change touches a second
system, rollback stops being simple, prod turns out to be in scope, or you hit an irreversible
action that wasn't pre-approved. Say you're escalating and why.

## Work package template (Tier 2 and 3)

```
WP-0X: <short name>
Objective: <what this accomplishes, one line>
Depends on: <WP-0Y, or "none">
Environment: <dev | staging | prod>
Verification: <the exact command you will run, and what output means pass>
Rollback: <how this is undone>
Success criteria:
  - <specific, checkable condition>
Evidence required: <command output, test pass, log line, curl response>
```

Not done until the evidence exists and has been looked at — not described as forthcoming.

## Evidence, concretely

- test suite output with an actual pass count, not "tests should pass"
- build/compile output, exit codes
- `curl -w '%{http_code}'` against the running service
- `kubectl get/describe/logs` showing the expected state
- `systemctl status` / `journalctl`
- `ansible-playbook` recap (`ok=/changed=/failed=`)
- `az` CLI confirming a resource's actual state
- a before/after diff against the PLAN baseline

---

# Report format

Three shapes. Keep them short.

**Success**
```
Done: <what was accomplished>
Evidence: <the specific proof, redacted>
Open: <anything unresolved, or "none">
```

**Blocked** — needs something only the user can give
```
Blocked: <what stopped it>
Needed: <access, credential, decision, or approval required>
State: <what changed so far, or "nothing changed">
```

**Failed** — iterated to a dead end
```
Attempted: <what was tried, how many cycles, what each showed>
Result: <what the evidence actually showed>
State: <current state vs baseline — rolled back, or left as-is and why>
Theory: <best current hypothesis>
```

For Tier 2/3, roll up per work package, then one overall summary line.

---

# Worked example (Tier 2)

> **User:** add rate limiting to the API

**PLAN — questions asked in one batch:**

> Before I start, six things:
> 1. Which service — `api-gateway`, or the Flask app in `services/core`?
> 2. Limit per what — IP, API key, or authenticated user?
> 3. Target numbers? I'd suggest 100 req/min per key, burst 20 — correct me.
> 4. Behaviour on limit: 429 with `Retry-After`, or queue?
> 5. Backing store — you have Redis in compose; use it, or in-memory per-instance?
> 6. Is this going to prod this week, or dev only for now?
>
> Assumptions unless you say otherwise: Python 3.11 per `pyproject.toml`, pytest for tests,
> I'll add `slowapi` as a dependency.
>
> **Verification:** I'll run the app with `uvicorn` on :8000, fire 120 requests in 60s with a
> script, and assert the first 100 return 200 and the rest 429 with `Retry-After` set. Plus
> unit tests for the limiter. All self-run — nothing needed from you.
>
> **Rollback:** feature-branched; `git checkout main` reverts everything.
> **Irreversible actions needed:** none.
>
> Tier 2 — touches app code, config and the compose file.

**Execute (unattended after sign-off):**

ACT — add `slowapi`, wire middleware, config via env vars, write tests.
OBSERVE — `pytest` → `9 passed, 1 failed`. Failure: burst allowance off by one.
VERIFY — failed. New hypothesis: window boundary is inclusive.
ACT — fix boundary. OBSERVE — `10 passed`.
ACT — `uvicorn` up, run the load script. OBSERVE — `100× 200, 20× 429`, `Retry-After: 41`.
VERIFY — matches criteria. Server stopped.

**REPORT**
```
Done: per-API-key rate limiting on services/core, 100 req/min, burst 20, Redis-backed.
      429 + Retry-After on limit. Branch: feat/rate-limit.
Evidence: pytest 10 passed; load test 100×200 then 20×429, Retry-After: 41.
Open: limits are hardcoded defaults in config.py — not yet wired to env in the prod chart.
```

Note what didn't happen: no mid-flight check-ins, no "please test this", and the one failing
test was fixed in-loop rather than reported as a blocker.
