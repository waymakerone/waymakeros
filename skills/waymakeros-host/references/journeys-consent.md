# Journeys consent: keeping an app's opt-outs in step

**Setting it up from nothing?** The worked example is
https://docs.waymakerone.com/host/journeys-connect-your-app. The order is: canvas (copy its id) → context key
(`waymaker host apps context-key create <app-id>`, then deploy) → an org admin adds the grants → pull, suppress,
upsert/archive, reinstate → a scheduled Ambassador (`--schedule "*/15 * * * *"`) calls a secret-guarded sync route in
the app. Every call is `Authorization: Bearer $WAYMAKER_CONTEXT_KEY`; `X-API-Key` + `X-App-Id` is the deprecated older
path, and every Journeys write refuses it.

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
- **These grants only suppress.** A person who wants mail again opts back in through a Journeys form, or an admin
  reinstates them in Journeys. If they opt back in through the app, see "Reinstating consent" below: its own grant.

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

## Being pushed changes

Subscribe the app (org admin only) with MCP `host_app_event_subscribe`
`{ app_id, source: "journeys", url, event_types: ["suppression.changed"] }`. The app needs only
`journeys:suppressions:read`. The secret is shown once: store it with `waymaker host apps env set` at once.
Each change arrives as a signed POST whose body is the feed's entry (`id`, `email`, `outcome`, `reason`,
`source`, `journey_id`, `occurred_at`), verified as shared inbox events are.

- **Keep the daily poll.** Push is at least once, unordered, kept 7 days. The feed is the record.
- **Your own suppressions are not pushed back to you**; reinstatements always are, your app's own included.
- **Other subscribers see the writer**: `source` is `app:<app id>` of the app that wrote the change.
- **Apply as state**, newest `occurred_at` per `id` wins. Push and feed `event_id`s differ.
- `waymaker host apps events list <app-id>` shows `org_enqueue_failures_7d`; above 0, read the feed.

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

## Putting the app's people on a canvas

Full reference: https://docs.waymakerone.com/host/journeys-audience

- **Creating the canvas and finding its id:** in Commander, Journeys → new canvas → **Canvas Settings** shows
  **Canvas ID** and copies the grant name. Or MCP `commander_journey_canvas_create` / `commander_journey_canvas_list`
  (`waymaker journeys canvases create|list` from the next CLI release). One
  canvas per app: an app only ever changes the memberships it created.

- **The grant is per canvas: `journeys:audience:write:<canvas id>`**, added by an org admin, never through `*`.
- **The context key only**, `journeys_audience_upsert` / `journeys_audience_archive` with
  `{ canvas_id, members: [...] }`; each upsert member needs `email` and `lawful_basis` (not `none`;
  `legitimate_interest_b2b` needs `contact_type: "business"`).
- **Your own memberships only.** Someone else's comes back `not_owned` and is untouched. Your own answer the same
  (`created`, `updated`, `unchanged`, `archived`) whatever their mail state; an opt-out or a bounce is never lifted.
  Archive is never an unsubscribe. To learn who opted out, read the suppression list (its own grant).
- **A membership is not permission to mail.** Each send checks the basis again, the person's own opt-out and the
  suppression list. For someone who already opted out, `created` records a membership that is not mailable. Your
  results are identical whether or not the person unsubscribed, bounced or is suppressed; for mail state, read the
  suppression list.
- 500 a call, 5,000 per app per hour; a 429 has `Retry-After` and wrote nothing.

## Reinstating consent

Full reference: https://docs.waymakerone.com/host/journeys-consent

- **Only for a person who opted back in through the app, expressly.** `journeys_consent_reinstate` with
  `{ members: [{ email, basis: "express_opt_in", consented_at, note }] }`, the context key only. The grant is
  `journeys:consent:reinstate`, by name, org admin only, never through `*`; ask the admin, do not work around it.
- **The person must be on a canvas the app writes** (`journeys:audience:write:<canvas id>`). That guards against
  mistakes, not abuse: an app with both grants can reinstate any eligible address it presents newer consent for, which
  is why both are admin-only. Only reinstate people who consented to you.
- **`not_lifted` is one bare answer** for a stale consent, a complaint, a hard bounce, someone else's manual
  suppression or a person not on your canvas. Do not retry it, and do not try to tell the cases apart.
- **`consented_at` is when the person consented, not when you call**: ISO 8601 with an offset or `Z`. It must be later
  than their latest opt-out, reinstatement and your last recorded consent for them. Never "fix" a `not_lifted` by
  sending the current time: that records a consent the person did not give at that moment.
- **The note is required** (200 characters, no address): how they opted in. It is kept in Journeys' history, not in
  the feed or pushed events.
- `unchanged` means nothing to lift. A retry after `reinstated` answers it, or `not_lifted` if they opted out again
  meanwhile. 100 a call, 1,000 per app per hour.
