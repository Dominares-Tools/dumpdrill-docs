# dumpdrill — Prove a backup restores before you need it

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/dumpdrill**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

dumpdrill restores a pg_dump into a scratch database and checks the rows against a manifest you write, reads tar archives end to end, fails when the newest backup is stale, and rehearses a restore twice with a safety copy.

## Get it

Buy dumpdrill at https://dominares.org/tools/dumpdrill: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the dumpdrill command, with an offline demo and the test suite.

Requirements: Python 3.10 or newer, standard library only. The Postgres checks need PostgreSQL's command-line tools (psql, pg_restore, and initdb and pg_ctl for the built-in throwaway server) on your machine.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

A backup job that exits 0 has proved that a file was written. It has not proved that you can come back from it. These are the ways that goes wrong, and each has a check:

- **`pg_dump` of an empty database** writes a perfect file. So does a dump where every row is present and every `body` is `{}`. `dumpdrill pg` loads the dump into a scratch database and asks the restored copy your questions.
- **A valid archive full of empty files** passes the download check, the byte count and `tar tzf`. `dumpdrill archive` reads it end to end and checks the files you name are present, non-empty and parse.
- **A half backup**: a list inside the archive names four customers, the archive holds two. `dumpdrill archive` can cross-check one against the other.
- **A schedule that says "configured" while nothing has run for four days.** `dumpdrill fresh` asks the list of backups, not the schedule.
- **A restore that overwrites live work with nothing to go back to.** `dumpdrill snapshot` copies aside exactly the files a restore would replace, and `dumpdrill drill` rehearses the whole restore, twice, in a scratch directory.

Every command exits `0` (verified), `1` (a real gap) or `2` (it could not check: no `psql`, no such file, an unreadable answer). `2` is not `0` on purpose: "I was unable to verify the backup" must never read as a good night. Both non-zero codes are failures to a cron job or CI step. When one run finds both a real gap and something it could not check, the exit is `1`: a proven gap is reported as one.

#### Postgres dumps: `dumpdrill pg`

```bash
dumpdrill pg nightly.sql.gz --manifest manifest.json
```

Accepts a plain `.sql`, a gzipped `.sql.gz`, a custom-format dump (`pg_dump -Fc`, also gzipped), a tar-format dump (`-Ft`) or a directory dump (`-Fd`). It restores into a scratch database that exists only for the run:

- by default, **a private PostgreSQL server it starts itself** in a temp directory (unix socket only, no network port, deleted afterwards). It needs `initdb` and `pg_ctl` from a PostgreSQL install; nothing else of yours is touched;
- with `--server-dsn DSN`, **an existing local server**: a uniquely named database is created, restored into and dropped (`--keep` leaves it);
- with `--target-dsn DSN`, **one existing empty database** (never dropped; a database that already holds anything is refused).

Anything that is not provably this machine (a remote host, `service=`, a `PGHOST` that points elsewhere) is refused before a byte is sent, unless you pass `--allow-remote-target`.

You say what a restore must bring back in a manifest (JSON; TOML on Python 3.11+):

```json
{
  "postgres": {
    "tables": [
      {"name": "orders", "min_rows": 100, "columns": ["id", "total"]},
      {"name": "documents", "min_rows": 20, "filled": [{"column": "body", "min": 20}]}
    ],
    "probes": [
      {"name": "recent writes", "sql": "SELECT max(created_at) > now() - interval '3 days' FROM orders"},
      {"name": "no orphans", "sql": "SELECT count(*) FROM documents d LEFT JOIN orders o ON o.id = d.order_id WHERE o.id IS NULL", "expect": {"equals": 0}}
    ]
  }
}
```

