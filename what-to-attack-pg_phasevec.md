# What to attack: pg_phasevec 0.20.0 (preview, 0.x)

Set-up: the quickstart's cluster, plus more roles as you need them: a second author, a table
writer that is not an author, a reader with `SELECT` under a row-level policy, a reader with
only `USAGE` on the schema, a verifier granted only `verify()`. Keep a superuser psql to read
`phasevec.truth` and the watched table directly.

**Preview.** Section 5 (authorship) attacks the newest rules, which have open findings (the
README's "Known limits"): a new way through them is a finding; one listed there is not.

## How to tell a real finding from a documented limit

A real finding breaks a numbered promise in `promises-and-limits-pg_phasevec.md`, on a
supported configuration (UTF8 database, plain table, simple unique key), without a superuser
acting after set-up, and is not in the README's "Known limits" (nor in doc/review-history.md's
Limits) -- or it is a consequence of a limit that the README does not state. Check what is
*served* (`brief()`, `_verdict()`, `find()`, `truth_coverage()`), not just what is stored, and
check it against the row as it is now, read by a superuser.

Not findings: a key, score or reading state seen through `search`/`find`/`find_ivf`/
`find_npu` by a role with `USAGE` (open to `PUBLIC` on purpose); a verdict that serves until
its horizon (`max_age`) because judgment was stalled (documented: the horizon is the bound);
a non-deterministic recipe whose text and hash disagree; a superuser's proof running as a
superuser; C functions from other extensions (`dblink`) doing what they do. A crash of a
backend or a worker, from any non-superuser role, is always worth reporting.

## Attack ideas

**1. Make a stale reading look fresh** (promises 1-4)
* Change a row's text by a path the watch trigger may miss: `COPY`, `MERGE`, `INSERT ... ON
  CONFLICT`, a cascade, a partition move (should be refused as a shape), `ALTER TABLE ... TYPE`
  that changes how a value prints, a collation or a domain change, logical replication apply,
  `session_replication_role = replica` (documented for `TRUNCATE`: look for others).
* Feed `record()` a hash it should refuse: the old text's, another row's, another reader's.
* Get readings of two readers compared: `reader_switch()` mid-fill, two watches on one table.
* Get an answer from `find()` below `min_coverage` without `p_partial`.

**2. Get out of the sealed room** (promises 5-6)
* From a recipe or a proof: write anything that survives (a table, a sequence, an advisory
  lock, `NOTIFY`, a temp object, a setting that outlives the guard), run a refused command,
  reach a cursor or a portal opened before the guard, reach a volatile system function not on
  the reviewed list.
* Make a recipe render differently by who triggers it: session settings, `search_path`,
  placeholders (`app.x`), row-level security, the triggering role's rights.

**3. Make a verdict lie** (promises 7-10; the fuzzer's invariant "a served `verified` means
the proof held for exactly the current text, settings and author")
* Get a proof judged with the verifier's rights or settings instead of the author's: wrappers
  with `SET` clauses, `SECURITY DEFINER` callers, `DateStyle` halves, role-level settings.
* Change the row, the proof's data, or the author between judging and serving; race
  `verify()`, `verify_one()`, `assert()`, `refute()` and row writes in parallel sessions.
* Make `verified_at` move backward, or a `verified` outlive `max_age`.
* Drop, rename or recreate an author; restore a dump; upgrade from 0.19.0 with claims in
  every state. A verdict that survives where the README says it reads `unproven` is a finding.

**4. See what you may not** (promise 11)
* As a reader under row-level security, or with no `SELECT`: learn about a hidden row's
  verdict, line, count, dispute or replacement chain through any door (`brief`, `recall`,
  `disputes`, `replacement`, `truth_coverage`, `_verdict`, error messages, timing).

**5. Act on another author's claim** (README, "Claims and proofs": who may change a claim)
* As a second author or a tenant under row-level security: overwrite, refute, supersede,
  unsupersede or adopt a claim another author holds; claim a key on a row you cannot see or
  may not `UPDATE`; take over a claim after its author is dropped, after its row is deleted and
  written again, or on a copy of the cluster after `rebind()`.
* As a revoked steward: change a claim you took only as a steward.

**6. Hold the retired down** (the fuzzer's invariant "a retired claim comes back only through
`assert()`")
* `refute()`, then any mix of row edits, `supersede()`/`unsupersede()`, upgrade, dump/restore,
  `adopt()`: does a retired claim come back into `brief()` some other way?

**7. Memory safety and resources** (the C code: `src/phasevec_*.c`)
* Hand-crafted codes and arguments to `find`, `find_ivf`, `alike`, `record`: wrong dimension
  counts, empty and `NULL` arrays, extreme `k`, `m`, `nprobe`; corrupted rows in
  `phasevec.ivf_lists` (as the owner, to test the load-time checks).
* Exhaust `max_worker_processes` with proofs; cancel and terminate verifiers mid-call.

The project tests each fix with a regression file in `test/sql/` or a script in `test/`;
a report in that shape (set-up, attack, a superuser's check) is easiest to turn into one.
The project's model-based fuzzer, with its spec and threat model, is in the repository's
`fuzz/` directory (`fuzz/README.md`): a sequence it shrinks is a ready-made report.
