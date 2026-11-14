# pg_agent_gate and its family

**A claim is only as good as the check PostgreSQL runs on it, and how old that check is.**

Every piece below applies that one idea to a different kind of claim. The gate applies it to
an agent's statement: PostgreSQL itself decides whether it runs. pg_living_assertions applies
it to any fact you state about a database: the SQL that proves it, and the date it last did.
The guards apply it to what a schema promises, to the identifiers an LLM may emit, to query
plans and to vector-index recall. pg_phasevec (a 0.x preview) applies it to what a model read
from a row: a reading is only served while the text it read is still the row's. All are
PostgreSQL extensions, Apache-2.0, by Manuel Reyes Bravo, each in its own repository under
github.com/Manuelreyesbravo.

One more piece is not an extension: ygg-agent, the model's side of the gate. The model writes
SQL, it travels over PostgreSQL's own protocol, and the gate decides: no MCP server and no
JSON between the model and its data, and with the model in the same process, no HTTP at all.

## The pieces

| piece | what it checks | version | PostgreSQL | needs |
|---|---|---|---|---|
| [pg_agent_gate][gate] | an agent's SQL, before it runs | 0.4.3 | 18, 19 | (living >= 0.5.6) |
| [pg_living_assertions][living] | any stated fact, with its date | 0.5.10 | 11-19 | nothing |
| [pg_promise_guard][promise] | schema guarantees no longer enforced | 0.2.10 | 11-19 | living |
| [pg_grammar_guard][grammar] | identifiers an LLM may emit | 0.4.9 | 11-19 | living |
| [pg_plan_guard][plan] | drift from an approved plan | 1.1.11 | 19 | pg_plan_advice (contrib) |
| [pg_recall_guard][recall] | vector-index recall drift | 0.2.10 | 10-19 | tsm_system_rows |
| [pg_phasevec][phasevec] | **preview, 0.x**: readings with provenance | 0.20.0 | 18, 19 | nothing |
| [ygg-agent][agent] | not an extension: the model's loop, no MCP | 0.2.0 | any | gate >= 0.4.0 |

Versions are the latest release of each repository. "PostgreSQL" is the majors each
repository's CI runs on; "living" is pg_living_assertions; parentheses mean optional.

[gate]: https://github.com/Manuelreyesbravo/pg_agent_gate
[living]: https://github.com/Manuelreyesbravo/pg_living_assertions
[promise]: https://github.com/Manuelreyesbravo/pg_promise_guard
[grammar]: https://github.com/Manuelreyesbravo/pg_grammar_guard
[plan]: https://github.com/Manuelreyesbravo/pg_plan_guard
[recall]: https://github.com/Manuelreyesbravo/pg_recall_guard
[phasevec]: https://github.com/Manuelreyesbravo/pg_phasevec
[agent]: https://github.com/Manuelreyesbravo/ygg-agent

### pg_agent_gate (the flagship)

A PostgreSQL 18+ extension (Rust/pgrx) for teams that connect LLM agents to a real database.
In a session that belongs to an agent role, SQL does not execute. The agent can only call
seven verbs (`discover`, `propose`, `dry_run`, `commit`, `propose_and_commit`, `acts`,
`whoami`). PostgreSQL checks each proposed statement with its own parser, analyzer, catalog
and privileges. It shows the exact before/after of every affected row, and runs the statement
under the agent's own rights within a per-agent row limit and budget. Every proposal and
refusal goes to an append-only record. The enforcement is in server hooks, so no client,
driver or protocol gets around it. An optional MCP shim (`gated-mcp/`) serves clients that
only speak MCP, and it adds no power.

### ygg-agent (the model's side, no MCP)

A Python package (`pip install "ygg-agent[binary]"`, on PyPI; not on PGXN, as it runs outside
the database) for running an LLM agent against the gate with no MCP server. The model writes
one SQL statement; ygg-agent hands it to `agent_gate.propose()` over PostgreSQL's own
protocol; the verdict goes back to the model. It has no power of its own: it connects as the
agent role, refuses to work unless `whoami()` says the session is behind the gate, never
executes the model's SQL itself, and stops after three refusals in a row. Since 0.2.0
`LlamaCpp` loads a GGUF model in the same process: no model server and no HTTP, and the only
connection the loop opens is PostgreSQL's. Any `/v1/chat/completions` server works too.

### pg_living_assertions (the foundation)

A pure-SQL extension (PostgreSQL 11 and later, no C library) for DBAs and for tools that must
prove a database still meets the guarantees it claims. Each assertion stores a claim, the SQL
that proves it, and who declared it. `run()` evaluates that SQL read-only, inside a
subtransaction that is always rolled back, as the role that declared it. Every verdict
(holds, broken, unknown, erroring, unchecked, retired, unregistered) carries the date it was
reached, so a stale "holds" cannot pass for a fresh one.

### pg_promise_guard

Reports the guarantees your schema advertises but no longer enforces: invalid UNIQUE indexes
left by a failed CONCURRENTLY build, NOT VALID or NOT ENFORCED constraints, disabled
triggers, RLS that is not forced, and disabled rules and event triggers. It tells a live
`breach` from a deliberate mid-migration `gap`. Catalog-only and lock-free; `watch()` records
each scan in pg_living_assertions. For DBAs and platform teams.

### pg_grammar_guard

Builds a GBNF / JSON-Schema grammar from the live PostgreSQL catalog (tables, columns, enum
labels, table->column correlation), so a constrained LLM cannot emit an identifier that does
not exist. Approved grammars are watched for drift through pg_living_assertions. For teams
running text-to-SQL or tool-calling agents against PostgreSQL. Pure SQL, no C.

### pg_plan_guard

