# Ambassadors

An Ambassador is a serverless function that runs **without your app being asked**: on a schedule,
retried, long-running, or holding a capability the app does not have (such as writing to a storage
bucket). If the job happens inside a request the app already handles, it is a route in the app,
not an Ambassador.

## Create and deploy

```bash
waymaker host ambassadors create <name> \
  --repo https://github.com/<owner>/<repo> \   # no trailing .git
  --branch main \
  --path index.ts \                             # the file that exports the handler
  [--schedule "0 9 * * mon-fri"] \
  [--auth required]                             # required (default), api-key or public; see below

waymaker host ambassadors deploy <id>           # builds from the repo; returns at once
waymaker host ambassadors status <id>           # poll until Status is active (or error)
```

The handler is the file's default export:

```ts
export default async function handler(request: Request, ctx) {
  // ctx.env       your environment variables
  // ctx.storage   buckets you have granted it (storage.md)
  return Response.json({ ok: true })
}
```

- **Deploy builds from the repo, never from your disk.** Push first. A push to the configured
  branch also deploys it automatically, unless the Solution uses the `release` policy (below).
- **`deploy` answers "building" before anything is built.** It reports success when the build has
  been *started*. Poll `ambassadors status` until it reads `active` or `error`. The MCP tool
  `host_ambassador_deployment_logs` shows the build's steps and its last log lines; use it when a
  deploy ends in `error`.
