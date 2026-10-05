---
name: waymakeros-host
description: >-
  Build and deploy an app on Waymaker Host — the order the steps go in, the flags that are
  permanent, and the failures that return success. Use before creating an app container,
  provisioning a database, writing or running migrations, or planning and applying a Solution
  release; before adding end-user sign-in, app email, storage, an Ambassador or a custom domain;
  and when something fails and you need to know whether it is your fault or the platform's.
  Trigger phrases: "deploy on Waymaker Host", "provision a Waymaker app", "waymaker host apps
  create", "set up a Waymaker database", "release a Solution", "plan a release", "release
  migrations", "build for a client on Waymaker", "password reset for my app", "app sign-in",
  "send email from my app", "sending domain", "upload files from my app", "storage bucket", "call
  my Ambassador", "schedule an Ambassador", "custom domain", "host doctor", "test against a
  database branch", "call AI from a Host app", "Host billing".
---

# Building on Waymaker Host

This is the order to do things in, and the things that will not tell you when they go wrong.

**Read this page before running the first command.** Three of the steps below are **permanent**,
and several of the failures are **silent**. Load a reference page when the job reaches it:

| When the job involves | Read |
|---|---|
| Your app's own users signing in, password reset, verification, second factor | [references/sign-in.md](references/sign-in.md) |
| The app sending email (receipts, notifications), sending domains, DNS for email | [references/email.md](references/email.md) |
| Files: buckets, uploads, signed links, getting a file from the app to storage | [references/storage.md](references/storage.md) |
| Ambassadors: deploy, env, schedules, logs, calling one from the app | [references/ambassadors.md](references/ambassadors.md) |
| The client's own domain on the app | [references/domains.md](references/domains.md) |
| Tests that touch a database | [references/testing.md](references/testing.md) |

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

Order matters. Steps 1 and 2 contain one-way doors.

```bash
# 1. The app. This also creates a Solution with the app's slug, with you as its owner.
waymaker host apps create <name> \
  --repo https://github.com/<org>/<repo> \
  --branch main \
  --framework <framework> \
  --runtime <server|static> \
  --build-command "npm run build" \
  --type <cx|ex>                       # PERMANENT — see below

# 2. The database. --region is PERMANENT. It joins the app's Solution automatically.
waymaker host db create <app> --region <syd|sin|fra|lon|iad>

# 3. Connect it — this injects the connection string. Nothing else does.
waymaker host db connect <app>         # takes effect on the app's NEXT build

# 4. Read the Solution: what is in it, and how it releases
waymaker host solution surfaces <app-slug>
waymaker host solution get <app-slug> --json     # "release_policy": "release" or "push"

# 5. Schema: commit migrations to db/migrations/ in the app's repo, and PUSH them
#    (a release reads the repo at a commit — never your working tree)

# 6. Release (see "Releases" below)
waymaker host solution plan <slug>                   # saves a plan — read it before applying
waymaker host solution release apply <release-id>    # call again until it is no longer "applying"
waymaker host solution release status <release-id>
```

**Do not create a second Solution for the app.** `apps create` already made one. Running
`solution create` and then `solution surface add --kind app` is refused: an app belongs to exactly
one Solution. Add other pieces (an Ambassador, a bucket, a domain) to the app's Solution with
`solution surface add <slug> --kind <kind> --id <id>`.

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
the app's hosted environment — attaching the database to the Solution does **not** do it. Without
it the app deploys with no database connection, and at a first release **that is invisible**:
every page is supposed to render empty at that point, so a missing connection and a correct first
release look identical. And the variable reaches the app only on its next build.

**2. Editing a migration file after it has run.** Applied migrations are recorded **by file name**.
An edited file is never pending again, so it is skipped forever, silently — by `db migrate` and by
every release — and your database will not match your files. **To change anything, add a new
file.** Zero-pad names (`001_`, `002_`, `010_`): they run in plain byte order, so `10_x.sql` runs
*before* `2_y.sql`.

**3. Expecting a release to see migrations you have not pushed.** A release plan reads
`db/migrations/*.sql` (only that directory, not subdirectories) **in the repo, at the release
commit**: the staged build's commit under the `release` policy, or the tip of the tracked branch
under `push`. A file on your disk, on another branch, or elsewhere in the repo does not exist to
it. The plan names the commit and every file it will run — check them against what you expect.
**`no migrations found at db/migrations … — set a path override or pass migrations` is not a
refusal**: the plan proceeds and the app deploys against the schema already live. If your files
live elsewhere, point the database at them:

