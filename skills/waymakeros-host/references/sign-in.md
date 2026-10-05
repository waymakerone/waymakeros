# End-user sign-in (database sign-in)

Your app's own users signing in to the app you built. They never need a Waymaker account. This is
**not** Waymaker sign-in: if the app's users are your colleagues, put the app in Commander
workspaces instead.

Read [the RLS section in SKILL.md](../SKILL.md#if-you-enable-database-sign-in) first. Turning
sign-in on changes who can read your tables.

## Turning it on

```bash
waymaker host db create <app> --region <region>   # if the app has no database yet
waymaker host db connect <app>
waymaker host db auth provision <app>              # installs the sign-in tables
waymaker host db auth deploy <app>                 # starts the sign-in service, prints its address
# then commit the migrations that use wm_user_id(), push, and release (SKILL.md, "Releases")
```

- **`deploy` refuses until `provision` has run.** Both are safe to run again.
- **The sign-in service is live when `deploy` returns. The app is not.** `deploy` puts two new
  variables into the app's environment (`WAYMAKER_DATABASE_AUTHENTICATED_URL`,
  `WAYMAKER_AUTH_JWKS_URL`), and the app only sees them after its next production build. Make sure
  one happens after `auth deploy`, and check the app's deployments list for it.
- **If `deploy` says the key endpoint has not propagated (503, retryable), wait a minute and run it
  again.** It is idempotent.
- **Read the sign-in address from `deploy`'s output.** It is a separate host from your app, with an
  organisation-specific suffix. Do not build it from the app's URL: the two can use different
  organisation names. Health check: `<sign-in address>/api/auth/ok`.
- `waymaker host db auth status <app>` shows whether it is live. `waymaker host doctor <app>`
  reports it too, and says if the sign-in schema is out of date (re-run `auth provision`).

`<app>` is the app's id or its slug.

## Methods

Two methods exist. Nothing else is available: no magic links, passkeys, social or phone sign-in.

| Method | State |
|---|---|
| Email + password | Always on |
| Emailed one-time code (6 digits, valid 10 minutes) | Off until you turn it on |

```bash
waymaker host db auth methods <app>              # show
waymaker host db auth methods <app> --otp on     # set
waymaker host db auth deploy <app>               # REQUIRED: a method change does nothing until deploy
```

`methods --otp on` prints "Not in effect yet". Believe it: until you redeploy, the one-time-code
routes do not exist and return 404.

**Sign-up is open.** Anyone who can reach the sign-in address can create an account. With codes on,
signing in with a code for an unknown email creates the account. Gate what an account can *see*
with row-level security, not with the sign-up form.

## The routes your app calls

Everything is under `<sign-in address>/api/auth/`. Call it from the browser with credentials
included. Test from a browser, not only `curl`: `curl` sends no `Origin` header, so an origin your
sign-in service does not accept passes in a script and fails for real users.

| Purpose | Route | Body |
|---|---|---|
| Sign up | `POST /api/auth/sign-up/email` | `{name, email, password}` |
| Sign in | `POST /api/auth/sign-in/email` | `{email, password}` |
| Sign out | `POST /api/auth/sign-out` | |
| Current session | `GET /api/auth/get-session` | |
| Token for your database | `GET /api/auth/token` | (uses the session cookie) |
| Health | `GET /api/auth/ok` | |

Your app's server passes that token to `forUser(token)` from `@waymakeros/db`. That connection runs
as the signed-in user, so your row-level security applies. The default `sql`/`db` export connects
as the database owner and **bypasses** your rules: use it only for work the app does on its own
behalf.

### Password reset: two flows, pick by method

**There is no `POST /api/auth/forget-password`.** It returns 404. Use one of these:

**With one-time codes on** (recommended: the user gets a 6-digit code):

```
POST /api/auth/email-otp/request-password-reset   {email}
POST /api/auth/email-otp/reset-password           {email, otp, password}
```

