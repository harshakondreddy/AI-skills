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
