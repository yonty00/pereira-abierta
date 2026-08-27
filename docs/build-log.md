# Build Log — Pereira Abierta

A condensed, sanitized account of how this project was built, from a spare desktop
to a hardened, publicly reachable full-stack app. (Server IPs, MAC addresses,
usernames, key names, and personal emails are intentionally omitted.)

## 1. Objective

Repurpose a spare desktop into a **dedicated, public-facing web server** — hosting
a real site, exposed securely with a minimal attack surface, and used as hands-on
practice for a cybersecurity career path (Security+, Hack The Box, hardening
fundamentals).

## 2. Architecture decided

| Layer | Choice | Rationale |
|---|---|---|
| OS | Ubuntu Server LTS, headless | Industry standard, lean, long security-update window |
| Install | Bare metal (wiped) | Eliminates legacy driver cruft; smaller attack surface than host+VM |
| Web server | Caddy | Single binary, simple config, easy security headers |
| Backend | PocketBase | Go single binary, SQLite, REST API + admin UI, quick to model data |
| Public exposure | Cloudflare Tunnel | **Outbound-only — no router ports opened**, hides IPs, HTTPS + DDoS |
| Remote admin | Tailscale + SSH keys | Private mesh, reach from anywhere, nothing exposed publicly |

## 3. A hardware-debugging detour (and the lesson)

The original plan was to keep Windows and run Linux in a VM. While stripping the
Windows install down (removing an antivirus, a VPN client, and some vendor
software), the machine began **freezing hard about two minutes after every boot**.

Debugging narrowed it to software, not hardware:

- **Safe Mode with Networking was stable** → a third-party driver/service was at fault.
- Event Viewer showed a flood of DistributedCOM 10005 errors — traced to a service
  that simply isn't allowed to start in Safe Mode, i.e. a **red herring**.
- Device Manager revealed a leftover **VPN WireGuard adapter** and a half-enabled
  **Hyper-V alongside VirtualBox**. Removing the adapter and disabling the hypervisor
  did **not** fix it.

**Conclusion / lesson:** security software and VPN clients install kernel-level
drivers that often don't uninstall cleanly. Rather than keep pulling suspect drivers
one at a time, wiping to a clean bare-metal Linux install was faster, safer, and
better aligned with the end goal.

## 4. Base setup & hardening

- Full system update; **UFW** default-deny inbound (SSH only)
- **SSH key-based auth** (ed25519), with **password and root login disabled**
- **fail2ban** on SSH (repeated failures → temporary ban)
- **unattended-upgrades** for automatic security patches
- Installed **Caddy**, served a starter page, verified over the LAN

## 5. The application

- Modeled a single **PocketBase collection** for businesses (name, category, status,
  address, phone, sector/barrio, description, social, geo point), with API rules:
  public read + create, admin-only edit/delete.
- Built a **single self-contained HTML frontend** — Leaflet + markercluster map on
  OpenStreetMap tiles, Nominatim geocoding, a registration form with a map location
  picker, and a business detail sheet. Spanish-only, mobile-first.
- Connected the frontend to the PocketBase REST API (load pins, submit new businesses).

## 6. Going public — safely

- Configured **Cloudflare Tunnel** (`cloudflared`) as a systemd service: an
  outbound-only connection to Cloudflare, so **no inbound ports are opened** and the
  home IP stays hidden.
- Put **Caddy** in front of PocketBase to **block the admin UI (`/_/`) and non-public
  API paths at the edge** (return 404), while proxying the public collection.
- Added a domain, Cloudflare **Email Routing**, a `.com → .co` redirect, and
  **rate-limiting** for the submission endpoint (at both Cloudflare and PocketBase,
  using the real client IP via a trusted proxy header) as anti-spam.
- Installed **Tailscale** so administration happens over a private mesh — SSH stays
  key-only and nothing new is exposed.

## 7. Iteration

Marker clustering; filter by category and by sector/barrio; a **"Solo abiertos ahora"**
toggle; a location-confirm gate so business pins land accurately; a black-and-white
city-photo backdrop; a short branded splash intro; category set refined from real
usage (Salud, Comederos, Ferreterías, Moda, Belleza, Otros).

## 8. Security-headers pass

Added a full set of HTTP security headers plus a tuned Content-Security-Policy at the
Caddy edge, reaching **grade A** on securityheaders.com. Details in
[`security-headers.md`](security-headers.md).

## 9. Roadmap

- Migrate to a **VPS** for reliability as usage grows
- **PWA** (installable) + optional native app wrappers
- Self-host the map libraries (drop the CDN dependency); automatic offsite backups;
  optional human-verification (Turnstile) on the form
- Structured business hours → automatic "open now"
