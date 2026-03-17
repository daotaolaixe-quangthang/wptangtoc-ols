# WPTangToc OLS — Core rules (template repo)

## Scope & assumptions

- This repo is a **template/bootstrap**.
- Runtime on servers is `/etc/wptt/*` + `/usr/local/lsws/*`.

## Always do (mandatory)

- Identify **Context → Impact layer → State changes → Rollback → Verify** in your response.
- For firewall/anti‑DDoS changes:
  - **safe‑by‑default**
  - mention **risk of lockout**
  - provide **rollback‑first** plan
  - pre‑check actual SSH port (never assume 22)
- For secrets:
  - never print tokens/passwords
  - only document location + permission expectations

## Token discipline

- Read `docs/README.md` first if you are unsure where information lives.
- Prefer reading the **minimum** doc section + the **specific** scripts referenced by `docs/02-MODULES-INDEX.md`.
- Avoid repo‑wide scans unless strictly required.

