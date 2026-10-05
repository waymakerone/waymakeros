# App email (transactional)

Email your app sends **as itself**: a receipt, a booking confirmation, a "your report is ready".
It is sent from a route in your app (or an Ambassador) through Host, and it is metered to the app's
Solution.

**It is not sign-in email.** Sign-in codes, password reset and email verification come from the
platform's own sender and need nothing from you or anyone's DNS. See
[sign-in.md](sign-in.md#where-the-emails-come-from). If the job is "make password reset work", you
do not need anything on this page.

## Before you touch DNS: read this

> **A change is in progress** that gives every app a Waymaker-owned default sending domain that
> needs no DNS work at all. Until it ships, **do not send DNS records to a client's IT team, or ask
> them to change DNS, without asking Waymaker first.** A DNS request to a client's IT is slow,
> visible and hard to take back, and it may soon be unnecessary.
>
> When you draft anything for a client about email, never paste raw records from command output
> into it. Ask Waymaker what the client needs to do.

## The model

- **One sending domain per app.** An app has at most one, and a domain belongs to one app. A second
  `register` for the same app is refused, and there is no command to remove or swap a registered
  domain. Choose it deliberately; ask Waymaker if you need to change it.
- **No verified domain, no send.** `send` refuses with "this app has no sending domain" or "the
  sending domain … is not verified yet". There is no sandbox or test mode.
- **You choose only the part before the `@`.** The address is always `<local-part>@<the app's
  sending domain>` (default local part `noreply`).
- **It is a paid capability, metered per send.** It is charged to the Solution's owner only when
  the sending domain is attached to the Solution. Otherwise it is charged to whoever sends.
  Registering a domain does not attach it, and neither the CLI nor the MCP can attach this kind
  yet, so ask Waymaker to attach it for a client's Solution. Never quote a price; rates are at
  https://waymakeros.com/pricing.

## Commands

```bash
waymaker host email domain register <app-id> <domain>   # prints the domain id, status and DNS records
waymaker host email domain verify <domain-id>           # re-checks DNS and SAVES the result
waymaker host email domain get <domain-id>              # reads the saved status only
waymaker host email domain list [--app <app-id>]
waymaker host email send <app-id> --to <addr> --subject "<s>" --text "<t>" [--html "<h>"] [--from <local-part>] [--reply-to <addr>]
waymaker host email status <send-id>
waymaker host email quota [--app <app-id>]
```

All take `--json`. The MCP tools are `host_email_*`.

## Traps

**`domain get` and `domain list` lag.** They show the status saved at the last `verify`. Nothing
re-checks in the background, so a domain whose DNS is correct shows `pending` until someone runs
`verify`. Run `verify`, not `get`, after DNS changes. Statuses are `pending`, `verified` and
`failed`.

**`status` means "accepted for delivery", not "delivered".** A send is `sent` or `failed`. There is
no delivered, bounced or opened state. A bounce after acceptance is still `sent`, and still charged.

**One recipient per send.** No cc, bcc, attachments or templates. Build the HTML in your app.

**A failed send says little.** It returns a generic failure. Check the domain is verified and the
local part is plain (`a-z`, `0-9`, `.`, `_`, `%`, `+`, `-`).

**`quota` is informational.** It reports a per-second rate and sends in the last 30 days. Send at a
steady rate, not in bursts.

## Sending from the app

Send from your app's **server** (a route), never from the browser. A confirmation email is a route
that calls Host transactional email. It does not need an Ambassador.

**The app has no email credential of its own yet.** A send is authorised as a Waymaker user or API
key in the organisation, the same as the CLI. Sending from app code therefore means holding a
Waymaker API key in the app's server-side environment. Before you do that in a client's app, ask
Waymaker which key and endpoint to use.