```bash
waymaker host db migrations-source <database-id> --path database/migrations
waymaker host db migrations-source <database-id> --repo https://github.com/<org>/<other-app-repo>
```

**4. Pushing code that needs a migration, under the `push` policy.** Under `push`, **a push to the
tracked branch deploys the app straight to production**, without a release and without running its
migrations. Code that reads a new column goes live before the column exists. Under `push`: push the
migration on its own, release it (or `db migrate`), then push the code that uses it — or keep every
migration additive so old and new code both work. Under `release` this cannot happen: see below.

**5. Reading only the `Ran:` line.** `host db migrate` prints `Ran:` and `Skipped (already
applied):`. An empty `Ran:` on a run you expected to do work means every file was already recorded —
which is failure 2. Verify against the schema itself, not the output.

**6. Changing settings that only apply on the next deploy.** Each of these succeeds at once and
changes nothing until the next build or deploy: `db connect`; app and Ambassador environment
variables; an Ambassador's schedule; sign-in methods (`db auth methods`), which need
`db auth deploy`. If a change "did nothing", check whether anything has been deployed since.

## Releases

### Which policy is this Solution on?

`waymaker host solution get <slug> --json` shows `release_policy` (the plain-text output does not).

- **`release`** is the default for every Solution created since 4 October 2026 — including the one
  `apps create` makes. A push builds a **staged** copy; production changes **only** through a
  release.
- **`push`** is what older Solutions have, unless their owner switched. A push deploys straight to
  production (failure 4).
- **Only a Solution owner can switch**, in the Solution's settings in Host. There is no CLI command
  or MCP tool for it. Switching back to `push` removes the Solution's staged builds.

### What a push does under `release`

- **Each app in the Solution gets a staged build** at its own address, replaced on every push. Find
  it with `waymaker host apps preview list <app>` (the row with branch `__staged__`).
- **If the app has a database, the staged build gets its own database branch**, and pending
  migrations run on that branch before the build. A failing migration fails the staged build, not
  production. (If the database's migrations live in another app's repo, this push runs none.)
- **Ambassadors are not deployed by a push.** They change only in a release.
- **A brand-new app's first build is staged too**, so its production address stays empty until the
  first release.
- **Every other route to production is refused** while the Solution is on `release`
  (`apps deploy`, promote, `ambassadors deploy`), with a message telling you to release.
  `apps rollback` is the exception: see below.
- **A plan needs a staged build.** "no staged build of X yet — push to <branch> to stage one" means
  exactly that.

### What a release does, in order

A release is derived from the Solution, in a fixed order you cannot change:

1. run pending **migrations** (only if a file is pending);
2. deploy **end-user sign-in**, if the database has it;
3. deploy **Ambassadors**, then any agents;
4. deploy **apps** at the release commit;
5. check each **custom domain** is live.

Each deploy step counts as done only when the deployment succeeded at the pinned commit **and** the
address answers. A build that has not settled in 30 minutes fails the step. A release stops at the
first failure and skips the rest.

### Applying: call `apply` until it settles

**One `apply` does not finish a release.** It does what it can, then returns while a build runs,
with the state still `applying`. **Call `apply` again** (it continues from where it stopped and
re-runs nothing) **until the state is no longer `applying`**:

| Release state | Means |
|---|---|
| `planned` | saved, nothing run |
| `applying` | in progress — call `apply` again (a minute apart is plenty) |
| `applied` | every step settled |
| `partially_applied` | it stopped, and something is live |
| `failed` | it stopped, and nothing is live |
| `refused` / `abandoned` | it will not run |

- **Read `waymaker host solution release status <id>` after every `apply`.** The CLI's `apply`
  reports a `partially_applied` or `failed` release as a bare "API error" without the steps. The
  status command shows each step and its `refusal_reason`.
- **In `release status`, read the state word, not the mark.** A step that is still `pending` or
  `running` is marked ✖, the same as a failure.
- Two `apply` calls at once: the second is refused ("Another call is advancing this release").
  Wait and call again.
- `waymaker host solution release list <slug>` shows recent releases and their states.

### Planning