- `tables`: must exist (tables, views or materialized views), have the listed `columns`, and at least `min_rows` rows (default 1; `0` allows an empty table). `filled` counts the rows where a column holds something: not NULL, not blank, not `{}`, `[]`, `null` or `""`. That is the check that catches rows that are present and worthless.
- `probes`: any SQL that returns one value. With no `expect` it must return true. `expect` takes `equals`, `not_equals`, `min`, `max` and `matches` (a regular expression). Probes run read-only. A probe that asks about something the dump lacks is a failure; a probe with a syntax error is "could not check".
- With no manifest at all, the check is only that the restore loaded and produced at least one table with at least one row.
- A misspelt key (`min_row`) is an error, not ignored.

Plain dumps are scanned before anything runs, with a reader that follows psql's own rules for quotes, dollar-quoted bodies, comments, parentheses and `COPY` blocks. A dump that ends with a string, comment or statement still open, with no closing `-- PostgreSQL database dump complete` line, or with a `COPY` block that never ends, is **truncated** (psql would quietly load the rows it got) and fails. The scan also refuses what `pg_dump` never writes: a psql command (`\connect`, `\!`, `\i`, `\o |cmd`, at the start of a line or in the middle of one), `COPY ... TO PROGRAM` / `FROM PROGRAM`, a `SET` of `client_encoding` or `standard_conforming_strings` other than the two lines `pg_dump` writes, a number run into letters and then a quote or `$` (PostgreSQL versions read that differently, so it cannot be scanned safely), a plain `'...'` string with an unpaired backslash before a quote (it ends somewhere else if `standard_conforming_strings` is switched off while the script runs; `pg_dump` writes one only for a value that ends in a backslash), and a NUL byte. `--allow-psql-commands` is the opt-in.

**A dump is a script, so treat it as trusted input.** The scan is a tripwire for the commands above, not a sandbox: it was checked against real psql, including a seeded fuzz, but it is a second lexer written to agree with psql's, and what remains is SQL. By default that SQL runs as the superuser of the scratch server (which can, for instance, read and write files that server's user can, and run programs). With `--server-dsn` or `--target-dsn` it runs with the privileges of the role in your DSN, and its SQL can reach other databases on that server; give those a role that is not a superuser. For a dump you do not fully trust, add **`--unprivileged`**: the throwaway server restores it as a role that is not a superuser (it owns only the scratch database), so `COPY ... PROGRAM`, `pg_read_file` and the like are refused by the server itself and the result is `2`. The price is that a dump that genuinely needs superuser rights (objects owned by a superuser, an extension that is not trusted) cannot load that way and also gives `2`, with a message saying so. Dumps made without owner statements (`pg_dump -O`), and any custom, tar or directory dump (those are restored without owners), are the ones that load cleanly.

`--unprivileged` limits what the server can do. The commands psql itself would run on your machine are stopped only by the scan, and only plain dumps go through psql (a custom, tar or directory dump is restored by `pg_restore`, which has no such commands). For a dump you believe to be hostile, run the check in a disposable container or virtual machine as well.

Options: `--manifest FILE`, `--server-dsn DSN`, `--target-dsn DSN`, `--allow-remote-target`, `--keep`, `--create-role NAME` (create a role the dump names as an owner; repeatable), `--unprivileged`, `--allow-psql-commands`, `--no-complete-marker-check`, `--pg-bin DIR` (or `$DUMPDRILL_PG_BIN`), `--timeout SECONDS` (default 3600), `--json`.

If the scratch server cannot host the dump because of **its own environment** (an owner role that does not exist, an extension that is not installed, a PostgreSQL older than the one that made the dump), the result is `2` with that reason, not `1`: that is the scratch server, not necessarily the backup. `--create-role` and a matching `--pg-bin` fix it.

#### Tar archives: `dumpdrill archive`

```bash
dumpdrill archive nightly.tar.gz --manifest manifest.json
```

Reads `.tar`, `.tar.gz`, `.tar.bz2` and `.tar.xz` end to end (so a truncated or corrupt file fails, including one missing only its checksum trailer), refuses absolute paths, `..` and links that point outside the archive, and fails an archive whose files are all empty. The manifest's `archive` section:

