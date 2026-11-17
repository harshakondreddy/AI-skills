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
