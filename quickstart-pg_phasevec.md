# Quickstart: pg_phasevec 0.20.0 (preview, 0.x)

From nothing to a reading that goes stale when its row changes, and a claim whose proof is
judged `verified` and then `refuted` -- on a throwaway cluster, in about ten minutes. No model
is needed: a toy SQL "reader" stands in for one. Every command below was run as written; the
output is in `quickstart-evidence-pg_phasevec.txt`.

**Preview (0.x).** Interfaces can still change between 0.x releases, and the project lists its
open findings under "Known limits" in its README; `promises-and-limits-pg_phasevec.md` has them.

## What you need

* PostgreSQL **18 or 19** (tested; newer is untested) with its server headers (`pg_config`;
  Debian/Ubuntu: `postgresql-server-dev-18`), write access to its `pkglibdir` and `sharedir` (or
  `sudo` for the install step), a C compiler and `make`. No other extension.
* A **UTF8** database: `CREATE EXTENSION` refuses any other encoding.
* Free slots of `max_worker_processes` (8 by default): `verify()` uses two at a time,
  `verify_one()` one.

## 1. Get the source, build and install

```sh
git clone https://github.com/Manuelreyesbravo/pg_phasevec
cd pg_phasevec
git checkout v0.20.0      # = fdd19cb, the 0.20.0 preview
export PG_CONFIG=/path/to/pg_config
make PG_CONFIG="$PG_CONFIG"
make install PG_CONFIG="$PG_CONFIG"
```

The build is portable (`-O3` over the server's own flags); `make PHASEVEC_NATIVE=1` adds
`-march=native` for a binary that only has to run where it was built. No
`shared_preload_libraries` is needed.

## 2. A throwaway cluster

```sh
export PATH="$($PG_CONFIG --bindir):$PATH"
export DATA=/tmp/pv-trial PGPORT=5572 PGHOST=/tmp/pv-trial
initdb -D "$DATA" --auth=trust -E UTF8 --locale=C -U postgres
pg_ctl -D "$DATA" -l "$DATA/server.log" -w \
  -o "-p $PGPORT -k $DATA -c listen_addresses=''" start
```

## 3. A reader, a watched table, and its readings

```sh
psql -X -U postgres -d postgres <<'SQL'
\set VERBOSITY terse
CREATE EXTENSION pg_phasevec;
-- a reader, sealed by the sha256 of its weights (a fake one here) and its dimensions
SELECT phasevec.reader_register('toy', repeat('a', 64), 4);
CREATE TABLE docs (id int PRIMARY KEY, body text);
INSERT INTO docs VALUES (1, 'apple'), (2, 'banana'), (3, 'cherry');
-- watch the table: read r.body of each row, keyed by id
SELECT phasevec.watch('docs', 'docs'::regclass, 'id', 'r.body', 'toy');
SELECT state, n, share FROM phasevec.coverage('docs') WHERE n > 0;
-- a toy "reader": hand-made codes instead of a model
CREATE FUNCTION toy_reader(body text, OUT code64 bigint, OUT code smallint[])
LANGUAGE sql IMMUTABLE AS $$
  SELECT CASE body WHEN 'apple' THEN 1 WHEN 'banana' THEN 2 ELSE 4 END::bigint,
         CASE body WHEN 'apple'  THEN ARRAY[100,0,0,0]
                   WHEN 'banana' THEN ARRAY[0,100,0,0]
                   ELSE ARRAY[0,0,100,0] END::smallint[] $$;
SELECT count(phasevec.record('docs', n.key, 'toy', n.text_hash, c.code64, c.code)) AS recorded
  FROM phasevec.next('docs') n, toy_reader(n.body) c;
SELECT state, n, share FROM phasevec.coverage('docs') WHERE n > 0;
SELECT key, round(score::numeric, 4) AS score, hamming, state
  FROM phasevec.find('docs', 1, ARRAY[90,10,0,0]::smallint[], 3);
SQL
```

Coverage goes from `pending 3` to `fresh 3`, and `find` ranks key `1` first (score 0.9939).

## 4. Change a row

```sh
psql -X -U postgres -d postgres <<'SQL'
\set VERBOSITY terse
UPDATE docs SET body = 'apricot' WHERE id = 1;
SELECT key, state FROM phasevec.reading_states ORDER BY key;
SELECT key FROM phasevec.find('docs', 1, ARRAY[90,10,0,0]::smallint[], 3);
SELECT key, state
  FROM phasevec.find('docs', 1, ARRAY[90,10,0,0]::smallint[], 3, p_partial => true);
SQL
```

```
 1   | stale
ERROR:  phasevec: watch docs covers 66.7% of its rows and declared 95.0%: stale 33.3%
 key | state           (with p_partial => true: only 2 and 3, both fresh)
```

The stale row is not searched; below the watch's `min_coverage` `find` refuses unless you ask
for a partial answer.

## 5. A claim, judged in a session of its author

```sh
psql -X -U postgres -d postgres <<'SQL'
CREATE TABLE tasks (id int PRIMARY KEY, done boolean);
INSERT INTO tasks VALUES (5, true);
INSERT INTO docs VALUES (4, 'task 5 is done');
CREATE ROLE author LOGIN;
GRANT USAGE ON SCHEMA phasevec TO author;
GRANT SELECT, UPDATE ON docs TO author;   -- the first claim on a row: a role that may UPDATE it
GRANT SELECT ON tasks TO author;
GRANT EXECUTE ON FUNCTION phasevec.assert, phasevec.verify, phasevec.brief TO author;
SQL
psql -X -U author -d postgres <<'SQL'
SELECT phasevec.assert('docs', '4', 'EXISTS (SELECT 1 FROM public.tasks WHERE id = 5 AND done)');
SELECT key, verdict, error FROM phasevec.verify('docs', p_every => interval '0');
SELECT key, verdict, line FROM phasevec.brief('docs') ORDER BY key;
SQL
psql -X -U postgres -d postgres -c "UPDATE tasks SET done = false WHERE id = 5"
psql -X -U author -d postgres <<'SQL'
SELECT key, verdict, error FROM phasevec.verify('docs', p_every => interval '0');
SELECT key, verdict, line FROM phasevec.brief('docs') ORDER BY key;
SQL
```

The first `verify` answers `4 | verified` and `brief` serves the line `task 5 is done`
as verified; after the world changes, `verify` answers `4 | refuted` and `brief` no longer
serves that line. An `assert()` and a `verify()` of its claim must be in two transactions (a
transaction that has written may not verify). The first claim on a row goes to the table's
owner, a steward of the watch, or a role that may `UPDATE` the row: `SELECT` alone is not
enough, which is why `author` is granted `UPDATE` on `docs`.

## 6. Next steps, and cleaning up

* `make installcheck PG_CONFIG="$PG_CONFIG"` against a running cluster, and the `test/*.sh`
  scripts (not run for this kit).
* Then `what-to-attack-pg_phasevec.md`.

```sh
pg_ctl -D "$DATA" -m fast -w stop && rm -rf "$DATA"
```

Tested for this kit on 2026-11-14 (UTC): steps 1-6 as written, step 1 from GitHub (`git clone`,
`git checkout v0.20.0` = `fdd19cb`), on PostgreSQL 19beta2 (a release build) and 18.6 (an
assertion-enabled build): the results are identical (`quickstart-evidence-pg_phasevec.txt`).
