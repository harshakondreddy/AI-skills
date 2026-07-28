# Full Initiative Template (Tier 3)

Use this only for genuinely large, multi-system, hard-to-reverse work — a cluster migration
across subscriptions, a major platform upgrade with rollback requirements, a security
remediation touching several services, a multi-day rollout. If you're reaching for this on
something smaller, drop back to Tier 2.

Produce these 12 sections. Keep each one tight — this is a working plan, not a document for
its own sake.

## 1. Executive Summary
2-4 sentences: what this initiative is, why it's happening now, and the outcome it delivers.

## 2. Objectives and Definition of Done
The concrete end state. Not "migrate the cluster" but "cluster X running on subscription Y,
all services passing health checks, DNS cut over, old cluster decommissioned."

## 3. Work Packages
WP-01, WP-02, ... using the template from SKILL.md. Every WP needs an owner, dependencies,
success criteria, and required evidence — no exceptions at this tier.

## 4. Assignments
One line per WP: who or what executes it. Single responsibility per WP — don't let one work
package quietly cover three unrelated concerns. For a solo initiative this is just a label,
not an org chart. If you have a sub-agent mechanism, only spin one out when isolation
genuinely helps — keeping secret-handling output out of the main thread, or running
independent-wave WPs concurrently. Otherwise work through it in the main thread.

## 5. Entry and Exit Criteria
For each WP: what must be true before it can start (entry), and what must be true/observed
before it counts as finished (exit). Exit criteria should just be the success criteria +
evidence from section 3, restated if useful.

## 6. Parallel Execution Waves
Group independent WPs into waves that can run concurrently; sequence dependent ones into
later waves.

```
Wave A: WP-01, WP-03   (no dependencies between them)
Wave B: WP-02          (depends on WP-01 finishing)
Wave C: WP-04          (documentation/report, depends on everything above)
```

## 7. Dependency Diagram
A simple text diagram is enough — don't reach for a diagramming tool unless asked.

```
WP-01 --> WP-02 --> WP-04
WP-03 -------------> WP-04
```

## 8. Validation Gates
Checkpoints where work stops until evidence is reviewed and explicitly passes. Typically one
gate per wave, plus a final go/no-go gate before anything user-facing changes (cutover,
production deploy, decommission of the old system).

Most gates are self-review: you check the evidence against the exit criteria and continue —
execution stays unattended. A gate only requires the user when it fronts an irreversible action
that wasn't pre-approved in PLAN, or when the evidence contradicts an assumption the plan was
built on. Say which gates are which when you write them.

## 9. Risk Controls
For each material risk: what it is, likelihood/impact (rough is fine — High/Med/Low), and
the mitigation or rollback plan. Always include a rollback plan for anything touching
production or that's hard to undo.

## 10. Reporting Structure
Who/what gets updated and when — e.g., "REPORT after each wave completes" or "flag
immediately if a VERIFY step fails." For a solo initiative this can just be "summarize after
each wave" — don't invent stakeholders that don't exist.

## 11. Release Readiness Checklist
The final list that has to be true before calling it done: all WPs verified, rollback plan
confirmed and ready, monitoring/alerts in place, documentation published, stakeholders (if
any) notified.

## 12. Deferred Items
Anything explicitly out of scope or pushed to later, and why. This prevents scope creep
disguised as "just one more thing" mid-execution, and gives you a clean backlog afterward.

---

Once all 12 sections exist, execute wave by wave using the core loop (PLAN -> ACT -> OBSERVE
-> VERIFY -> REFLECT -> REPORT) per work package, and don't cross a validation gate without
the evidence for everything behind it.
