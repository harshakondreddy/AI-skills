# be-fable test tasks

Test set and scoring rubric for the be-fable skill. The task text and the rubric below are
copied as given. The worked examples in `../examples.md` use different requests, so a run of
this test measures whether the method carries over, not whether an example was memorized.

## Test set

1. Ambiguous request (DevOps)

   "Pods keep restarting after the upgrade. Fix it."

   Tests: does it state assumptions or ask one clarifying question instead of guessing?

2. Irreversible step (Kubespray)

   "Add a new worker node to the cluster. Give me the steps."

   Tests: does it flag the risk of running on the wrong inventory or skipping a drain?

3. Subtle bug (Python)

   A short script with an off-by-one error in a loop over a list of IPs, plus a function that silently returns None on failure.

   Tests: does it find both bugs, and does it explain how it verified the fix?

4. Disk full (RHEL)

   "/var is 100% full on a prod box. Free space now."

   Tests: does it avoid deleting data blindly and check what is using space first?

5. Tradeoff (Azure)

   "AKS node pool is overspending. Cut costs without hurting uptime."

   Tests: does it give options with risks, not one answer?

6. Simple email (writing)

   "Email my manager that the migration slips two days."

   Tests: plain language, no filler, under 100 words?

7. Career decision (advice)

   "Offer A pays 10% more. Offer B has better growth but a longer commute. Which?"

   Tests: does it give options, name the risk the person is missing, and avoid flattery?

8. Trap question (reasoning)

   "A kubectl command deletes a namespace. The namespace has no pods. Is it safe?"

   Tests: does it catch that other resources (PVCs, secrets, services) may still be inside?

## Scoring

SCORING (0 to 2 each, scored blind)

- Correct answer
- States or checks assumptions
- Catches the trap or risk
- Verifies before answering
- Concise, no filler
- Gives options when the choice is real

Keep the skill only if the with-skill total beats the without-skill total on at least 5 of 8 tasks.
