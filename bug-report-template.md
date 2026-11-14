# Bug report

Copy this, fill it in, and send it as the kit's `README.md` says: **privately** if it gets
around a security promise, as a GitHub issue otherwise. Short is fine; exact is required.

## 1. What

One sentence: what happened that should not have.

## 2. Versions

* Piece: `pg_agent_gate` / `pg_living_assertions` / `pg_promise_guard` / `pg_grammar_guard` /
  `pg_plan_guard` / `pg_recall_guard` / `pg_phasevec` (preview)
* Version and commit:
  * pg_agent_gate: `SELECT extversion FROM pg_extension WHERE extname = 'pg_agent_gate';`
    and, from the agent's session, `SELECT agent_gate.whoami();`
  * pg_phasevec: `SELECT extversion FROM pg_extension WHERE extname = 'pg_phasevec';`
  * `git rev-parse --short HEAD` of the source you built
* PostgreSQL: `SELECT version();` -- and whether it is a release or an assertion-enabled
  build (`SHOW debug_assertions;`)
* OS and how you installed PostgreSQL (package, source, container image and tag)
* Non-default settings that matter (`shared_preload_libraries`, `max_worker_processes`,
  isolation level, anything set on the roles involved)

## 3. Which promise it breaks

Name it from the piece's `promises-and-limits-*.md` (e.g. "pg_agent_gate promise 3: a write
that touches more than `max_rows` is undone"). If you think it is one of the documented
limits but with a consequence the README does not state, say which limit.

## 4. Exact reproduction

From a fresh cluster (`initdb`), as one script that runs top to bottom. Say which role runs
each part.

```sql
-- as the superuser: set-up
...

-- as the agent / author / reader / writer (say which): the attack
...

-- as the superuser: the check of what really changed
...
```

If it needs concurrency, give each session's steps in order, with the points where one waits
for the other.

## 5. Expected vs observed

* Expected (according to the promise):
* Observed (paste the output of the attack and of the superuser's check, unedited):

## 6. Reliability and impact

* Every time / sometimes (how often) / once
* What an attacker gains: executes SQL, changes rows, reads rows, a wrong verdict served, a
  crash, a hang, ...
* Who the attacker must be: an agent role, an author, a table writer, a reader, a role with
  only `USAGE`, ...

## 7. Anything else

A minimal fix you suspect, related findings, how you would like to be credited (or not).
