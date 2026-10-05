# Testing against a database

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

## The test-harness rule

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

## Proving a connection did *not* reach a database

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
