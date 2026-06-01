# Hosting notes — noteshq.ai

The marketing site is plain static HTML; it works on every major
static host. The only host-specific bit is the **404 handler** —
make sure it's wired so visitors who hit a broken URL land on the
brand-matching `/404.html` page (which exists in this folder)
**and** the response carries an HTTP 404 status code (not 200).

## Per-host setup

### Netlify / Cloudflare Pages

Already wired via the **`_redirects`** file at the repo root:

```
/*    /404.html   404
```

No dashboard config needed. Drop the folder into the build output
and it's done.

### Vercel

`vercel.json` (create at repo root if you migrate):

```json
{
  "routes": [
    { "src": "/(.*)", "status": 404, "dest": "/404.html" }
  ]
}
```

Or simpler — Vercel auto-serves `/404.html` on any 404 by
convention. The `_redirects` file is ignored on Vercel.

### Apache (cPanel, classic LAMP)

Already wired via the **`.htaccess`** file at the repo root:

```
ErrorDocument 404 /404.html
```

Make sure `mod_rewrite` and `mod_headers` are enabled (default
on most cPanel installs).

### nginx (self-hosted / Hetzner / DigitalOcean) — **THIS IS THE ACTIVE HOST**

A complete, ready-to-deploy config lives in **`nginx.conf`** at the
repo root. To deploy it:

```bash
# 1. Copy the static files
sudo mkdir -p /var/www/noteshq.ai
sudo rsync -av --delete ./ /var/www/noteshq.ai/ \
    --exclude=.git --exclude=HOSTING.md --exclude=nginx.conf \
    --exclude=_redirects --exclude=.htaccess

# 2. Install the nginx config
sudo cp nginx.conf /etc/nginx/sites-available/noteshq.ai
sudo ln -sf /etc/nginx/sites-available/noteshq.ai /etc/nginx/sites-enabled/
sudo nginx -t                  # syntax check
sudo systemctl reload nginx    # apply

# 3. First-time only: issue Let's Encrypt certificates
sudo certbot --nginx -d noteshq.ai -d www.noteshq.ai
```

The config covers:

- **Apex as canonical** — `www.noteshq.ai` 301-redirects to bare
  `noteshq.ai`, so search engines collapse link equity into one URL.
- **404 handler** — `error_page 404 /404.html` with `internal;` on
  the location block so a direct GET to `/404.html` still returns a
  real 404 status (not 200).
- **Clean URLs** — `try_files $uri $uri.html $uri/index.html =404`
  lets `/pricing` resolve to `/pricing.html` without a trailing slash.
- **`.well-known` passthrough** — needed for Let's Encrypt renewals
  AND for the Android App Links `assetlinks.json` that the digest
  email's "Open NotesHQ" button relies on for direct app handoff.
- **Cache headers** — long cache on `/assets/*` (immutable, 30d),
  short cache on HTML so deploys are visible within minutes.
- **Dotfile guard** — anything starting with `.` returns 404 except
  the `/.well-known/` passthrough above (no `.git`, `.env`, etc. leaks).

After `sudo certbot --nginx ...` runs, certbot edits the config in
place to add the HTTPS server block + the HTTP→HTTPS 301. The
commented-out 443 block in `nginx.conf` shows what certbot will add.

### GitHub Pages

GitHub Pages auto-serves `/404.html` from the repo root for any
404 response. No config needed — just push `404.html`.

## Testing

After deploy, verify:

```bash
# Should return a 404 status AND the brand-matching page body.
curl -sI https://noteshq.ai/this-path-does-not-exist
# Look for:   HTTP/2 404
# NOT:        HTTP/2 200
```

If you get 200, the host is serving the page but not setting the
status — which means the SPA-style "always serve index.html" rule
is winning over the 404 rule. Fix the order in `_redirects` /
`nginx.conf` so the explicit 404 rule fires first.

## What the 404 page does

`404.html` mirrors the marketing brand (Geist + Instrument Serif,
purple accent, same logo SVG), echoes back the broken URL the
visitor tried (via JS reading `window.location.pathname`), and
offers six links back into the site. `<meta name="robots"
content="noindex,nofollow">` keeps crawlers from indexing 404
pages as legitimate content.
