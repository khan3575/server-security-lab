# Server Security Lab: Project Instructions

## Purpose

Build a small test website on a temporary Hetzner server, secure the server and the website, measure the result, and publish the evidence as a Fiverr portfolio item and a GitHub repo.

Why: the owner's Fiverr profile has no portfolio yet (profile strength 11/12). This project proves the skills sold in two gigs, Linux server hardening and Cloudflare setup, and produces a reusable checklist for real orders.

Source guide: `~/Documents/Server Security Lab Portfolio Project Guide.pdf` (26 pages). Follow it step by step. If this file and the guide disagree, ask the owner.

## Architecture

Six layers, top to bottom:

1. Visitor's browser opens `https://lab.sakib-khan.com`
2. Cloudflare: DNS, proxy, SSL Full (strict), WAF rule, rate limit
3. UFW firewall on the lab server: ports 22, 80, 443 only (later 80 and 443 only from Cloudflare ranges)
4. Ubuntu 24.04 lab server: SSH keys only, no root login, fail2ban, unattended-upgrades
5. Nginx on the server: HTTPS with Cloudflare origin certificate, security headers, forwards to `127.0.0.1:8080`
6. Docker container `nginx:alpine`: serves `index.html`, listens on `127.0.0.1:8080` only

The site is one static page. No login, forms, database or JavaScript.

## Steps

1. Create the server and measure "before" (lynis, ss, nmap). About 1 to 1.5 h.
2. Harden the server (sudo user, SSH lockdown, UFW, fail2ban, unattended-upgrades). 3 to 4 h.
3. Website in Docker and Nginx, including the Docker-bypasses-UFW demo. 2 to 3 h.
4. Cloudflare (DNS record, origin certificate, SSL Full strict, Nginx HTTPS and headers, Cloudflare-only firewall, WAF rule, rate limit). 2 to 3 h.
5. Measure "after", SSL Labs, securityheaders.com, screenshots, portfolio item, delete the server. 2 h.

## Hard rules

1. Never run any command from the guide on `ubuntu-1`. It is the owner's existing server, shared with their brother, and runs their site. Check the hostname in the terminal prompt before every command. The lab server prompt shows `lab`.
2. Never change Cloudflare records for `sakib-khan.com` or `api.sakib-khan.com`. Only add and change the `lab` record.
3. Before changing SSH or firewall settings, keep one SSH session open and test the new login in a second terminal. Close the first only after the test works. Run `sudo sshd -t` before every SSH restart and never restart on an error. Allow `22/tcp` in UFW before `ufw enable`.
4. Never put real passwords, `.env` files or data from the owner's other projects on the lab server.
5. Scan only the lab server with `nmap`. Never scan servers the owner does not own.
6. Do not invent or round measurement numbers. Use exactly what lynis, nmap, SSL Labs and securityheaders.com print.
7. Do not run destructive or irreversible commands (deleting servers, Cloudflare records or certificates, force pushes) without the owner's explicit confirmation in that moment.

## Secrets and privacy (applies to git, screenshots and Fiverr)

Never commit or publish:
- Anything in `~/.ssh/`, especially `lab_key`
- `/etc/ssl/cloudflare/lab.key` or `lab.pem`
- The server's real IP: write `LAB_IP`
- The owner's home IP (appears in fail2ban output)
- The Cloudflare account email, API tokens, any `.env`

`.gitignore` already blocks `*.key`, `*.pem`, `.env`, `lab_key*`. Before every commit that includes results or screenshots, run `grep -rE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' .` and review `git diff --staged`. A secret that was pushed stays in git history: rotate or revoke it instead of only deleting the file.

## Git workflow

- The repo lives only on the owner's Mac. The server never gets it. Copy files off the server with `scp -i ~/.ssh/lab_key sakib@LAB_IP:<file> .` before the server is deleted.
- Commit at the end of every lab step, so the history reads as the lab log. Example: `Step 2.5: lock down SSH`.
- Append each command run, in order, to `docs/hardening-runbook.md` with a one-line note on any problem and its fix. Add problems to `docs/troubleshooting.md`.
- Copy real config files into `configs/` after they work on the server.
- Put raw outputs in `results/before/` and `results/after/` (lynis, ports, nmap), with the IP replaced by `LAB_IP`.
- Put screenshots in `screenshots/` with numbered names, IPs and emails hidden.
- Write `README.md` last, with the real before/after numbers.

## File structure

```
server-security-lab/
├── CLAUDE.md
├── README.md
├── .gitignore
├── docs/
│   ├── hardening-runbook.md
│   └── troubleshooting.md
├── configs/
│   ├── ssh/            00-hardening.conf
│   ├── fail2ban/       jail.local
│   ├── nginx/          lab.sakib-khan.com
│   └── docker/         docker-compose.yml, site/index.html, site/style.css
├── results/
│   ├── before/         lynis.txt, ports.txt, nmap.txt
│   └── after/          lynis.txt, ports.txt, nmap.txt
└── screenshots/        01-nmap-before-after.png ... 08-site-padlock.png
```

## Screenshots to take

1. nmap before and after, side by side
2. lynis hardening index before and after
3. `ufw status verbose`
4. `fail2ban-client status sshd` with a banned IP
5. SSL Labs result
6. securityheaders.com result
7. Cloudflare Security Events showing the blocked `/wp-login.php` request
8. The page in the browser with the padlock

Use 3 to 4 of the clearest for Fiverr.

## Fiverr portfolio item

- Title: Hardened Ubuntu VPS + Cloudflare WAF setup (before/after)
- Images: best 3 to 4 screenshots, before/after first
- Description: use the template in guide section 5.4, filled with real numbers
- Linked gigs: the hardening gig and the Cloudflare gig
- Link the GitHub repo from the description, and link Fiverr from the README

## Clean up (end of Step 5)

- Hetzner: delete the `lab` server and any snapshot. Billing stops on deletion.
- Cloudflare: delete the `lab` DNS record, revoke the origin certificate, delete or pause the WAF and rate-limit rules.
- Keep `~/.ssh/lab_key` for future lab servers.
- Confirm the repo contains no secrets before making it public.

## How Claude should work in this project

- Guide one step at a time. Give the commands for that step, say where each runs (Mac, lab server, Cloudflare dashboard, Hetzner console), then wait for the owner to confirm before the next step.
- Pair every lab step with its git step.
- Do not SSH into servers or change Cloudflare or Hetzner on the owner's behalf unless asked. Local files and git in this folder are fine.
- Prefer exact commands from the guide. Explain flags only when asked.
- If something fails, ask for the exact error and the command that caused it, then use the guide's troubleshooting table first.
- Budget: about 12 to 14 hours and under 2 euros of Hetzner credit.