**`plan` is not a dry run.** It saves a release, and **abandons any earlier plan for that Solution
that applied nothing**. Two people or agents planning the same Solution replace each other's plans,
and applying the replaced one is refused. Plan once, read it, apply that id. Saving a plan needs
the Solution's `owner` or `releaser` role.

- **A release that is `applying` or `partially_applied` blocks every new plan** until it finishes,
  is resumed or is abandoned. `release resume <id>` continues without re-running what applied (it
  cannot start while a build is in flight); `release abandon <id> --reason "…"` stops it and
  **undoes nothing**. Read `release status <id>` first.
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

### Rolling back

`waymaker host apps rollback <app> [deployment-id]` puts a previous production deployment back,
instantly and without a rebuild. **It is allowed under `release`**, because it is the emergency
lever, and it is recorded in the Solution's activity in Host as **rolled back outside a release**
(`rollback_outside_release`), with who did it and to what. The rollback response does not mention
this, and only apps can be rolled back this way.

A rollback is not a release. The next release deploys the commit it pins. So after a rollback, fix
forward, push, and release, rather than leaving production on a version no release recorded.

## If you enable database sign-in

`waymaker host db auth provision` / `deploy` give your app its own end-user accounts (the flows,
routes and emails are in [references/sign-in.md](references/sign-in.md)). Two things follow
immediately, and the second is the one people miss.

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

**Your rules only apply on the user's connection.** In the app, query as the signed-in user with
`forUser(token)` from `@waymakeros/db`. The default export connects as the database owner and
bypasses every rule.

## Choosing the piece

Host already gives an app storage, request/response compute, Ambassadors, transactional email and
an AI gateway. Pick the lightest piece that does the job:

- **"Record this"** (a waitlist, a ledger, a sign-up) is **a migration plus a route.** Anything
  heavier needs a reason.
- **Sign-in, password reset and verification emails need nothing from you.** The platform sends
  them. They do not use a sending domain and need no DNS.
- **A confirmation email from the app is a route that sends through Host transactional email**
  ([references/email.md](references/email.md)). It does **not** need an Ambassador.
- **Files go in a storage bucket, through an Ambassador.** An app cannot write to a bucket itself
  ([references/storage.md](references/storage.md)).
- **Reach for an Ambassador when work must happen *without* a request** — scheduled, retried, or
  long-running — or to hold storage access. "An email is involved" is not a reason.

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

## Your fault or the platform's?

Spend ten minutes finding out before you spend an afternoon changing code.

1. **Read the failure where it is recorded**, not where it surfaced: `waymaker host apps logs
   <app>` (build) or `--runtime` (requests); `host_ambassador_deployment_logs` (an Ambassador
   build); `release status <id>` (a release step's `refusal_reason`).
2. **Run `waymaker host doctor <app>`.** It checks the app's build state, runtime, database
   (region, suspension, connection), sign-in, and whether pushes can rebuild. Each problem comes
   with a fix command.
   - **Known false positive:** "No GitHub connection for this organization — pushes will not
     rebuild" can appear when pushes *do* rebuild. Check `waymaker host apps deployments <app>`:
     if your recent commits appear there, ignore that line.
   - Doctor prints the app's address and custom domain but does **not** test them.
3. **Signs it is the platform, not you:** the step that failed is one you do not control (setting
   up the environment, an "unresponsive" build, a timeout with no output from your code); something
   that worked before fails with no change on your side; or a minimal fresh resource (a new
   hello-world Ambassador, an empty app) fails the same way.
4. **Then stop and ask Waymaker**, with the resource id, the deployment or release id, the time
   and the exact message. Do not work around a platform fault by recreating apps, databases or
   Solutions: the recreated one is billed, the data is split, and the fault is still there.

For example: when new Ambassadors fail their first deploy at a platform step while existing ones
redeploy fine, comparing against an existing Ambassador shows it is the platform in five minutes.

## Reading what a Solution has used

`waymaker host solution credits` shows, per Solution:

- **email credits** — app transactional email sent;
- **compute credits** — **all-time** credits used by the Solution's apps and Ambassadors handling
  requests, updated daily. It **does not include database compute.** Blank means nothing has been
  recorded yet, not zero.

Do not read either number as the Solution's bill.

## Getting help

`waymaker host <group> <command> --help` lists every flag — read it for `--region` on
`db create` and `--dir` on `db migrate` before you run either. If it prints the parent group's
help instead, your CLI is old: upgrade, then use `waymaker host <group> help <command>`.
