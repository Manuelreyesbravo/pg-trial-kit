# pg_phasevec 0.20.0 (preview, 0.x): what it promises, and what it does not

Taken from the project's README at commit `fdd19cb` (sections "Status: what this preview is",
"Claims and proofs", "Backup, restore and copies" and "Known limits"), shortened, and from
`doc/review-history.md` (each review round's findings, fixes and limits): read both before
reporting. When this page and the README disagree, the README wins -- and the disagreement is
worth a report.

> **Preview (0.x).** The rules for who may change a claim are the newest part of 0.20.0 and the
> one with open findings (listed below under "Open findings"). They may still change in 0.21.

## The model, in two layers

* **Readings.** A *reader* (an embedding model) is sealed by the sha256 of its weights. A
  *watch* names a table, its key column and one *recipe* (an SQL expression over the row) for
  the text to read. A *reading* stores the reader's codes with the hash of the exact text read.
  States: pending, fresh, stale, foreign, orphan, failed.
* **Claims (the truth layer).** A role *asserts* a claim about a row, optionally with a
  *proof* (one SQL expression). `verify()` / `verify_one()` judge each proof; `brief()` serves
  what a watch still holds as true; `refute()`, `supersede()`, `unsupersede()` and `adopt()`
  change a claim's life.

## The threat model in plain words

The roles involved (the project's internal threat model, checked against the rules of
`fdd19cb`):

* **Owner** of the extension: trusted; every internal runs as it.
* **Author**: a role granted `assert()`. Its proofs run with its rights, in a session of its
  own, with the settings it had when it asserted.
* **Table owner and stewards**: the watched table's owner changes every claim of its watch, and
  names *stewards* (`steward()`) who change the claims on the rows they see.
* **Table writer**: changes the text a claim is about; may make the first claim on a row only
  if it may `UPDATE` that row, and never changes another author's live claim.
* **Reader with access**: may `SELECT` the watched table (under row-level security, some
  rows) and see the verdicts of what it can see -- and nothing else.
* **Reader without access**: `USAGE` on the schema only: gets nothing from `brief()` or
  `_verdict()`, but does get keys, scores and reading states from the search functions open to
  `PUBLIC` (see below).
* **Verifier**: granted `verify()`/`verify_one()`; may run the proofs of authors it holds,
  may not choose an author, forge a verdict, or judge with its own rights.
* **Superuser**: out of scope as an attacker; in scope as a careless caller.

The reader (the model) is yours: pg_phasevec holds codes to the reader's sha, it does not
vouch for what the weights do. Recipes and proofs are code that is not phasevec's: they run
in one guard, a "sealed room".

## What it promises

Readings:

1. A reading names the reader and the exact text it was taken from. A row changed after it was
   read is reported `stale` (in the same transaction), not matched as fresh; a reading of the
   old text is refused.
2. Readings from different readers are never compared.
3. A deleted row leaves an `orphan` reading you can see, not a silent ghost.
4. `find()` refuses below the watch's `min_coverage` unless asked for a partial answer.

Code that is not phasevec's (README, Security):

5. A recipe and a proof run in a subtransaction of their own, always rolled back, as one named
   role, in a security-restricted operation, read-only. Only a listed set of commands runs
   there; of the system's volatile functions only a reviewed few.
6. A recipe renders one text: as the table's owner, at default settings with a pinned set on
   top (`DateStyle` ISO, `TimeZone` UTC, `search_path` `pg_catalog, pg_temp`, ...), whoever
   triggers it.

Claims and verdicts:

7. A proof runs in a background worker, in a new session **of its author**, by OID, with the
   settings the author's session had changed when it asserted (`truth.proof_env`): the
   verifier's own settings and rights decide nothing.
8. A verdict is about a committed state and is bound to the text it judged: a claim whose row
   now says something else reads `unproven`.
9. A `verified` or `refuted` serves only while it was judged within the watch's horizon
   (`max_age`, 1 hour by default); then it reads `unproven`. Safety does not depend on
   liveness: whoever can stall judgment cannot keep an old verdict past the horizon.
10. Only the proof's own logic decides its verdict; only its own work refutes it by time.
11. What shows verdicts (`recall`, `disputes`, `replacement`, `truth_coverage`, `brief`) shows
    a watch only to a role that may `SELECT` its table; `_verdict()` answers `NULL` otherwise.
12. A writer never waits on a verifier, and two verifiers do not wait on each other.

Supported configurations (refused with an error otherwise): a UTF8 database; a watch over a
plain table (not partitioned, not a partition, no inheritance either way); a key column with
its own unique, non-deferrable, non-partial, non-expression btree index. Every read checks the
shape again after DDL.

## What it exposes on purpose

* `search`, `find`, `find_ivf` and `find_npu` are `SECURITY DEFINER` and open to `PUBLIC`: any
  role with `USAGE` on the schema gets the keys, scores and reading states of every watch (not
  the text, not verdicts). Revoke them where keys are sensitive.
* A recipe sees, beyond its row, what the table's owner may see, and its text goes to any role
  that may `SELECT` the table.

## Documented limits (the README's "Known limits" and doc/review-history.md's Limits, shortened)

Authorship and grants:

* A key has one claim, about one row. A key with no claim, or with a *dead* claim (its row is
  gone and another is under the key), is claimed only by the table's owner, a steward, or a role
  that may `UPDATE` that row (row-level security included); the row must exist, and `SELECT`
  is never enough. A live claim is changed by its author (same OID and name, in this cluster),
  the table's owner, or a steward; a claim whose author is gone waits for the owner or a steward
  (`adopt()`).
* A copy of the cluster's files (`pg_basebackup`, `pg_createsubscriber`, `pg_upgrade --copy`)
  carries its authors: run `phasevec.rebind()` in every database of a copy you do not fully
  trust.
* `assert()` takes its author from whoever calls it; a non-`SECURITY DEFINER` trigger lets its
  owner assert in the name of whoever fires it (PostgreSQL's trigger trap).
* A proof whose author is a superuser runs as a superuser (read-only, restricted). A verifier
  that may act as an author runs that author's code: grant `assert()` only to roles whose code
  you would run as them.
* A caller with the author's privileges lends a claim its own session's settings.
* A verdict is bound to its author by OID and name: a role dropped and recreated with the same
  name and OID (after OID wraparound) is the same author.

Rendering and recipes:

* A recipe is evaluated twice per row (text and hash): a non-deterministic recipe can have its
  text and hash disagree.
* An `ERROR` from the server or another session during a render ends the statement even if the
  recipe catches it; a recipe's own `RAISE` of class 53 is taken as the server's.
* The guard checks functions when an expression is set up: a PL/pgSQL expression the session
  already cached before a guard is not re-checked. C functions of extensions (`dblink`) are not
  classified. An extended-protocol portal is not listed in `pg_cursors`: rows it holds can be
  read by a recipe rendering in that session.
* Temporary objects: a name looked up at run time (`to_regclass(text)`) can find one; the walk
  for temporary objects covers a stated list of node kinds.

Verification, time and workers:

* An author, or anyone who can hold a claim's row or a lock its proof needs, can keep a claim
  from being judged -- not keep its verdict past the horizon. Choose `max_age` for what a stale
  verdict costs you.
* A proof that catches `query_canceled` keeps its worker until it returns; C code that ignores
  interrupts can hold a `max_worker_processes` slot. With no free slot, nothing is judged.
* `verify()` and `verify_one()` refuse read-only transactions and standbys; a transaction that
  has written may not verify. `refute()` in a transaction older than the last verdict stamps
  that verdict's time ("not before").
* A proof reads what any session of its author reads (`pg_stat_activity`, `pg_locks`). The
  database's configuration (`ALTER DATABASE/ROLE ... SET`) can change what proofs compute.
* Whether an author may still read the table is read once per `verify()` call.

Keys, shape, restore, upgrade:

* A key is matched by its text, byte for byte; an operator class whose equality is finer than
  the type's text output could let a key name two rows (the index's shape is checked, not its
  operator class).
* A verifier checks the table's shape without its lock: a change committed in between is
  caught by the next call.
* A restore trusts its dump; into another cluster every author reads as gone until `adopt()`.
  A 0.19.0 dump restored under 0.20.0 needs `phasevec.migrate()` before writes resume. Plain,
  custom, directory and parallel (`pg_restore -j N`) restores bring back the same rows,
  verdicts and counts (fixed at `fdd19cb`; `test/review30.sh`).
* `_verdict(t)` read directly from `phasevec.truth` compares with the trigger's cache, not the
  row; a `TRUNCATE` with triggers off is a change the watch never saw.
* `TRUNCATE` of a watched table costs per row (about 3 s for 100000 rows on a release build).
* `ALTER EXTENSION ... UPDATE` to 0.20.0 takes locks and can stop on a busy or failing recipe;
  two at once: the second fails with `tuple concurrently updated`.

Open findings (the last red-team pass, round 30; the README's "Known limits"):

* A writer's right to claim a dead claim's row is checked before the claim's lock is taken and
  used after it: with a concurrent delete and re-insert by another tenant, a claim can be taken
  over on that tenant's new row (RV30-1).
* `rebind()` leaves the cut authors' refutations and replacements in force, and `adopt()` does
  not list them (RT29PA-REBIND-RETIRED); it does not hold on a live `pg_createsubscriber`
  subscription (RT29PL-1); it works per database and cuts authorship, not table privileges.
* A lexical configuration is checked by its owner's name, not OID (RT29PL-3).
* The update with `lock_timeout = 0` and any enabled `ddl_command_*` event trigger retries until
  `statement_timeout` (RT29PU-2).

Timings and sizes in `doc/review-history.md` are measurements on the author's machine, not
promises; the README quotes none.