```json
{
  "archive": {
    "min_entries": 20,
    "members": [
      {"path": "config/settings.json", "format": "json", "why": "the switches an operator set"},
      {"path": "registry.json", "expect_json": {"ok": true}}
    ],
    "references": [
      {"list_in": "registry.json", "at": "accounts[].id", "exists_as": "accounts/{value}"}
    ]
  }
}
```

- `members`: each `path` must exist, have at least `min_bytes` (default 1), and with `format` `json` or `text` must parse and hold something (`{}`, `[]`, `null`, `""` and blank text count as nothing; `"non_trivial": false` turns that off). `expect_json` asserts values at dotted paths. A `path` matches whole directory names at the end of a member's path: `config/app.json` matches `data/config/app.json` and not `data/old_config/app.json`, and when a name matches several members every one is checked and you are told.
- `references`: every value found by `at` in the JSON member `list_in` (`name[]` fans out over a list) must have a directory `exists_as` in the archive. A listed account with no files is a failure. Directories present but not listed are a warning (`"warn_orphans": false` silences it). A path that is not in the document at all (a misspelt key, or a list that is not a list) is a failure, never a pass over zero values, and so is a list that comes back with fewer than `min_values` values (default 1; `"min_values": 0` allows an empty list).
- `max_empty_files` fails the archive when more than that many files are empty (otherwise empty files are a warning, and an archive where all are empty always fails).

#### Freshness: `dumpdrill fresh`

```bash
dumpdrill fresh --max-age-hours 36 --github OWNER/REPO --artifact nightly-backup
dumpdrill fresh --max-age-hours 26 --glob '/backups/*.dump' --min-size-bytes 1000000
dumpdrill fresh --max-age-hours 36 --timestamps backup-times.txt
```

Fails when the newest backup is older than the limit, and when there is no backup at all (never run, or all expired or deleted). Sources, each judged on its own and combinable:

- `--github OWNER/REPO --artifact NAME`: unexpired Actions artifacts of that name, token from `$GITHUB_TOKEN` or `$GH_TOKEN` (or `--github-token-env NAME`). This also catches a scheduled workflow GitHub has disabled.
- `--glob PATTERN`: the newest file matching the pattern, by modification time (`**` recurses).
- `--timestamps FILE` (`-` for stdin) or `--at TIME`: for anything that can print when each backup was made. One ISO 8601 time or epoch-seconds value per line; blank lines and `#` comments are ignored; any other line is "could not check", never silently dropped. A time without a zone is read as UTC.

`--max-gap-hours N` also fails a hole between consecutive backups wider than N hours (a fresh newest backup can hide a four-day outage in the history). `--min-size-bytes N` fails a newest backup smaller than N (GitHub artifacts and files only). A newest backup dated more than five minutes in the future is "could not check".

#### Safety copy: `dumpdrill snapshot`

```bash
dumpdrill snapshot /srv/app/data nightly.tar.gz --out /srv/app/undo
```

