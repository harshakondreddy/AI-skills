# be-fable procedures and worked examples

One section per task type. Each section stands alone. Read the one that matches the task and
skip the rest. To load a single section, list the section headers with their line numbers,
then read from that header to the next one:

    grep -n '^## ' examples.md

Every section has the same parts in the same order: questions to ask first, assumptions,
steps, decision rules, common mistakes, checks before answering, stating confidence, worked
examples. A worked example shows the request and the final answer only. The core rules in
`SKILL.md` apply on top of every section.

Sections:

- [Coding](#coding)
- [Debugging](#debugging)
- [Infrastructure and DevOps](#infrastructure-and-devops)
- [Writing](#writing)
- [Research](#research)
- [Math](#math)
- [Planning](#planning)

---

## Coding

Use for writing new code, changing existing code, reviewing code, and refactoring.

### Questions to ask first

Ask only what the request, the repository, and the environment do not already answer.

- What should it do? Ask for one example input and the expected output.
- Where does it run: language version, operating system, framework, allowed dependencies?
- What must not change: public interfaces, file formats, behavior other code relies on?
- How will it be checked: existing tests, a command, a manual step?
- What is out of scope?

### Assumptions

State:

- The language and runtime version the code is written for.
- The shape and size of the input, and what happens on bad input.
- That no new dependency is added. Say so first if one is needed.

Check, do not assume:

- The function, file, or API you call exists and has the signature you expect, in the
  installed version.
- How the project already does the same thing: naming, error handling, test layout.
- The tests pass before your change. Record the failures that were already there.

### Steps

1. Read the code you will change, the code that calls it, and its tests.
2. Run the existing tests or the program to get a baseline.
3. State the change in one sentence: input, output, behavior on bad input.
4. Write the smallest change that does it, in the style already used in the file.
5. Handle the edges: empty input, one item, the last item, duplicates, wrong types, missing
   files, timeouts.
6. Make failures loud. Raise or return an explicit error. Do not return `None`, swallow an
   exception, or ignore an exit code.
7. Add or update a test that fails without the change and passes with it.
8. Run the tests and the program. Read the output.
9. Re-read the diff line by line. Remove anything the request did not need.
10. Report what changed and where, how it was verified, and what was not verified.

### Decision rules

| If | Then |
|---|---|
| The codebase already has a helper or pattern for this | Use it. Do not add a second way to do the same thing. |
| The change needs a new dependency | Say so before adding it, and offer the standard-library route if one exists. |
| You cannot run the code | Say so. Trace one normal input and one edge input by hand and label the result "not run". |
| A test you did not touch fails | Check whether it failed before your change. Report it. Do not edit the test to make it pass. |
| Making a test pass means changing what the test expects | Stop and confirm the expected behavior with the person. |
| The change touches a public interface, a schema, or a file format | Say so and list what depends on it. |
| The request says "refactor" | Behavior stays identical. Run the same tests before and after. |
| You are unsure of an API's name or arguments | Look it up in the installed version (`help()`, the docs, the source). Do not write it from memory. |
| The existing code has a behavior your change will alter | Name the behavior change and ask whether anything depends on the old behavior. |
| The schema, data, or surrounding code was not provided | State the names you assumed, write the code, and mark it "not run". |
| You see a bug outside the request | Report it in one line. Leave it out of this change. |
| The code deletes or overwrites (`rm`, `DROP`, `TRUNCATE`, a file opened for writing) | Before anything else, check what it does when its input is empty, missing, or wrong. |

### Common mistakes and how to catch them

| Mistake | How to catch it |
|---|---|
| Off-by-one in ranges, slices, and indexes | Trace the first, last, and empty cases by hand. Test with lists of length 0, 1, and 2. |
| Silent failure: a bare `except`, returning `None`, an unchecked exit code | Search the diff for `except:`, `except Exception: pass`, `return None`, and output sent to `/dev/null`. |
| Calling an API that does not exist in the installed version | Check the version and read the signature before use. |
| Fixing the symptom where it shows, not where the bad value is created | Follow the bad value back to where it first appears. Fix it there. |
| Changing more than was asked | Read the diff. Every hunk must trace back to the request. |
| A test that passes without testing anything | Break the code on purpose. The test must fail. |
| Code that works only for the example input | Test a second input with a different size and shape. |
| Unquoted shell variables and an unchecked `cd` | Run `bash -n` and `shellcheck`. Ask what runs if the variable is empty. |
| Mutable default arguments, float equality, naive datetimes, integer division | Search for `def f(x=[])`, `==` on floats, and `datetime.now()` without a time zone. |
| Secrets, debug prints, or commented-out code left in | Search the diff before reporting. |

### Checks before answering

- The code was run, or the answer says it was not.
- Tests: the count passing before and after, and the new test fails without the change.
- The edge cases from step 5 are handled or declared out of scope.
- Errors are explicit, not silent.
- The diff contains only what the request needs.
- Behavior changes are named.
- The answer says which files and functions changed.

### Stating confidence

- Verified: "Ran `pytest tests/test_orders.py`: 12 passed. The new test fails on the old code."
- Not run: "Not run: I do not have your database. Traced by hand for an empty list and a list
  of three."
- Unsure: "Unsure whether `connect()` accepts `timeout` in your version. Check with
  `pip show <package>` and the changelog."

### Worked examples

#### Coding example 1: a new function

**Request**

> Write a Python function that splits a list into chunks of size n.

**Final answer**

```python
def chunks(items, n):
    """Split a list into consecutive chunks of length n. The last chunk may be shorter."""
    if n < 1:
        raise ValueError(f"n must be at least 1, got {n}")
    return [items[i:i + n] for i in range(0, len(items), n)]
```

Assuming the last chunk is kept when it is shorter than `n`. If you want it dropped or padded,
say which.

The `n < 1` check is there because without it `n = 0` fails with an unclear message
(`range() arg 3 must not be zero`) and a negative `n` returns `[]` with no error.

Verified: ran these, all passed.

```python
assert chunks([1, 2, 3, 4, 5], 2) == [[1, 2], [3, 4], [5]]
assert chunks([1, 2, 3, 4], 2) == [[1, 2], [3, 4]]
assert chunks([1], 3) == [[1]]
assert chunks([], 3) == []
```

`chunks([1, 2], 0)` and `chunks([1, 2], -1)` raise `ValueError`.

On Python 3.12 or later, `itertools.batched(items, n)` does the same for any iterable and
yields tuples.

#### Coding example 2: a change to existing code

**Request**

> Add retries to this:
>
> ```python
> import requests
>
> def fetch(url):
>     return requests.get(url).json()
> ```

**Final answer**

```python
import time
import requests

RETRY_STATUSES = {429, 500, 502, 503, 504}


def fetch(url, attempts=3, backoff=1.0, timeout=10):
    """GET url and return the parsed JSON body.

    Retries connection errors, timeouts, and the statuses in RETRY_STATUSES.
    Raises the last error when every attempt fails.
    """
    for attempt in range(1, attempts + 1):
        try:
            response = requests.get(url, timeout=timeout)
            if response.status_code not in RETRY_STATUSES:
                response.raise_for_status()
                return response.json()
            error = requests.HTTPError(
                f"HTTP {response.status_code} from {url}", response=response
            )
        except (requests.ConnectionError, requests.Timeout) as exc:
            error = exc
        if attempt == attempts:
            raise error
        time.sleep(backoff * 2 ** (attempt - 1))
```

What it does:

- Up to 3 attempts, waiting 1 second and then 2 seconds between them.
- Retries connection errors, timeouts, and HTTP 429, 500, 502, 503, 504.
- Does not retry other 4xx statuses. Repeating the request would give the same result.

Two changes beyond the retries. Confirm both:

1. **A 10-second timeout.** The original had none, so a stalled connection would wait forever
   and no retry would ever start.
2. **Error statuses now raise.** The original never checked the status. I confirmed that a 503
   with a JSON body came back as a normal return value. This version raises
   `requests.HTTPError`. If a caller depends on reading error bodies, tell me and I will keep
   that behavior.

Assuming the request has no side effects, so repeating it is safe.

Verified against a local test server:

- 503, 503, then 200: returned the JSON after 3 requests.
- 503 every time: raised `HTTPError` after 3 requests.
- 404: raised `HTTPError` after 1 request.
- A response slower than the timeout, with `attempts=2`: raised `ReadTimeout` after 2 requests.
- Nothing listening on the port: raised `ConnectionError`.

Not verified: your real endpoint. Not handled: the `Retry-After` header on 429 responses.

#### Coding example 3: a review of a script that deletes

**Request**

> Review this cleanup script before I put it in cron:
>
> ```bash
> #!/bin/bash
> BACKUP_DIR=$1
> cd $BACKUP_DIR
> rm -rf *
> ```

**Final answer**

Do not schedule this as written. It can delete the wrong directory in three ways, and it exits
with code 0 each time, so cron will not report a failure.

1. **No argument.** `cd` with no argument goes to the home directory. `rm -rf *` then empties
   it.
2. **The path does not exist or is not mounted.** `cd` fails, the script carries on, and
   `rm -rf *` runs in the directory the script started in. Under cron that is the home
   directory.
3. **The path contains a space.** `$BACKUP_DIR` is unquoted, so `cd` receives a broken path,
   fails, and case 2 follows.

Replacement:

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="${1:?usage: cleanup.sh <backup-dir>}"

if [[ ! -d "$BACKUP_DIR" ]]; then
    echo "error: not a directory: $BACKUP_DIR" >&2
    exit 1
fi

# Refuse anything outside the expected parent. Change /backups to your real parent path.
case "$(realpath "$BACKUP_DIR")" in
    /backups/*) ;;
    *) echo "error: $BACKUP_DIR is not under /backups" >&2; exit 1 ;;
esac

rm -rf -- "${BACKUP_DIR:?}"/*
```

What changed:

- `${1:?...}` stops with a usage message when the argument is missing.
- The `-d` test stops when the path is missing or is not a directory.
- The `case` block refuses any path outside one parent directory. `/backups` is a placeholder.
  Set it to your real parent path.
- There is no `cd`. `rm` receives the full, quoted path, so it cannot run somewhere else.
- `set -euo pipefail` stops the script at the first failed command or unset variable.

Verified in a scratch directory with a fake home directory.

The original:

- No argument: deleted every non-hidden file in the home directory. Exit code 0.
- A path that does not exist: deleted the files in the starting directory. Exit code 0.
- A path with a space: deleted the files in the starting directory. Exit code 0.

The replacement:

- No argument, a missing path, a path outside the allowed parent: error, exit code 1, nothing
  deleted.
- A valid path, with and without a space in the name: emptied that directory only.
- An empty directory: exit code 0.

Two things to confirm:

- **Hidden files.** As in the original, `*` skips names that start with a dot, so those files
  stay.
- **Everything, or only old backups?** If only old ones should go, delete by age. List first,
  then delete:

```bash
find "$BACKUP_DIR" -type f -mtime +30 -print     # list files older than 30 days
find "$BACKUP_DIR" -type f -mtime +30 -delete    # delete them
```

#### Coding example 4: code that cannot be run

**Request**

> Write a SQL query for the top 5 customers by revenue last month.

**Final answer**

Assuming PostgreSQL and these tables: `customers(id, name)` and
`orders(id, customer_id, total_amount, status, created_at)`. Change the names to match your
schema.

```sql
SELECT c.id,
       c.name,
       SUM(o.total_amount) AS revenue
FROM orders AS o
JOIN customers AS c ON c.id = o.customer_id
WHERE o.created_at >= date_trunc('month', CURRENT_DATE) - INTERVAL '1 month'
  AND o.created_at <  date_trunc('month', CURRENT_DATE)
  AND o.status = 'completed'
GROUP BY c.id, c.name
ORDER BY revenue DESC
LIMIT 5;
```

Three choices in this query change the result. Confirm each:

1. **"Last month"** is the previous calendar month in the database session's time zone. If
   `created_at` is stored in UTC and you report in local time, orders near the month boundary
   land in the wrong month.
2. **"Revenue"** is the sum of `total_amount` over completed orders. Refunds, cancellations,
   tax, and shipping are not handled. Adjust the `status` filter and the column to your
   definition.
3. **Ties.** `LIMIT 5` cuts arbitrarily among customers tied for fifth place. To include ties
   on PostgreSQL 13 or later, replace `LIMIT 5` with `FETCH FIRST 5 ROWS WITH TIES`.

Not run: I do not have your schema or data. To check it, run the query, then compare one
customer's figure against a manual sum for that customer.

---

## Debugging

Use when something fails or gives a wrong result: an error message, a failing test, a crash,
wrong output, a slowdown, or "it worked yesterday".

### Questions to ask first

- What did you expect, and what happened instead? Ask for the exact error text and the full
  stack trace.
- When did it last work, and what changed since: a deploy, an upgrade, a config edit, new
  data, more traffic?
- Does it happen every time? On which inputs, hosts, or users?
- Where does it happen (production, staging, local), and can it be reproduced there?
- What has already been tried?

### Assumptions

State:

- Which version or commit you are reading.
- Whether the cause is confirmed by a reproduction or inferred from the evidence.

Check, do not assume:

- The code that is running is the code you are reading: the deployed version, a stale build, a
  cached artifact.
- The exact error text and its timestamp, compared with the time of the change.
- The configuration that was loaded, not the file you expect was loaded.

Never assume:

- The most recent change is the cause.
- The first plausible cause is the only one.
- The last line of a stack trace is where the bug is.

### Steps

1. Get the exact symptom: full error text, stack trace, exit code, log lines with timestamps.
2. Reproduce it. Record the exact command and input. If it cannot be reproduced, collect more
   evidence (logging, the failing input) and do not guess.
3. Find what changed: `git log`, the deploy history, package versions, config diffs, data.
4. Narrow the location. Bisect by commit (`git bisect`), by input (halve it), or by code path
   (log at the midpoint).
5. State one hypothesis that explains every symptom, and name the observation that would
   prove it wrong.
6. Test it with one read-only check or one small experiment. Change one thing at a time.
7. Fix the cause, not the symptom, with the smallest change.
8. Verify. Re-run the reproduction: it must now pass. Run the wider tests. Confirm the symptom
   is gone in the place it was reported.
9. Search for the same bug elsewhere in the code.
10. Report the symptom, the cause, the evidence, the fix, how it was verified, and what is
    still unknown. Add the test or check that would have caught it.

### Decision rules

| If | Then |
|---|---|
| You cannot reproduce it | Do not claim a fix. Say what to log or capture on the next occurrence. |
| The error appeared right after a change | Diff that change first. Confirm the mechanism before blaming it. |
| There are several errors | Start with the earliest in time. Later ones are often consequences. |
| A change makes the symptom go away and you cannot explain why | Treat it as not fixed. Keep looking, or say so. |
| Two attempts at the same idea have failed | Drop the idea and re-read the evidence. |
| The bug is in production and causing harm | Mitigate first (roll back, disable the feature, add capacity), then diagnose. Say which of the two you are doing. Capture logs before a restart removes them. |
| The evidence points at a dependency or the platform | Check its version, changelog, and known issues before concluding. |
| You find a second bug | Report it separately. Fix one at a time and re-test after each. |
| The failure is intermittent | Compute how often it should fail under your hypothesis and compare that with how often it does. Look at concurrency, timeouts, caches, shared state, time of day, resource limits, and one bad host among many. |
| The person wants a fix and the cause is unknown | Give ranked hypotheses and the check that separates them. Do not present one guess as the answer. |
| A failing test looks wrong | Find out whether the test or the code is wrong before changing either. A failing test can be correct. |
| The problem happens only outside your shell (cron, CI, a container, a service) | Compare the two environments: `PATH`, variables, working directory, user, permissions. |

### Common mistakes and how to catch them

| Mistake | How to catch it |
|---|---|
| Guessing the cause from the last line of the error | Read the whole trace from the top. Find the first frame in your own code. |
| Changing several things at once | One change, one run. |
| Debugging code that is not the code running | Print the version, commit, or image digest that is running. |
| Stopping at the first bug found | Re-run the full reproduction after the fix. Check every reported symptom. |
| Hiding the error with a new `try/except`, a retry, or a discarded exit code | Search the diff for new exception handlers and ignored return values. |
| Declaring it fixed without re-running | Show the output before and after. |
| Reading a summary of the log and not the log | Open the log. Search for the first error by timestamp. |
| "It broke after X, so X caused it" | Name the line and the value that connect X to the symptom. |
| Changing the test to match the bug | Ask what the correct behavior is before touching the assertion. |
| Restarting first | Capture logs, state, and timestamps before any restart. |

### Checks before answering

- A reproduction existed and now passes. If there was none, the answer says so.
- The cause explains every reported symptom. Unexplained symptoms are listed.
- The fix was tested and the wider tests still pass.
- Each claim is marked as verified or inferred.
- A mitigation is not described as a fix.
- The answer includes the check that confirms the result on the person's side.

### Stating confidence

- Verified: "Cause confirmed. Reproduced with input X, applied the fix, the reproduction now
  passes."
- Likely: "Likely cause, not reproduced: the connection pool is exhausted. To confirm, compare
  connections in use with the pool size at the time of the errors."
- Unsure: "Two candidates fit. This check separates them: ..."
- Split a mixed answer by claim: "Problem 1 is certain from the line itself. Problem 2 applies
  only if the script is outside `/usr/bin` and `/bin`."

### Worked examples

#### Debugging example 1: an error that can be reproduced

**Request**

> My script crashes with `TypeError: can't compare offset-naive and offset-aware datetimes`.
>
> ```python
> from datetime import datetime
>
> def is_expired(expires_at_iso):
>     expires_at = datetime.fromisoformat(expires_at_iso)
>     return expires_at < datetime.now()
>
> print(is_expired("2025-03-01T12:00:00+00:00"))
> ```

**Final answer**

**Cause.** The input string carries an offset (`+00:00`), so `fromisoformat` returns a
datetime with a time zone. `datetime.now()` returns one without. Python refuses to compare the
two.

**Fix.** Compare against the current time in UTC:

```python
from datetime import datetime, timezone

def is_expired(expires_at_iso):
    expires_at = datetime.fromisoformat(expires_at_iso)
    if expires_at.tzinfo is None:
        raise ValueError(f"timestamp has no UTC offset: {expires_at_iso!r}")
    return expires_at < datetime.now(timezone.utc)
```

The `tzinfo` check makes a timestamp without an offset fail with a clear message. Without the
check, that input brings the same `TypeError` back. If your data has timestamps without
offsets and you know they are UTC, replace the `raise` line with
`expires_at = expires_at.replace(tzinfo=timezone.utc)`.

**Verified** on Python 3.9:

- Your script reproduces the error.
- With the fix, `2025-03-01T12:00:00+00:00` returns `True`, `2999-01-01T00:00:00+00:00`
  returns `False`, `2025-03-01T12:00:00+05:30` returns `True`, and a string with no offset
  raises `ValueError`.

**One related failure to expect.** On Python 3.10 and earlier, `fromisoformat` rejects a
trailing `Z`: `2025-03-01T12:00:00Z` raises `ValueError: Invalid isoformat string`. I
confirmed this on 3.9. The Python documentation says 3.11 and later accept it. I did not run
3.11. If your timestamps end in `Z` on an older Python, convert first:
`expires_at_iso.replace("Z", "+00:00")`.

#### Debugging example 2: an intermittent failure

**Request**

> This test fails about 1 run in 10 in CI and I can't see why:
>
> ```python
> import random, time
>
> def make_token():
>     return f"{int(time.time())}-{random.randint(0, 9)}"
>
> def test_tokens_are_unique():
>     assert make_token() != make_token()
> ```

**Final answer**

**Cause.** A token has only 10 possible values in any one second. Both calls in the test run
in the same second, so the timestamp parts match, and the test fails whenever the two random
digits match. `random.randint(0, 9)` includes both ends, so that is 1 time in 10, which is the
failure rate you see.

**Verified.** Generated 100,000 pairs with the current function: 9,985 pairs were equal
(10.0%).

**The test is correct. The function is wrong.** Outside the test it will hand out duplicate
tokens at the same rate whenever two are created in the same second.

**Fix.**

```python
import secrets

def make_token():
    return secrets.token_hex(16)
```

`secrets.token_hex(16)` returns 32 hexadecimal characters built from 16 random bytes.

**Verified.** 100,000 pairs: 0 equal. 1,000,000 tokens: all unique.

**Check before merging.** The token format changes: 32 hex characters and no timestamp
prefix. Search for code that reads the timestamp out of a token or stores tokens in a
fixed-width column. If these tokens are used for sessions, password resets, or anything else
that must not be guessable, the old function was also a security problem: the Python
documentation says the `random` module should not be used for security purposes.

#### Debugging example 3: not enough information

**Request**

> Our API got slow yesterday. Why?

**Final answer**

I cannot tell yet. Nothing in the request shows where the time goes. One question:

**What changed yesterday: a deploy, a config or dependency update, more traffic, or nothing
you know of?**

While you find out, these read-only checks separate the usual causes:

| Check | What the result means |
|---|---|
| Latency per endpoint (median and 95th percentile), before and after | One endpoint slow: look at that code path and its queries. Every endpoint slow: a shared cause such as the database, the hosts, the network, or a dependency. |
| The time the slowdown started, against the deploy and change log | A match points at that change. Diff it. |
| Request rate, before and after | More traffic on the same code: a capacity problem. |
| CPU, memory, and swap on the application hosts | A saturated host. |
| Database: the slow query log, connections in use against the pool size, lock waits | A slow query, an exhausted pool, or locking. |
| Response time of calls to external services | A slow dependency. |

Send me the first two results and I can narrow it down.

I am not proposing a fix yet. A restart or a scale-up now could hide the evidence without
showing the cause. The exception: if users are affected right now, say so. Then mitigation
comes first, and if yesterday had a deploy, rolling it back is the first candidate.

#### Debugging example 4: works by hand, fails under a scheduler

**Request**

> This cron job works when I run the command by hand but does nothing from cron:
>
> ```
> 0 2 * * * backup.sh > /var/log/backup-$(date +%F).log 2>&1
> ```

**Final answer**

There are two problems in the line. Either one stops the job.

1. **`%` is special in a crontab.** Cron cuts the command at the first unescaped `%` and
   passes the rest as input. What runs is `backup.sh > /var/log/backup-$(date +`. That is
   certain from the line itself, and I checked the result: `/bin/sh` rejects that text with
   `syntax error: unexpected end of file`, so the script never starts. Write `\%`.
2. **`backup.sh` has no path.** Cron runs with a short `PATH`, usually `/usr/bin:/bin`, and
   starts in your home directory. This applies unless the script is in one of those two
   directories. Use the full path.

Corrected line. Replace `/opt/scripts` with the real location:

```
0 2 * * * /opt/scripts/backup.sh > /var/log/backup-$(date +\%F).log 2>&1
```

Not verified: I cannot see your machine. Confirm in this order.

1. See whether cron tried to run the job. On RHEL:

   ```bash
   sudo grep CRON /var/log/cron | tail -20
   ```

2. Read the error cron captured. Cron mails a job's output to its owner when a mail service
   is installed:

   ```bash
   tail -50 /var/spool/mail/$USER
   ```

3. Run the script the way cron does, with an empty environment:

   ```bash
   env -i HOME="$HOME" PATH=/usr/bin:/bin /bin/sh -c '/opt/scripts/backup.sh'
   ```

   If this fails, the script itself depends on something from your login shell: a variable, a
   relative path, or a command outside `/usr/bin` and `/bin`.

4. Check that the job's user can write to `/var/log`. A non-root user usually cannot, and then
   the redirect fails before the script starts.

After the fix, set the schedule to two minutes from now, confirm that the log file appears
with the expected content, then restore `0 2 * * *`.

---

## Infrastructure and DevOps

Use for Kubernetes, RHEL, Azure, Kubespray, Ansible and other automation, and any change to a
running system.

### Questions to ask first

- Which environment, exactly: production, staging, or test? Which cluster, subscription, or
  host, by name?
- What changed last, and when: an upgrade, a deploy, a config or certificate change, a node
  change?
- What is the impact right now (down, degraded, at risk), and is there a change window?
- What access do you have (read-only, admin, sudo), and who approves changes?
- Is there a backup or snapshot, and has a restore from it been tested?
- Which versions: Kubernetes, the OS release, the Kubespray release, the CLI?

### Assumptions

State:

- Whether you can see the system. If not, say: "I cannot see your system. These commands are
  for you to run."
- The versions the commands were written for.
- The environment you assumed. Treat an unnamed environment as production.

Check with a read-only command, do not assume:

- The target: `kubectl config current-context`, `az account show`, `hostname`,
  `cat /etc/os-release`, `ansible-inventory -i <inventory> --graph`.
- A backup exists and is recent: an etcd snapshot, a VM or disk snapshot, a database backup.
- Spare capacity, before removing any (drain, scale down, resize).
- What depends on the thing you are about to change.

Never assume:

- "No pods" or "nothing is running" means empty.
- A drain is safe without checking PodDisruptionBudgets, single-replica workloads, and local
  data.
- Deleting a file frees its space. A process may still hold it open.
- The default namespace, context, subscription, or inventory is the intended one.
- A `--limit` or a filter was applied.
- The previous step succeeded.

### Steps

1. Identify the target and prove it. Print the context, subscription, inventory, or hostname.
2. Look before changing. Run read-only commands first: `get`, `describe`, `logs`, `events`,
   `df`, `du`, `journalctl`, `az ... show`, `az ... list`.
3. Find what changed and when. Line the timestamps up with the symptom.
4. Choose the smallest change that addresses the cause.
5. Preview it where the tool allows: `kubectl diff`, `--dry-run=server`,
   `ansible-playbook --check --diff`, `az deployment group what-if`, `terraform plan`.
6. Before any step that deletes, drains, restarts, reboots, scales down, or rewrites
   configuration: save the current state (export the YAML, copy the file, take a snapshot),
   write down the rollback, list what else is affected, and get a yes.
7. Apply to one target first: one node, one pod, one host. Check it. Then widen.
8. Verify with the same read-only commands as in step 2. Compare before and after.
9. Record what was done: the commands, the time, the result, and what to watch next.

### Decision rules

| If | Then |
|---|---|
| A command changes or deletes anything | Put a read-only command before it that proves the target is the intended one. |
| The environment is production, or was not stated | Choose the reversible form: cordon before drain, scale to zero before delete, move or compress before delete, snapshot before change. State the rollback. |
| The tool has a dry-run or diff mode | Run it first and read the output. |
| You cannot see the system | Give the commands, the expected output, and what each result means. Do not state the cause as a fact. |
| The request says "now" | Give the first safe action first. Keep the pre-check: it takes seconds, and a wrong delete does not. |
| The action removes capacity (drain, scale down, resize, spot instances) | Check that what remains can carry the load and that PodDisruptionBudgets allow it. |
| A name was not given (namespace, resource group, node, path) | Write a placeholder such as `<namespace>` and say it must be filled in. Do not invent a name. |
| The proposed fix is a restart | Capture logs, events, and state first. Say what the restart will not explain. |
| Automation will run on more than one host | List the hosts first (`--list-hosts`), run on one, then in batches (`serial`, `--limit`). |
| A secret, key, or token appears in output | Do not repeat it. If it was exposed, say it should be rotated. |
| A command differs between versions | Say which version it was written for and how to check (`--help`, `--version`, the release notes). |
| The request is to cut cost | Get the numbers first: what costs what. Give options with the risk of each. Do not trade away redundancy without saying so. |
| The request is to turn off a safeguard (SELinux, a firewall, a PodDisruptionBudget, TLS verification) | Find what the safeguard is blocking and fix that. If it must be relaxed, relax it for one service, not the whole system. |

**Kubernetes**

- Run `kubectl config current-context` first. Pass `-n <namespace>` explicitly.
- Pods restart or crash: read in this order. `kubectl get pods -o wide`, then
  `kubectl describe pod` (Last State, Reason, Exit Code, Events), then
  `kubectl logs --previous`, then `kubectl get events --sort-by=.lastTimestamp`. Exit code 137
  or `OOMKilled`: compare memory use with the limit. Probe failures in Events: check the probe
  path, port, and timing. `ImagePullBackOff`: check the image name, tag, and pull secret.
- Pods stay `Pending`: `kubectl describe pod` and read the `FailedScheduling` event. It names
  the cause.
- Apply a manifest: `kubectl diff -f <file>` first. After it, `kubectl rollout status`. To
  undo, `kubectl rollout undo`.
- Drain a node: check `kubectl get pdb -A`, the pods on the node, single-replica workloads,
  and `emptyDir` data. Cordon first. Uncordon after.
- Delete a namespace: list everything in it first. `kubectl get all` leaves out Secrets,
  ConfigMaps, PersistentVolumeClaims, Ingresses, and custom resources. This lists every type:

  ```bash
  kubectl api-resources --verbs=list --namespaced -o name \
    | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <namespace>
  ```

- Delete a PersistentVolumeClaim or PersistentVolume: check the reclaim policy
  (`kubectl get pv`). With `Delete`, the storage behind it is removed too.
- Upgrade: one minor version at a time. Back up etcd first. Check for APIs removed in the
  target version.

**RHEL**

- Confirm the host and release first: `hostname`, `cat /etc/os-release`. RHEL 7 uses `yum`.
  RHEL 8 and later use `dnf`.
- A filesystem is full: find what uses the space before removing anything.

  ```bash
  df -h <mount>; df -i <mount>                   # space, then inodes
  sudo du -xh --max-depth=1 <mount> | sort -h    # largest directories on this filesystem
  sudo lsof +L1                                  # deleted files a process still holds open
  ```

  Free space in this order: the package cache (`dnf clean all`), the journal
  (`journalctl --vacuum-size=500M`), old rotated logs, then anything else only after naming
  its owner. Truncate a log that a process holds open (`truncate -s 0 <file>`). Deleting it
  frees nothing until the process closes it. Do not delete under `/var/lib` without knowing
  which service owns the data.
- Changing `sshd`, the firewall, or the network on a remote host: keep the current session
  open, validate the config (`sshd -t`), and test a second session before closing the first.
- A service fails: `systemctl status <unit>`, then `journalctl -u <unit> -b`, then the
  service's own log.
- SELinux denies something: read the denial (`ausearch -m AVC -ts recent`) and fix the label,
  port, or boolean. Do not disable SELinux for the whole host.
- Editing a config file: copy it first, validate it (`sshd -t`, `nginx -t`, `visudo -c`), and
  reload instead of restarting when the service supports it.
- Patching: `dnf check-update`, take a snapshot or backup, patch one host, check
  `dnf needs-restarting -r`, reboot in a window, verify the services. Roll back with
  `dnf history undo <id>` or by booting the previous kernel.

**Azure**

- Confirm the subscription first: `az account show --query "{name:name, id:id}" -o table`.
  Pass `-g <resource-group>` explicitly on every command that changes something.
- Before deleting a resource group, list what is in it (`az resource list -g <rg> -o table`)
  and its locks (`az lock list -g <rg> -o table`). Deleting the group deletes everything in
  it.
- Resizing, deallocating, or redeploying a VM restarts it. Check what depends on it, whether
  its temporary disk holds data, and whether its public IP is static.
- Before a cost change, get the cost per resource and per meter from Cost Management. Then
  list options with the risk of each.
- Reservations and savings plans are commitments. Check the term and the exchange rules
  before buying.
- Spot capacity can be taken back at short notice. Do not put the only copy of a workload on
  it.
- Preview infrastructure-as-code changes: `az deployment group what-if`, `terraform plan`.

**Kubespray**

- Print the inventory before every playbook run:
  `ansible-inventory -i inventory/<cluster>/hosts.yaml --graph`. A run against the wrong
  inventory changes the wrong cluster.
- Use the Kubespray release the cluster was deployed or last upgraded with
  (`git describe --tags`), and read that release's documentation. Playbook behavior changes
  between releases.
- Know what each playbook does before running it. `cluster.yml` deploys or reconfigures the
  whole cluster. `scale.yml` adds worker nodes. `remove-node.yml` removes a node.
  `upgrade-cluster.yml` upgrades node by node. `reset.yml` wipes Kubernetes from the hosts in
  the inventory: it is never a repair step.
- Limit the run to the hosts you mean: `--limit=<node>` for `scale.yml`, `-e node=<node>` for
  `remove-node.yml`. Add `--list-hosts` to see the hosts without running anything. Before
  using `--limit`, run the `facts.yml` playbook without a limit to refresh the fact cache.
- Back up etcd before an upgrade and before any control-plane or etcd change.
- Upgrade one minor version at a time.
- After any run: `kubectl get nodes -o wide`, then look for pods that are not `Running`.

**Automation (Ansible, shell, scheduled jobs)**

- Ansible: `--list-hosts`, then `--check --diff`, then one host with `--limit`, then the rest
  in batches with `serial`. A second run should report `changed=0`.
- Shell: `set -euo pipefail`, quote every variable, guard deletes with `${VAR:?}`, and never
  run a command after a `cd` that was not checked.
- Scheduled jobs: full paths, output to a log, an alert on failure, and `flock` to stop two
  runs overlapping.
- Secrets: not in the repository, the command line, or the log. Use the vault or secret store.
- Production changes go through the pipeline or the source of truth. Record a manual change
  and feed it back.

### Common mistakes and how to catch them

| Mistake | How to catch it |
|---|---|
| Wrong target: context, subscription, inventory, or host | Make printing the target the first command of every procedure. |
| Deleting to free space or tidy up without knowing what the data is | List and measure first. Move or compress before deleting. |
| Restarting as the fix and losing the evidence | Save logs, events, and `describe` output before the restart. |
| Treating "no pods" as "empty" | List every resource type, not only the one that was named. |
| Draining without checking disruption budgets, single replicas, or local data | `kubectl get pdb -A` and list the pods on the node first. |
| Running a playbook without a limit, or against a stale inventory | `--list-hosts` and `ansible-inventory --graph` before the run. |
| Upgrading or migrating without a tested backup | Name the backup, its age, and the restore command before step 1. |
| Giving commands for a different version | Ask for or check the version. Say which version the commands are for. |
| Several changes at once during an incident | One change, observe, then the next. |
| Editing live objects by hand and leaving the source of truth behind | Apply through Git or the pipeline, or record the manual change and reconcile it. |
| Turning off a safeguard to make an error go away | Read what the safeguard reported. Fix that one thing. |
| Inventing resource names | Use `<placeholders>` and say they must be filled in. |

### Checks before answering

- The target is named, and a command that confirms it comes first.
- Read-only checks come before changes.
- Every destructive step is marked, with what it affects, a pre-check, and a rollback.
- Placeholders are marked. No names were invented.
- Versions are stated or asked for.
- Verification commands and the expected output are included.
- The answer says what could not be seen or verified.
- Where the choice is real (cost, redundancy, downtime), options and their risks are given.

### Stating confidence

- No access: "I cannot see your cluster. The cause is not known until the output of step 2 is
  in."
- Verified: "Checked on the host: `/var/log` is 41 GB, 38 GB of it in one file."
- From documentation: "The flags are from the kubectl documentation for current releases.
  Confirm with `kubectl drain --help`."
- Version-dependent: "Recent releases refuse to run without this variable. Older ones did
  not. Check which yours does with ..."
- Unsure: "This fits two causes. The event text separates them."

### Worked examples

#### Infrastructure example 1: Kubernetes, a symptom and "fix it"

**Request**

> Half the pods of my deployment are stuck in Pending. Fix it.

**Final answer**

I cannot see your cluster, so the cause is not known yet. `Pending` means the scheduler has
not found a node for the pod, and the pod's events say why. These three commands are
read-only:

```bash
kubectl config current-context                       # confirm this is the cluster you mean
kubectl get pods -n <namespace> -o wide              # which pods are Pending
kubectl describe pod <pending-pod> -n <namespace>    # read the Events at the bottom
```

Match the `FailedScheduling` message. The wording varies a little between Kubernetes versions.

| The message contains | Cause | What to do |
|---|---|---|
| `Insufficient cpu` or `Insufficient memory` | The pod's requests do not fit on any node | Compare the requests with the free capacity in `kubectl describe nodes` ("Allocated resources"). Lower the requests if they are above real use, or add nodes. |
| `had untolerated taint` | The nodes are tainted and the pod has no matching toleration | `kubectl describe node <node>` and read Taints. Add the toleration, or remove the taint if it is not meant to be there. |
| `were unschedulable` | The nodes are cordoned | `kubectl get nodes`. Uncordon a node only if the cordon is not deliberate. |
| `didn't match Pod's node affinity/selector` | The node selector or affinity matches no node | Compare the pod's `nodeSelector` and affinity with `kubectl get nodes --show-labels`. |
| `didn't match pod anti-affinity rules` | Anti-affinity allows one replica per node and there are more replicas than nodes | Add nodes, lower the replica count, or change the rule from required to preferred. |
| `unbound immediate PersistentVolumeClaims` | A volume claim is not bound | `kubectl get pvc -n <namespace>`, then `kubectl describe pvc <name>`. |
| `Too many pods` | The node's pod limit is reached | Add nodes. |

"Half the pods" fits two of these more than the others: there is room for only some of the
replicas, or an anti-affinity rule meets fewer nodes than replicas. That is a guess until the
event text is in.

Send me the Events output and I will give the exact change. I am not suggesting deleting pods
or restarting anything. A Pending pod never started, so a restart changes nothing.

#### Infrastructure example 2: Kubernetes, a step that evicts workloads

**Request**

> Give me the command to drain worker-3 for maintenance.

**Final answer**

The command is in step 3. Run steps 1 and 2 first. A drain evicts every pod on the node, and
three things can turn that into an outage or lost data.

1. Confirm the cluster and see what is on the node:

   ```bash
   kubectl config current-context
   kubectl get pods -A -o wide --field-selector spec.nodeName=worker-3
   ```

2. Check what the drain could break:

   ```bash
   kubectl get pdb -A     # a budget with ALLOWED DISRUPTIONS 0 will block the drain
   kubectl get nodes      # are the other nodes Ready, with room for these pods?
   ```

   In the pod list from step 1, look for:

   - **Single-replica workloads.** They are down until the pod starts on another node. Scale
     them up first, or accept the gap.
   - **`emptyDir` data.** The drain deletes it. Confirm nothing needs what is stored there.
   - **Pods with no controller.** They are deleted and do not come back. The drain refuses to
     remove them unless `--force` is added. Do not add `--force` before reading which pods
     they are.

3. Cordon, then drain:

   ```bash
   kubectl cordon worker-3
   kubectl drain worker-3 --ignore-daemonsets --delete-emptydir-data --timeout=10m
   ```

   - `--ignore-daemonsets` is needed when DaemonSet pods are on the node. They stay.
   - `--delete-emptydir-data` is needed when a pod uses `emptyDir`. It is the flag that
     discards that data.
   - If the drain waits on a PodDisruptionBudget, do not bypass it with `--disable-eviction`.
     Find out why the budget has no room, usually an unhealthy replica somewhere else, and
     fix that.

4. Verify. Only DaemonSet pods remain on the node, and nothing is stuck:

   ```bash
   kubectl get pods -A -o wide --field-selector spec.nodeName=worker-3
   kubectl get pods -A --field-selector status.phase=Pending
   ```

5. After the maintenance:

   ```bash
   kubectl uncordon worker-3
   kubectl get node worker-3     # STATUS is Ready, without SchedulingDisabled
   ```

Rollback at any point: `kubectl uncordon worker-3`. Pods that were already evicted stay where
they landed. They do not move back on their own.

Not verified: I cannot see your cluster. The flags are for current kubectl. Before 1.20,
`--delete-emptydir-data` was called `--delete-local-data`. Confirm with `kubectl drain --help`.

#### Infrastructure example 3: Kubespray, a step that cannot be undone in place

**Request**

> Remove worker node7 from our cluster. We use Kubespray.

**Final answer**

Removing a node drains it and then wipes Kubernetes from the host. It cannot be undone in
place. To get the node back, you add it again with `scale.yml`. Two mistakes cause most of the
damage: running against the wrong inventory, and leaving out `-e node=`. The steps check for
both.

I cannot see your environment. `<cluster>` is a placeholder for your inventory directory. The
commands follow the Kubespray documentation (`docs/operations/nodes.md`). Read the copy in
your own checkout, because playbooks change between releases.

1. Use the Kubespray release your cluster runs, and confirm the inventory is this cluster:

   ```bash
   cd kubespray
   git describe --tags                                            # the release the cluster was deployed with
   ansible-inventory -i inventory/<cluster>/hosts.yaml --graph    # the right cluster, with node7 in it?
   kubectl config current-context
   kubectl get nodes -o wide                                      # do names and IPs match the inventory?
   ```

2. Confirm node7 is a worker only. In the `--graph` output it should appear under `kube_node`
   and not under `kube_control_plane` or `etcd`. If it is in either of those, stop. Removing a
   control-plane or etcd member is a different procedure in the same document.

3. Check what the removal disrupts:

   ```bash
   kubectl get pods -A -o wide --field-selector spec.nodeName=node7
   kubectl get pdb -A
   ```

   Look for single-replica workloads, pods that keep data in `emptyDir` or on local volumes
   on node7 (that data is lost), and whether the other nodes have room for these pods.

4. Remove it. Keep node7 in the inventory for this step:

   ```bash
   ansible-playbook -i inventory/<cluster>/hosts.yaml -b remove-node.yml -e node=node7
   ```

   The playbook asks for confirmation, drains node7, removes it from the cluster, and resets
   the host.

   - **Always pass `-e node=node7`.** It is what limits the run to one node. Recent Kubespray
     releases stop with an error when it is missing. Older releases fell back to every node in
     the group. See which yours does:

     ```bash
     grep -n "default(" remove-node.yml playbooks/remove_node.yml 2>/dev/null
     ```

   - If node7 is already dead or unreachable, add
     `-e '{"reset_nodes": false, "allow_ungraceful_removal": true}'`.
   - Do not use `reset.yml` for this. It wipes Kubernetes from the hosts in the inventory.

5. Verify:

   ```bash
   kubectl get nodes                                             # node7 is gone
   kubectl get pods -A --field-selector status.phase=Pending     # nothing is left unscheduled
   ```

6. Remove node7 from `inventory/<cluster>/hosts.yaml` and commit the change. Otherwise the
   next `cluster.yml` or `scale.yml` run adds it back.

Confidence: the playbook name and variables are from the Kubespray documentation, not from
your checkout. Verify them in your release before step 4.

#### Infrastructure example 4: RHEL, a change that can lock you out

**Request**

> Change SSH to port 2222 on our RHEL 9 server. I'm connected over SSH.

**Final answer**

The risk is locking yourself out. If SELinux or the firewall is not updated before `sshd`
restarts, the new port refuses connections and the old one is gone. These steps keep port 22
open until port 2222 is proven to work. Keep your current session open until step 6 is done.

If the server has a console (cloud serial console, iDRAC, iLO, the hypervisor), make sure you
can reach it before you start. It is the way back in if both ports fail.

1. Confirm the host and back up the config:

   ```bash
   hostname; head -2 /etc/os-release
   sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak-$(date +%F)
   ```

2. Allow the new port in SELinux and in the firewall:

   ```bash
   sudo semanage port -a -t ssh_port_t -p tcp 2222
   sudo firewall-cmd --permanent --add-port=2222/tcp
   sudo firewall-cmd --reload
   ```

   If `semanage` is missing: `sudo dnf install policycoreutils-python-utils`.

3. Make `sshd` listen on both ports for now. In `/etc/ssh/sshd_config`:

   ```
   Port 22
   Port 2222
   ```

4. Validate, then restart:

   ```bash
   sudo sshd -t && sudo systemctl restart sshd
   sudo ss -tlnp | grep sshd     # expect lines for :22 and :2222
   ```

   `sshd -t` prints nothing when the file is valid. If it prints an error, fix it before
   restarting. The restart does not close your current session.

5. From a second terminal, without closing the first:

   ```bash
   ssh -p 2222 <user>@<server>
   ```

   If it fails, port 22 and your first session still work. Check
   `sudo journalctl -u sshd -n 20` and `sudo ausearch -m AVC -ts recent`.

6. Only after step 5 works, delete the `Port 22` line and close the old port:

   ```bash
   sudo sshd -t && sudo systemctl restart sshd
   sudo firewall-cmd --permanent --remove-service=ssh
   sudo firewall-cmd --reload
   ```

   Test `ssh -p 2222` from a new terminal again before closing anything.

Rollback: restore the backup and restart.

```bash
sudo cp -a /etc/ssh/sshd_config.bak-<date> /etc/ssh/sshd_config
sudo sshd -t && sudo systemctl restart sshd
```

Also update whatever connects on port 22 today: Ansible inventories (`ansible_port`),
monitoring checks, jump hosts, and `~/.ssh/config` entries. If a cloud firewall or security
group sits in front of the server, open 2222 there before step 5.

Not verified: I cannot see your server. The commands are for RHEL 9 with firewalld running
and SELinux enforcing, which is the default.

#### Infrastructure example 5: RHEL, a request to turn off a safeguard

**Request**

> SELinux is blocking my app on RHEL 8. How do I disable it?

**Final answer**

Disabling SELinux would make the error go away. It would also remove a protection from every
service on the host in order to fix a problem in one. The block is usually one wrong label,
port, or boolean, and takes minutes to fix. Find the denial first. If you still want SELinux
off after that, step 4 says how.

1. Confirm SELinux is the cause, and read what it denied:

   ```bash
   getenforce                                  # Enforcing, Permissive, or Disabled
   sudo ausearch -m AVC,USER_AVC -ts recent    # denials from the last 10 minutes
   ```

   Each denial names the process (`comm=`, `scontext=`), the target (`path=` or `dest=`,
   `tcontext=`), and the action in braces, such as `{ read }` or `{ name_bind }`. If there are
   no denials, SELinux is not the cause. Look at file permissions and the firewall.

2. Apply the fix that matches the denial:

   | The denial shows | Usual cause | Fix |
   |---|---|---|
   | `name_bind` on a port | The service listens on a non-default port | `sudo semanage port -a -t <type> -p tcp <port>`. For a web server the type is `http_port_t`. |
   | `read`, `open`, or `write` on a path whose `tcontext` is `default_t` or `user_home_t` | Files were moved or created outside the standard location | `sudo semanage fcontext -a -t <type> "/path(/.*)?"`, then `sudo restorecon -Rv /path` |
   | `name_connect` | The service opens an outbound connection, such as a web app to a database | A boolean, such as `sudo setsebool -P httpd_can_network_connect_db on` |

   For an explanation of each denial with a suggested fix:

   ```bash
   sudo dnf install setroubleshoot-server
   sudo sealert -a /var/log/audit/audit.log
   ```

   Read a suggestion before applying it. Do not feed denials to `audit2allow` and load the
   result unread. That grants whatever was denied, including things that should stay denied.

3. To prove SELinux is the only thing in the way, switch to permissive mode for a short test.
   It logs denials without blocking them and lasts until the next reboot:

   ```bash
   sudo setenforce 0     # test the app now
   sudo setenforce 1     # switch back
   ```

   To relax one service while you work on it and leave the rest enforcing:
   `sudo semanage permissive -a <domain>`, for example `httpd_t`.

4. If you still decide to turn it off for the whole host, set `SELINUX=permissive` in
   `/etc/selinux/config` and reboot. Permissive keeps the labels and the log, so turning it
   back on is one edit. `SELINUX=disabled` stops labelling, and re-enabling it later needs a
   full relabel of the filesystem at boot.

Send me the `ausearch` output and I will give the exact fix line. Not verified: I have not
seen your denial, so the table shows the common cases and may not include yours.

#### Infrastructure example 6: Azure, a command that restarts production

**Request**

> Our prod VM app01 is slow. Give me the command to resize it to a bigger size.

**Final answer**

The command is in step 4. Two things first:

- **A resize restarts the VM.** That is downtime for app01. If the new size is not available
  on the hardware the VM sits on, the VM has to be deallocated first, which takes longer. Do
  this in a maintenance window.
- **A bigger size helps only if the VM is short of CPU or memory.** If the limit is the disk
  or the application, you pay more and it stays slow. Step 2 checks this.

`<resource-group>` and `<new-size>` are placeholders.

1. Confirm the subscription and the VM:

   ```bash
   az account show --query "{subscription:name, id:id}" -o table
   az vm show -g <resource-group> -n app01 --query "{size:hardwareProfile.vmSize, location:location}" -o table
   ```

2. Check what is slow, over the last 7 days:

   ```bash
   VM_ID=$(az vm show -g <resource-group> -n app01 --query id -o tsv)
   az monitor metrics list --resource "$VM_ID" --metric "Percentage CPU" \
     --offset 7d --interval PT1H --aggregation Average Maximum -o table
   ```

   Repeat with `"Available Memory Bytes"` and `"OS Disk IOPS Consumed Percentage"`.

   - CPU near 100%, or available memory near zero: a larger size helps.
   - Disk IOPS consumed near 100%: the disk is the limit. Change the disk tier or size.
   - None of these: the cause is in the application or something it calls. A resize will not
     fix it.

3. List the sizes this VM can move to where it is now:

   ```bash
   az vm list-vm-resize-options -g <resource-group> -n app01 -o table
   ```

4. Resize, in the window:

   ```bash
   az vm resize -g <resource-group> -n app01 --size <new-size>
   ```

   If `<new-size>` was not in the list from step 3, the VM must be deallocated first:

   ```bash
   az vm deallocate -g <resource-group> -n app01
   az vm resize -g <resource-group> -n app01 --size <new-size>
   az vm start -g <resource-group> -n app01
   ```

   Deallocating has two side effects. Check both before running it:

   - **The temporary disk is wiped** (`/mnt` or `/mnt/resource` on Linux, `D:` on Windows).
     The OS and data disks are not affected.
   - **A dynamic public IP is released** and can come back different. A static one is kept:

     ```bash
     az network public-ip list -g <resource-group> \
       --query "[].{name:name, allocation:publicIPAllocationMethod, ip:ipAddress}" -o table
     ```

5. Verify:

   ```bash
   az vm get-instance-view -g <resource-group> -n app01 \
     --query "instanceView.statuses[].displayStatus" -o tsv     # includes "VM running"
   az vm show -g <resource-group> -n app01 --query hardwareProfile.vmSize -o tsv
   ```

   Then check the application and the metric from step 2.

Rollback: resize back to the old size with the same command. That restarts the VM again.

Not verified: I cannot see your subscription. A larger size costs more: check the price of
`<new-size>` in your region first. On Windows, a VM cannot move between a size that has a
temporary disk and one that does not.

#### Infrastructure example 7: Azure, a cost request with a real tradeoff

**Request**

> Our Azure Blob storage bill doubled this quarter. Cut it.

**Final answer**

I cannot see the bill, so I do not know which part doubled. A storage bill has four parts:
stored data, operations, data read out, and redundancy. Each has a different fix, and two of
the options below cost more when applied to the wrong data. So: one check, then the options.

**Check first.** In the portal, open Cost Management, then Cost analysis. Scope it to the
storage account, group by Meter, and compare this quarter with the last one. That shows which
part grew.

**Options, by what grew:**

| Option | It saves when | Risk or cost |
|---|---|---|
| A. A lifecycle rule that moves blobs untouched for N days to the Cool, Cold, or Archive tier | Stored data grew and most of it is rarely read | Cooler tiers charge more per read and have minimum storage periods: 30 days for Cool, 90 for Cold, 180 for Archive. Moving or deleting sooner is charged. Archive is offline: a blob must be rehydrated before it can be read, which can take up to 15 hours. Wrong for data that is read often or needed at once. |
| B. Delete what is no longer needed: old blob versions, snapshots, soft-deleted data, stale backups | Stored data grew because of versioning, snapshots, or long retention | Deletion is permanent once the soft-delete period ends. Confirm the owner and the retention requirement for each data set first. |
| C. Lower the redundancy, for example from geo-redundant to zone-redundant or locally redundant | The data can be recreated, or does not need to survive the loss of a region | You give up the copy in the second region. This is a disaster-recovery decision, not only a cost one. Not for the only copy of important data. |
| D. Reduce operations | The transactions meter grew | The cause is usually one client listing or polling in a loop, or logs written as many small blobs. The fix is in that client, so you have to find it first. |
| E. Reduce data read out | The data transfer meter grew | Find who is downloading. Move that consumer into the same region, or put a cache or CDN in front. |

**Recommendation.** Run the check. If stored data is the driver, start with B for old
versions and snapshots, then A with a cautious rule: Cool after 60 to 90 days without access,
and Archive only for data with no need for a fast read. I would not use C to reach a cost
target unless the person who owns the recovery requirements agrees.

**What would change this.** If the meter view shows that operations or data transfer grew,
A, B, and C will not help. Go to D or E.

Not verified: I have no figures from your account. The tier minimums and the rehydration time
are from the Azure documentation. Confirm them, and current prices, on the Azure Blob Storage
pricing page before you rely on them.

#### Infrastructure example 8: automation that touches every server

**Request**

> Write an Ansible playbook to restart nginx on all our web servers.

**Final answer**

```yaml
---
- name: Restart nginx, one host at a time
  hosts: web
  become: true
  serial: 1
  any_errors_fatal: true

  tasks:
    - name: Validate the nginx configuration
      ansible.builtin.command: nginx -t
      changed_when: false
      check_mode: false

    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted

    - name: Wait for nginx to accept connections
      ansible.builtin.wait_for:
        port: 80
        timeout: 30

    - name: Check that the site answers
      ansible.builtin.uri:
        url: http://127.0.0.1/
        status_code: 200
```

Why it is written this way:

- `serial: 1` restarts one server at a time, so the others keep serving. Without it, every
  server restarts at about the same moment and the site drops.
- `nginx -t` runs first. A restart with a broken config leaves nginx down on that host.
- `any_errors_fatal: true` stops the whole run at the first failure, so a problem stops at
  one host.
- The last two tasks confirm the host is serving again before the play moves on.

Assumptions:

- The inventory group is named `web`. Change `hosts:` if it is not.
- nginx listens on port 80 and `/` returns 200 on the host itself. Point the last task at
  your health-check URL and its expected status.

Run it in this order:

```bash
ansible-playbook -i <inventory> restart-nginx.yml --syntax-check
ansible-playbook -i <inventory> restart-nginx.yml --list-hosts          # which hosts will it touch?
ansible-playbook -i <inventory> restart-nginx.yml --check               # dry run; nginx -t still runs
ansible-playbook -i <inventory> restart-nginx.yml --limit <one-host>    # one server first
ansible-playbook -i <inventory> restart-nginx.yml                       # then the rest
```

One question that may make this simpler: do you need a restart, or a reload? To pick up a
config change, `state: reloaded` applies it without dropping connections. A restart is needed
after an nginx upgrade.

Not run: I do not have your inventory or servers. The file parses as valid YAML.

---

## Writing

Use for emails, messages, documents, summaries, commit messages, and edits to existing text.

### Questions to ask first

- Who reads it, and what do they already know?
- What should the reader do or know after reading?
- What is the one main point?
- What tone and channel: email, chat, document? Is there a length limit?
- Which facts are missing: dates, names, numbers, reasons?

### Assumptions

State:

- The tone you chose, if none was given. The default is plain and direct.
- Each placeholder you left, and what goes in it.
- Anything you added that the request did not contain.

Check, do not assume:

- Every name, date, and number in the draft against what the person gave you.

Never invent:

- A reason, a date, a number, a name, a quote, or a commitment the person did not give. Use a
  `[placeholder]` for a missing fact the text cannot do without.

### Steps

1. Write the main point in one sentence. It becomes the first line, or the subject line.
2. List what the reader needs in order to act: what happened, what it means for them, what
   happens next, what you need from them.
3. Draft in that order. One idea per sentence.
4. Cut: a second greeting, a second apology, intensifiers, hedges, restatements, and openers
   such as "I wanted to let you know" or "I hope this finds you well".
5. Replace vague words with specific ones: a date for "soon", a number for "significant", a
   name for "the team".
6. Compare every fact in the draft with the request. Mark gaps with `[brackets]`.
7. Count the words if there is a limit. Read the draft once as the recipient: can they act
   without writing back to ask something?
8. Deliver the text ready to send. After it, add at most two lines: the placeholders to fill
   and the assumptions you made.

### Decision rules

| If | Then |
|---|---|
| No reason was given for the news | Do not invent one. Leave it out, or add `[reason]` and say so. |
| The message carries bad news | Put the news in the first two sentences. Then the impact, the new date or plan, and what you need from the reader. |
| A date is relative ("two days", "next week") | Keep the relative wording. Add the calendar date only if the person gave the original date. Otherwise use a placeholder. |
| The person asks you to edit their text | Keep their meaning and voice. Change only what the request names. Say what you cut or changed. |
| The reader is senior or busy | Lead with the decision or the ask. Put detail after it. |
| The task is a summary | Include only what is in the source. Keep numbers exact. Add no new claims. Mark any number you computed. |
| No tone was given | Match the relationship: direct and respectful to a manager, clear and polite to a customer, casual to a friend. |
| No length was given | A single-point email: under 100 words. A chat message: one to three sentences. A document: as long as the reader needs in order to act. |
| The text is meant to persuade | Use claims that are true and can be supported. Do not invent statistics or quotes. |
| The channel does not render Markdown (email, SMS, a ticket field) | Write plain text. No headers, no bold, no tables. |
| The request could mean "draft it" or "send it" | Draft it. Sending is a separate step that needs a clear yes. |
| The source text contradicts itself or contains an obvious error | Point it out. Do not silently fix it and do not carry it over. |

### Common mistakes and how to catch them

| Mistake | How to catch it |
|---|---|
| Filler at the start and the end | Delete the first and the last sentence. If nothing is lost, leave them out. |
| The point is buried | The first sentence must hold the news or the ask. |
| Invented details | Check each fact in the draft against the request. |
| Too many apologies or explanations | One apology at most. One reason at most. |
| Vague words: "soon", "some issues", "ASAP" | Replace each with a date, a number, or a name. |
| An edit that changes the meaning | Compare each sentence with the original. |
| Text written for the writer, not the reader | Remove what the reader does not need in order to act. |
| A summary that says more than the source | Find the source sentence behind each summary sentence. |
| Over the word limit | Count the words. Do not estimate. |
| The wrong register | Read it as the sender speaking to that recipient. |

### Checks before answering

- The main point is in the first line.
- Every fact comes from the request or is a marked placeholder.
- The word count is within the limit, and it was counted.
- No filler remains.
- The reader knows what happens next and what is wanted from them.
- Placeholders, additions, and assumptions are listed after the text, in at most two lines.

### Stating confidence

In a writing task the uncertainty is about missing facts, not about the wording.

- A missing fact: use a placeholder and name it. "Fill in `[new date]`."
- An assumption: "Assuming a direct, neutral tone. Tell me if you want it warmer."
- An addition: "The last line is mine. Remove it if you do not want replies."
- Coverage of a summary: "This covers the whole report", or "This covers pages 1 to 12. The
  rest was not provided."
- Do not put hedges inside the text itself unless the sender is unsure of the fact.

### Worked examples

#### Writing example 1: a status update with the reason given

**Request**

> Tell the team the release moves from Tuesday to Thursday because QA found a login bug.

**Final answer**

```text
Subject: Release moves from Tuesday to Thursday

Team,

The release moves from Tuesday to Thursday. QA found a login bug, and it has to be fixed before we ship.

If Thursday causes a problem for you, tell me.
```

31 words. Written as an email. For chat, drop the subject line and the greeting. The last
line is my addition. Remove it if you do not want replies.

#### Writing example 2: bad news with missing facts

**Request**

> Email the customer that their order is late.

**Final answer**

```text
Subject: Your order [order number] is delayed

Hello [customer name],

Your order [order number] is late. It will now arrive on [new delivery date].

[One sentence giving the reason, if you want to give one.]

I am sorry for the delay. If the new date does not work for you, reply to this email and we will [what you can offer: a refund, a cancellation, faster shipping].

[Your name]
```

Fill in the bracketed parts. I did not write a reason or an offer, because none was given. If
the new date is not known yet, replace the second sentence with the date the customer will
next hear from you.

#### Writing example 3: an edit

**Request**

> Make this shorter:
>
> "I just wanted to reach out and let you know that, after a great deal of careful
> consideration and many discussions with the wider team, we have ultimately come to the
> decision that we will not be moving forward with renewing the contract with Acme when it
> comes up for renewal at the end of March. I would like to take this opportunity to thank you
> for all of your hard work on this account over the past two years."

**Final answer**

```text
We have decided not to renew the Acme contract at the end of March. Thank you for your work on this account over the past two years.
```

27 words, down from 80. Kept: the decision, the timing, the thanks. Cut: the opener and the
description of how the decision was reached. If the reader needs to know that the team was
consulted, add "after discussion with the team" to the first sentence.

#### Writing example 4: a summary

**Request**

> Summarize this for my director in three bullets:
>
> "On 12 May at 09:14 UTC the payments API began returning errors. The cause was an expired
> TLS certificate on the internal gateway. The certificate had been due for renewal on 10 May,
> but the renewal job had been failing silently since April because its service account had
> lost a permission. Errors continued until 10:02 UTC, when the on-call engineer installed a
> new certificate by hand. About 4,300 payment attempts failed during the incident; 3,900 were
> retried successfully by customers within the hour and roughly 400 were abandoned. We have
> fixed the service account permission, added an alert for certificates within 14 days of
> expiry, and will audit the other renewal jobs by 30 May."

**Final answer**

- **What happened:** On 12 May the payments API returned errors for 48 minutes (09:14 to
  10:02 UTC) because a TLS certificate on the internal gateway had expired.
- **Impact:** About 4,300 payment attempts failed. 3,900 were retried successfully within the
  hour. About 400 were abandoned.
- **Cause and changes:** The renewal job had been failing silently since April, after its
  service account lost a permission. The permission is fixed, an alert now covers
  certificates within 14 days of expiry, and the other renewal jobs will be audited by 30 May.

The 48 minutes is computed from the two times in the report. Everything else is stated in it.

#### Writing example 5: a commit message

**Request**

> Write a commit message. The change: parse_date() used to return None for a date it could not
> read. Now it raises ValueError with the bad string in the message. I updated the three
> callers to handle it and added tests.

**Final answer**

```text
Raise ValueError from parse_date() on unreadable dates

parse_date() returned None when it could not read a date. It now
raises ValueError with the bad string in the message.

Update the three callers to handle the exception. Add tests.
```

The message does not say why the change was made, because the request did not. If the old
behavior caused a specific bug, add one sentence naming it.

#### Writing example 6: a short refusal

**Request**

> Reply to a vendor who wants a demo call next week. I'm not interested right now.

**Final answer**

```text
Hi [name],

Thank you for the offer. I am not interested right now, so I will pass on the call.

If that changes, I will get in touch.

[Your name]
```

30 words. Fill in the two names.

---

## Research

Use for finding facts, comparing options, checking a claim, summarizing sources, and "what is
the current ..." questions.

### Questions to ask first

- What decision or output will this feed?
- How current must it be? Is there a cutoff date?
- Which sources count: official documentation, peer-reviewed work, vendor material?
- How deep: a short answer with a source, or a full comparison?
- Which version, region, or context applies?

### Assumptions

State:

- Whether you consulted live sources in this session or are answering from memory, and the
  date your information runs to.
- The version, region, or scope the answer applies to.

Check, do not assume:

- The claim in a primary source: the official documentation, the standard, the paper, the
  release notes, the data set.
- The date of each source.
- That the source is about the same version, region, and conditions as the question.

Never assume:

- A widely repeated number is correct.
- A source exists because the claim sounds familiar.
- A figure measured under one set of conditions holds under another.

### Steps

1. Restate the question as something that can be checked. Split it into sub-questions.
2. For each one, find the primary source. Use secondary sources to locate primary ones, not to
   replace them.
3. Record for each claim: the source, its date, and the exact figure or wording.
4. Cross-check each important claim against a second, independent source. Note where they
   disagree.
5. Test what you can yourself: run the command, read the code, recompute the number.
6. Separate what the sources say from what you infer.
7. Check recency. Versions, prices, limits, and laws change. Note the date you checked.
8. Answer the question first. Then the support. Then what is uncertain or disputed.
9. Give the sources in a form the reader can follow: title, publisher, date, and a link if
   you have one.

### Decision rules

| If | Then |
|---|---|
| You have no access to current sources | Say so first. Give what you know with its date. Say where to confirm it, with the exact page or command. |
| Two sources disagree | Report both. Say which is more direct, more recent, or more independent. Do not average them. |
| A number matters | Give it with its unit, date, and source. Keep the source's precision. |
| You cannot find a source for a claim | Drop the claim or label it "unsourced". |
| You are asked for a citation and are not certain it exists | Do not produce one. Say what you recall, label it as unverified, and say how to check it. |
| The question asks what is "recommended" or "standard" | Name who recommends it. |
| The source is a vendor describing its own product | Treat performance and comparison figures as claims, and say so. |
| The topic moves fast (software versions, prices, regulations) | Point to the official page and tell the person to confirm there. |
| The evidence is thin | Say "the evidence is limited to ...". Do not round it up to a conclusion. |
| The question rests on a common belief | Check the belief before answering inside it. |
| The claim can be tested in a minute | Test it, and report the result along with the source. |
| Part of the question is a matter of judgment or prediction | Say which part is fact and which is judgment. |

### Common mistakes and how to catch them

| Mistake | How to catch it |
|---|---|
| A citation that was made up, or attributed to the wrong work | Cite only what you read in this session or are certain exists. Mark everything else. |
| An outdated fact stated as current | Put a date on every time-sensitive claim. |
| One source treated as settled | Find a second, independent source for each important claim. |
| A summary that says more than the source | Re-read the source passage next to your sentence. |
| Versions or regions mixed together | Write the version or region next to each claim. |
| Searching only for support | Search for the opposite claim too. |
| A correlation reported as a cause | Check what kind of study it was and what it measured. |
| Precision the source does not have | Keep the source's rounding. |
| A finding about one thing applied to another | Compare what was measured with what is being claimed. |

### Checks before answering

- The direct answer comes first.
- Each claim has a source, or is labeled as inference or as unverified.
- Time-sensitive claims carry a date.
- Disagreements between sources are reported, not hidden.
- The answer says what was not checked.
- The reader has what they need to confirm it: the page, the command, the reference.

### Stating confidence

- Confirmed: "Confirmed in the built-in help for `list.sort`, and by running it."
- From memory: "From memory, as of my information. Confirm on the releases page."
- No access: "I cannot check current sources from here, so I am not giving a version number as
  current."
- Disagreement: "The sources disagree. A reports 40% in its own test. B measured 8% to 12%
  under conditions closer to yours."
- Unverified recollection: "I recall this from a book and have not checked the page. Verify
  it before you cite it."

### Worked examples

#### Research example 1: a fact with a source and a test

**Request**

> Does Python's sort keep equal elements in their original order?

**Final answer**

Yes. Python's sort is stable: elements that compare equal keep their original order. This
holds for `list.sort()` and for `sorted()`, with or without `key=`.

Sources:

- The built-in help for `list.sort` (`python3 -c "help(list.sort)"`) says the sort is
  "stable (i.e. the order of two equal elements is maintained)".
- The Sorting HOW TO in the Python documentation says sorts are guaranteed to be stable.

Verified by running it on Python 3.9:

```python
records = [("bob", 2), ("alice", 1), ("carol", 2), ("dave", 1)]
print(sorted(records, key=lambda r: r[1]))
# [('alice', 1), ('dave', 1), ('bob', 2), ('carol', 2)]
```

`alice` stays ahead of `dave`, and `bob` ahead of `carol`, as in the input.

One case that surprises people: `reverse=True` also keeps equal elements in their original
order. It does not flip them.

```python
print(sorted(records, key=lambda r: r[1], reverse=True))
# [('bob', 2), ('carol', 2), ('alice', 1), ('dave', 1)]
```

#### Research example 2: a fast-moving fact and no live access

**Request**

> What is the latest Kubernetes version?

**Final answer**

I cannot check current sources from here, so I am not giving a version number as current.
Kubernetes ships a new minor release about three times a year, so a number from memory is
likely to be out of date.

To get the answer now:

```bash
curl -L -s https://dl.k8s.io/release/stable.txt     # prints the latest stable release
```

Or read the release page, https://kubernetes.io/releases/. It lists the supported minor
releases and their end-of-life dates.

Two questions that often sit behind this one:

- **What does my cluster run?** `kubectl version` shows the client and server versions.
- **What can I upgrade to?** The project maintains the three most recent minor releases, and a
  cluster is upgraded one minor version at a time. The target is normally the next minor
  release above your server version, not the latest one.

Tell me your current version and I will lay out the upgrade path.

#### Research example 3: two sources that disagree

**Request**

> Should I tell my team the new storage driver makes things 40% faster? The vendor's blog says
> 40% higher throughput. An independent benchmark I found measured 8 to 12%. The vendor tested
> sequential writes with 1 MB blocks on NVMe. The independent test ran a mixed read/write
> database workload on the same class of hardware.

**Final answer**

No. Do not tell them 40%. The two numbers measure different things, so report both with what
each one measured.

| Source | Result | Workload | Who ran it |
|---|---|---|---|
| Vendor blog | 40% higher throughput | Sequential writes, 1 MB blocks, NVMe | The vendor, on its own product |
| Independent benchmark | 8% to 12% | Mixed read/write database workload, same class of hardware | A third party |

Both can be true. A likely reason for the gap: large sequential writes are a workload where a
driver change shows up strongly, while a database workload is also limited by random I/O,
locking, and CPU. That explanation is my inference, not something either source states.

Treat the vendor's figure as the upper end for one workload, reported by the party selling
the product.

Wording you can defend:

```text
The vendor reports 40% higher throughput on large sequential writes. An independent test on a
database workload measured 8% to 12%. Our workload is closer to [which one], so we should
expect [that range] until we measure it ourselves.
```

What would settle it: run your own workload on the old and the new driver on one test node
and compare. That is the only number that applies to you.

Not verified: I have not seen either source. I am taking the test details as you described
them.

#### Research example 4: a question built on a common belief

**Request**

> How long does DNS propagation take? I heard 48 hours. We're moving our site to a new IP on
> Friday.

**Final answer**

There is no fixed propagation time, and 48 hours is not a rule for this change. Each resolver
keeps your old record until its TTL (time to live) runs out, then asks again. The wait is at
most the TTL of the record you are changing, and you control that.

For Friday:

1. Check the current TTL. The second column is the TTL in seconds:

   ```bash
   dig +noall +answer www.example.com A @<your-nameserver>
   ```

2. At least one full TTL before Friday, lower the TTL to 300 seconds. If it is 86400 (24
   hours) now, do this by Thursday morning at the latest.
3. On Friday, change the IP. Resolvers pick up the new address within about 5 minutes.
4. Keep the old server running for a day. Some resolvers and applications hold a record
   longer than its TTL says.
5. When it is stable, raise the TTL again.

Where the 48 hours comes from: changing a domain's **nameservers** at the registrar. The
delegation records held by the `.com` registry carry a TTL of 172800 seconds, which is 48
hours, and you cannot lower it. That does not apply to changing an A record at the same DNS
provider.

Verified: `dig +norec @a.gtld-servers.net example.com NS` shows the 172800-second TTL on a
`.com` delegation.

Not verified: your record's current TTL. Step 1 shows it. If you are also switching DNS
providers on Friday, tell me. Then the 48 hours does apply, and the plan changes.

#### Research example 5: a request for a citation

**Request**

> I need a citation for "code review catches 60% of defects" for a slide.

**Final answer**

I cannot give you a citation I have checked, and I will not write one that only looks right.
Here is what I recall, labeled as unverified:

- Steve McConnell's *Code Complete* (2nd edition), in the chapter on software quality, has a
  table of defect-detection rates. As I recall it, **formal code inspections** find 45% to 70%
  of defects, with 60% as the typical figure, and **informal code reviews** find 20% to 35%.
  The table is adapted from data by Capers Jones and others.

Two cautions before this goes on a slide:

1. **I have not checked the page.** This is from memory. Open the book and confirm the table
   and the numbers.
2. **The claim and the source do not match.** If my recollection is right, 60% is for formal
   inspections: a structured process with prepared reviewers and defined roles. Everyday
   pull-request review is closer to the informal row, which is much lower. "Code review
   catches 60% of defects" would overstate it.

Wording the source would support, once you have verified it:

```text
Formal code inspections typically find around 60% of defects (McConnell, Code Complete, 2nd ed.).
```

If you cannot get the book, drop the number and make the point without a figure.

---

## Math

Use for calculations, word problems, estimates, statistics, unit conversions, and proofs.

### Questions to ask first

- What exactly is wanted: a value, a range, an estimate, a proof? In which units, and to what
  precision?
- Are the given numbers exact or rounded?
- Which convention applies: the compounding period, inclusive or exclusive bounds, with or
  without replacement, decimal or binary units?
- Is an exact answer needed, or is an estimate enough?

### Assumptions

State:

- The reading you chose when the wording allows two.
- The units, the rounding, and the convention used (for example a 30-day month, or
  1 TB = 10^12 bytes).
- Independence, a distribution, or any other condition the method needs.

Check, do not assume:

- The units agree on both sides.
- The conditions of a formula or theorem hold before you use it.
- The given data is consistent with itself.

### Steps

1. Restate the problem: the knowns, the unknown, the units.
2. Choose the method, and check that its conditions hold.
3. Compute step by step. Carry the units. Keep full precision until the end.
4. Use a calculator or code for arithmetic beyond a few digits.
5. Check the result a second way: the inverse operation, substituting back, a different
   method, a small case counted by hand, an order-of-magnitude estimate, or a simulation.
6. Check that the result is reasonable: its sign, its size, its behavior at the extremes.
7. Re-read the question. Confirm the result answers what was asked.
8. Give the answer with units and precision, then the key steps, then the assumptions.

### Decision rules

| If | Then |
|---|---|
| The wording has two readings | State the reading you used. If both are cheap to compute, give both. |
| The problem involves probability | Define the sample space, and say whether the events are independent, before computing. |
| The arithmetic has many steps or more than about four significant digits | Compute it with a tool and report the tool's result. |
| The result is surprising | Verify it by a second method before presenting it. |
| It is an estimate | Give a range, and name the assumption that moves it the most. |
| A proof is asked for | State what is given and what is to be shown. Justify each step. Test the claim on an example. |
| Rounding is needed | Round once, at the end, and say to how many places. |
| The problem looks like a known trap (percent of percent, average of rates, base rates, adding percentiles) | Compute from the definitions with concrete numbers. Do not answer from the pattern. |
| Units are mixed | Convert first and show the conversion. |
| The answer cannot be computed from what was given | Say so, say what is missing, and give what can be said: a bound or a range. |
| The given data contradicts itself | Point it out. Do not force an answer. |
| Items come in whole units (pods, people, servers, boxes) | Divide per container, not in total, and round in the direction the situation requires. |

### Common mistakes and how to catch them

| Mistake | How to catch it |
|---|---|
| Averaging rates or percentages directly | Compute from totals: total distance over total time, total hits over total tries. |
| Percent of what? Percent mixed up with percentage points | Name the base of every percentage. |
| Off-by-one in counting (fence posts, inclusive ranges) | Count a small case by hand. |
| Unit slips: MB and MiB, bits and bytes, monthly and annual, ms and s | Write the unit on every line. |
| Assuming independence | State it. Ask what would make the events move together. |
| Arithmetic slips | Recompute with a tool, or apply the inverse operation. |
| Rounding too early | Keep full precision until the last step. |
| Answering a different question, such as P(A given B) for P(B given A) | Re-read the question after solving. |
| Correct arithmetic on the wrong model | Compare the result with something known. |
| Adding or averaging percentiles | Compute the percentile of the per-item totals, not from the parts' percentiles. |

### Checks before answering

- The answer has units and a stated precision.
- A second method or a substitution confirmed it.
- The assumptions and conventions are stated.
- The result answers the question that was asked.
- Whole-unit quantities were rounded in the right direction.

### Stating confidence

- Exact: "Exactly 40 mph."
- Checked: "Checked by substituting back", or "Checked with one million simulated cases."
- Estimate: "Between 5 and 6.5 hours. The share of the link you get is what moves it."
- Conditional: "About 9%, if '99% accurate' means the false-positive rate and the
  false-negative rate are both 1%."
- Not computable: "This cannot be computed from the two numbers given. What can be said
  is ..."

### Worked examples

#### Math example 1: a percentage of a percentage

**Request**

> A price went up 20% and then down 20%. Is it back where it started?

**Final answer**

No. It ends 4% below where it started.

Start at 100. Up 20% gives 120. Down 20% of 120 is 24, which gives 96.

The two percentages have different bases. The rise is 20% of 100. The fall is 20% of 120. The
order does not matter: down 20% and then up 20% also gives 96.

To get from 120 back to 100, the drop has to be 16.67% (20 out of 120).

Checked: 100 x 1.2 x 0.8 = 96.

#### Math example 2: an average of rates

**Request**

> I drive to work at 30 mph and back at 60 mph. What's my average speed?

**Final answer**

40 mph, not 45.

Average speed is total distance divided by total time. The slower leg takes twice as long, so
it counts for more.

Take 60 miles each way. Out takes 2 hours. Back takes 1 hour. That is 120 miles in 3 hours:
40 mph.

The distance does not change the answer. Checked with one-way distances of 1, 15, 60, and 240
miles: 40 mph each time.

Assuming the same route in both directions. If the two legs took equal time, not equal
distance, the answer would be 45 mph.

#### Math example 3: whole units and capacity

**Request**

> We have 3 nodes with 16 GiB of memory each. Each pod requests 1.5 GiB. How many pods fit?

**Final answer**

At most 30, and fewer in practice: about 27, or 18 if the pods must survive the loss of a
node.

- **Not 32.** 48 GiB divided by 1.5 GiB is 32, but that treats the cluster as one pool. A pod
  cannot be split across nodes, so divide per node and round down.
- **Upper bound: 30.** 16 divided by 1.5 is 10.67, so 10 pods per node, times 3 nodes.
- **Realistic: about 27.** Pods are scheduled against a node's allocatable memory, which is
  its capacity minus what is reserved for the system and the kubelet. If allocatable is
  14.5 GiB, that is 9 pods per node (9 x 1.5 = 13.5 fits, 10 x 1.5 = 15 does not), so 27.
  Pods already on the nodes, such as DaemonSets, lower it further.
- **With one node down: 18.** Two nodes times 9. If these pods must keep running through a
  node failure or a drain, plan for 18.

Get the real numbers from your cluster:

```bash
kubectl describe node <node> | grep -A 7 "Allocatable:"
kubectl describe node <node> | grep -A 8 "Allocated resources:"
```

Per node, pods = (allocatable memory - memory already requested) / 1.5 GiB, rounded down. Add
up the nodes.

Assumptions: memory is the only limit. CPU requests and the per-node pod limit (110 by
default) can cap it sooner. The 14.5 GiB is an example, not your value.

#### Math example 4: a base rate

**Request**

> A test for a disease is 99% accurate. 1 person in 1,000 has the disease. I tested positive.
> What is the chance I have it?

**Final answer**

About 9%, not 99%.

Assuming "99% accurate" means both of these: 99% of people with the disease test positive, and
99% of people without it test negative.

Take 100,000 people:

- 100 have the disease. 99 of them test positive.
- 99,900 do not. 1% of them, 999 people, test positive anyway.

So 1,098 people test positive and 99 of them have the disease: 99 / 1,098 = 9.0%.

The result is low because the disease is rare. The healthy group is so large that its 1% of
false positives outnumbers the true positives ten to one.

Checked with Bayes' rule: (0.001 x 0.99) / (0.001 x 0.99 + 0.999 x 0.01) = 11/122 = 0.0902.

What changes the answer:

- The 1 in 1,000 is the rate for the whole population. If you were tested because of symptoms
  or a known exposure, your starting chance is higher, and so is the result. At 1 in 10 it
  would be 92%.
- A second, independent positive test raises it sharply.

This is the arithmetic only. What it means for you is a question for the doctor who ordered
the test.

#### Math example 5: a conversion with a convention

**Request**

> How much downtime does 99.9% uptime allow per month?

**Final answer**

43.2 minutes in a 30-day month.

A 30-day month has 30 x 24 x 60 = 43,200 minutes. 0.1% of that is 43.2 minutes.

| Period | Downtime allowed at 99.9% |
|---|---|
| 30-day month | 43.2 minutes |
| 31-day month | 44.64 minutes |
| 365-day year | 8.76 hours (8 h 45 m 36 s) |

For comparison, 99.95% allows 21.6 minutes in a 30-day month, and 99.99% allows 4.32 minutes.

Before relying on the figure, check how your agreement measures it: the period (calendar
month, 30 days, or rolling), whether planned maintenance counts, and what counts as "down".
Those change the number more than the arithmetic does.

#### Math example 6: an estimate

**Request**

> How long will it take to copy 2 TB over our 1 Gbps link?

**Final answer**

About 4.5 hours at the theoretical maximum. Plan for 5 to 6.5 hours.

- 2 TB = 2 x 10^12 bytes = 1.6 x 10^13 bits.
- 1 Gbps = 10^9 bits per second.
- 1.6 x 10^13 / 10^9 = 16,000 seconds = 4 h 26 m 40 s.

A link rarely carries its nominal rate end to end. Protocol overhead, other traffic, and disk
speed each take a share:

| Share of the link you get | Time |
|---|---|
| 100% | 4 h 27 m |
| 90% | 4 h 56 m |
| 80% | 5 h 33 m |
| 70% | 6 h 21 m |

Assumptions that move the result:

- **Units.** If the tool reports 2 TiB (binary units), that is about 10% more data: 4 h 53 m
  at the full rate.
- **The slowest part sets the pace.** 1 Gbps is 125 MB per second. A disk that reads or
  writes more slowly than that is the limit, not the link. Many small files copy more slowly
  than a few large ones.
- **Sharing.** If the link also carries production traffic, either the copy slows down or the
  traffic does. Consider a rate limit, such as `rsync --bwlimit`.

To replace the estimate with a measurement, copy a 10 GB sample, time it, and scale up.

#### Math example 7: a proof

**Request**

> Prove that the sum of two odd numbers is even.

**Final answer**

**Given:** two odd integers a and b. **To show:** a + b is even.

An integer is odd when it can be written as 2k + 1 for some integer k. It is even when it can
be written as 2k.

1. a is odd, so a = 2m + 1 for some integer m.
2. b is odd, so b = 2n + 1 for some integer n.
3. a + b = (2m + 1) + (2n + 1) = 2m + 2n + 2 = 2(m + n + 1).
4. m + n + 1 is an integer, so a + b is 2 times an integer. That is the definition of even.

Checked on examples: 3 + 5 = 8, -7 + 1 = -6, 1 + 1 = 2. The last one shows that a and b may
be the same number, which is why the proof uses separate letters m and n.

#### Math example 8: a question that cannot be answered from the data given

**Request**

> Service A has a p95 latency of 200 ms and service B has 300 ms. A request calls A and then
> B. Is the p95 of the whole request 500 ms?

**Final answer**

Not necessarily. Percentiles do not add, and the p95 of the total cannot be computed from
those two numbers. It depends on whether the same requests are slow in both services.

| If | Then the p95 of the total is |
|---|---|
| The same requests are slow in both services | 500 ms |
| Slowness in A and in B is independent | Below 500 ms. About 400 ms in a simulation with exponential latencies. |
| Different requests are slow in each service | Possibly far above 500 ms. 1,200 ms in a constructed case. |

What the two numbers do guarantee: at least 90% of requests finish within 500 ms. A request
can take longer than 500 ms only if A took more than 200 ms or B took more than 300 ms, and
each of those happens to at most 5% of requests.

To get the real figure, measure the total time per request, from the caller or from traces,
and take the 95th percentile of that.

Checked by simulation with one million requests per case. The three results in the table are
from those runs. In the constructed case, A is slow on 4.9% of requests and B is slow on a
different 4.9%.

---

## Planning

Use for project plans, migration plans, maintenance windows, schedules, and breaking work into
tasks.

### Questions to ask first

- What is the goal, and how will we know it has been reached?
- What is the deadline, and is it fixed?
- Who and what is available: people, hours, budget, tools, access?
- Which constraints apply: change windows, freeze dates, approvals, other teams?
- What must not break: uptime, data, compliance?
- What has already been decided or done?

### Assumptions

State:

- The team size, the hours available, and the start date you planned with.
- The unknowns the plan depends on, and what you assumed for each.
- What the estimates are based on.

Check, do not assume:

- Lead times outside your control: approvals, access, procurement, another team's work.
- Fixed dates.
- That the estimates add up to less than the time available.

Never assume:

- Every hour of a working day is available for the work.
- A step that worked in a test will take the same time in production.
- A rollback exists because nobody said it does not.

### Steps

1. Write the goal as an outcome with a check that shows it is done.
2. List the constraints and the fixed dates.
3. Work backward from the done state. List the deliverables, then the tasks for each.
4. Order the tasks by dependency. Mark what can run in parallel. Find the longest chain.
5. Estimate each task as a range. Add up the low ends and the high ends. Compare both with
   the time and capacity available.
6. Put the least-known item first, as a short test, so the plan can change early.
7. Add checkpoints with a go or no-go test. For a change to a live system, add the backup,
   the rollback, and the point of no return.
8. Name an owner for each task, or write "owner needed".
9. List the main risks, each with the sign that it is happening and the response.
10. Present the goal, the timeline, the tasks, the risks, the open questions, and the first
    step to take now.

### Decision rules

| If | Then |
|---|---|
| The estimates add up to more than the time available | Say so first. Give options: cut scope, move the date, or add people and count their ramp-up time. Do not shrink the estimates to fit. |
| A task is longer than about two days or holds an unknown | Split it, or put a time-boxed investigation in front of it. |
| The plan changes a production system | Include a backup, a rehearsal outside production, a canary, a rollback, a communication step, and the point of no return. |
| A dependency is outside the team's control | Start it first and name a fallback. |
| Information is missing | Plan with stated assumptions and list the questions that would change the plan. Do not wait for all the answers. |
| One level of the plan has more than about seven steps | Group them into phases. |
| The plan covers a day or a week of one person's time | Size it to the real hours. Leave slack for meetings and interruptions. |
| Success depends on one person or one system | Name it as a single point of failure. |
| A step cannot be undone (a cutover, a deletion, a decommission) | Mark it. Put a go or no-go check before it. Keep the old system until the new one is verified. |
| Two orders are possible | Choose the order that produces something testable sooner. |
| The request names a time limit (a downtime budget, a window) | Build the plan to the limit, and say which steps consume it. |
| A step's duration is unknown | Rehearse it and measure it before promising a time. |

### Common mistakes and how to catch them

| Mistake | How to catch it |
|---|---|
| No definition of done | Write the acceptance check before the first task. |
| Estimates as single numbers | Use ranges. Add up the high ends to see the worst case. |
| Hidden dependencies: access, approvals, DNS TTLs, data transfer time, reviews | For each task ask what must be true before it can start. |
| No rollback | Write the undo next to every step that changes something. |
| Only the case where everything works is planned | List the three most likely failures, with the sign of each and the response. |
| Too much detail far ahead, too little for next week | Detail the next two weeks. Outline the rest. |
| Meetings, holidays, and on-call time ignored | Count working days and the hours really available. |
| Tasks with no owner | Name one, or write "owner needed". |
| A plan that hides the fact that it does not fit | Put the gap in the first sentence. |
| The point of no return is not marked | Find the step after which the old state cannot be restored, and label it. |

### Checks before answering

- The goal can be tested.
- The tasks are in dependency order.
- The estimates add up, and any gap against the deadline is stated.
- A live change has a backup, a rollback, and a marked point of no return.
- The risks have triggers and responses.
- The open questions are listed, with what each one changes.
- The first step is clear.

### Stating confidence

- Estimates: give ranges. "3 to 5 hours one at a time."
- Assumption-driven: "This plan assumes the database is under 1 TB. A larger one adds copy
  time, not a different design."
- Weak spot: "Phase 3 is the least certain estimate. The rehearsal will replace it with a
  measurement."
- Not verified: "I have not seen your systems. The first step produces the facts the rest is
  sized from."

### Worked examples

#### Planning example 1: a week that does not fit

**Request**

> I have 5 working days. Tasks: write the design doc (2 days), review 3 PRs (half a day), fix
> the login bug (1 to 2 days), prepare the quarterly demo (1 day), onboard the new hire
> (1 day). Plan my week.

**Final answer**

It does not fit. The tasks add up to 5.5 to 6.5 days and you have 5, before any meetings or
interruptions. Something has to move, shrink, or go to someone else.

| Task | Estimate in days |
|---|---|
| Design doc | 2 |
| Review 3 PRs | 0.5 |
| Fix the login bug | 1 to 2 |
| Prepare the quarterly demo | 1 |
| Onboard the new hire | 1 |
| **Total** | **5.5 to 6.5** |

Options:

| Option | Frees | Risk |
|---|---|---|
| A. Move the second half of the design doc to next week | 1 day | Whoever is waiting for the doc waits longer. |
| B. Hand the PR reviews to a teammate, and share the onboarding with a buddy | 1 day | Depends on two other people having time. |
| C. Stop work on the bug after 1.5 days and escalate if it is not fixed | Up to 0.5 day | The bug may stay open. |

Recommendation: A, with the limit from C. It needs nobody else's time, and it protects the
bug fix, which has the least certain estimate. That brings the week to 4.5 to 5 days.

| Day | Morning | Afternoon |
|---|---|---|
| Mon | Login bug | Login bug |
| Tue | Login bug, or design doc if the bug is fixed | PR reviews |
| Wed | Onboarding | Onboarding |
| Thu | Design doc, first half | Design doc, first half |
| Fri | Demo preparation | Demo preparation |

This plan has no slack. Every hour of meetings comes out of the design doc on Thursday.

Two things I assumed. Tell me if either is wrong:

- The bug comes first because it affects users.
- Onboarding is on Wednesday. Move it to the day the new hire starts.

What would change the recommendation: if the design doc has a fixed date this week, choose B.

#### Planning example 2: a maintenance window

**Request**

> Plan a maintenance window to patch 12 RHEL 8 servers this Saturday.

**Final answer**

Assumptions, because the request does not say: the servers are production, each service runs
on more than one of them so it can lose one member at a time, and reboots are acceptable in
the window. If any server has no redundant peer, such as a standalone database, tell me. It
needs its own slot and either a failover or a downtime notice.

**Goal.** All 12 servers on the current packages and kernel, every service healthy, no
unplanned outage.

**Before Saturday**

1. Inventory. List the 12 hosts with role, owner, and whether each has a redundant peer.
   Split them into three batches so that no batch holds every member of one service.
2. See what will change, on one host per role:

   ```bash
   sudo dnf check-update
   sudo dnf updateinfo list --security
   ```

   Note the kernel, glibc, openssl, and database packages. They need a reboot or a service
   restart.
3. Patch a non-production host of each role first, if one exists, and run the application
   checks on it.
4. Confirm that each host has a snapshot or a tested backup scheduled just before the window.
   Write down the restore step.
5. Write the health checks per role: the service is active, the port answers, the health URL
   returns 200, the host is back in the load balancer.
6. Announce the window: time, hosts, expected impact, contact.
7. Agree the stop rule in advance: if a host fails its checks and is not fixed in 30 minutes,
   roll that host back and patch nothing else.

**Saturday**

| Step | What | Check before moving on |
|---|---|---|
| 1 | Snapshot all 12 hosts | Every snapshot exists |
| 2 | Batch 1: one canary host. Remove it from the load balancer, patch, reboot. | Health checks pass. Watch it for 15 minutes. |
| 3 | Batch 2: 5 hosts, never two of the same service at once | Each host is healthy before the next of its service starts |
| 4 | Batch 3: the remaining 6, same rule | Same |
| 5 | Final check on all 12 | New kernel running, no failed units, services healthy |

Per host:

```bash
sudo dnf update -y
sudo dnf needs-restarting -r      # exit code 1 means a reboot is needed
sudo systemctl reboot
```

After it is back:

```bash
uname -r                          # the new kernel
systemctl --failed                # expect no units listed
sudo dnf history info last        # what was changed
```

`dnf needs-restarting` comes from the `dnf-utils` package.

**Rollback**

- A package problem: `sudo dnf history undo last` on that host.
- A kernel problem: boot the previous kernel from the GRUB menu, then make it the default
  with `sudo grubby --set-default /boot/vmlinuz-<previous-version>`.
- Anything else: restore the snapshot from step 1.

**Time.** Each host takes 15 to 25 minutes to patch, reboot, and check. One at a time, that is
3 to 5 hours. With two or three hosts of different services in parallel after the canary, it
is 1.5 to 2.5 hours. Add 30 minutes for snapshots and 30 minutes of slack. Book 4 hours.

**Risks**

| Risk | Sign | Response |
|---|---|---|
| A service does not start after the reboot | `systemctl --failed` lists it | Fix it or roll that host back. Stop the batch. |
| A host does not come back | No SSH after 10 minutes | Use the console. Boot the previous kernel, or restore the snapshot. |
| A third-party kernel module or agent fails on the new kernel | The module does not load | Boot the previous kernel. Hold the kernel back until the vendor updates the module. |
| An update brings a version the application does not support | The application check fails | `dnf history undo last`, then pin the package. |

**Open questions.** Each one changes the plan:

1. Which hosts have no redundant peer?
2. Is there a non-production copy to patch first?
3. Security updates only (`dnf update --security`), or everything?
4. Who confirms that each service is healthy?

**First step today.** The inventory. The batches depend on it.

Not verified: I have not seen the servers. The commands are for RHEL 8.

#### Planning example 3: a migration with a downtime limit

**Request**

> Plan the migration of our PostgreSQL database from our data center to Azure. We can afford
> about 30 minutes of downtime.

**Final answer**

A 30-minute limit rules out dump-and-restore for anything but a small database. This plan
copies the data to Azure with continuous replication and then makes a short cutover. Three
facts I do not have would change it. The plan states what it assumes for each, and they come
back as open questions at the end.

**Assumptions**

- The target is Azure Database for PostgreSQL flexible server, the managed service. For
  PostgreSQL on a VM, phases 2 and 3 change.
- The database is under 1 TB. A larger one adds copy time, not a different design.
- The source is PostgreSQL 10 or later, and you can set `wal_level = logical`. That setting
  needs a restart of the source.

**Done means.** The application runs against Azure. Row counts match on every table and
checksums match on the agreed tables. Write downtime was 30 minutes or less. The old database
is intact and can be switched back to.

**Phases**

| Phase | What | Exit check | Working days |
|---|---|---|---|
| 1. Assess | Size, version, extensions, large objects, tables without a primary key, peak write rate, network path to Azure | A written list of blockers, each with an owner | 2 to 4 |
| 2. Build | Create the target server: same major version, extensions enabled, sized from phase 1. Private network path. Roles and schema loaded. | The application's tests pass against the empty schema | 3 to 5 |
| 3. Rehearse | A full migration into a throwaway target. Measure the copy time and the replication lag. Practice the cutover and the rollback with a test copy of the application. | The rehearsed cutover took 15 minutes or less, and the rollback worked | 5 to 10 |
| 4. Replicate | Start the real replication ahead of the cutover. Freeze schema changes. Watch the lag. | Lag stays at a few seconds or less through two days of peak load | 3 to 7 |
| 5. Cut over | The runbook below | Its checks pass inside the window | 1 |
| 6. Stabilize | Watch errors, latency, and slow queries. Keep the old database as the fallback. | An agreed number of clean days. Decommissioning is a separate, approved step. | 5 to 10 |

Total: 19 to 37 working days, about 4 to 7.5 weeks. Phase 3 is the least certain estimate. The
rehearsal replaces it with a measurement.

**Cutover runbook, 30 minutes**

| Minute | Step | Go or no-go |
|---|---|---|
| 0 | Announce the start. Put the application into maintenance mode so writes stop. | |
| 0 to 5 | Confirm nothing is writing to the source. Wait for the replication lag to reach zero. | Lag is zero |
| 5 to 10 | Copy what replication does not carry: the sequence values. Run the data checks. | Every check matches. If not, reopen the application on the old database. |
| 10 to 15 | Point the application at Azure and restart it. | |
| 15 to 25 | Smoke tests: log in, read, write, one of each critical flow. | All pass. If not by minute 25, switch back to the old database. |
| 25 to 30 | Reopen to users. Announce. | |

**The point of no return** is the first write accepted on Azure after reopening. Before it,
rollback means pointing the application back at the old database, which has not changed.
After it, going back means carrying the new writes back too. Decide before the window whether
to set up reverse replication or to accept a defined amount of data loss on a late rollback.

**What logical replication does not copy.** Each item needs its own step: sequence values,
schema changes made after the initial load, and large objects. A table needs a primary key or
a replica identity for its updates and deletes to replicate.

**Risks**

| Risk | Sign | Response |
|---|---|---|
| The initial copy is slower than planned | The rehearsal copy runs over its estimate | More bandwidth, or start phase 4 earlier |
| The lag does not reach zero | Lag grows at peak | Cut over at the quietest hour. Find the long transaction. Resize the target. |
| An extension or feature is missing on the managed service | Found in phase 1 | Replace it, or move to PostgreSQL on a VM |
| Queries are slower on Azure | Slow queries in the phase 3 tests | Tune and resize before the cutover, not after |
| A rollback is needed after reopening | Errors after minute 30 | The decision made in advance: reverse replication, or a defined data loss |

**Open questions**

1. How large is the database, and what is the peak write rate? That sets the copy time and
   whether the lag can reach zero.
2. Which PostgreSQL version, and which extensions?
3. Is the 30 minutes a total, or 30 minutes without writes while reads continue?
4. Can the application be put into a maintenance or read-only mode?
5. Who makes the go or no-go call during the window?

**First step.** Run the assessment this week: size, version, extensions, tables without
primary keys. Everything else is sized from it.

Not verified: I have not seen your database. Check the supported versions and the extension
list for the target service in the current Azure documentation. Both change.
