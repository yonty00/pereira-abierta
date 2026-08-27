# How to publish this repo (and keep it clean)

This folder is already sanitized and ready to push. Follow the steps below so no
sensitive data ever lands in git history.

## 0. Pre-flight redaction checklist

Everything here has already been scrubbed, but before you push, do a final grep to
be 100% sure none of these ever appear in any committed file:

- [ ] Server IPs — Tailscale (`100.x.x.x`) and LAN (`192.168.x.x`)
- [ ] NIC MAC address
- [ ] Server username / login
- [ ] SSH private keys or key filenames
- [ ] Personal email addresses (Gmail / Proton)
- [ ] Any PocketBase admin credentials or tokens

Quick self-check from inside the folder:

```bash
grep -rInE "100\.|192\.168\.|[0-9a-f]{2}(:[0-9a-f]{2}){5}|@gmail|@proton" . && \
  echo "^ review these before pushing" || echo "clean"
```

## 1. Scan for secrets (recommended — nice cybersecurity habit)

Use a secret scanner before the first push and before every push after. Either works:

```bash
# gitleaks (https://github.com/gitleaks/gitleaks)
gitleaks detect --source . --no-git -v

# or trufflehog (https://github.com/trufflesecurity/trufflehog)
trufflehog filesystem . --only-verified
```

If a scanner flags something, fix it **before** committing. A secret that reaches a
commit stays in history even after you delete it — you'd have to rewrite history
(`git filter-repo`) to truly remove it. Cleanest path: never commit it in the first place.

## 2. Add your screenshots

Drop images into `screenshots/` and they'll show in the README:

- `home.png` — the home screen
- `map.png` — the map with clustered pins
- `securityheaders-A.png` — the A grade from securityheaders.com

## 3. Initialize and push

```bash
cd pereira-abierta-repo
git init
git add .
git status                     # confirm .gitignore is excluding Caddyfile / index.html / pb_data etc.
git commit -m "Pereira Abierta — portfolio: build log, architecture, security"

# create an EMPTY public repo on github.com first (no README/license — this folder has them),
# then connect and push:
git branch -M main
git remote add origin https://github.com/<your-username>/pereira-abierta.git
git push -u origin main
```

> Tip: with the GitHub CLI you can do steps in one line —
> `gh repo create pereira-abierta --public --source . --push`

## 4. Polish (optional)

- Add repo **topics** on GitHub: `self-hosting`, `pocketbase`, `caddy`, `cloudflare-tunnel`,
  `leaflet`, `cybersecurity`, `homelab` — helps discovery.
- Pin the repo on your GitHub profile.
- Link it from your LinkedIn and résumé; the build log + A-grade write-up is the part
  that stands out for a cybersecurity role.
