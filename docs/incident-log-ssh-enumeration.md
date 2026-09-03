# Incident Log: SSH Enumeration and Brute Force Against a New Host

**Host:** `pereiraabierta-vps` — Vultr VPS, Miami, Ubuntu 24.04.4 LTS
**Window:** 2 September 2026, 01:47–02:45 UTC
**Impact:** None. No unauthorized access. No service disruption.
**Classification:** Opportunistic, non-targeted. Internet background radiation.

---

## Summary

A newly provisioned VPS received automated SSH authentication attempts against the `root` account within the
first hour of existence. All attempts failed at the pre-authentication stage. `fail2ban` recorded **17 failed
attempts** and **2 IP bans** in that window.

The host was not targeted. It was enumerated by mass scanning of the IPv4 address space and then handed to a
credential-guessing routine.

Two distinct classes of traffic appear in the logs, from two different sources with different intent. Telling
them apart is the analytically useful part of this incident.

---

## Timeline (UTC)

| Time | Event |
|---|---|
| 01:47:56 | Instance created |
| ~01:48 | First boot; SSH listening on `0.0.0.0:22` |
| 02:08:20 | `fail2ban` started, `sshd` jail active |
| 02:12 | Key-based authentication verified for non-root user from an external host |
| 02:34:45 | `sshd` restarted with hardening drop-in applied |
| **02:35:36** | **First hostile authentication attempt in the post-restart log window** — `92.118.39.14` |
| 02:37:02 | Second attempt, same source |
| 02:38:30 | Third attempt, same source |
| 02:39:38 | Connection from `185.242.226.17` — no authentication attempted |
| 02:39:55 | Fourth attempt, `92.118.39.14` |
| 02:41:28 | Fifth attempt, `92.118.39.14` |
| ~02:45 | `fail2ban`: 17 total failures, 2 IPs banned |

**Elapsed from instance creation to first observed hostile attempt: 47 minutes, 40 seconds.**

Note on precision: `fail2ban` began counting at 02:08:20 and recorded 17 total failures by 02:45, while the
`sshd` journal excerpt below begins at the 02:34:45 service restart. Attempts occurring between 02:08 and
02:34 are therefore counted but not shown. 02:35:36 is the earliest attempt *visible in the captured log
window*, not necessarily the earliest overall.

---

## Evidence

### sshd journal

```
Sep 02 02:34:45 pereiraabierta-vps sshd[10895]: Server listening on 0.0.0.0 port 22.
Sep 02 02:35:36 pereiraabierta-vps sshd[10901]: Connection closed by authenticating user root 92.118.39.14 port 35092 [preauth]
Sep 02 02:37:02 pereiraabierta-vps sshd[10903]: Connection closed by authenticating user root 92.118.39.14 port 37002 [preauth]
Sep 02 02:38:30 pereiraabierta-vps sshd[10910]: Connection closed by authenticating user root 92.118.39.14 port 38898 [preauth]
Sep 02 02:39:38 pereiraabierta-vps sshd[10915]: Connection closed by 185.242.226.17 port 39072 [preauth]
Sep 02 02:39:55 pereiraabierta-vps sshd[10917]: Connection closed by authenticating user root 92.118.39.14 port 40794 [preauth]
Sep 02 02:41:28 pereiraabierta-vps sshd[11000]: Connection closed by authenticating user root 92.118.39.14 port 42692 [preauth]
```

### fail2ban

```
$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     17
`- Actions
   |- Currently banned: 0
   |- Total banned:     2
```

`Currently banned: 0` reflects expiry of the default 10-minute ban window, not absence of bans.

---

## Analysis: two different things in the same log

The two sources look similar at a glance and are behaviorally distinct. The log lines themselves carry the
distinction:

```
Connection closed by authenticating user root 92.118.39.14 port 35092 [preauth]
                     ^^^^^^^^^^^^^^^^^^^^^^^^
Connection closed by 185.242.226.17 port 39072 [preauth]
                     ^ no user, no authentication attempted
