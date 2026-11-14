# Trial kit: pg_agent_gate, pg_phasevec, pg_living_assertions and the guards

This kit is for PostgreSQL hackers and technical early users who want to install these
extensions on a throwaway cluster, try them, and attack them. It assumes you know PostgreSQL
internals and are comfortable building an extension from source. A finding is most useful
when it names the promise it breaks.

The whole family is described in one page, [family.md](family.md): the pieces, the majors each
supports, how they depend on each other, and how to reproduce the published numbers. Every
published number is tied to the run that produced it in
[pg-evidence](https://github.com/Manuelreyesbravo/pg-evidence), where `make validate`
recomputes it from the raw data.

## What is in this package

| piece | role | version | PostgreSQL | in this kit |
|---|---|---|---|---|
| pg_agent_gate | flagship | 0.4.3 | 18, 19 | quickstart, promises, attacks |
| pg_phasevec | **preview, 0.x** | 0.20.0 | 18, 19 | quickstart, promises, attacks |
| pg_living_assertions | foundation | 0.5.10 | 11-19 | install notes below |
| pg_promise_guard | guard | 0.2.10 | 11-19 | install notes below |
| pg_grammar_guard | guard | 0.4.9 | 11-19 | install notes below |
| pg_plan_guard | guard | 1.1.11 | 19 only | install notes below |
| pg_recall_guard | guard | 0.2.10 | 10-19 (CI matrix) | install notes below |

pg_phasevec is a **0.x preview**: its interface and its authorship rules can still change
before 1.0, and its README lists the open findings under "Known limits". It is public (GitHub
tag `v0.20.0`, PGXN as `testing`). Try it and attack it; do not put data you cannot rebuild
behind it yet.

Without building anything: the public image `ghcr.io/latamflows/ygg-pg:0.3.6` (PostgreSQL
19rc2 on Red Hat UBI 9) ships pg_agent_gate, pg_living_assertions, pg_promise_guard,
pg_plan_guard, pg_grammar_guard and pg_phasevec (installed, not created). The quickstarts
below build from source, which is how they were tested.

## pg_agent_gate 0.4.3 (the flagship)

Rust, pgrx 0.19.2; PostgreSQL 18 and 19. In a session that belongs to an AI agent, SQL does
not execute. The agent may only *propose* one statement through seven verbs (`discover`,
`propose`, `dry_run`, `commit`, `propose_and_commit`, `acts`, `whoami`); PostgreSQL verifies
it against its own parser, planner and catalog, shows the exact before/after of every row a
write touches, runs it with the agent's own privileges, and keeps it only if every guard
agrees (a row limit per statement and per time window, bound assertions, no DDL unless
allowed). Everything the agent proposed, and why anything was refused, is recorded
append-only. It is enforced by hooks inside the server, so there is no client, driver or
protocol path around it. The adversary it is built against is the agent.

Bound assertions come from pg_living_assertions: use **0.5.6 or later** (0.5.10 is current).
With 0.5.5 the gate detects a `search_path` leak and refuses writes that have bound
assertions (gate README; META.json recommends 0.5.6).

## pg_living_assertions and the four guards

All pure SQL, Apache-2.0, each with its own repository, CI and `SECURITY.md`. There is no
separate quickstart for them in this kit: follow each repository's README, with the notes
below (what the 2026-11-11 review found when it ran those READMEs literally).

* **pg_living_assertions 0.5.10** (PostgreSQL 11-19). A claim, the SQL that proves it, and the
  date it was last proved. `run()` evaluates the SQL read-only, in a subtransaction that is
  always rolled back, as the role that declared it. Note: `make installcheck` after
  `make cluster` needs `PGHOST=$PWD/.testcluster`; `make cluster` and `check-suites` need
  PostgreSQL 18 or later.
* **pg_promise_guard 0.2.10** (11-19). Guarantees the schema advertises but no longer enforces
  (invalid UNIQUE indexes, NOT VALID constraints, disabled triggers, RLS not forced...).
  Requires pg_living_assertions: `CREATE EXTENSION pg_promise_guard CASCADE`. Its objects live
  in schema `promise_guard`: write `promise_guard.promise_breaks`, `promise_guard.promises_kept()`.
* **pg_grammar_guard 0.4.9** (11-19). A GBNF / JSON-Schema grammar built from the live
  catalog, so a constrained LLM cannot emit an identifier that does not exist; drift is
  watched through pg_living_assertions. Requires it: `CREATE EXTENSION pg_grammar_guard
  CASCADE` (`make install` does not install pg_living_assertions).
* **pg_plan_guard 1.1.11** (PostgreSQL 19 only). Records the plan advice you approved
  (PostgreSQL 19's `pg_plan_advice`) and reports when the planner's choice drifts. On 18 the
  `CREATE EXTENSION` succeeds but the first capture fails. pg_living_assertions is optional
  (only `watch()` needs it).
* **pg_recall_guard 0.2.10** (CI matrix 10-19). Measures a vector index's recall against exact
  ground truth and reports drift from the baseline you approved. Requires `tsm_system_rows`
  (contrib); you need a vector index (pgvector, pgvectorscale, gist or spgist) to watch.

## pg_phasevec 0.20.0 (preview, 0.x)

C and SQL; PostgreSQL 18 and newer, UTF8 databases. Readings with provenance: for each row of
a watched table it stores the codes a sealed, local reader (an embedding model identified by
the sha256 of its weights) produced, together with the hash of the exact text it read. A row
edited after it was read becomes `stale` and is not searched; readings of different readers
are never compared; a search refuses to answer below the watch's declared coverage. On top of
that sits a truth layer: a role may *assert* a claim about a row with a proof (one SQL
expression), and `verify()` judges each proof in a background worker, in a session of the
claim's author, so a verdict (`verified`, `refuted`, `unproven`) means what it meant to the
author and is bound to the text it judged. **Preview:** the rules for who may change a claim
(owner, author, stewards, the row's writer) are the newest part, with open findings that its
README lists under "Known limits". Its model-based fuzzer, spec and threat model are in the
repository's `fuzz/` directory.

## What is in the kit

| file | what |
|---|---|
| `family.md` | the family in one page: pieces, dependencies, the published numbers |
| `quickstart-pg_agent_gate.md` | from zero to a refused `DELETE` and a verified write |
| `promises-and-limits-pg_agent_gate.md` | what it promises, threat model, documented limits |
| `what-to-attack-pg_agent_gate.md` | attack ideas, and how to tell a finding from a limit |
| `quickstart-pg_phasevec.md` | preview: from zero to a stale reading and a judged claim |
| `promises-and-limits-pg_phasevec.md` | preview: the same for pg_phasevec |
| `what-to-attack-pg_phasevec.md` | preview: the same for pg_phasevec |
| `bug-report-template.md` | what a report needs |
| `quickstart-evidence.txt` | the pg_agent_gate quickstart, run as written, with its output |
| `quickstart-evidence-pg_phasevec.txt` | the pg_phasevec quickstart, run as written |

## Versions this kit was written against

* pg_agent_gate 0.4.3: commit `2e4da3b` (GitHub `main`). Its extension code is that of tag
  `v0.4.3` (`856cbcf`); what came after is `Cargo.lock`, the Setup fix and formatting (its
  CHANGELOG, "Unreleased"). The tag `v0.4.3` was moved once, on 2026-11-11, from `4c0f6a0` to
  `856cbcf` (same gate code); the CHANGELOG says so, and release tags are no longer moved.
* pg_phasevec 0.20.0: tag `v0.20.0` = commit `fdd19cb` (preview).
* pg_living_assertions and the guards: their latest tags (`v0.5.10`, `v0.2.10`, `v0.4.9`,
  `v1.1.11`, `v0.2.10`).

## How to report

1. Reproduce on a throwaway cluster (the quickstarts make one). Never attack a server you
   do not own.
2. Read the piece's `promises-and-limits-*.md` (or its README's limits). If what you found is
   listed there as a limit, it is not a bug -- but a sharper statement of the limit, or a
   consequence it does not mention, is still a welcome report.
3. Fill in `bug-report-template.md`. The smallest SQL that shows it, run from a fresh
   cluster, plus a superuser's check of what really changed, is the ideal report.
4. Send it:
   * **A way around a security promise** (an agent executing SQL, a verdict served that should
     not be, a reader seeing what it may not): **privately**, not as a public issue: GitHub's
     private vulnerability reporting (the repository's Security tab), or email
     `manuelreyesbravo@gmail.com` with a subject starting `[<piece> security]` (each
     repository's `SECURITY.md`).
   * **Everything else** (a crash that is not a bypass, a wrong error, unclear docs): a GitHub
     issue on the piece's repository.

pg_agent_gate's `SECURITY.md` promises an acknowledgement within 72 hours, a fix before
disclosure, a regression case in `tests/`, and credit in the CHANGELOG unless you prefer
otherwise; pg_living_assertions and the guards have the same kind of `SECURITY.md`.
pg_phasevec's `SECURITY.md` (at `fdd19cb`) promises the same.

pg_agent_gate does not accept code pull requests for now (its `CONTRIBUTING.md`): reports,
attack cases and documentation feedback are what helps. The same holds for
pg_living_assertions, the guards and pg_phasevec (their `CONTRIBUTING.md`).
