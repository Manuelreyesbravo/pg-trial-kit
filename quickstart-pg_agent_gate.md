# Quickstart: pg_agent_gate 0.4.3

From nothing to an agent whose `DELETE` is refused and whose proposed `UPDATE` is verified,
shown, and kept -- on a throwaway cluster, in about ten minutes (most of it the first build).
Every command below was run as written; the output is in `quickstart-evidence.txt`.

## What you need

* PostgreSQL **18 or 19** with its server headers: `pg_config` must work
  (Debian/Ubuntu: `postgresql-server-dev-18`; Fedora/RHEL: `postgresql-server-devel`; a source
  build: it is there). You need write access to its `pkglibdir` and `sharedir`, or run the
  install step with `sudo`. A PostgreSQL you built into your home directory is easiest.
* Rust 1.98 (`rustup`), `clang` and `libclang` (pgrx runs bindgen), `pkg-config`, `make`.
* About 2 GB of disk for the build.

## 1. Get the source and the build tool

```sh
git clone https://github.com/Manuelreyesbravo/pg_agent_gate
cd pg_agent_gate
git checkout 2e4da3b       # main: the code of 0.4.3, with Cargo.lock (README.md, "Versions")
cargo install cargo-pgrx --version 0.19.2 --locked
```

`Cargo.lock` is tracked (since `e0763df`), so you build the dependency versions that were
tested.

## 2. Build and install into your PostgreSQL

```sh
export PG_CONFIG=/path/to/pg_config               # the PostgreSQL you will run
cargo pgrx init --pg19 "$PG_CONFIG"               # --pg18 for PostgreSQL 18
cargo pgrx install --release --pg-config "$PG_CONFIG"
```

`cargo pgrx init` with a `pg_config` downloads nothing; it only records where your PostgreSQL
is. Without it, `install` stops with `config.toml not found. Have you run cargo pgrx init
yet?` (the gate README's Setup section says the same). With crates
already cached the build took about 40 s on a 24-core machine; a first build downloads and
compiles more.

## 3. A throwaway cluster with the gate preloaded

```sh
export PATH="$($PG_CONFIG --bindir):$PATH"
export DATA=/tmp/gate-trial PGPORT=5571 PGHOST=/tmp/gate-trial
initdb -D "$DATA" --auth=trust -E UTF8 -U postgres
echo "shared_preload_libraries = 'pg_agent_gate'" >> "$DATA/postgresql.conf"
pg_ctl -D "$DATA" -l "$DATA/server.log" -w \
  -o "-p $PGPORT -k $DATA -c listen_addresses=''" start
```

`--auth=trust` and no TCP listener: this cluster is for you alone. Use
`shared_preload_libraries` -- without the library loaded before an agent's first statement
the gate fails open (README, "What it does not cover").

## 4. A table, an agent role, and its registration (as a superuser)

```sh
psql -X -At -U postgres -d postgres -v ON_ERROR_STOP=1 <<'SQL'
CREATE EXTENSION pg_agent_gate;
CREATE TABLE customers (id int PRIMARY KEY, name text NOT NULL, plan text NOT NULL);
INSERT INTO customers VALUES (1, 'Ana', 'free'), (2, 'Bo', 'free'), (3, 'Cy', 'free');
CREATE ROLE billing_agent LOGIN;            -- never a superuser
GRANT USAGE ON SCHEMA public TO billing_agent;
GRANT SELECT, UPDATE ON customers TO billing_agent;
SELECT agent_gate.register_agent('billing', 'billing_agent',
  'answers billing questions and upgrades plans', p_max_rows => 1, p_allow_ddl => false);
SQL
```

`register_agent` returns, among others, `"enforced_by": "shared_preload_libraries"` and
`"max_rows_per_window": 1`. It takes effect on the role's **next** connection.

## 5. Be the agent

```sh
psql -X -At -U billing_agent -d postgres <<'SQL'
SELECT agent_gate.whoami();
-- 1. typed directly: refused, nothing runs
DELETE FROM customers;
-- 2. proposed: verified, nothing runs yet
SELECT agent_gate.propose('update customers set plan = ''pro'' where id = $1::int',
                          'upgrade the customer who asked', ARRAY['1']);
-- 3. see the exact effect, nothing kept
SELECT agent_gate.dry_run(1);    -- 1 = the "proposal" number propose returned
-- 4. keep it
SELECT agent_gate.commit(1);
-- 5. a write wider than max_rows (3 rows, limit 1) is undone
SELECT agent_gate.propose_and_commit('update customers set plan = ''free''',
                                     'reset every plan');
-- 6. a statement outside the agent's kind is refused at propose
SELECT agent_gate.propose('drop table customers', 'clean up');
-- 7. what the record says
SELECT agent_gate.acts(10);
SQL
psql -X -U postgres -d postgres -c 'TABLE customers ORDER BY id'
```

What you should see (abridged from the real run):

```
{"role": "billing_agent", "agent": "billing", ..., "version": "0.4.3", "enforced": true, ...}
ERROR:  pg_agent_gate: this session belongs to agent "billing": it proposes, it does not execute
DETAIL:  DELETE does not reach the database from an agent session
{"ok": true, "kind": "write", ..., "checks": [{"check": "parses", ...}, ... ], "proposal": 1, ...}
{"kind": "write", "mode": "dry_run", ..., "rows": [{"after": {"id": 1, "name": "Ana",
  "plan": "pro"}, "before": {... "plan": "free"}}], ..., "outcome": "rolled_back", ...}
{"kind": "write", "mode": "commit", ..., "outcome": "kept", ..., "rows_affected": 1, ...}
{"commit": {..., "reason": "it touched 3 rows, more than this agent may change in one
  statement", "outcome": "aborted", ...}, "outcome": "aborted", ...}
{"ok": false, "kind": "ddl", ..., {"check": "kind_allowed", "detail": "T_DropStmt changes the
  schema or the server, and this agent is not allowed DDL", "passed": false}], ...}
 id | name | plan
----+------+------
  1 | Ana  | pro
  2 | Bo   | free
  3 | Cy   | free
```

The agent cannot pull a value out of a verb's JSON in SQL (`(agent_gate.propose(...))->>...`
is refused: an agent session may only run `SELECT agent_gate.<verb>(...)` with literals or
parameters). Read the proposal number from the output, or let your client parse it.

## 6. Next steps, and cleaning up

* `make demo PG_CONFIG="$PG_CONFIG"` -- the same role with and without the gate, side by side
  (not run for this kit).
* `make verify PG_CONFIG="$PG_CONFIG"` -- every suite of the README against its own throwaway
  cluster (not run for this kit).
* Then `what-to-attack-pg_agent_gate.md`.

```sh
pg_ctl -D "$DATA" -m fast -w stop && rm -rf "$DATA"
```

Tested for this kit on 2026-11-14 (UTC): steps 1-6 as written, step 1 from GitHub (a fresh
`git clone`, `git checkout 2e4da3b`, and `cargo install cargo-pgrx` into an empty Cargo home),
on PostgreSQL 19beta2 (a release build) and 18.6 (an assertion-enabled build; `--pg18` in
step 2), identical outcomes (`quickstart-evidence.txt`).
