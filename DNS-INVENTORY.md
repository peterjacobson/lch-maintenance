# DNS inventory & migration — livingcomposthubs.org.nz

Captured 2026-06-21 from the **live** zone while it was still authoritative on the
old (inaccessible) Cloudflare account. This is the backup for rebuilding the zone
on a new Cloudflare account. All values below are public DNS data.

## Current state (for rollback reference)

- **Registrar:** Porkbun
- **Old nameservers:** `chin.ns.cloudflare.com`, `hunts.ns.cloudflare.com`
  (old Cloudflare account — no access)
- **Email:** Zoho Mail (AU) — live, must not break
- **cdn:** `Cloudflare → CloudFront → S3` (bucket `living-compost-hubs`, ap-southeast-2)

## Complete record inventory

> ⚠️ Cloudflare's auto-scan when you add the site will import the **proxied** apex/www/app/cdn
> records as literal Cloudflare anycast IPs (`104.21.90.90` / `172.67.198.88`) — that's garbage,
> delete them. It may also miss the DKIM TXT. Reconcile everything against this table.

### Email — recreate EXACTLY (DNS only / grey cloud)

| Type | Name | Value | Priority |
|------|------|-------|----------|
| MX | `@` | `mx.zoho.com.au` | 10 |
| MX | `@` | `mx2.zoho.com.au` | 20 |
| MX | `@` | `mx3.zoho.com.au` | 50 |
| TXT | `@` | `v=spf1 include:zohomail.com.au include:_spf.google.com -all` | — |
| TXT | `@` | `zoho-verification=zb05777276.zmverify.zoho.com.au` | — |
| TXT | `@` | `google-site-verification=ukrcGJNYu0LYvoY2duKFxtf5EmNTBrNnMxIhZ6OSR60` | — |
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:75894eabed804e67a0563d226a6eea23@dmarc-reports.cloudflare.net` | — |
| TXT | `zmail._domainkey` | (Zoho DKIM, below) | — |

Zoho DKIM TXT value for `zmail._domainkey`:

```
v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCKSvgeQwsITczIl4KTv4IWuWnckLf76OH+MvZ/MFv7recMULXOTVmqeDdudZjaeruza911JhPRZzzw3kkCyUqlh/+/W7ptZp41PR+r2eOROUopgI4R6jCLp+2hbvr2xRGcVvfHhU90Ogn6q6p8au05AIoJfCzUijVXOc08PlJyAQIDAQAB
```

> The DMARC `rua` mailbox belongs to the **old** Cloudflare account's DMARC Management. It keeps
> working (just an inbox), but the old account owner receives the reports. Optional: repoint later.

### Web — new values (replacing the down app)

| Type | Name | Value | Proxy | Note |
|------|------|-------|-------|------|
| CNAME | `@` | `peterjacobson.github.io` | **DNS only** | Maintenance page (Cloudflare flattens apex CNAME) |
| CNAME | `www` | `peterjacobson.github.io` | **DNS only** | Maintenance page |
| CNAME | `cdn` | `<CloudFront-dist>.cloudfront.net` | DNS only* | **Get dist domain from AWS CloudFront console** |
| — | `app` | (decide — see below) | | Old app subdomain |

\* If CloudFront already lists `cdn.livingcomposthubs.org.nz` as an alternate domain name with a
valid ACM cert (it was serving HTTPS), a **DNS-only** CNAME straight to the CloudFront domain
works and drops the redundant Cloudflare layer. If not, set it **proxied (orange)** to keep
Cloudflare terminating TLS as before.

**apex/www must be DNS only (grey)** so GitHub Pages can provision its Let's Encrypt cert.
Proxying (orange) breaks the cert challenge and risks redirect loops.

**`app` subdomain:** currently a separate proxied host (the old app). It will NOT show the GitHub
Pages site via a plain CNAME (Pages only serves the domain in its `CNAME` file → 404). Either:
- leave it out (it goes dark — fine if unused), or
- add a Cloudflare **Redirect Rule**: `app.livingcomposthubs.org.nz/*` → `https://livingcomposthubs.org.nz/$1` (302), with `app` proxied (orange).

## Migration procedure

1. **New Cloudflare account** → Add a site → `livingcomposthubs.org.nz` → Free plan.
2. **Recreate records** from the table above. Email exactly; delete any bogus anycast-IP A records
   the scan imported; set apex/www to GitHub Pages (DNS only); handle `cdn` and `app`.
3. **Copy the new account's assigned nameservers** (shown on the "complete your setup" screen —
   a unique pair like `xxx.ns.cloudflare.com` / `yyy.ns.cloudflare.com`).
4. **Porkbun** → domain → **Authoritative Nameservers** → replace `chin`/`hunts.ns.cloudflare.com`
   with the new pair → save.
5. **Wait & verify** (below). Cloudflare emails you when the zone is active. Then in the GitHub repo
   → Settings → Pages, confirm the domain goes green and tick **Enforce HTTPS** once the cert issues.

## Verify after cutover

```bash
dig +short livingcomposthubs.org.nz NS        # → new Cloudflare pair
dig +short livingcomposthubs.org.nz           # → 185.199.108–111.153 (GitHub Pages)
dig +short livingcomposthubs.org.nz MX        # → mx.zoho.com.au etc. (email intact)
dig +short zmail._domainkey.livingcomposthubs.org.nz TXT   # → DKIM intact
curl -sI https://livingcomposthubs.org.nz | head -1
```

## Rollback

The safest rollback is this inventory (re-enter records anywhere). NS-flip-back to the old
account is unreliable — Cloudflare only keeps a zone active on one account at a time.
