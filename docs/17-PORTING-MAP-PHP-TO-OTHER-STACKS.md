# Porting Map -- PHP/WordPress/OLS To Other Stacks

Muc tieu: giup AI Agent map logic tu `wptangtoc-ols` sang stack khac nhu Node.js, Python, Go, Java, hoac app custom tren VPS production.

File nay khong de "dich tung lenh shell". No de map y nghia van hanh.

---

## 1) Mapping tong quat

| Trong WPTangToc OLS | Y nghia generic | Node.js / stack khac thuong tuong ung |
|---|---|---|
| WordPress site | mot app instance | Node app / Python app / Go binary service |
| OLS vhost | site routing definition | Nginx server block / Caddy site / Traefik router |
| OLS listener | frontend listener | reverse proxy listener / ingress entrypoint |
| `wp-config.php` | app config + DB creds | `.env`, config file, secret mount |
| `lsphpXX` | runtime version binding | Node version, Python venv, JDK version, Go binary build target |
| `php.ini` | runtime tuning | Node process flags, PM2/ecosystem config, env vars, uvicorn/gunicorn settings |
| `.htaccess` | app/proxy rewrite and security rules | Nginx rewrite, middleware, reverse-proxy security rules |
| wp-cli | app-aware admin CLI | app management CLI / npm scripts / custom admin command |
| MariaDB per-site DB | app data store | PostgreSQL/MySQL/Redis/Mongo per app |
| cron scripts | scheduled automation | cron, systemd timers, queue workers, scheduler service |
| systemd units | service supervision | systemd, PM2, Supervisor, Docker compose restart policy |
| SSH/SFTP per-site user | scoped operator access | deploy user, app user, restricted shell/container exec |

---

## 2) Porting theo capability

### 2.1 Provision app

- **PHP/WordPress hien tai**:
  - tao domain, docroot, DB, config, install app
- **Node.js tuong ung**:
  - tao app directory
  - tao env file
  - install dependencies
  - tao process definition (`systemd`, `pm2`, `docker compose`)
  - map reverse proxy
- **Porting rule**:
  - giu lai flow `provision -> bind runtime -> bind domain -> verify -> rollback`
  - bo phu thuoc `wp-cli`, thay bang app-specific bootstrap command

### 2.2 Runtime version switching

- **PHP hien tai**:
  - per-domain PHP version
- **Node.js tuong ung**:
  - per-app Node version (`nvm`, `fnm`, package manager, distro package`)
  - hoac container image version
- **Porting rule**:
  - van can tach `global default runtime` va `per-app runtime`
  - verify ca CLI version va process runtime that

### 2.3 Config editing

- **PHP/OLS hien tai**:
  - sua `php.ini`, vhost config, `.htaccess`
- **Node.js tuong ung**:
  - sua `.env`, process config, reverse proxy config, app settings file
- **Porting rule**:
  - van phai co backup config, syntax/health test, restart controlled
  - khong port syntax; chi port quy trinh verify/rollback

### 2.4 Backup / restore / clone / migrate

- **PHP hien tai**:
  - dump/import DB, zip/rsync code, search-replace URL
- **Node.js tuong ung**:
  - dump/import DB
  - backup persistent volume/uploads
  - copy app artefacts/build
  - remap env/domain/origin URL
- **Porting rule**:
  - khong gan restore voi WordPress-specific assumptions
  - giu nguyen tu duy `data plane + app plane + routing plane`

### 2.5 Security hardening

- **PHP hien tai**:
  - `.htaccess`, permissions, fail2ban, nftables, XDP
- **Node.js tuong ung**:
  - reverse proxy hardening
  - app middleware hardening
  - process user isolation
  - host firewall / IDS / rate limiting
- **Porting rule**:
  - tach 3 lop:
    - app-layer
    - proxy-layer
    - host/network-layer
  - rollback-first van la bat buoc

### 2.6 Cache and performance

- **PHP hien tai**:
  - object cache, page cache, preload, Redis, disk tuning
- **Node.js tuong ung**:
  - Redis cache, CDN cache, reverse proxy cache, warmup requests, process tuning
- **Porting rule**:
  - giu logic "turn on cache -> warm cache -> verify headers/perf -> rollback if stale/broken"

### 2.7 Scheduled jobs and background workers

- **PHP hien tai**:
  - cron + systemd services
- **Node.js tuong ung**:
  - cron jobs, queue workers, PM2 processes, systemd timers/services
- **Porting rule**:
  - moi job/service phai co inventory, source-of-truth path, verify and disable path

### 2.8 Admin/operator access

- **PHP hien tai**:
  - SSH user, SFTP jail, web admin tools
- **Node.js tuong ung**:
  - deploy user, app logs access, restricted shell, dashboard, admin route
- **Porting rule**:
  - khong copy nguyen SFTP jail neu stack moi khong can
  - giu nguyen nguyen tac least privilege + operator audit path

---

## 3) Nhung phan co the copy gan nguyen ven

- control plane chia theo module
- state-driven operations
- inventories cho cron/systemd/shell hooks
- verify and rollback contract
- separate global state vs per-app state
- backup-first cho destructive ops
- runtime truth uu tien hon template truth

## 4) Nhung phan phai viet lai theo stack moi

- webserver/proxy syntax
- runtime manager logic
- app install/bootstrap steps
- app-specific CLI commands
- file permissions/layout
- health checks dac thu app

---

## 5) Porting checklist cho AI Agent

Khi clone 1 module sang stack moi, agent phai tra loi:

1. Day la capability nao trong `16`?
2. Trong stack moi, source of truth la file/service/state nao?
3. Thay cho `wp-config.php`, `.htaccess`, `lsphpXX`, OLS vhost` bang cai gi?
4. Verify thanh cong bang app-specific signal nao?
5. Rollback toi thieu co the lam trong production la gi?

Neu chua tra loi duoc 5 cau hoi nay, chua nen copy logic module do.

---

## 6) Vi du map nhanh sang Node.js production tren VPS

| Bai toan trong WPTT | Node.js phien ban tuong ung |
|---|---|
| Tao website WordPress moi | Tao Node app moi + `.env` + install deps + `systemd`/`pm2` + reverse proxy |
| Doi PHP version theo domain | Doi Node runtime theo app hoac deploy image version moi |
| Sua `.htaccess` | Sua Nginx/Caddy config hoac middleware app |
| Backup web + DB | Backup app files can thiet + upload dir + DB + env/secret refs |
| Khoi phuc website | Rehydrate app artefacts + env + DB + restart service |
| Preload cache | warmup HTTP endpoints / CDN / SSR caches |
| Khoa IP/fail2ban | reverse proxy deny rules + fail2ban/firewall host |

Ket luan: clone logic dung la clone capability + verify/rollback model, khong phai clone ten script.