Companion to PostgreSQL 19's pg_plan_advice: records the plan advice you approved for a query
and reports, from a schedule, when the planner's choice drifts (the silent "same rows, now a
seq scan" regression). It can push approved advice into pg_stash_advice. Baselines are
re-planned sealed (read-only subtransaction, rolled back, inside a temporary SECURITY DEFINER
frame owned by their author). For DBAs and SREs on PostgreSQL 19+.

### pg_recall_guard

Measures the recall of a vector index against exact ground truth, records the number you
accepted as good, and reports when it drifts: a degraded ANN index otherwise just returns
fewer of the right neighbours, with no error. It finds indexes through a catalog property,
so one neutral tool covers hnsw, ivfflat (pgvector), diskann (pgvectorscale), gist and
spgist. For pgvector users.

### pg_phasevec (preview, 0.x)

Readings with provenance for PostgreSQL. It stores, next to each row of a watched table, what
a local embedding model ("sealed reader", identified by the sha256 of its weights) produced
for that row, together with the hash of the exact text it read. Searches only answer from
readings that still match the row: a row edited after it was read is reported `stale`,
readings of different models are never mixed, and a search refuses to answer below a
declared coverage. An optional "truth" layer lets roles attach SQL proofs ("claims") to rows;
a background worker re-checks them. C + SQL, no other extension, no network. **This is a
preview:** its claim-authorship rules are still changing, and it is offered for trial only.

## How they depend on each other

```
pg_living_assertions  <-- requires --  pg_promise_guard
                      <-- requires --  pg_grammar_guard
                      <-- optional --  pg_plan_guard   (watch() only; needs pg_plan_advice, PG 19)
                      <-- optional --  pg_agent_gate   (bound assertions; use 0.5.6 or later)
pg_recall_guard       (tsm_system_rows; a vector index to watch)
pg_phasevec           (stands alone)
ygg-agent             (Python, outside the database) -- talks SQL to --> pg_agent_gate >= 0.4.0
```

`make install` of a guard does not install pg_living_assertions: install it first, then
`CREATE EXTENSION <guard> CASCADE`. With pg_living_assertions 0.5.5 the gate refuses writes
that have bound assertions (a `search_path` leak fixed in 0.5.6); 0.5.10 is current.

## Reporting bugs

One channel: **a GitHub issue on the piece's repository**. If what you found gets around a
security promise (an agent executing SQL, a verdict served that should not be, a reader
seeing what it may not), do **not** open an issue: report it privately as each
repository's `SECURITY.md` says (GitHub private vulnerability reporting, or
manuelreyesbravo@gmail.com with the subject the file gives). Acknowledgement within 72 hours.
A report template and attack ideas are in this kit (README.md).

## Reproducing the headline numbers

Each number below is a claim in [pg-evidence](https://github.com/Manuelreyesbravo/pg-evidence),
tied to the raw data of its run: `make validate` there recomputes every one of them, and a
rerun of yours can be submitted next to the cited one.

The benchmark compares four ways of connecting the same model to the same database: same
model, same tasks, a superuser checking what really changed. Through a popular Postgres MCP
server, 4 of 7 trap tasks broke out. With the database deciding, 0 of 7. With the same gate
behind both, the agent without MCP used 5x fewer prompt tokens than through MCP (24k vs 126k
for 13 tasks) and finished each task 13x faster: a median of 14 s against 185 s, on a
CPU-only CI runner anyone can re-run. Writing one statement and letting the database decide
took a median of 6 s per task.

These numbers were measured with **pg_agent_gate 0.4.0**, not the current release:

* Run: https://github.com/Manuelreyesbravo/agent-gate-demo/actions/runs/41184106078
  (agent-gate-demo commit `4cef786`, one repetition, GitHub `ubuntu-24.04` runner, CPU only).
* Pinned in that commit's `.github/workflows/compare.yml`: pg_agent_gate `v0.4.0`
  (`33d0acb`), PostgreSQL 18 (PGDG), llama.cpp `bd4eeaa`, the public Qwen3-8B Q4_K_M GGUF
  from unsloth (sha256 `120307ba529eb2439d6c430d94104dabd578497bc7bfe7e322b5d9933b449bd4`;
  the base model, not fine-tuned), a 4096-token context, and the stock MCP server
  `crystaldba/postgres-mcp` pinned by digest.
* To rerun it: fork agent-gate-demo, push a branch at `4cef786`, and run the `compare`
  workflow on that branch with "runs per task and arm" set to 1.
* What the workflow pins stays fixed; what it does not (the PostgreSQL 18 minor release, the
  runner image, the gate's crate versions, as pg_agent_gate did not track `Cargo.lock` then) has
  moved since, so a rerun is not guaranteed to match byte for byte.
* `compare/quickstart.sh` on agent-gate-demo's master branch runs the same comparison on your
  machine against the public image; it is not the cited configuration.

The container image: built on Red Hat UBI; passes all 11 Red Hat container certification
checks (preflight) — reproduce it with one command:

```sh
preflight check container \
  ghcr.io/latamflows/ygg-pg@sha256:ec1aabe4be95f2f84d9c06aa612edc6346079dc7ebfe93280332cd7905c0a9f1
```

(preflight 1.22.0, no credentials needed). That image is ygg-pg 0.3.6: PostgreSQL 19rc2 with
pg_agent_gate 0.4.3, pg_living_assertions 0.5.10, pg_promise_guard 0.2.10, pg_plan_guard
1.1.11, pg_grammar_guard 0.4.9 and pg_phasevec 0.20.0 (preview, installed but not created); it
does not include pg_recall_guard. LatamFlows is a Red Hat partner; container certification is
the next step.

The test suites are the other numbers worth checking yourself: `make verify` in pg_agent_gate
(PostgreSQL 18 and 19), and `make installcheck` in pg_living_assertions and each guard, as
their CI does.
