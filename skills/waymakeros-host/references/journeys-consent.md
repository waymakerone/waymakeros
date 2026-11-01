# Journeys consent: keeping an app's opt-outs in step

An app (a CRM, say) can read its organisation's Journeys suppression list: every address Journeys must
not mail, and why. It can also add addresses, so Journeys stops mailing people who opted out in the app.
Full reference: https://docs.waymakerone.com/host/journeys-suppressions

## Before you write any code

- **The grants are `journeys:suppressions:read` and `journeys:suppressions:write`, by name.** `*` does not
  include them and neither includes the other. Only an **org admin** can add them, so if you are not one,
  ask them; do not work around it. Check what the app holds with the MCP
  tool `host_app_get` (`commander_permissions`).
- **Adding it replaces the list.** `--commander-permissions` sets the whole list, so include what the app
  already has: `waymaker host apps update <app-id> --commander-permissions "team_ids,journeys:suppressions:read"`.
- **An app can only suppress.** Nothing through this API lifts a suppression. If a person wants mail again, they
  opt back in through a Journeys form, or an admin reinstates them in Journeys. The feed then shows it.

## The call

`POST https://apps.waymakerone.com/functions/v1/host-context-api`, with
`Authorization: Bearer $WAYMAKER_CONTEXT_KEY` and
`{ "action": "journeys_suppressions", "data": { "after": "<cursor>", "limit": 500 } }`.

It returns `{ suppressions, next, has_more, settle_seconds }`. No `after` means start from the first entry.

## Backfill, then poll

```js
let after = await loadCursor()            // undefined on the first run: the whole list
for (;;) {
  const page = await call({ after, limit: 500 })
  for (const s of page.suppressions) {
    if (s.source === `app:${MY_APP_ID}`) continue           // your own write coming back
    if (await alreadyApplied(s.event_id)) continue          // event_id, not id, is the idempotency key
    await applyToContact(s.email, s.outcome, s.reason)
  }
  after = page.next
  await saveCursor(after)                 // after applying: a crash re-reads, never skips
  if (!page.has_more) break
}
// Run again every few minutes from the saved cursor. An empty page still returns `next`.
// Once a day, also run it from no cursor to the end (skipping event_ids already applied): a change that
// took long to save can, rarely, land just behind a cursor already handed out.
```

## Suppressing from the app

`{ "action": "journeys_suppress", "data": { "addresses": [{ "email", "note?" }], "note": "why" } }`, with the
app's own context key (a person's token or the old env key is refused).

```js
const res = await call('journeys_suppress', { addresses: batch.map((email) => ({ email })), note: 'opted out in the CRM' })
if (res.status === 429 || res.status === 503) return retryLater(batch, Number(res.headers.get('Retry-After')) || 30) // nothing was written
// 200: { results: [{ email, status: created|unchanged|resuppressed|invalid, id?, outcome? }], counts }
```

- **Up to 500 addresses a call, 5,000 per app per hour.** Over either, the whole call is refused and nothing is
  written. Retry the same batch; sending it twice is safe (the second time it is all `unchanged`).
- **Every address needs a note**, its own or the call's (up to 200 characters). No note is a 400 `note_required`.
- **`unchanged` means something already stands.** `outcome` says what: an existing complaint stays a complaint;
  you cannot overwrite it.
- **Tell the admin what the write grant allows before they add it.** The results show whether any address
  was already suppressed and why (a complaint included), even without the read grant, and submitting an
  address suppresses it. It can also suppress again an address an admin reinstated (Journeys keeps the
  history). Capped at 5,000 an hour.
- **Your writes come back in the feed** with `source: app:<your app id>`. Skip them there.

## Traps

- **`id` repeats; `event_id` does not.** An address that is reinstated and then suppressed again comes
  round again with the same `id`. Deduplicate on `event_id`.
- **The feed is the latest state per address, not a history.** If an address changed twice between
  two polls you see only the second change. Apply `outcome` as a state, not as a delta.
- **`occurred_at` is not the feed order.** A bounce can be reported hours late; it still arrives after
  your cursor, because the feed is ordered by when Journeys recorded it.
- **New changes take a minute to appear** (`settle_seconds: 60`). That is deliberate; do not poll faster
  to "catch" them, and do not use the feed as a send gate. Journeys already refuses to mail a suppressed
  address.
- **A 403 with `code: "missing_grant"` is not a bug in your app.** The grant was never added or has been
  removed. The body names the grant to ask for.

## Outcomes

| `outcome` | `reason` | Meaning |
|---|---|---|
| `unsubscribed` | `unsubscribe` | Unsubscribe link |
| `hard_bounced` | `bounce_hard` | Permanent bounce |
| `soft_bounce_suppressed` | `bounce_soft` | Repeated temporary bounces past Journeys' threshold |
| `complained` | `complaint` | Marked as spam |
| `suppressed` | `manual` | An app or a person suppressed it |
| `reinstated` | the reason it had been suppressed for | May be mailed again |

**Not in the feed:** the campaign (`journey_id`) for entries recorded before November 2026 (it is
`null`); a bounce below the threshold; opens and clicks.
