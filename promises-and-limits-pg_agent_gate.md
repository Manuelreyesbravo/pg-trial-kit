# pg_agent_gate 0.4.3: what it promises, and what it does not

Everything here is taken from the project's README (sections "Threat model" and "What it
does not cover") and `SECURITY.md` at commit `856cbcf` (tag v0.4.3), shortened. When this
page and the README disagree, the README wins -- and the disagreement is worth a report.

## The threat model in plain words

**The adversary is the agent.** A model that hallucinates, one that was prompt-injected, or
a fully hostile client. It holds a connection as its own role and can send anything the wire
protocol carries: any statement, any startup parameter, the extended protocol, a cursor,
garbage.

**Where the line is.** The gate bounds what the agent *names* -- the one statement it
proposes and the rows that statement changes -- plus the row count. What a DBA attaches to a
table (a trigger, a `CHECK` or `DEFAULT`, a generated column, a view) is trusted code that
runs by its own rules: the gate refuses the nested *utility* such code fires and counts the
rows it writes, but it does not read a function's body or re-decide what a view may show.

**Trusted, out of scope:** superusers and whoever can edit `postgresql.conf` or restart the
server (they can unload the gate); the DBA's grants (`GRANT` is still the authorization); the
functions and views the agent is granted; the operating system and the server binaries.

**Not protected, by design: intent.** A well-formed statement, within the agent's privileges
and under `max_rows`, runs -- even if it is not what the human meant. The README's own
example: asked to "set every customer's plan to free", a model wrote `UPDATE ... WHERE plan
<> 'free'`, which touched 2 rows under a limit of 5, and the gate kept it, correctly.

## What it promises (each has a suite in `tests/`)

1. Nothing an agent session types executes except the seven verbs.
2. A proposal runs only after PostgreSQL's parser, planner, analyzer and rewriter accept it as
   one statement of a kind the agent may run.
3. A write that touches more than `max_rows` rows, or breaks a bound assertion, is undone; its
   writes over a time window stay within its budget (0.4.0). An `allow_ddl` agent's
   `CREATE TABLE AS` / `SELECT INTO` cannot carry a write, and rows a DDL's default,
   constraint or index expression changes in other tables are counted like a write's.
4. A read changes nothing the record would not show: no sequence, lock or large object,
   wherever the code that would change it sits.
5. No other role can become the agent: an agent role has no members, so no session but its
   own login can `SET ROLE` into it.
6. The context its row-level policies read cannot be moved: not by `SET`, not at connection
   start, not by `set_config()` inside a proposal (in the statement, in SQL passed as text, in
   a function body, a constraint, a default, a domain's `CHECK`, a type's support function),
   nor by a function's `SET` clause.
7. Everything it proposed and did is recorded append-only, including why anything was
   refused.

`SECURITY.md` names what counts as a vulnerability: an agent session executing SQL other than
the verbs, a proposal passing verification that should not, more rows changed than
`max_rows`, the policy context moved, or the record altered.

## Documented limits (not bugs -- unless you find a consequence the README does not state)

Configuration and deployment:

* **Without the library loaded before the agent's first statement, the gate fails open.**
  Use `shared_preload_libraries`; `whoami()` reports `enforced`.
* Existing connections of a role are not behind the gate until they reconnect.
* `pg_dump` does not carry who is an agent (the mark is on the role; `pg_dumpall
  --globals-only` has it). Restored without globals, the record says "agent" and the role is
  outside the gate.
* `register_agent` refuses superusers, `REPLICATION`/`BYPASSRLS` roles and members of such
  roles or of `pg_write_all_data`, `pg_execute_server_program`, `pg_write_server_files`.

Functions and code the gate does not read:

* **A function's body is trusted to be what the function is marked.** A DBA's function marked
  `STABLE`/`IMMUTABLE` that writes runs; only the row-count backstop sees what it changed.
  PostgreSQL grants `EXECUTE` to `PUBLIC` on every new function: `REVOKE` it where needed.
