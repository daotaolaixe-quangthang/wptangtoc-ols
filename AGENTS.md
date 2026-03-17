# AGENTS.md — WPTangToc OLS (Must‑read for any AI Agent)

## 60‑second brief (read this first)

- This repo is the **template**; production runtime is `/etc/wptt/*` + `/usr/local/lsws/*`.
- Always answer with: **Context → Impact layer → State changes → Rollback → Verify**.
- Firewall/anti‑DDoS changes: **safe‑by‑default**, mention **lockout risk**, include **rollback‑first**, pre‑check real SSH port.
- Never print secrets (only locations).
- Do not scan the whole repo; read the **minimum** doc section + the **specific** module scripts.

---

This repository is a **template/bootstrap** for WPTangToc OLS.
After installation, the **real runtime** lives on servers at:

- `/etc/wptt/*` (scripts + configs used in production)
- `/usr/local/lsws/*` (OpenLiteSpeed + vhosts + docroots)

> **Rule 0 (Context trap)**: When debugging production, prefer reading `/etc/wptt/*` on the server.
> When editing this repo, you are editing **the template** (future installs / docs).

---

## 1) Non‑negotiable output contract

Every change request MUST be answered using this structure (even if short):

1) **Context**: which module/entrypoint/scripts are involved  
2) **Impact layer**: choose from:
   - OLS global config
   - OLS per‑vhost config
   - `.htaccess` per‑site rules
   - filesystem hardening (chmod/chattr/ownership)
   - system services (systemd)
   - cron/automation
   - kernel/eBPF (XDP)
   - cloud integration (Cloudflare/rclone/Telegram)
3) **State changes**: files/paths that will change (exact paths)
4) **Rollback plan**: how to revert + which services to restart
5) **Verify**: minimal checks after change

---

## 2) Safety rules (lockout / data loss)

### Firewall / anti‑DDoS

- Must be **safe‑by‑default**
- Must mention **risk of lockout**
- Must include **rollback‑first** steps (how to regain SSH access)
- Must pre‑check **actual SSH port** (never assume 22)

### Database / Restore

- Wipe/drop/restore are **destructive**.
- Must require a **backup before change** (or explicitly note none exists).

### Secrets

Never print secrets. Only document locations and permissions.
Common locations:

- Telegram: `/etc/wptt/.wptt.conf`
- Cloudflare API: `/etc/wptt/cloudflare-cdn-cache/.wptt-<domain>.conf`
- rclone tokens: `/root/.config/rclone/rclone.conf`

---

## 3) Token discipline (do not waste reads)

Before searching or reading widely:

- **Decide the target** (docs vs code vs both).
- Read **only** the relevant doc section first:
  - project assumptions → `docs/00-OVERVIEW.md`
  - architecture/flow → `docs/01-ARCHITECTURE-FLOW.md`
  - module logic → `docs/02-MODULES-INDEX.md` (the specific section)
  - agent rules → `docs/03-AI-AGENT-GUIDE.md`

**Hard rule**: Do not scan the whole repo unless the task truly requires it.

---

## 4) Where to start

- Docs index: `docs/README.md`
- Roadmap / phase tasks: `docs/04-ROADMAP-PHASE-TASKS.md`
- Runbooks rollback: `docs/05-RUNBOOKS.md`

## 5) Task/Phase workflow (plan → approve → execute → review → fix → report)

For non-trivial tasks/phases:

- First provide a **plan** and wait for user approval.
- Then execute strictly within the approved scope.
- Iterate on user review feedback until re-approved.
- Finish with a concise **final report** (files touched + rollback/verify summary).

