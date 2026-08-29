---
name: waymakeros-host
description: >-
  Build and deploy an app on Waymaker Host — the order the steps go in, the flags that are
  permanent, and the failures that return success. Use before creating an app container,
  provisioning a database, running migrations, or planning a Solution release. Covers the steps
  that are easy to omit because their absence is invisible. Trigger phrases: "deploy on Waymaker
  Host", "provision a Waymaker app", "waymaker host apps create", "set up a Waymaker database",
  "release a Solution".
---

# Building on Waymaker Host

This is the order to do things in, and the things that will not tell you when they go wrong.

**Read the whole page before running the first command.** Three of the steps below are
**permanent** and two of the failures are **silent**. Both facts are cheap now and expensive later.

## Before anything

```bash
npm i -g @waymakeros/cli@latest     # 2.11.0 or newer
waymaker --version                   # confirm which binary actually runs
which -a waymaker                    # more than one path = a shadowed install, fix that first
waymaker auth status                 # must report YOUR organisation
```

**`waymaker --version` is not enough.** If a repo-local `node_modules/.bin/waymaker` shadows the
global one, `--version` can report a version you are not running. Check `which -a`.

**"Not authenticated" and "bad key" render identically.** If `auth status` says not authenticated,
the usual cause is that `WAYMAKER_API_KEY` is not in the environment the CLI reads — not that the
key is wrong. Confirm which file holds it before assuming the credential is bad.

## The sequence

Order matters. Steps 2 and 3 are one-way doors.

```bash
# 1. The app container
waymaker host apps create <name> \
  --repo https://github.com/<org>/<repo> \
  --branch main \
  --framework <framework> \
  --runtime <server|static> \
  --build-command "npm run build" \
  --type <cx|ex>                       # PERMANENT — see below

# 2. The database. --region is PERMANENT.
waymaker host db create <name> --region <syd|sin|fra|lon|iad>

# 3. Connect it — this injects the connection string. Nothing else does.
waymaker host db connect <name>

# 4. Schema. --dir defaults to migrations/ — pass it if yours live elsewhere.
waymaker host db migrate <name> --dir db/migrations

# 5-7. Compose and release
waymaker host solution create "<Name>" --region <same as the database>
waymaker host solution surface add <slug> --kind app --id <app-id>
waymaker host solution surface add <slug> --kind database --id <db-id>
waymaker host solution plan <slug>          # applies nothing; read it
waymaker host solution release apply <release-id>
```

## The three permanent decisions

**Decide these before step 1. None can be changed afterwards.**

| Flag | Default | Why it is permanent |
|---|---|---|
| `--type` | **`ex`** | `ex` requires an authenticated Waymaker user on **every request** — anonymous visitors get 401. `cx` authenticates optionally and never refuses. **`app_type` is absent from the update whitelist, so an update naming it is silently dropped, not rejected.** If your app has any public page — a sign-in page, a share link, a public listing — you need `cx`. |
| `--region` | **`iad` (US East)** | Region is fixed for the life of the database. **An unset region silently places customer data in the United States.** For an Australian customer, pass `--region syd`. A data-residency outcome must never be produced by a fallback constant. |
| `--framework` | none | Detected per deploy from the repo, but the container records what you declare. |

## Failures that return success

**These are the ones that cost days, because nothing goes red.**

**1. Skipping `db connect`.** Nothing else injects `WAYMAKER_DATABASE_URL` — attaching the database
as a Solution surface does **not** do it. Without it the app deploys with no database connection,
and at a first release **that is invisible**: every page is supposed to render empty at that point,
so a missing connection and a correct first release look identical.

**2. Editing a migration file after it has run.** Applied migrations are tracked **by filename, with
no checksum**. An edited file is skipped forever, silently, and reports success — your database will
not match your files. **To change anything, add a new file.** Zero-pad names (`001_`, `002_`, `010_`)
because they run in plain alphabetical order: `10_x.sql` runs *before* `2_y.sql`.

**3. Passing migrations to a release.** A release cannot carry migrations from the CLI, the MCP, or
the web UI. If you supply them anyway they are **silently ignored** and the release still reports
success. **Schema moves at `host db migrate`, before the release.** Expect the plan to say *no
schema change is part of this release* — that is correct, not an error.

**4. Reading only the `Ran:` line.** `host db migrate` prints `Ran:` and `Skipped (already
applied):`. An empty `Ran:` on a run you expected to do work means every file was already recorded —
which is failure mode 2. Verify against the schema itself, not the output.

## If you enable database sign-in

`waymaker host db auth provision` / `deploy` give your app its own end-user accounts. Two things
follow immediately, and the second is the one people miss.

**Every table you create in `public` from then on is readable *and writable* by any signed-in
end user of your app, by default.** It is a standing default, not a one-off grant, and per-table
`REVOKE`s do not survive a view being replaced or a restore.

**So: enable row-level security on every table you create, and write a policy.** A table with RLS
off is open to your end users. If data must never be reachable by an end user, put it in a
**separate schema** and do not grant usage on it — per-table revokes are not durable.

Write access rules as `USING (owner_id = wm_user_id())`. `wm_user_id()` returns NULL for an
unidentified caller, so rules deny rather than expose. **It does not exist until end-user sign-in
is enabled** — a migration referencing it before then fails and rolls back the whole file, so keep
access-rule migrations in a separate directory and run them with `--dir` once auth is on.

**Do not define your own `wm_user_id()`.** The platform creates it with `CREATE OR REPLACE` and
will silently replace yours.

## Reading a release plan

The plan applies nothing. Read it before you apply it.

- **`no schema change is part of this release`** — correct when schema moved at `db migrate`.
- **`changed_since_last_deploy: unknown`** — normal, not an error.
- **`would_proceed: true`** — means no *blocking condition was detected*. It is not a statement
  that the release will succeed. Read the steps.

## Getting help

`waymaker host <group> --help` works. If a subcommand's `--help` prints the parent group's help
instead of its own, use `waymaker host <group> help <subcommand>` — some flags are only visible
that way, including `--region` on `solution create` and `--dir` on `db migrate`.
