# Deploying the maintenance page

A self-contained static site (`index.html` + `404.html`) for GitHub Pages.
`404.html` is a copy of `index.html` so **any path** on the domain shows the
maintenance notice. `CNAME` binds the custom domain.

## 1. Create the repo and push

```bash
cd /Users/pete/projects/living-compost-hubs/lch-maintenance
git init -b main
git add .
git commit -m "Maintenance page for livingcomposthubs.org.nz"

# Replace <OWNER> with the GitHub account/org that should host this.
gh repo create <OWNER>/lch-maintenance --public --source=. --remote=origin --push
```

## 2. Enable GitHub Pages

```bash
gh api -X POST repos/<OWNER>/lch-maintenance/pages \
  -f 'source[branch]=main' -f 'source[path]=/'
```

(Or: repo **Settings → Pages → Source: Deploy from a branch → main / root**.)

The `CNAME` file sets the custom domain to `livingcomposthubs.org.nz`. Confirm it
under Settings → Pages, and enable **Enforce HTTPS** once the cert provisions
(can take a few minutes after DNS resolves).

## 3. Repoint DNS

> ⚠️ This is the live cutover. Verify GitHub's current IPs before changing records:
> https://docs.github.com/pages/configuring-a-custom-domain-for-your-github-pages-site

**Apex `livingcomposthubs.org.nz`** — four A records to GitHub Pages:

```
A   @   185.199.108.153
A   @   185.199.109.153
A   @   185.199.110.153
A   @   185.199.111.153
```

(Optional IPv6 AAAA records:)

```
AAAA  @  2606:50c0:8000::153
AAAA  @  2606:50c0:8001::153
AAAA  @  2606:50c0:8002::153
AAAA  @  2606:50c0:8003::153
```

**`www` subdomain** — CNAME to your Pages host:

```
CNAME  www   <OWNER>.github.io.
```

Lower the TTL beforehand (e.g. 300s) so you can switch back quickly when the new
app is ready. Propagation is usually minutes but can take longer depending on the
registrar.

## 4. Verify

```bash
dig +short livingcomposthubs.org.nz
curl -sI https://livingcomposthubs.org.nz | head -n 1
```

You should see the GitHub Pages IPs and a `200`/`301` from the maintenance page.
