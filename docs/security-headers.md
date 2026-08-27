# HTTP Security Headers & CSP

The site sets a full set of HTTP response security headers at the edge via Caddy
(see [`../config/Caddyfile.example`](../config/Caddyfile.example)). Result:
**grade A** on [securityheaders.com](https://securityheaders.com/?q=pereiraabierta.co).

## Headers

| Header | Value | Purpose |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS for a year |
| `X-Frame-Options` | `DENY` | Anti-clickjacking (paired with `frame-ancestors 'none'`) |
| `X-Content-Type-Options` | `nosniff` | Stop MIME-type sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Trim referrer leakage |
| `Permissions-Policy` | `geolocation=(self), camera=(), microphone=()` | App uses geolocation only |
| `Content-Security-Policy` | see below | Restrict where resources can load from |

## Content-Security-Policy

The CSP is scoped to exactly what the app loads — nothing more:

| Directive | Allowed sources | Why |
|---|---|---|
| `default-src` | `'self'` | Same-origin by default |
| `img-src` | `'self' data: blob:` + OpenStreetMap tiles + `unpkg.com` | Map tiles + base64 backgrounds/logo/markers |
| `style-src` | `'self' 'unsafe-inline'` + `unpkg.com` + `fonts.googleapis.com` | Leaflet CSS, inline styles, Google Fonts |
| `script-src` | `'self' 'unsafe-inline'` + `unpkg.com` + `static.cloudflareinsights.com` | Leaflet/markercluster, inline app JS, Cloudflare beacon |
| `connect-src` | `'self'` + `nominatim.openstreetmap.org` + `unpkg.com` + `cloudflareinsights.com` | Geocoding, source maps, analytics |
| `font-src` | `'self'` + `fonts.gstatic.com` | Nunito web font files |
| `object-src` / `base-uri` / `frame-ancestors` / `form-action` | locked down | Defense in depth |

### Note on the A vs A+ grade

The grade is capped at **A** (not A+) because the CSP includes `'unsafe-inline'`
in `script-src` / `style-src`. The frontend is a single self-contained HTML file
with inline `<script>` and `<style>` blocks, which require it. Reaching A+ would
mean moving that code to external files and adopting nonces or hashes — a
worthwhile future refactor, but a conscious trade-off for now.

## Verifying

```bash
# List the security headers the live site returns
curl -sI https://YOUR_DOMAIN | grep -iE \
  "content-security-policy|strict-transport|x-frame|x-content-type|referrer-policy|permissions-policy"
```

Then load the site with the browser console open and confirm there are no
`Refused to load … Content-Security-Policy` violations. Any blocked resource is
named explicitly in the console, which makes tuning the policy straightforward.

## Applying changes safely (no downtime)

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak            # back up first
sudo cp ~/Caddyfile /etc/caddy/Caddyfile                          # install new
sudo caddy validate --adapter caddyfile --config /etc/caddy/Caddyfile
sudo systemctl reload caddy                                       # graceful reload
# rollback if needed:
# sudo cp /etc/caddy/Caddyfile.bak /etc/caddy/Caddyfile && sudo systemctl reload caddy
```
