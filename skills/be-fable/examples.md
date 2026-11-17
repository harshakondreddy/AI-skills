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
