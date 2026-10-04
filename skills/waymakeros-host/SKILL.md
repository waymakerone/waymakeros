---
name: waymakeros-host
description: >-
  Build and deploy an app on Waymaker Host — the order the steps go in, the flags that are
  permanent, and the failures that return success. Use before creating an app container,
  provisioning a database, writing or running migrations, or planning and applying a Solution
  release. Covers the steps that are easy to omit because their absence is invisible. Trigger
  phrases: "deploy on Waymaker Host", "provision a Waymaker app", "waymaker host apps create",
  "set up a Waymaker database", "release a Solution", "plan a release", "release migrations",
  "build for a client on Waymaker", "test against a database branch", "write a database test
  harness", "schedule an Ambassador", "call AI from a Host app", "Host billing".
---

# Building on Waymaker Host

This is the order to do things in, and the things that will not tell you when they go wrong.

**Read the whole page before running the first command.** Three of the steps below are
**permanent**, and several of the failures are **silent**. Both facts are cheap now and expensive
later.

## Before anything

```bash
npm i -g @waymakeros/cli@latest     # 2.14.0 or newer
waymaker --version                   # confirm which binary actually runs
which -a waymaker                    # more than one path = a shadowed install, fix that first
waymaker auth status                 # prints the Organization this key acts in
```

**`waymaker --version` is not enough.** If a repo-local `node_modules/.bin/waymaker` shadows the
global one, `--version` can report a version you are not running. Check `which -a`.

**"Not authenticated" and "bad key" render identically.** If `auth status` says not authenticated,
the usual cause is that `WAYMAKER_API_KEY` is not in the environment the CLI reads — not that the
key is wrong. Confirm which file holds it before assuming the credential is bad.

### Confirm the organisation before you create anything

Everything you create lands in **the organisation your credential acts in**, and nothing warns you
when that is the wrong one. A create in the wrong organisation **succeeds silently**: the app,
database or Solution is real, billed there, and invisible from the organisation you meant.

- **CLI:** the API key decides. `waymaker auth status` prints its `Organization` — read it.
- **MCP:** the connection acts in the user's **active** organisation, which is not necessarily
  the one this repository belongs to. Ask for your identity (`get_my_info`) and check the
  organisation before the first create of a session.
- **Building for a client?** Use an API key created **inside the client's organisation**. Never
  build a client's product from your own organisation's key and plan to move it later — there is
  no move. You would recreate everything, including the database.

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
waymaker host db connect <name>        # takes effect on the app's NEXT deploy

# 4. Compose the Solution — what releases together, and what is billed together
waymaker host solution create "<Name>" --region <same as the database>
waymaker host solution surface add <slug> --kind app      --id <app-id>
waymaker host solution surface add <slug> --kind database --id <db-id>

# 5. Schema: commit migrations to db/migrations/ in the app's repo, and PUSH them
#    (a release reads the repo at a commit — never your working tree)

