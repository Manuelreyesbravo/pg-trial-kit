# What to attack: pg_agent_gate 0.4.3

Set-up: the quickstart's cluster, one agent role (`billing_agent`), and a second psql as
`postgres` to check what really changed. Raise the stakes as you go: a tenant table with a
row-level policy over `current_setting('app.tenant_id')`, a trigger, a foreign key with
`ON DELETE CASCADE`, an `allow_ddl` agent, a bound assertion.

## How to tell a real finding from a documented limit

A real finding is **all three**:

1. **The agent did it**, from a session that `whoami()` reports as `"enforced": true`, with the
   library in `shared_preload_libraries`, and no superuser help after `register_agent`.
2. **The database changed, or something leaked**, as seen from the superuser session -- not the
   gate's own JSON. A rows count, a sequence value, a lock in `pg_locks`, another tenant's row
   in the agent's output, a record row missing or edited.
3. **It breaks a promise** in `promises-and-limits-pg_agent_gate.md` (numbered 1-7) and is not
   one of its documented limits -- or it is a consequence of a limit that the README does not
   state.

What is **not** a finding: a PostgreSQL permission error (it proves nothing about the gate); a
statement within the agent's grants and `max_rows` that does what the agent wrote but not
what a human meant (intent is out of scope by design); anything that needs a superuser,
`postgresql.conf`, or the gate not preloaded; a view without `security_invoker` showing its
owner's rows; a DBA's `STABLE`/`IMMUTABLE` function that writes (documented, counted by the
backstop). A crash of the backend from an agent session is always worth reporting.

## Attack ideas

**1. Execute anything but the verbs** (promise 1; suite `adversarial.sh`, 59 checks)
* Every channel: simple and extended protocol, a named portal, `DECLARE CURSOR`, `PREPARE` /
  `EXECUTE`, `COPY`, `DO`, `CALL`, `EXPLAIN ANALYZE`, the fast-path function call (`PQfn`, e.g.
  libpq's large-object API), pipelining several messages.
* The verb's own arguments: an expression, a subquery, a cast, a function call, `VARIADIC`,
  a composite, a `varchar[]` parameter, a default argument. Only literals and parameters
  should pass.
* After an error inside a verb, after a cancel mid-act, inside an explicit transaction that
  is rolled back: is the next statement still refused?

**2. Get a proposal past verification** (promise 2)
* Hide a write in a read: a writing CTE, `SELECT ... FOR UPDATE`, a function that runs SQL
  from text, a built-in that changes state (sequences, advisory locks, large objects,
  `pg_notify`, WAL, statistics). On PostgreSQL 19, look for built-ins the reviewed list may
  not know (the list was derived on 18.6 and 19beta2).
* Hide a function where the walker does not look -- the README calls this "a class, not a
  fixed list": row comparisons, an aggregate's transition/final function, I/O and user casts,
  a range type's support function, a domain `CHECK`, a generated column, an expression index.
  The claim to break is not "the walker sees it" but "it cannot change rows the backstop
  misses, run a nested utility, or run a `SECURITY DEFINER` body".
* Two statements in one: comments, dollar quoting, `standard_conforming_strings`, encodings.

**3. Change more rows than allowed** (promise 3; `budget.sh`, `audit9.sh`)
* Slice a refused write into pieces within one window; across two agents sharing a table.
* Rows the backstop may count wrong: an `UPDATE` moving rows between partitions, a `MERGE`,
  a foreign or non-heap partition, a cascade on an allow-listed table, `TRUNCATE` reached from
  a trigger.
* An `allow_ddl` agent: `CREATE TABLE AS` over a write, a `DEFAULT` or index expression that
  writes elsewhere, `ALTER TYPE` / `DROP ... CASCADE` reaching a DBA's object.

**4. Move the tenant context** (promise 6; `rls_isolation.sh`, `drivers.sh`)
* `SET`, `set_config` anywhere, a function's `SET` clause, startup `options`, `PGOPTIONS`,
  pgjdbc/node-pg properties, a module placeholder defined later in the session, `RESET`,
  `DISCARD`, a policy helper that reads an unqualified name (documented: test the edges).
* `TimeZone`/`DateStyle` against a policy that uses an explicit zone (documented when it does
  not).

**5. Become the agent, or act as another** (promise 5; `membership.sh`, `privileges.sh`)
* `SET ROLE` / `SET SESSION AUTHORIZATION` from another role; membership granted to or from
  the agent role **after** `register_agent` (registration refuses risky memberships -- is the
  same held later?); `ALTER ROLE` changes after registration.

**6. Leave a trace that the record does not show, or alter the record** (promises 4 and 7;
`hostile.sh`, `rollback.sh`, `dump_restore.sh`)
* A read that advances a sequence, takes a lock or leaves a large object.
* Rewrite, delete or forge record rows from the agent; make an act disappear by rollback in a
  way the README does not describe.

**7. Change the world between propose and commit** (`hostile.sh`, `isolation.sh`, `audit4.sh`)
* Swap a table, function, policy or setting after `propose`; commit one proposal twice under
  any isolation level; let a proposal expire (`expires_in_seconds`).

**8. Garbage** (`hostile.sh`, `make fuzz`)
* Huge or malformed SQL, intent and parameters; invalid UTF-8; NUL bytes; deep nesting. A
  crash, an assertion failure on an assertion-enabled build, or a hang is a finding.

The project's own fuzzer (`make fuzz`, `FUZZ_ITERS`, `FUZZ_SEED`) is a good starting point;
a case in the style of `tests/*.sh` (the attack plus a superuser's check) becomes the
regression test.