- **Under the `release` policy** (the default for new Solutions), a push does not deploy an
  Ambassador and `ambassadors deploy` is refused. The Ambassador changes only through a release.
  See "Releases" in [SKILL.md](../SKILL.md#releases).

### Two deploy behaviours that look like your fault and are not

- **Overlapping deploys make the status flap.** If you deploy while another deploy of the same
  Ambassador is in flight (a push you just made counts), `status` can read `error` for a minute or
  two and then settle on `active`. **Wait for the in-flight deploy to finish before starting
  another, and read `status` again before you debug.**
- **A first deploy that fails at a platform step is not your code.** If a new Ambassador's first
  deploy ends in `error` at a step you don't control (setting up its environment, for example),
  deploy it once more before changing any code. If it fails the same way again, ask Waymaker.

## Who can call it: auth mode and the invoke key

Every Ambassador has an auth mode. Set it at create (`--auth`, or `auth_mode` on
`host_ambassador_create`). Leave it out and you get `required`.

| Mode | An HTTP request to its address |
|---|---|
| `required` (the default) | Must carry the Ambassador's **invoke key** |
| `api-key` | Same: must carry the invoke key |
| `public` | Runs for anyone who has the address |

- **Send the key as `X-API-Key: <key>` or `Authorization: Bearer <key>`.** Use one, not both: if
  `X-API-Key` is present, that is the one checked. A request without a valid key gets
  `401 {"error":"Unauthorized"}`, your handler never runs, and **nothing appears in
  `ambassadors logs`**. A 401 with an empty log means the caller's key is missing or wrong.
- **The key is removed from the request before your handler sees it.** Don't look for it in the
  handler, and don't expect it in your own logs.
- **Scheduled runs never need the key.** When you invoke it by hand, you don't send it either:
  `waymaker host ambassadors invoke`, `host_ambassador_invoke` and the console's **Trigger** send it
  for you.
- **Getting the key is a person's job.** Someone with deploy access opens the Ambassador in the
  Host console and clicks **Show invoke key** (under Auth Mode, on its Overview). The CLI and the
  MCP tools do not return it, by design: it is a standing credential. As an agent, ask the person
  to put it into the calling app's environment variable themselves, not to paste it into the chat.
- **The key does not change between deploys**, so it is set once per caller. It can't be rotated
  yet. Keep it in server-side environment variables only (never in browser code or the repo), and
  if it may have leaked, ask Waymaker.
- **A mode change takes effect on the next deploy** (or release). The Ambassador enforces the mode
  it was last deployed with. Change the mode in the console (the Ambassador's Settings); the CLI
  sets it only at create.
- **Still check the caller in your handler.** The key decides who reaches the handler; your own
  check (below) is the second lock, and the only one on a `public` Ambassador.

## Environment variables

```bash
waymaker host ambassadors env list <id>
waymaker host ambassadors env set <id> KEY=value
waymaker host ambassadors env delete <id> KEY
```

- **A change does nothing until the next deploy.** `env set` succeeds immediately; the running
  Ambassador keeps the old values until it is redeployed (or released).
- **Names starting with `_` are reserved for the platform** and are refused. The same rule applies
  to app environment variables.
- Up to 50 variables; values up to 8 KB. Read them in the handler as `ctx.env.KEY`.

## Schedules

Write a schedule as **standard five-field cron**. Day-of-week numbers mean what they mean
everywhere (`0` or `7` = Sunday, `1` = Monday) and are stored as names (`MON`). Names such as
`mon-fri` are the clearest to write. Non-standard extensions (`L`, `W`, `#`, `?`) are refused.

- **A schedule takes effect on the next deploy.** Setting it is not the same as it running.
- A scheduled run calls the same handler with a request carrying the header `X-Trigger: cron`. It
  needs no invoke key, whatever the auth mode.
- **"Did it run?" is answered by its logs**, not by the schedule shown back to you. A schedule
  displays faithfully whether or not it fires. If the logs are empty, invoke it once by hand and
  check that the manual run appears before concluding the schedule never fired.

## Logs

```bash
waymaker host ambassadors logs <id> [--limit 50]
```

Each run: trigger (`cron`, or `http` for every request including a manual invoke), status, duration
and error. Newest first. Build and
deploy history is separate: `host_ambassador_deployment_logs` (MCP).

## Invoking it by hand

```bash
waymaker host ambassadors invoke <id>     # POST to the Ambassador, prints its HTTP status and body
```

Use `waymaker host ambassadors invoke`, not the older `waymaker ambassador invoke`, which prints
"invoked" whatever the Ambassador answered. The MCP tool `host_ambassador_invoke` also takes a
`payload`, a `method` and `headers`. It refuses an Ambassador that is not `active`. Both send the
invoke key for you, so they work on a protected Ambassador. Your own `curl` to its address does
not: it gets a 401 unless you send the key.

## Calling an Ambassador from your app

The usual reason: the app needs to store or fetch a file ([storage.md](storage.md)).

```
browser → app server route → POST <Ambassador URL> with its invoke key + your secret → ctx.storage
```

1. **Get the Ambassador's address** from `waymaker host ambassadors status <id>` (`URL`). Nothing
   puts it into the app for you. Set it as an app environment variable, for example
   `PHOTOS_AMBASSADOR_URL`.
2. **Give the app the invoke key** (every mode except `public`). A person copies it from **Show
   invoke key** in the Host console and sets it as an app environment variable, for example
   `PHOTOS_AMBASSADOR_KEY`. See "Who can call it" above.
3. **Make a shared secret** as well: a long random value. Set the same value as an environment
   variable on the app *and* on the Ambassador, for example `PHOTOS_AMBASSADOR_SECRET`. Redeploy
   (or release) both.
4. **The app sends both** from its server, never from the browser:
   `X-API-Key: <invoke key>` and `X-App-Secret: <secret>`.
5. **The Ambassador checks the secret first**, on every request, and answers 401 when it is wrong.
   The key has already been checked by then; this is the handler's own check, kept as defence in
   depth:

```ts
export default async function handler(request: Request, ctx) {
  if (request.headers.get('x-app-secret') !== ctx.env.PHOTOS_AMBASSADOR_SECRET) {
    return new Response('Unauthorized', { status: 401 })
  }
  // ... ctx.storage work
}
```

- **Authenticate every request in the handler**, before anything else it does, unless the
  Ambassador is meant to be public. Keep the check on a protected Ambassador too.
- **Use a custom header for your secret, not `Authorization`.** `host_ambassador_invoke` cannot set
  `Authorization` or `X-API-Key` (it sends the invoke key itself), but it can send `X-App-Secret`
  in `headers`, so you can test the secured handler by hand.
- **If the app's call gets a 401 and nothing shows in `ambassadors logs`**, the invoke key is
  missing or wrong. Check the app's environment variable, and that the app was rebuilt after it
  was set. If the run *does* appear in the logs, the 401 came from your handler's own check.
- **If the app's call never reaches the Ambassador at all** (no response, nothing in the logs)
  while the same call works from your terminal, rebuild the app once (push, or release): apps built
  before early October 2026 could not call an Ambassador. Nothing in your code needs to change.
- **An Ambassador cannot yet call another Ambassador at its address.** Do that work in one
  Ambassador, or route it through the app.
