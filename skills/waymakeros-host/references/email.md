# App email (transactional)

This is email your app sends **as itself**: a receipt, a booking confirmation, a "your report is ready".
It goes from a route in your app (or an Ambassador) through Host, and it is metered to the app's
Solution.

**It is not sign-in email.** Sign-in codes, password reset and email verification come from the
platform's own sender and need nothing from you or anyone's DNS. See
[sign-in.md](sign-in.md#where-the-emails-come-from). If the job is "make password reset work", you
do not need anything on this page.

## Every app already has an address. No DNS.

An app sends from its own ready-made address:

```
noreply@<app>.<org>.waymakermail.com
```

Waymaker owns that domain and publishes its records. **Nothing is needed from you or from a client's
IT team.** The address is created on the app's **first real send** and is ready within a minute or
two. The sender's display name is the app's name.

- You choose only the part before the `@`: `--from receipts` sends as `receipts@<app>.<org>.waymakermail.com`.
- `waymaker host email domain register <app-id>`, with no domain, **shows** the address the app will
  use. It does not create anything.
- If the first send says the address "is still being set up", send again in a minute.

**Default to this address.** Don't propose a client's own domain unless the client asks for one.

## A client's own domain (optional)

If a client wants mail to come from their own domain (for example `mail.client.com`):

```bash
waymaker host email domain register <app-id> mail.client.com
```

Host returns a short list of **CNAME records only**, and every one points at a Waymaker hostname. The
client's IT adds them, then you run `verify`.

- **Paste exactly what Host shows.** One record name contains a fixed selector label that looks like
  a vendor word. It is required as written. Never rename, shorten or "tidy" a record name or value.
- Never paste records from anywhere other than Host's output, and never add records Host didn't show.
- Use a subdomain the client doesn't already send mail from (`mail.`, `appmail.`), so nothing clashes.
- If Host says the records are "still being prepared", read the domain again in a minute.

## The model

- **One sending address per app.** It is either the ready-made one or a custom domain. A second
  `register` for the same app is refused, and there is no command to swap it. Ask Waymaker if it has
  to change.
- **It is a paid capability, metered per accepted send.** It is charged to the Solution's owner when
  the sending domain is attached to the Solution, otherwise to whoever sends. Never quote a price;
  rates are at https://waymakeros.com/pricing. An account that cannot pay gets 402 on `send` and
  on `register`. That is a billing question for the account owner, not something to retry.
- **Transactional only.** Use it for mail the recipient expects. Newsletters and marketing belong in
  Journeys.

## Sending health: what can stop an app sending

Host tracks bounces and spam complaints **per app**. One app's bad list can't hurt another app's mail.

- **A new address starts with a daily limit that grows each day.** If `send` returns 429, the app
  has hit today's limit. Spread the sends out; the limit rises on its own.
- **People who bounced or complained aren't mailed again by that app.** A send to a suppressed
  recipient is refused with 422 and recorded. Nothing is sent and nothing is charged. Don't try to
  route around it.
- **A sustained spike in bounces or complaints stops the app's sending automatically.** `send` then
  returns 403 "Sending is paused". The organisation owner and Waymaker are emailed with the reason.
  **There is no self-service resume.** Fix the cause (who is being mailed, and what), then ask
  Waymaker support to resume it. Sending never resumes on its own.
- **"Sending capacity is full — contact Waymaker"** means Waymaker has to add capacity before a new
  address can be made. Nothing is wrong with the app. Tell the person to contact Waymaker. Don't retry
  in a loop.

Check it with:

```bash
waymaker host email health <app-id>      # per-day sends and outcomes, rates, paused or not and why, suppressed count
```

The MCP tool is `host_email_health`. Host's **Tools → Transactional Email** shows the same thing
under each address.

## Commands

```bash
waymaker host email send <app-id> --to <addr> --subject "<s>" --text "<t>" [--html "<h>"] [--from <local-part>] [--reply-to <addr>]
waymaker host email domain register <app-id> [<domain>]   # no domain: show the ready-made address; a domain: get its CNAMEs
waymaker host email domain verify <domain-id>             # re-checks a custom domain's DNS and SAVES the result
waymaker host email domain get <domain-id>                # reads the saved status only
waymaker host email domain list [--app <app-id>]
waymaker host email status <send-id>
waymaker host email health <app-id>
waymaker host email quota [--app <app-id>]
waymaker host email key create <app-id> [--env NAME | --show]   # the app's own sending key (see below)
waymaker host email key rotate <app-id> [--env NAME | --show]
waymaker host email key revoke <app-id> --confirm
waymaker host email key list <app-id>
```

All take `--json`. The MCP tools are `host_email_*`.

## Traps

**`domain get` and `domain list` lag for a custom domain.** They show the status saved at the last
`verify`. Run `verify`, not `get`, after the client's DNS changes.

**`status` means "accepted for delivery", not "delivered".** A send is `sent`, `failed` or
`suppressed`. Delivery outcomes (delivered, bounced, complaints) show in `health`, per day, not on
the send.

**No cc, bcc, attachments or templates.** Build the HTML in your app. Send to one person per call
unless the mail really is for several people.

**A failed send says little.** It returns a generic failure. Check the local part is plain (`a-z`,
`0-9`, `.`, `_`, `%`, `+`, `-`) and run `health` to see whether the app is paused.

## Sending from the app

Send from your app's **server** (a route), never from the browser. A confirmation email is a route
that calls Host transactional email. It does not need an Ambassador.

**Give the app its own sending key.** It can send this app's email, read this app's send status and
health, and do nothing else: not another app's mail, not any other Host or Commander action. Never
put a person's Waymaker API key in an app to send mail.

```bash
waymaker host email key create <app-id>     # puts it in the app's env as WAYMAKER_EMAIL_KEY, never shows it
waymaker host apps deploy <app-id>          # the app sees the variable after its next deploy
```

Then, from the app's server:

```
POST https://apps.waymakerone.com/functions/v1/host-transactional-email
Authorization: Bearer $WAYMAKER_EMAIL_KEY
Content-Type: application/json

{"action": "send", "data": {"to": "customer@example.com", "subject": "Your receipt", "text": "…", "html": "…", "from_local_part": "receipts"}}
```

`data.app_id` can be left out: the key decides the app. Naming any other app, or calling any other
action, answers 404.

- **Putting it in the env is the default and the safe path.** `--env NAME` picks another variable.
  Names starting with `_`, and browser-bundle prefixes such as `VITE_` or `NEXT_PUBLIC_`, are refused,
  because the key must stay on the server. `--show` prints it once instead (for a server hosted
  somewhere else); store it then, it is never shown again.
- **One key per app.** `rotate` replaces it and the old key stops working **at once**, so redeploy the
  app straight after. `revoke` stops it and removes it from the env.
- Creating, rotating and revoking need the owner or releaser role on the app's Solution (for an app
  in no Solution: its creator or an organisation admin).
- The MCP tools are `host_email_key_create`, `_rotate`, `_revoke` and `_list`. Through MCP the key
  always goes into the app's env and is never returned.