`POST /api/auth/forget-password/email-otp {email}` also works as an older name for the first step.

**With codes off:**

```
POST /api/auth/request-password-reset   {email}
POST /api/auth/reset-password           {newPassword, token}
```

With codes off, the email contains a **long token, not a link** and not a short code. Your app needs
a page with a field the user can paste the token into. The token is valid for an hour, though the
email text says it expires sooner.

**A reset request for an unknown email still succeeds** and sends nothing. That is deliberate, so
attackers cannot discover accounts. It means a 200 is not evidence an email went out. Test with an
address you can read.

### Signing in with a code (codes on)

```
POST /api/auth/email-otp/send-verification-otp   {email, type: "sign-in"}
POST /api/auth/sign-in/email-otp                 {email, otp}
```

### Email verification

**Not required.** Accounts can sign in without verifying. Link-based verification is not available
(`/send-verification-email` refuses). With codes on, you can verify with a code:

```
POST /api/auth/email-otp/send-verification-otp   {email, type: "email-verification"}
POST /api/auth/email-otp/verify-email            {email, otp}
```

Signing in with a code, or resetting with a code, also marks the email verified.

A code allows 3 wrong attempts. Then request a new one.

## Second factor (authenticator app)

The second factor is an authenticator app (TOTP), plus 10 single-use backup codes. There is no
email or SMS second factor.

```
POST /api/auth/two-factor/enable                  {password, issuer: "<Your app name>"}
POST /api/auth/two-factor/verify-totp             {code}       # confirms enrolment, and is the 2nd sign-in step
POST /api/auth/two-factor/verify-backup-code      {code}
POST /api/auth/two-factor/generate-backup-codes   {password}
POST /api/auth/two-factor/disable                 {password}
```

- **Pass `issuer` with your app's name** on `enable`. Without it, the user's authenticator app shows
  a generic name and they cannot tell which account it is.
- With a second factor on, password sign-in (`sign-in/email`) answers with
  `twoFactorRedirect: true` and **no session**: show the code prompt, then `verify-totp`.
- **The second factor applies to password sign-in.** If an app needs a second factor, keep
  one-time-code sign-in off (`methods --otp off`), and ask Waymaker before combining the two.
- Show the backup codes once and tell the user to keep them. They are what makes a lost phone
  self-service.

## When a user is locked out

```bash
waymaker host db auth reset-user <app> --email them@example.com    # or --user-id <id>
```

It does exactly two things: **removes their second factor** and **ends every session they have**.
It records who did it (`waymaker host db auth audit <app>`). Only the database's creator can run
it; anyone else is refused.

- **It does not change their password.** There is no command, tool or route that lets you set an
  end user's password. A user who forgot their password uses the reset flow above, and nobody else
  can do it for them.
- Do not write to the sign-in tables with SQL to get around this. Those tables are the platform's.

## Where the emails come from

Sign-in codes, reset and verification emails are sent **by the platform, from its own sender, with
your app's name as the display name** (subjects such as "Reset your <App> password").

- They **do not use your app's sending domain**, and they need **no DNS work by anyone**. They work
  the moment `auth deploy` succeeds.
- So never tell a client's IT team that DNS records are needed for sign-in or password-reset email.
  If reset emails are not arriving, check the address, the spam folder and the route your app calls
  first, then ask Waymaker.
- A failed send is an error, not a false "sent".

App email (receipts, notifications) is different and does use a sending domain. See
[email.md](email.md).

## Changes that need `auth deploy` again

Each of these saves and reports success, then does nothing until the next `auth deploy`:

- `methods --otp on|off`
- adding or changing the app's custom domain (the sign-in service only accepts requests from the
  app's addresses as of its last deploy, so a new domain's sign-in requests are refused until then)

## Not covered here

Turning sign-in off, and requiring a second factor for particular roles, are not documented in this
playbook yet. Ask Waymaker before using either.
