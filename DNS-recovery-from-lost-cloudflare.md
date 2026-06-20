# DNS recovery — livingcomposthubs.org.nz (lost Cloudflare account)

**Action this runbook only if the original Cloudflare account cannot be recovered.**
It rebuilds the DNS zone on a **new Cloudflare account** and repoints the registrar
(Porkbun) to it, while preserving live email and pointing the web to the maintenance page.

Captured 2026-06-21 from the **live** zone while it was still authoritative on the old
account. All values below are public DNS data.

---

## Step 0 — first, try to regain the old account

Migrating is a one-way-ish move (see Rollback). Before doing it, exhaust recovery of the
account holding nameservers `chin.ns.cloudflare.com` / `hunts.ns.cloudflare.com`:

- Cloudflare login / password reset against any email plausibly used to sign it up
  (check Porkbun contact email, org shared inboxes, a former dev/contractor).
- Cloudflare support can help prove ownership (you control the registrar at Porkbun).
- Search inboxes for "Cloudflare" onboarding / billing mail to identify the account email.

If none of that works, proceed below.

---

## Current state (reference)

- **Registrar:** Porkbun
- **Old nameservers:** `chin.ns.cloudflare.com`, `hunts.ns.cloudflare.com` (inaccessible account)
- **Email:** Zoho Mail (AU) — **live, must not break**
- **`cdn.`:** `Cloudflare → CloudFront → S3` (bucket `living-compost-hubs`, ap-southeast-2)
- **Maintenance page:** already deployed — GitHub Pages repo `peterjacobson/lch-maintenance`,
  custom domain `livingcomposthubs.org.nz` already set in the repo's `CNAME`.

---

## The one hard dependency: `cdn`

`cdn.livingcomposthubs.org.nz` proxies through Cloudflare to a **CloudFront distribution**
(confirmed by `x-amz-cf-pop` / `x-amz-cf-id` headers; the S3 bucket itself has no website
config). To recreate `cdn` you need the **CloudFront distribution domain** (`dXXXX.cloudfront.net`)
from the **AWS CloudFront console** — it is hidden behind the proxy and cannot be read from DNS.

This does **not** block the maintenance cutover (the maintenance page doesn't use `cdn`).
Recreate `cdn` as a follow-up once you have the CloudFront domain.

---

## Complete record inventory

> ⚠️ When you add the site, Cloudflare's auto-scan will import the **proxied** apex/www/app/cdn
> records as literal Cloudflare anycast IPs (`104.21.90.90` / `172.67.198.88`) — that's garbage,
> **delete those**. The scan may also miss the DKIM TXT. Reconcile everything against this table.

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
| TXT | `zmail._domainkey` | (Zoho DKIM — value below) | — |

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

---

## Migration procedure

1. **New Cloudflare account** → *Add a site* → `livingcomposthubs.org.nz` → **Free** plan.
2. **Recreate records** from the inventory above. Email exactly; delete the bogus anycast-IP A
   records the scan imported; set apex/www to GitHub Pages (DNS only); handle `cdn` and `app`.
3. **Copy the new account's two assigned nameservers** (shown on the "complete your setup" screen
   — a unique pair like `xxx.ns.cloudflare.com` / `yyy.ns.cloudflare.com`, **not** chin/hunts).
4. **Porkbun** → the domain → **Authoritative Nameservers** → replace `chin`/`hunts.ns.cloudflare.com`
   with the new pair → save.
5. Cloudflare emails you when the zone is active. Then GitHub repo → **Settings → Pages** →
   confirm the domain banner goes green → tick **Enforce HTTPS** once the cert issues.

---

## Verify after cutover

```bash
dig +short livingcomposthubs.org.nz NS                      # → new Cloudflare pair
dig +short livingcomposthubs.org.nz                         # → 185.199.108–111.153 (GitHub Pages)
dig +short livingcomposthubs.org.nz MX                      # → mx.zoho.com.au etc. (email intact)
dig +short zmail._domainkey.livingcomposthubs.org.nz TXT    # → DKIM intact
curl -sI https://livingcomposthubs.org.nz | head -1        # → 200/301 from the maintenance page
```

---

## Rollback

The safe rollback is **this inventory** — re-enter the records anywhere. NS-flip-back to the old
account is unreliable: Cloudflare keeps a zone active on only one account at a time.

---

## Alternative: skip Cloudflare, use Porkbun DNS

Since every record is recreated by hand either way, you can instead point Porkbun at **Porkbun's
own nameservers** and manage DNS there — plain authoritative DNS, no proxy/cert gotchas (apex/www
to GitHub Pages just work). Same record reconstruction. `cdn` becomes a DNS-only CNAME straight to
the CloudFront domain. Only choose Cloudflare if you specifically want its proxy/CDN/WAF in front.