# 6. Release
waymaker host solution plan <slug>                    # saves a plan — read it before applying
waymaker host solution release apply <release-id>
```

**In the app, use `@waymakeros/db`, not `pg`.** `pg` does not bundle for this runtime. Open the
connection inside each request; a pool held across requests hangs some requests and not others.

**`host db migrate` still exists** — for a database outside a Solution, or to bootstrap one before
its first release. Its `--dir` defaults to `migrations/`, while a release reads `db/migrations/`.
Pass `--dir db/migrations` so both read the same files; both record a file by its name, so a file
`db migrate` already ran is not pending in the next release.

## The three permanent decisions

**Decide these before step 1. None can be changed afterwards.**

| Flag | Default | Why it is permanent |
|---|---|---|
| `--type` | **`ex`** | `ex` requires an authenticated Waymaker user on **every request** — anonymous visitors get 401. `cx` authenticates optionally and never refuses. **An update that names the type is ignored, not rejected.** If your app has any public page — a sign-in page, a share link, a public listing — you need `cx`. |
| `--region` | **`iad` (US East)** | Region is fixed for the life of the database. **An unset region silently places customer data in the United States.** For an Australian customer, pass `--region syd`. A data-residency outcome must never be produced by a default. |
| `--framework` | none | Detected per deploy from the repo, but the container records what you declare. |

## Billing — before you create a Solution

A database, app compute and email all cost someone money. Say so before you create them.

- **Billing is moving to per Solution, and is rolling out.** Each Solution will be billed on its
  own: one free start per paid seat (once, not monthly, shared across the organisation's
  Solutions), and a card added on the Solution when it is created or when the free start runs out.
  **Until a Solution's page in Host asks for a card, do not tell the user to add one** — today, Host
  usage draws on the organisation's credits.
- **When a database stops answering or deploys start being refused**, run
  `waymaker host doctor <app>` before debugging code. It reports a database suspended for
  non-payment and says what resumes it. Follow what it says, not this page.
- **Do not create a second Solution or database to get around a billing stop.** It is the same
  organisation and the same bill, and you will have split the data.
- Never quote prices to the user. Rates are at https://waymakeros.com/pricing.

## Failures that return success

**These are the ones that cost days, because nothing goes red.**

**1. Skipping `db connect`.** Nothing else injects `DATABASE_URL` (and `WAYMAKER_DATABASE_URL`) into
the app's hosted environment — attaching the database as a Solution surface does **not** do it.
Without it the app deploys with no database connection, and at a first release **that is
invisible**: every page is supposed to render empty at that point, so a missing connection and a
correct first release look identical. And the variable reaches the app only on its next deploy.

**2. Editing a migration file after it has run.** Applied migrations are recorded **by file name**.
An edited file is never pending again, so it is skipped forever, silently — by `db migrate` and by
every release — and your database will not match your files. **To change anything, add a new
file.** Zero-pad names (`001_`, `002_`, `010_`): they run in plain byte order, so `10_x.sql` runs
*before* `2_y.sql`.

**3. Expecting a release to see migrations you have not pushed.** A release plan reads
`db/migrations/*.sql` (only that directory, not subdirectories) **in the repo, at the release
commit**: the tip of the tracked branch, or the staged build's commit under the `release` policy. A
file on your disk, on another branch, or elsewhere in the repo does not exist to it. The plan names
the commit and every file it will run — check them against what you expect.
**`no migrations found at db/migrations … — set a path override or pass migrations` is not a
refusal**: the plan proceeds and the app deploys against the schema already live. If your files
live elsewhere, point the database at them:

```bash
waymaker host db migrations-source <database-id> --path database/migrations
waymaker host db migrations-source <database-id> --repo https://github.com/<org>/<other-app-repo>
```

**4. Pushing code that needs a migration (the `push` policy).** Every Solution releases by **`push`**
unless its owner changes it: **a push to the tracked branch deploys the app straight to
production**, without a release and without running its migrations. Code that reads a new column
goes live before the column exists. Under `push`: push the migration on its own, release it (or
`db migrate`), then push the code that uses it — or keep every migration additive so old and new
code both work. Under the **`release`** policy (switched by a Solution owner in the Solution's
settings in Host), a push builds a **staged** copy at its own URL, runs pending migrations on the
staged copy's own database branch, and production changes only through a release.

**5. Reading only the `Ran:` line.** `host db migrate` prints `Ran:` and `Skipped (already
applied):`. An empty `Ran:` on a run you expected to do work means every file was already recorded —
which is failure 2. Verify against the schema itself, not the output.

## Planning and applying a release

**`plan` is not a dry run.** It saves a release, and **abandons any earlier plan for that Solution
that applied nothing**. Two people or agents planning the same Solution replace each other's plans,
and applying the replaced one is refused. Plan once, read it, apply that id.

- **A release that stopped part way blocks every new plan** until it is resumed or abandoned.
  `release resume <id>` continues without re-running what applied; `release abandon <id> --reason
  "…"` stops it and **undoes nothing**. Read `release status <id>` first.
- **Apply runs exactly what the plan reviewed.** The plan pins the commit and a fingerprint of each
  migration file; apply re-reads each one at that commit and refuses, running nothing, if any
  differs. Changed a file? Plan again.
- **`cannot tell which migrations are pending (…)` is a refusal**, deliberately: the plan does not
  assume "nothing pending". Fix what it names and plan again.
- **`… sort before the newest applied migration … the ledger and the repo disagree`** means the
  database ran files under other names, or a file was added out of order. Rename files to match what
  ran. Never "fix" it by renaming the new file to sort last after the fact.
- **No repo convention at all?** Pass migrations to `plan --migrations <file>` and **the same
  file** to `release apply --migrations <file>`. Passing `--migrations` to apply for a plan made
  without them is refused (409) — a migration the plan never reviewed does not run.
- **`would_proceed: true`** means no *blocking condition was detected*. It is not a statement that
  the release will succeed. Read the steps.
- **`changed_since_last_deploy: unknown`** is normal, not an error.

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
is provisioned.** A migration that references it before then fails and rolls back that whole file,
and in a release it stops the release before the app deploys. So provision sign-in **before** you
commit the first migration that uses it, then keep those migrations in `db/migrations/` with the
rest — a release reads only that one directory.

**Do not define your own `wm_user_id()`.** The platform creates it with `CREATE OR REPLACE` and
will silently replace yours.

## Choosing the piece

Host already gives an app storage, request/response compute, Ambassadors, transactional email and
an AI gateway. Pick the lightest piece that does the job:

- **"Record this"** (a waitlist, a ledger, a sign-up) is **a migration plus a route.** Anything
  heavier needs a reason.
- **A confirmation email is a route that sends through Host transactional email**
  (`waymaker host email` — register and verify the app's sending domain first). It does **not**
  need an Ambassador.
- **Reach for an Ambassador when work must happen *without* a request** — scheduled, retried, or
  long-running. "An email is involved" is not a reason.

### Calling an AI model from the app

The AI gateway is **in preview** and not open to every organisation yet. If key creation is refused,
it is not available to you — ask; do not put a model provider's own key in the app instead.

- **One key per Solution:** `waymaker host ai keys create <solution> <name>`. It is shown **once**;
  store it in the app's environment straight away. A lost key cannot be shown again — revoke it and
  create another.
- **Endpoint `https://ai.waymakerone.com/v1`**, compatible with the OpenAI and Anthropic SDKs: change
  the base URL and key only. Model names are `provider/model`.
- **Server side only** — the app's backend or an Ambassador. The endpoint does not answer browsers.
- **A request that could exceed a key's limit is refused up front**, not shortened. To raise a
  limit, create a new key with it, move the app, revoke the old one.
- **The gateway does not retry for you, and you should not retry blindly:** a retry is a second
  billed call.

## Testing against a database

**A test database is a branch, not a second database.** `waymaker host db branch create <app>
<name>` makes a copy-on-write branch of your database, and it carries the platform's
`wm_user_id()` and your row-level security with it. That makes RLS behaviour genuinely testable on
a branch — and **untestable in a throwaway local Postgres**, which has neither. **Never stub
`wm_user_id()`** to make a local database pass: you would be testing your stub, not your rules.

- `db branch` has `list`, `create`, `mask` and `delete --confirm`. Mask a branch before sharing it;
  delete branches when you are done — each one is a billed resource.
- **No command returns a connection string for a branch — or for the primary.** That is deliberate.
  It also means **a database-backed test has no CI path today**: nothing can hand CI a credential
  for a branch. Do not work around it by pointing tests at the app's own database URL. Run those
  tests where a branch credential is available to you, and treat CI coverage of them as a known gap.

### The test-harness rule

`@waymakeros/db` reads the **ambient** `DATABASE_URL`, falling back to `WAYMAKER_DATABASE_URL`.
`db connect` injects those into **the app's hosted environment** — the deployed app runtime and
its Ambassadors — as the **database owner**, which is exempt from row-level security. That is right
for the app and dangerous for a test: a harness that reads ambient env will insert into and delete
from whatever database it lands on, with owner rights and no confirmation.

(`db connect` does **not** write anything to your local shell or files. The hazard is any
environment where those variables hold the owner URL.)

**So a test harness never reads ambient database env:**

1. Take a **distinctly named** variable — e.g. `TEST_DATABASE_URL`, never `DATABASE_URL`.
2. **Refuse to run** if it is unset, empty, or equal to `DATABASE_URL` or `WAYMAKER_DATABASE_URL`.
3. Only then pass it **explicitly**: `createClient(url)`. Never import the default `db` / `sql` /
   `query` in a harness — those read ambient env.

**Order matters in step 3: `createClient(undefined)` silently falls back to ambient env.** Passing
the variable "explicitly" without refusing first puts you straight back on the owner URL. Test all
three refusal paths.

```ts
import { createClient } from '@waymakeros/db'

const url = process.env.TEST_DATABASE_URL
const ambient = [process.env.DATABASE_URL, process.env.WAYMAKER_DATABASE_URL].filter(Boolean)
if (!url || ambient.includes(url)) {
  throw new Error('Refusing to run: set TEST_DATABASE_URL to a branch, distinct from the app database')
}
const db = createClient(url)
```

### Proving a connection did *not* reach a database

**When you assert that a connection did NOT reach a database, prove the check would have seen it
if it had.** "The sentinel never appears" passes just as happily when the check is blind.

- **Do not read the destination out of error text.** With the current driver, `err.message` never
  names the host and `err.cause` is empty — the host is nested deeper, and where is
  driver-specific. A catch that prints `err.message` makes the assertion unfalsifiable.
- **Record the request instead.** The driver talks over HTTP, so in the test replace `fetch` with a
  stub that records the URL and throws. Nothing leaves the machine, and you see exactly where the
  client tried to go.
- **The driver rewrites the hostname before it dials** — the first label is replaced. Put the
  sentinel in a label that survives (`db.test-sentinel.invalid`, not `test-sentinel.invalid`), and
  use the reserved `.invalid` suffix so it can never resolve.
- **Break the guard on purpose and watch the assertion fail.** If it still passes with the guard
  removed, it was never testing the guard.

## Scheduled Ambassadors

Write a schedule as **standard five-field cron**. Day-of-week numbers mean what they mean
everywhere (`0` or `7` = Sunday, `1` = Monday), and are stored as names (`MON`) — names such as
`mon-fri` are the clearest to write. Non-standard extensions (`L`, `W`, `#`) are refused.

**"Did it run?" is answered by its logs** (`waymaker host ambassadors logs <id>`), not by the
schedule shown back to you. A schedule displays faithfully whether or not it fires.

## Reading what a Solution has used

`waymaker host solution credits` shows, per Solution:

- **email credits** — app transactional email sent;
- **compute credits** — **all-time** credits used by the Solution's apps and Ambassadors handling
  requests, updated daily. It **does not include database compute.** Blank means nothing has been
  recorded yet, not zero.

Do not read either number as the Solution's bill.

## Getting help

`waymaker host <group> <command> --help` lists every flag — read it for `--region` on
`solution create` and `--dir` on `db migrate` before you run either. If it prints the parent
group's help instead, your CLI is old: upgrade, then use `waymaker host <group> help <command>`.
