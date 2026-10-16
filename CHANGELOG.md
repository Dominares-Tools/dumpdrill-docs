# Changelog

## 1.0.1

Fixes from an adversarial review of 1.0.0.

- Security: the 1.0.0 claim that a plain dump "cannot run a shell command" was too strong, and two ways around the scan are closed or documented. The dump scan now follows psql's lexer (strings, `E''` escapes, dollar-quoted bodies, nested comments, parentheses, `BEGIN ATOMIC`, `COPY` blocks), so a fake `COPY` line inside a dollar-quoted string can no longer hide a `\!` after it, and a psql command in the middle of a line is found (it was only looked for at the start of a line). The scan is checked against real psql in the tests (a table of scripts, and a seeded fuzz of random scripts: whenever psql ran the planted command the scan had flagged it). The fuzz found one way the first version of this scan could be led astray (`1e--`: psql reads `1e-` as one number, so the dash is not a comment), and reading psql's lexer found two more (a bare carriage return ends a `--` comment; strings read differently when `standard_conforming_strings` is switched off at run time). All are closed and have tests. `COPY ... TO/FROM PROGRAM`, a `SET` of `client_encoding` / `standard_conforming_strings` other than pg_dump's own, a number run into letters then a quote or `$` (PostgreSQL versions lex that differently), a plain string with an unpaired backslash before a quote, a NUL byte and a malformed `\.` are refused too. A dump is still SQL run by a superuser on the scratch server, and the README now says so; new `--unprivileged` restores as a role that is not a superuser on the throwaway server, which is the boundary.
- A dump that ends inside an open string, comment or statement is reported as truncated.
- `archive` references: a path that is not in the document (a misspelt `at`) now fails instead of passing over zero values, and a list that yields fewer than `min_values` values (default 1, new key) fails.
- A DSN beginning with `-` is refused and passed to `psql` as `--dbname=...`; `PGHOSTADDR` always counts as a host, so `host=localhost` with a remote `PGHOSTADDR` is no longer taken for local.
- Exit code: a real gap (1) now wins over could-not-check (2) in one report.
- `fresh`: a compact date such as `20261004`, or any all-digit value under ten digits, is "could not check" instead of being read as 1970.
- `archive`: a reference list too large to parse is "could not check", not "unreadable".
- Housekeeping: removed Worker-only lines from `.gitignore`.

## 1.0.0

First release.

- `dumpdrill pg`: restore a `pg_dump` (plain, gzip, custom, tar or directory format) into a scratch
  database and check what came back against a manifest (required tables, row counts, columns,
  populated columns, SQL probes). The scratch database lives on a private throwaway server, or on a
  local server you name; non-local targets are refused. Truncated plain dumps and dumps carrying
  psql commands are caught before anything runs.
- `dumpdrill archive`: read a tar archive end to end and check members and cross-references
  declared in the manifest; refuse unsafe paths; fail an archive of empty files.
- `dumpdrill fresh`: fail when the newest backup is too old, from GitHub Actions artifacts, files, or
  a list of timestamps, with optional gap and size floors.
- `dumpdrill snapshot`: copy aside exactly the live files a restore would overwrite.
- `dumpdrill drill`: rehearse a restore in a scratch directory: byte comparison, a second restore
  over a live edit with the safety copy verified, and an optional boot command.
- `dumpdrill demo`: watch the checks catch broken backups, with nothing of yours involved.
- Exit codes: 0 verified, 1 a real gap, 2 could not check (never a pass).