* Built-ins reached from C code through a cache (type cache, RI check, index support) are not
  judged one by one.
* Code an `allow_ddl` agent plants runs for whoever else uses its objects (PostgreSQL's rule
  for any role that may create objects); within the agent's own session the judgment is by
  function, not call site, and follows only what `pg_depend` records.
* A persistent object of the agent can answer for a name a DBA's code does not qualify (a
  schema searched before the DBA's). Qualify names in policy helpers, or pin `search_path`.
* Built-ins are classified by PostgreSQL's volatility marking plus one reviewed list (derived on
  18.6 and 19beta2); a `STABLE`/`IMMUTABLE` built-in is trusted not to change state.
* `no_opaque_function` walks the proposal's query tree, not every place a function can hide
  (row comparisons, an aggregate's own functions, casts, catalog-attached code, view bodies) --
  "a class, not a fixed list". The commit backstop still counts rows; nested utility is refused
  (0.2.4); a `SECURITY DEFINER` call is stopped wherever it is reached (0.2.7).
* A cascade, trigger or rule that amplifies a write is refused; `allow_write` re-permits a
  table only for a non-`SECURITY DEFINER` trigger, whose extra rows are counted but not
  tenant-filtered. The backstop reads `pg_stat_xact_user_tables`: user tables only.
* A view without `security_invoker` is checked with its owner's privileges (PostgreSQL's rule).
* `EXPLAIN` folds constant calls to immutable functions: a DBA's function falsely marked
  `IMMUTABLE` runs at `propose`.
* A row-level policy that calls a `SECURITY DEFINER` function refuses every proposal on its
  table.

Session and context:

* The context is what the session started with; the gate refuses (does not reset) a session
  whose context moved. An agent session may change only allowlisted parameters; startup
  parameters outside the allowlist fail the session closed. `TimeZone`, `DateStyle` and
  `IntervalStyle` stay settable (every driver sends them): a policy that dates in the session's
  zone can be read in another zone -- write such policies with an explicit zone.
  `search_path` is deliberately not on the allowlist: a driver that sends a default schema at
  startup (pgjdbc's `currentSchema`) is refused; set the agent's schema on its role.
* The fast-path function-call protocol (`PQfn`) skips the parser; an agent session answers it
  only for the verbs (0.2.12).
* What `dry_run` showed is what `commit` keeps: the settings that decide how literals are read
  and values written (`DateStyle`, `TimeZone`, `bytea_output`, ...) are recorded at `propose`,
  and `dry_run`/`commit` refuse if any changed (0.2.14).
* The verbs run only under READ COMMITTED; `PREPARE TRANSACTION` is refused; the verbs refuse a
  read-only transaction and a hot standby (every proposal is written to the record first).
* `lastval()` names the gate's own sequence in an agent's session.

Execution and the record:

* **`dry_run` is a rollback, not a sandbox**: sequence values, session advisory locks, and
  anything outside the transaction (`dblink`, untrusted languages) are not undone.
* The record is transactional: an explicit transaction the caller rolls back takes the record
  with it (the server log keeps the verdicts). The server log can hold what an agent wrote.
* With `attempt_durability = fast`, a crash can lose records of attempts that changed nothing,
  never of changes.
* The record takes bounded input (`max_proposal_bytes` 1 MB, `max_intent_bytes` 64 kB) and has
  no retention helper.
* A reused agent name starts clean. Parameters travel as text (cast them in the SQL).
* **`allow_ddl` is schema power, not a bigger `max_rows`**: a granted `DROP TABLE` is kept, an
  `ALTER COLUMN ... TYPE ... USING` rewrites a table with 0 rows counted. Leave
  `allow_ddl => false` unless a specific agent needs it.

The README's full text of each item is longer and states the measured case behind it. Read
it before reporting something that sounds like one of these.
