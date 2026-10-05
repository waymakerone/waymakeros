# Storage from an app

Host Storage is buckets of objects (photos, uploads, exports) that belong to a Solution. Objects are
not rows in your database; store the object's **key** in a table and keep the bytes in the bucket.

## The route from an app to a bucket

**An app cannot write to a bucket itself.** Writing, and reading a `members` bucket, go through an
**Ambassador** that holds a grant and uses `ctx.storage`:

```
browser → your app's server route → your Ambassador (ctx.storage) → the bucket
```

`storage grant add <bucket> --app <id>` succeeds and says the app can read and write, but **no app
code path uses an app grant today.** Grant the Ambassador, not the app.

The one exception is reading: a bucket set to `app_users` or `anyone` can be read directly at its
address (below).

How the app calls its Ambassador, and how to secure that call, is in
[ambassadors.md](ambassadors.md#calling-an-ambassador-from-your-app).

## Setting up

```bash
# 1. The bucket. --solution or --app is required: a bucket always belongs to a Solution.
waymaker storage create <bucket> --solution <slug>        # private by default

# 2. Let the Ambassador use it. reader = get/list/signedUrl; writer = also put/delete. Default: writer.
waymaker storage grant add <bucket> --ambassador <ambassador-id> --role writer
waymaker storage grant list <bucket>
```

Bucket names belong to the organisation. A bucket another organisation already uses can't be
yours.

## Who can read it: three access modes

```bash
waymaker storage access <bucket> members|app-users|anyone [--app <app-id>]
```

| Mode | Who reads | How |
|---|---|---|
| `members` (default) | People you add with `storage member add`, and Ambassadors with a grant | Signed links only. Nothing is served at an address. |
| `app-users` | Anyone signed in to the app named by `--app` | `GET <bucket address>/<key>` with the end user's token as `Authorization: Bearer …` |
| `anyone` | The whole internet | The same address, no sign-in |

- **The bucket address is printed as `URL` by `storage access`** when a bucket becomes served.
  Use it exactly as printed (or a custom domain attached with `storage domain attach`). Never
  build it yourself: the organisation name in it is not always the one you expect.
- **`app-users` needs the app's end-user sign-in turned on** ([sign-in.md](sign-in.md)), or it is
  refused. The token goes in the `Authorization` header: **cookies are not accepted, so a plain
  `<img src>` does not work.** Fetch the object with the header, or have the Ambassador mint a
  signed link.
- **An access change takes about a minute to apply, in both directions.** Wait before you test,
  and do not "fix" a change that has not landed yet.
- **Making a bucket public is a deliberate act.** `storage create --public` succeeds even when
  serving it could not be switched on; read the response for a `warning`. And a revoked `anyone`
  object can stay in browser and network caches for a few minutes after the change applies.
- People: `storage member add <bucket> <user-id> --role reader|writer|manager`. Being in the
  organisation grants nothing by itself, and organisation admins can see a bucket's settings but
  not its contents.

## `ctx.storage` in an Ambassador

```ts
export default async function handler(request: Request, ctx) {
  const photos = ctx.storage.bucket('site-photos')

  await photos.put('jobs/123/before.jpg', await request.arrayBuffer(), { contentType: 'image/jpeg' })
  const res = await photos.get('jobs/123/before.jpg')       // a Response, or null if missing OR not granted
  const page = await photos.list({ prefix: 'jobs/123/', limit: 100 })   // { objects, truncated, next_token }
  const link = await photos.signedUrl('jobs/123/before.jpg', { expiresIn: 900 })
  await photos.delete('jobs/123/before.jpg')

  return Response.json({ link })
}
```

- **`get` returns `null` for a missing object, and throws when the Ambassador has no grant on the
  bucket**, with a message naming `grant add`. A `put`, `list` or `signedUrl` without a grant
  throws the same way. Catch errors separately from checking for `null`.
- **`put` reserves space before it uploads.** If the organisation's storage is full, it fails
  before writing anything.
- Grants and removals take effect on the Ambassador's next call, with no redeploy.

## Signed links

A signed link reads one object and expires. **Default 900 seconds, maximum 3600.** Give a browser a
signed link instead of making the bucket public.

- From an Ambassador: `bucket.signedUrl(key, { expiresIn, download })`.
- From a terminal: `waymaker storage url <bucket> <key> [--expires <s>] [--download]`. This needs
  you to be a member of the bucket.
- Do not store a signed link in your database. Store the key, and mint a link when it is needed.

HTML, XHTML and SVG objects are always served as downloads, and installer file types are refused.

## From a terminal

```bash
waymaker storage ls [bucket] [--prefix p/]
waymaker storage put <bucket> <file> [key]
waymaker storage rm <bucket> <key>
waymaker storage rmbucket <bucket> [--delete-objects]     # --delete-objects is not reversible
```
