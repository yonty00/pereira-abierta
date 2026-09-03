# Migrating Pereira Abierta from a Home Server to a VPS

**Migration:** 2 September 2026 — ~2.5 hours, including a live cutover
**Follow-up:** 3 September 2026 — offsite backups and home server decommission
**Downtime:** under 60 seconds
**Author:** Jonathan Romero

---

## Background

[Pereira Abierta](https://pereiraabierta.co) is a free community web application that maps open small and
medium businesses in Pereira, Colombia. It was built during an emergency, when residents needed to know
which pharmacies, hardware stores, and food vendors were actually operating.

It ran on a hardened Ubuntu server in my house. That was fine for an emergency response. It is not fine for
something people are going to depend on, because a residential internet connection, a consumer router, and a
machine sitting in a room in my home are three single points of failure I do not control.

This document covers moving it to a VPS without anyone noticing.

---

## Starting architecture

| Component | Role |
|---|---|
| PocketBase 0.39.11 | Backend API + SQLite database + static file serving |
| Caddy | Reverse proxy, security headers, API path allowlist |
| cloudflared | Named Cloudflare Tunnel — outbound only, no inbound ports |
| Tailscale | Private admin access |
| Ubuntu 26.04 LTS | Host OS (home server) |

The application is deliberately simple: a single Go binary, a SQLite file, one HTML page, and one image.
`pb_data` was 4.3 MB holding 113 business records.

The Cloudflare Tunnel matters more than anything else in this list. Because `cloudflared` dials *out* to
Cloudflare, the origin server never needs an inbound port open for web traffic. That property is what makes
the whole migration low risk.

---

## Target architecture

Same stack, different location.

| Choice | Value |
|---|---|
| Provider | Vultr |
| Plan | `vhp-1c-2gb` — 1 vCPU, 2 GB RAM, 50 GB NVMe |
| Region | **Miami** |
| OS | Ubuntu 24.04.4 LTS |
| Cost | ~$14.40/mo including automatic backups |

### Why Miami

Latency to end users was the deciding factor, not price. Miami is the primary interconnection hub for Latin
America, and Colombian ISPs peer there heavily. Hetzner is cheaper per unit of compute and Ashburn or
Frankfurt would have been fine on paper, but for an app whose entire user base is in Pereira, the shortest
network path wins.

### Why not AWS

The app uses no managed services. On AWS it would be a bare EC2 instance wrapped in VPCs, security groups,
IAM roles, and EBS volumes — more to misconfigure, for zero architectural benefit. Metered egress also
introduces bill risk on a free service funded out of pocket. Flat pricing is worth more than elasticity here.

---

## Sequence

### 1. Provision

Deployed with an SSH public key attached at creation time, so the instance never had a root password.

**Gotcha:** Vultr's SSH key field is a picker for keys already registered to the account, not a paste target.
The key must be added under *Account → SSH Keys* first. Navigating away from the create form also resets it —
worth using a second browser tab.

### 2. Harden the baseline

```bash
apt update && apt upgrade -y
apt install -y ufw fail2ban unattended-upgrades curl rsync

adduser --gecos "" jromero
usermod -aG sudo jromero
rsync --archive --chown=jromero:jromero ~/.ssh /home/jromero/
```

Then SSH lockdown:

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
EOF

sudo sshd -t && sudo systemctl restart ssh
```

**The important detail — filename ordering.** Ubuntu's `sshd_config` includes `/etc/ssh/sshd_config.d/*.conf`
at the *top* of the file, and for most keywords **the first value encountered wins**. Vultr's image ships:

```
$ sudo cat /etc/ssh/sshd_config.d/50-cloud-init.conf
PasswordAuthentication yes
```

A hardening drop-in named `99-hardening.conf` would be read *after* `50-cloud-init.conf` and silently lose.
Password authentication would remain enabled with no visible indication of failure. Naming it `10-` is what
makes it take precedence.

This is a good example of why config changes need verification rather than assumption. I confirmed the
lockdown from a separate machine rather than trusting that the file existed:

```bash
# expect: works
ssh -i ~/.ssh/id_ed25519_webserver jromero@<ip>

# expect: Permission denied (publickey)
ssh root@<ip>

# expect: denied without ever prompting
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password jromero@<ip>
```

Firewall, default deny inbound:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
```

Note that ports 80 and 443 are **never opened**. Cloudflare Tunnel is outbound-only, so there is nothing to
expose.

### 3. Join Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Key expiry disabled on the node — otherwise it deauthenticates in roughly six months and admin access breaks
at an inconvenient moment.

This put the VPS, the home server, and my laptop on one private network, which is how the data copy happened
without exposing anything.

### 4. Read the existing configuration

Rather than rebuild from memory, I pulled the real config off the home server: PocketBase version, systemd
unit definitions, the Caddyfile, and the `cloudflared` config. Matching the PocketBase version exactly
(`0.39.11`) mattered — a newer binary would have run schema migrations against production data on first start.

### 5. Install the stack

PocketBase pinned to the same version, Caddy from the official Cloudsmith repo, cloudflared from Cloudflare's
apt repo. systemd units copied from the home server with only the `cloudflared` binary path adjusted
(`/usr/local/bin` → `/usr/bin`).

### 6. Copy the data

Over Tailscale, using SSH agent forwarding so no private key ever landed on an intermediate host:

```bash
ssh -A -i ~/.ssh/id_ed25519_webserver jromero@<home-tailscale-ip>

rsync -av ~/pocketbase/pb_data ~/pocketbase/pb_public ~/pocketbase/pb_migrations \
      jromero@<vps-tailscale-ip>:~/pocketbase/
rsync -av ~/.cloudflared jromero@<vps-tailscale-ip>:~/
```

The tunnel credentials (`<tunnel-id>.json`, `cert.pem`) are secrets and were moved machine-to-machine over the
encrypted tailnet, never through a clipboard or a third party.

### 7. Verify before going live

Port 80 opened **only** on the Tailscale interface, so the new server was reachable from my devices and
invisible to the internet:

```bash
sudo ufw allow in on tailscale0 to any port 80
```

Then loaded the full site at `http://<vps-tailscale-ip>/` in a browser and confirmed the map rendered, marker
clusters populated, category filters worked, and the API returned `"totalItems":113`.

Meanwhile the production site kept serving from the home server, untouched.

---

## The cutover

### A correction worth recording

My initial plan was to run `cloudflared` on both machines simultaneously. A named Cloudflare Tunnel supports
multiple replicas and load-balances across them, which would have given a true zero-downtime drain.

**That plan was wrong for this application.** Each server has its own local SQLite database. With both
replicas live, a business submitted through Miami would be written to the Miami database and one submitted
through the home server to the home database — split brain, with one set of records discarded at
decommission.

Replica-based cutover is a stateless-service pattern. This is a stateful service with local storage, so the
correct approach is a short, deliberate gap.

### Actual sequence

```bash
# home server — stop serving, freeze the data
sudo systemctl stop cloudflared
sudo systemctl stop pocketbase

# vps — release the database file
sudo systemctl stop pocketbase

# home server — final consistent sync
rsync -av --delete ~/pocketbase/pb_data/  jromero@<vps-tailscale-ip>:~/pocketbase/pb_data/
rsync -av        ~/pocketbase/pb_public/ jromero@<vps-tailscale-ip>:~/pocketbase/pb_public/

# vps — come up and take the tunnel
sudo systemctl start pocketbase
sudo systemctl enable --now cloudflared
```

Stopping PocketBase before the final sync is what makes the copy trustworthy — it closes SQLite cleanly and
checkpoints the WAL, so the snapshot is consistent rather than mid-write.

### Result

```
$ curl -I https://pereiraabierta.co
HTTP/2 200
via: 1.1 Caddy
server: cloudflare
strict-transport-security: max-age=31536000; includeSubDomains
```

Tunnel established at **04:01:26 UTC**. Verified live at **04:01:37 UTC**. Read and write paths both confirmed
— a test business was submitted through the public site, appeared on the map, and was found in the database.

**Total downtime: under 60 seconds**, during the lowest-traffic hour of the day in Pereira.

---

## Post-migration hardening

With web traffic arriving exclusively through the tunnel, the public SSH port was the last thing reachable
from the internet. Removed:

```bash
sudo ufw allow in on tailscale0 to any port 22   # add replacement first
# verify Tailscale SSH works from a separate session
sudo ufw delete 4                                 # public 22/tcp (v6)
sudo ufw delete 1                                 # public 22/tcp
```

Delete the higher rule number first — UFW renumbers after each deletion.

Final firewall state:

```
[ 1] 80 on tailscale0        ALLOW IN    Anywhere
[ 2] 22 on tailscale0        ALLOW IN    Anywhere
[ 3] 80 (v6) on tailscale0   ALLOW IN    Anywhere (v6)
[ 4] 22 (v6) on tailscale0   ALLOW IN    Anywhere (v6)
```

Verification from outside:

```
$ ssh -o ConnectTimeout=5 jromero@<public-ip>
ssh: connect to host <public-ip> port 22: Operation timed out
```

A **timeout** rather than "connection refused" is the desired outcome. Refusal confirms a host is alive.
Dropping the packets silently gives a scanner nothing.

The server now has **zero publicly reachable ports** while remaining fully functional.

Out-of-band recovery is preserved through Vultr's noVNC console, which operates at the hypervisor level
independent of SSH, Tailscale, and the firewall. Locking down remote access is only safe when a path back in
exists that does not depend on what you just locked down.

---

## Offsite backups

Migrating created a new single point of failure. Before the cutover the data existed on two machines. After
it, every business registered since lived only on the VPS, protected by nothing but provider snapshots stored
in the same account as the server. A compromised or closed Vultr account would have taken the server and its
backups together.

PocketBase has a built-in backup scheduler with S3-compatible upload, so no external tooling was needed.

**Storage:** Cloudflare R2. Free at this scale (10 GB tier against 4.3 MB of data), no egress charges, and the
bucket sits in ENAM — the same coast as the server.

The tradeoff worth naming: Cloudflare already provides DNS and the tunnel, so putting backups there
concentrates dependency on one provider. A truly independent target such as Backblaze B2 would be stronger in
principle. The deciding argument was operational: one fewer account and one fewer credential to keep working,
and the correlated-failure case is survivable — losing the Cloudflare account costs DNS and the tunnel, but
the server and its data still exist.

### Scoped credentials

The R2 API token was created with **Object Read & Write** — not admin — and restricted to the single backup
bucket:

```
Permissions:      Object Read & Write
Specify buckets:  pereiraabierta-backups   (not "all buckets in this account")
TTL:              Forever
```

Scoping matters because this credential lives in plaintext in the PocketBase settings on a server that faces
the internet through a tunnel. If it leaks, the blast radius is one bucket of regenerable archives — not DNS,
not the tunnel, not future buckets.

`TTL: Forever` is deliberate. An expiring token means backups stop silently, and you discover it the day you
need one.

### PocketBase configuration

*Settings → Backups*

| Field | Value |
|---|---|
| Cron | `0 0 * * *` (daily, midnight UTC — 7pm in Pereira) |
| Max backups to keep | 30 |
| Endpoint | `https://<account-id>.r2.cloudflarestorage.com` |
| Bucket | `pereiraabierta-backups` |
| Region | `auto` |
| Force path-style addressing | **enabled** |

Two things that cost time:

**Path-style addressing must be on.** R2 addresses buckets as `<endpoint>/<bucket>`. Leaving this off produces
a configuration that looks correct and silently fails to connect.

**The endpoint excludes the bucket.** Cloudflare's dashboard displays the S3 API URL with the bucket appended.
PocketBase takes them as separate fields, so pasting the full URL causes the bucket to be appended twice and
every request 404s.

Retention was raised from the default 7 to 30. Seven daily backups is a one-week recovery window — if
corruption goes unnoticed for eight days, every copy contains it. Thirty days is roughly 130 MB, which is
free at this tier.

### Verification

A configuration that saves cleanly is not a backup. The archive was pulled back down and opened:

```bash
unzip -o pb_backup_acme_<timestamp>.zip -d pb-restore-test
sqlite3 pb-restore-test/data.db "SELECT COUNT(*) FROM Negocios;"
121
```

Valid archive, valid SQLite, correct record count. Silent backup failure is common enough — empty archives,
truncated writes, credentials that expired months ago — that "the file appeared in the bucket" is not
sufficient evidence.

The count also confirmed something useful independently: 113 businesses at migration, 121 a day later. The
write path was working in production with real users.

---

## Decommissioning the home server

The home server sat stopped but intact for a day as a rollback. Once the VPS had handled a full day of real
traffic, it was retired.

### Why it is not a failover

An obvious question is whether the home server should stand by to take over if the VPS fails. It should not,
and the reason is architectural rather than a matter of effort.

PocketBase uses local SQLite, which has no multi-writer replication. If the home server took over it would
serve a frozen snapshot — 113 businesses against the live 121 — and any submissions received during the
failover would be discarded when the VPS returned. If both ran simultaneously, that is the same split-brain
condition the cutover was sequenced to avoid, except unattended and undetected.

Failover for a single-writer stateful service is not a bolt-on. It needs Litestream, rqlite, or an equivalent
— real complexity for a free community application.

**The resilience story is restore, not failover:** provision a new VPS, restore the latest R2 archive, start
`cloudflared`. Roughly 30–45 minutes, against infrastructure failures measured in hours per year.

There is also an irony in the alternative. This project moved off home hosting precisely because a residential
connection and consumer hardware were the unreliable components. Designating them the fallback would
reintroduce what the migration removed.

### Procedure

The concern was not the machine existing. It was that it still held valid tunnel credentials and enabled
systemd units. A power cycle would have brought `cloudflared` up, registered it as a second tunnel replica,
and started serving a week-old database alongside Miami.

```bash
sudo systemctl disable --now cloudflared pocketbase caddy

# hand-written units live in /etc/systemd/system, so `mask` cannot
# create its symlink until the real files are moved aside
sudo mv /etc/systemd/system/cloudflared.service /etc/systemd/system/cloudflared.service.retired
sudo mv /etc/systemd/system/pocketbase.service  /etc/systemd/system/pocketbase.service.retired
sudo systemctl daemon-reload

sudo systemctl mask cloudflared pocketbase caddy
mv ~/.cloudflared ~/.cloudflared.RETIRED-<date>
```

`disable` prevents start at boot. `mask` symlinks the unit to `/dev/null` so it cannot be started manually or
pulled in by a dependency. Masking `caddy` matters specifically because it came from apt — a future package
operation could otherwise restore and re-enable its unit.

Credentials and unit files were renamed rather than deleted. Nothing is destroyed, and the VPS holds the
working copies.

### Verification

Configuration is not proof. The machine was rebooted:

```bash
sudo reboot
# ~60 seconds later
systemctl is-active cloudflared pocketbase caddy
inactive
inactive
inactive
```

A cold boot with all three services down is the actual guarantee that the machine can lose power and return
without rejoining the tunnel. `curl -I https://pereiraabierta.co` returned `HTTP/2 200` throughout.

Remaining physical task: wipe the disk before repurposing. It still holds retired tunnel credentials and a
stale copy of `pb_data`.

---

## What carried over for free

The home server's Caddyfile already contained a tuned CSP and a full set of security headers. Copying it
verbatim meant no regression:

- `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options: DENY`
- `Referrer-Policy`, `Permissions-Policy` scoped to geolocation only
- A CSP enumerating exactly the origins the app loads
- An allowlist blocking `/api/*` and the PocketBase admin UI at the edge, permitting only public reads of the
  `Negocios` collection

Retiring the home server also closes a pending item — LAN port 80 exposure — by removing the host entirely.

---

## Remaining work

- [x] Automatic offsite backups — Cloudflare R2, nightly, 30-day retention, restore-verified
- [x] Decommission the home server
- [ ] Wipe the retired machine's disk before repurposing
- [ ] Self-host Leaflet and markercluster instead of loading from unpkg
- [ ] Cloudflare Turnstile with server-side verification for submissions
- [ ] Re-allow `/api/realtime` in the Caddy allowlist for live updates
- [ ] Convert to an installable PWA
- [ ] Capacitor wrap for iOS and Android
- [ ] Monitoring — nothing currently alerts if a nightly backup fails to run

---

## Lessons

**Verify, do not assume.** The `50-cloud-init.conf` precedence trap fails silently. The only reason it was
caught is that the config was read back and the lockdown tested from an external machine.

**Keep a way back in.** An open root session during the SSH lockdown, the noVNC console during the firewall
changes, and the intact home server during the cutover. Every irreversible-looking step had a path back.

**Match the pattern to the architecture.** Tunnel replicas are elegant and wrong for a service with local
state. Recognizing that before the cutover avoided a data-loss incident that would have been invisible until
someone reported a missing business. The same reasoning ruled out keeping the home server as a failover
target — the constraint is SQLite's single-writer model, not a gap in configuration.

**Reduce surface rather than defend it.** The outcome is not a well-defended SSH port. It is no reachable port
at all.

**A backup you have not restored from is a hypothesis.** The archive was downloaded, extracted, and queried
before the work was called finished. Silent backup failure is common precisely because the failure mode
produces no error — just an object of plausible size sitting in a bucket.

---

## Related

Attack traffic observed against this host during its first hour online, and the analysis of it, is documented
separately in [`incident-log-ssh-enumeration.md`](./incident-log-ssh-enumeration.md).