```

`92.118.39.14` opened a session and attempted to authenticate as `root`, repeatedly, from rotating source
ports. That is credential guessing.

`185.242.226.17` connected once, took the SSH banner, and disconnected without attempting authentication.
That is fingerprinting.

**One is trying the door handle. The other is photographing the door.**

That distinction determines triage priority. The first warrants a ban. The second is a known research scanner
and is noise.

---

## Attribution

### 92.118.39.14 — brute force source

```
inetnum:      92.118.39.0 - 92.118.39.255
org:          ORG-DL591-RIPE
netname:      DMZHOST
descr:        https://dmzhost.co
country:      NL
% Abuse contact for '92.118.39.0 - 92.118.39.255' is 'dmzhostabuse@gmail.com'
```

Registered through **RIPE** (Europe / Middle East / Central Asia registry). The netblock belongs to an
offshore hosting provider. The attacker is a customer of that provider, not the provider itself — renting
servers is not the offense.

**On the abuse contact:** a free Gmail address published as the RIPE abuse contact for an entire /24 is
unusual. Legitimate providers typically run a domain-based abuse mailbox with ticketing behind it.

In fairness, there is a defensible innocent explanation: keeping a contact channel off your own network means
you remain reachable during an outage or DDoS, and some entirely mainstream providers do exactly this. Two
other commonly cited justifications — protecting the corporate domain from registrar action, and avoiding
email blocklisting — do not survive scrutiny. Registrars act on conduct, not on mailbox location, and
blocklists target senders, not recipients of complaint mail.

**The honest position:** the abuse contact is a weak signal, not proof of anything. What *is* documented is
the behavior recorded in the logs above. Motive is inference; the authentication attempts are fact.

### 185.242.226.17 — research scanner

```
inetnum:      185.242.226.0 - 185.242.226.255
netname:      AISPERA
descr:        Criminal IP Collector AiSpera
remarks:      | Criminal IP collects port information for only security/research purposes.
remarks:      | It only reads the response data from basic port requests,
remarks:      | and never utilizes vulnerability scanning or exploit scripts.
% Abuse contact ... is 'request@aispera.com'
```

Criminal IP is a commercial attack-surface and threat-intelligence platform, in the same category as Shodan
and Censys. They publish scanner attribution directly in the RIPE record specifically so operators do not
misclassify their traffic. Consistent with the observed behavior — a single connection, no authentication
attempt.

Worth noting that banner data is not trivial. An SSH version string identifies the distribution, approximate
release, and OpenSSH build, which is enough to cross-reference against a CVE database. These platforms are
used by defenders to find their own exposed assets, and equally by attackers to build target lists without
scanning themselves.

---

## Why the attempts failed

Three independent controls, any one of which was sufficient:

| Control | Effect |
|---|---|
| Deployed with SSH key only | `root` has **no password**. Nothing to guess. |
| `PasswordAuthentication no` | Server does not offer password auth as a method. |
| `PermitRootLogin no` | `root` cannot authenticate over SSH regardless of credentials. |

`[preauth]` in every log line confirms the connections terminated before authentication completed. No attempt
ever reached the point of being wrong about a password.

### The configuration trap that nearly undermined this

Ubuntu's `sshd_config` includes `/etc/ssh/sshd_config.d/*.conf` at the top of the file, and for most keywords
**the first value wins**. The Vultr image ships:

```
$ sudo cat /etc/ssh/sshd_config.d/50-cloud-init.conf
PasswordAuthentication yes
```

A hardening file named `99-hardening.conf` would be parsed *after* this and lose, leaving password
authentication enabled with no error, no warning, and no visible symptom. Naming it `10-hardening.conf` is
what makes it take precedence.

This was confirmed empirically rather than assumed:

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password jromero@<ip>
# denied without prompting for a password
```

---

## Remediation

No remediation was required — the controls held. Subsequent hardening was surface reduction rather than
incident response:

Public SSH access was removed entirely. Port 22 is now reachable only over the Tailscale interface:

```
[ 1] 80 on tailscale0        ALLOW IN    Anywhere
[ 2] 22 on tailscale0        ALLOW IN    Anywhere
[ 3] 80 (v6) on tailscale0   ALLOW IN    Anywhere (v6)
[ 4] 22 (v6) on tailscale0   ALLOW IN    Anywhere (v6)
```

Web traffic reaches the host exclusively through an outbound Cloudflare Tunnel, so ports 80 and 443 were never
opened. The host now presents **zero publicly reachable ports**.

External verification:

```
$ ssh -o ConnectTimeout=5 jromero@<public-ip>
ssh: connect to host <public-ip> port 22: Operation timed out
```

A timeout rather than `Connection refused` is the intended result. Refusal confirms a live host; silent packet
drop returns nothing to a scanner.

---

## Measuring the result

Claiming reduced attack surface is easy. The useful version is measured — and conveniently, the platforms that
index the internet can be queried about your own host.

### Shodan

```
shodan.io/host/<public-ip>
404: Not Found
Note: No information available for <public-ip>
```

No record at all. Shodan scans the entire address space but cycles each IP on the order of days to weeks, not
continuously. The public exposure window here was under three hours, and their pass did not fall inside it.

### Criminal IP

More interesting, because this is the platform that **appears in the logs above as an actor**. They connected
at 02:39:38 and took a banner. Their current record:

| Field | Value |
|---|---|
| Current Open Ports (TCP) | *No open port information* |
| Current Open Ports (UDP) | *No open port information* |
| Open Ports | 0 |
| Vulnerabilities | 0 |
| Abuse Record | 0 |
| **Connected Domains** | **0** |
| Representative Domain | N/A |
| SSL Certificate | N/A |
| IP Scoring (inbound / outbound) | Safe / Safe |

What remains visible is only what cannot be hidden and does not matter: ASN 20473 (The Constant Company /
Vultr), Miami, and a generic provider PTR of the form `<ip>.vultrusercontent.com`. Every instance on that
platform has one.

### The line that matters most

**`Connected Domains: 0`.**

Web traffic reaches this host exclusively through an outbound Cloudflare Tunnel. The public domain resolves to
Cloudflare's addresses and has never resolved to this server. No DNS record links them, and no TLS certificate
was ever issued to the origin, because the origin does not terminate TLS.

Origin IP disclosure is the standard way protection like this gets bypassed — an attacker who learns the real
address can go around the CDN entirely. Here, a commercial threat-intelligence platform that actively scanned
the host holds no association between it and the service it runs.

### Caveat

A `404` on Shodan means "no data," which is ambiguous between *never scanned* and *scanned and found nothing*.
The stronger evidence is persistence: re-checking after several scan cycles and still finding nothing
distinguishes an empty result from an absent one. Worth re-running in a few weeks.

---

## Takeaways

**Exposure is immediate and impersonal.** IPv4 is small enough to scan exhaustively in under an hour. Cloud
provider netblocks are published and prioritized because they are dense with real servers. A recycled IP may
already appear on target lists from a previous tenant. There is no period of obscurity for a new host.

**Reporting accomplishes nothing at this scale.** No intrusion occurred, no damage was done, and a handful of
rejected logins is below the threshold of any enforcement body. Cross-border action against offshore hosting
requires the provider to be linked to something far more serious. Upstream intervention is not part of a
realistic threat model.

**Configuration must be verified, not assumed.** The precedence trap fails silently and would have left
password authentication enabled against active brute forcing.

**The best outcome is not a well-defended port.** It is no port. Defending SSH against botnets is a
permanently ongoing task; removing it from the internet ends the task.

**Measure the claim.** "Reduced attack surface" is an assertion until a third party that scanned you reports
nothing. Criminal IP appears in these logs as an actor and now holds an empty record for the host — including
no association between the IP and the domain it serves.

---

## Reproducing this analysis

```bash
# attempts and outcomes
sudo journalctl -u ssh --no-pager | grep preauth

# jail state and ban counts
sudo fail2ban-client status sshd

# netblock ownership and abuse contact
whois <ip> | head -20

# current exposure
sudo ufw status numbered

# external reputation — check the IPs, and your own host
# https://www.abuseipdb.com/check/<ip>
# https://www.shodan.io/host/<your-ip>
```

Looking up your own host before and after surface reduction makes the change concrete.

---

## Related

The migration that produced this host, and the hardening described above, is documented in
[`pereira-abierta-vps-migration.md`](./pereira-abierta-vps-migration.md).
