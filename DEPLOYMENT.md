# Deployment Guide

Static site for `awesomeedwin.com`, deployed to **Cloudflare Pages** with Wrangler.

## Overview

| Thing | Value |
|---|---|
| Platform | Cloudflare Pages |
| Pages project | `awesomeedwin` |
| Cloudflare account | Edwin Encomienda (`a8ff9c451998ca7e86bc4cdb6d0a1bc1`) |
| Default URL | https://awesomeedwin.pages.dev |
| Custom domain | https://awesomeedwin.com (zone `222d46083fc72029e40166acdd3ad0ee`) |
| Build output | `public/` (plain static files, no build step) |
| Config | `wrangler.jsonc` (`pages_build_output_dir: "public"`) |

## Standard deploy

```bash
npm install        # first time only
npm run deploy     # production (deploys public/ to branch "main")
npm run deploy:preview  # preview deployment (current git branch)
```

There is no build step — edit files in `public/` and deploy. The custom domain
serves the latest production deployment automatically; no extra step needed
after deploying.

## Authentication (read this first — common gotcha)

Two credentials are in play, and one can silently break the other:

1. **Wrangler OAuth login** (`wrangler login`, stored in
   `~/Library/Preferences/.wrangler/config/default.toml`). Has Pages/Workers
   permissions but **no DNS permissions** (wrangler's OAuth doesn't offer a
   DNS scope).
2. **`CLOUDFLARE_API_TOKEN` env var**, exported by direnv from
   `~/Code/Personal/.envrc` for every project under `~/Code/Personal/`.
   When set, it **overrides** the OAuth login entirely.

Symptoms and fixes:

- `Failed to automatically retrieve account IDs` from any wrangler command →
  the env token is expired or under-scoped. Either fix the token in
  `~/Code/Personal/.envrc` (then `direnv allow`), or bypass it for one
  command: `unset CLOUDFLARE_API_TOKEN; npx wrangler <cmd>`.
- The API token in `.envrc` should have at least:
  - Account → **Cloudflare Pages: Edit**
  - Account → **Account Settings: Read**
  - Zone (`awesomeedwin.com`) → **DNS: Edit**, **Zone: Read**

Never commit or print token values.

## Custom domain setup (already done — reference for rebuilds)

The custom domain needs BOTH of these; the dashboard does both at once, but
via API/CLI they are separate steps and the domain stays `pending` with
"CNAME record not set" until both exist:

1. **Attach the domain to the Pages project.** Wrangler has no command for
   this; use the API:

   ```bash
   curl -X POST "https://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/pages/projects/awesomeedwin/domains" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name":"awesomeedwin.com"}'
   ```

2. **Create the DNS record** in the `awesomeedwin.com` zone:

   ```txt
   Type: CNAME, Name: @ (apex), Target: awesomeedwin.pages.dev, Proxied: On
   ```

   ```bash
   curl -X POST "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/dns_records" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"type":"CNAME","name":"awesomeedwin.com","content":"awesomeedwin.pages.dev","proxied":true}'
   ```

3. **Verification** happens automatically within a couple of minutes. To
   force a re-check or inspect status:

   ```bash
   # status ("pending" → "active")
   curl -s "https://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/pages/projects/awesomeedwin/domains" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

   # re-trigger validation
   curl -s -X PATCH "https://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/pages/projects/awesomeedwin/domains/awesomeedwin.com" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
   ```

Note: `www.awesomeedwin.com` is NOT configured. To add it, repeat steps 1–2
with `www.awesomeedwin.com` / a `www` CNAME.

## Verifying a deploy

```bash
npx wrangler pages deployment list --project-name awesomeedwin  # recent deploys
curl -s -o /dev/null -w "%{http_code}\n" https://awesomeedwin.com  # expect 200
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Site works on `*.pages.dev` but not the custom domain | Domain attached but DNS/verification incomplete — see "Custom domain setup" |
| Custom domain stuck `pending` / "CNAME record not set" | CNAME missing in zone, or verification not re-run — create record, then PATCH to re-validate |
| Wrangler auth errors despite being logged in | Stale `CLOUDFLARE_API_TOKEN` from `~/Code/Personal/.envrc` overriding OAuth — see "Authentication" |
| 522 on custom domain right after setup | Verification still propagating; wait 1–2 min and retry |
