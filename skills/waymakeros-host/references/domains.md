# Custom domains for an app

Putting an app on the client's own address (`app.example.com` or `example.com`).

## Order

```bash
waymaker host apps domain setup <app-id> <domain>    # records the domain, prints the DNS records; routes nothing yet
waymaker host apps domain dns <app-id>               # prints the records again
# ... the domain's owner adds the records at their DNS provider ...
waymaker host apps domain verify <app-id>            # checks, SAVES the result, and switches the domain on
waymaker host apps domain status <app-id>
waymaker host apps domain provision <app-id>         # retry the certificate if it is stuck
waymaker host apps domain remove <app-id> --confirm
```

The MCP tools are `host_domain_setup`, `host_domain_verify`, `host_domain_status`,
`host_domain_remove` and `host_domain_list` (every domain in the organisation).

## What the owner adds

`setup` prints the exact records. Copy them from there; never retype them from memory or from
another app.

- **A subdomain** (`app.example.com`) gets a **CNAME** to the address `setup` prints, plus a **TXT**
  record that proves ownership and a CNAME for the certificate.
- **An apex domain** (`example.com`) cannot hold a CNAME, so it gets **A records** to the addresses
  `setup` prints, plus a `www` CNAME, ownership TXT records and certificate CNAMEs. The `www`
  variant is added for you; `setup` refuses a `www.` domain on its own.
- **Nothing is served until ownership is proven.** The TXT record is not optional.
- DNS usually propagates within minutes but can take up to 48 hours. The certificate usually takes
  1 to 5 minutes after ownership is proven, and renews itself.

The same rule applies here as for email: when a client's IT team will make the change, send them
exactly what `setup` printed for that domain, and nothing else.

## Traps

**Run `verify`, not just `status`.** `verify` is what writes the result back and switches the
domain on: it is the only step that creates the route. `status` mostly reads the saved state. A
domain whose DNS is correct but which never had `verify` run stays `pending_dns` or `pending_ssl`
for ever. After DNS changes, run `verify`; if it says `pending_ssl`, wait a few minutes and run it
again. `verify` can be called about twice a minute.

**`status` does not notice a domain breaking.** It never moves a domain from `active` back down. If
the owner later removes the DNS records, `status` still says `active`. Check the address itself.

**A domain belongs to one app.** `setup` refuses a domain another app uses. Running `setup` again
for the same app replaces its earlier pending domain.

**Sign-in needs a redeploy after a domain change.** If the app uses end-user sign-in, run
`waymaker host db auth deploy <app>` after the domain goes `active`. Until then, sign-in requests
from the new address are refused. See [sign-in.md](sign-in.md).

**Under the `release` policy, a release also checks each domain** in the Solution: its DNS and
certificate must be live, and it must answer. An unverified domain refuses the plan.