Writes a `.tar.gz` of the live files under the root that the archive would overwrite, and nothing else: files the archive only adds are not copied, and no empty placeholder is written. Exit `0` with "Nothing to save" when the archive overwrites nothing; exit `2` with "Refuse to restore" when the copy could not be made, and your restore should stop there. Default location: `<root>/backups/pre_restore/`. `--prefix` names the folder inside the copy (default: the root's name). `--strip NAME` / `--no-strip`: by default the archive's first directory is dropped when it equals the root's own name (an archive of `/srv/app/data` carries `data/...`).

The same path rules are used everywhere (`dumpdrill.paths`): the copy never follows `..`, an absolute path, or a symlink that leads outside the root.

#### Restore drill: `dumpdrill drill`

```bash
dumpdrill drill nightly.tar.gz --manifest manifest.json --boot-cmd "./smoke-test.sh"
```

In a scratch directory it checks the archive, restores it into an empty tree, then compares every restored file with the archive's own bytes (sha256). It then makes a live edit to one restored file and restores **again** over the populated tree, which is the only way to exercise the safety copy: the copy must exist and hold the live edit byte for byte, and the file must be back to the archive's bytes. Finally `--boot-cmd` (run inside the restored tree, split like a shell would, no shell) must exit 0 within `--boot-timeout` seconds (default 120).

- Without `--boot-cmd` the boot phase is **skipped, and the result says so** ("DRILL PASSED, but not exercised: boot"). A pass that did not boot anything is not a claim that it boots.
- `--restore-cmd CMD` rehearses your own restore procedure instead of the built-in one. It runs with `$DUMPDRILL_ARCHIVE` and `$DUMPDRILL_TREE` set; the same byte comparison applies. Add `--safety-dir DIR` (exported as `$DUMPDRILL_SAFETY_DIR`) so the drill can find the copy your procedure keeps; without it that phase is skipped, loudly.
- `--root-name NAME` (default `data`), `--strip NAME` / `--no-strip`, `--workdir DIR` (must be empty), `--keep`.

Production is never a target: the tree is always created inside a scratch directory the drill makes.

#### As a library

```python
from dumpdrill.manifest import parse_pg
from dumpdrill.pgcheck import verify_dump
from dumpdrill.archive import check_archive
from dumpdrill.freshness import check_freshness
from dumpdrill.safety import snapshot_before_restore, restore_archive
from dumpdrill.drill import run_drill

report = verify_dump("nightly.sql.gz", parse_pg({"tables": [{"name": "orders", "min_rows": 100}]}))
print(report.render())
raise SystemExit(report.exit_code)
```

`snapshot_before_restore(root, archive)` returns `{"path", "files", "error"}`; a non-empty `error` means refuse to restore.

### Limits

- **Runtime is the Python standard library only (3.10 or newer).** The Postgres leg runs the `psql` and `pg_restore` programs, plus `initdb` and `pg_ctl` for the built-in scratch server. They are found on `PATH`, in the usual install directories, or via `--pg-bin`. Without them, `dumpdrill pg` exits `2`. The built-in scratch server needs unix sockets (macOS, Linux) and cannot run as root; elsewhere use `--server-dsn`.
- The scratch server must be at least as new as the server that made the dump, and have the extensions the dump creates; otherwise the result is `2` (see above).
- A DSN is passed to `psql` on its command line, so a password in it is visible to other users of the machine; use `PGPASSWORD` or a password file. A DSN that begins with `-` is refused, and a `PGHOSTADDR` in the environment counts as a host even when the DSN names `localhost`.
- The dump scan runs over the whole plain dump before the restore: `COPY` data goes through at hundreds of MB a second, a dump made of `INSERT` statements at roughly 5 MB a second.
- The dump scan reads the file the way psql does, but it is not a security boundary (see "treat it as trusted input" above). `--unprivileged` is the boundary, and only on the throwaway server.
- Restoring proves the dump loads and holds what you declared. It does not prove that your application works against it; for files, that is what `--boot-cmd` is for. For Postgres, write probes.
- A plain dump with a line that starts with a backslash inside a multi-line string or function body looks like a psql command and is refused until you pass `--allow-psql-commands`.
- `fresh` reads GitHub artifacts through the REST API, at most 1000 per name. Reading another store's backup list is up to you via `--timestamps`.
- A timestamp without a zone is read as UTC. A file's age is its modification time. A bare run of digits is epoch seconds only when it has ten or more; `20261004` is "could not check", not 1970.
- `archive` and `drill` read tar archives (gz, bz2, xz or none). Zip is not supported. A JSON or text member larger than 64 MB cannot be parsed and gives `2`.
- The built-in restorer writes regular files only: symlinks, hard links and special files in an archive are not restored (links that escape the root make the archive fail). File ownership is not restored; modes lose setuid, setgid and group/world write.
- The drill's byte comparison proves the tree matches what the restore plan says it should be. A wrong `--strip` therefore restores byte for byte to the wrong place and still compares equal; `--boot-cmd`, or `--keep` and a look at the tree, is what catches that.

---

Docs mirror of the private repository Dominares-Tools/dumpdrill; updated with each release.
