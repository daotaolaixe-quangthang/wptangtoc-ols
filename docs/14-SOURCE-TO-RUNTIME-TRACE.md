# Source To Runtime Trace -- WPTangToc OLS

Muc tieu: map nhanh tu menu/entrypoint sang script con, state runtime, service, logs, verify, rollback.

Dung file nay khi:

- can debug nhanh theo luong thuc thi
- can biet sua code o dau nhung verify o runtime o dau
- can tranh doc mo rong toan repo

---

## 1) Trace model chung

`wptangtoc` -> `wptt-*-main` -> script con trong repo/runtime -> file state/config -> service/hooks/background jobs -> verify -> rollback

Ghi nho:

- Repo la template.
- Runtime that thuong nam o `/etc/wptt/*`, `/usr/local/lsws/*`, `/etc/cron.d/*`, `/etc/systemd/system/*`.
- Neu production va template mau thuan nhau, runtime la uu tien so 1.

---

## 2) Trace theo cum module chinh

### WordPress / Domain / Vhost

- **Entry chính**:
  - `wptangtoc`
  - `wptt-wordpress-main`
  - `wptt-domain-main`
- **Script con thường gặp**:
  - `wordpress/*`
  - `domain/*`
- **Runtime state chinh**:
  - `/etc/wptt/vhost/.<domain>.conf`
  - `/usr/local/lsws/conf/httpd_config.conf`
  - `/usr/local/lsws/conf/vhosts/<domain>/<domain>.conf`
  - `/usr/local/lsws/<domain>/html/wp-config.php`
- **Service / reload**:
  - OLS reload/restart
  - MariaDB khi co DB ops
- **Logs / verify**:
  - OLS error/access logs
  - `wp-cli` theo domain
  - host header curl
- **Rollback**:
  - revert vhost/global config, state file, `.htaccess`, hoac restore site/DB

### SSL / HTTPS

- **Entry chính**:
  - `wptt-ssl-main`
- **Script con thường gặp**:
  - `ssl/*`
- **Runtime state chinh**:
  - letsencrypt/live cert paths hoac self-signed paths
  - OLS listener/vhost SSL config
- **Service / reload**:
  - OLS reload/restart
- **Logs / verify**:
  - `curl -I https://domain`
  - cert dates / browser test
- **Rollback**:
  - revert cert path/config, tra lai cert truoc do neu co

### Database / Backup / Restore

- **Entry chính**:
  - `wptt-database-main`
  - `wptt-backup-restore-main`
- **Script con thường gặp**:
  - `db/*`
  - `backup-restore/*`
- **Runtime state chinh**:
  - `/etc/wptt/vhost/.<domain>.conf`
  - `wp-config.php`
  - backup artefacts
  - MariaDB config
- **Service / reload**:
  - MariaDB service
  - co the dong/mo firewall neu remote DB
- **Logs / verify**:
  - login DB bang creds that
  - `wp db check`
  - backup file ton tai va import duoc
- **Rollback**:
  - restore lai backup, revert DB config, close remote access

### Cache / Performance

- **Entry chính**:
  - `wptt-cache-main`
  - `wptt-preload-cache2`
- **Script con thường gặp**:
  - `cache/*`
  - preload helpers runtime
- **Runtime state chinh**:
  - plugin cache settings
  - `.htaccess`
  - OLS cache rules
- **Service / reload**:
  - OLS reload khi doi rule
- **Logs / verify**:
  - response headers
  - cache files/warm-cache artefacts
- **Rollback**:
  - xoa rule moi them, tat cache layer moi bat

### Security / Hardening / Firewall

- **Entry chính**:
  - `wptt-bao-mat-main`
  - `wptt-khoa-ip-main`
- **Script con thường gặp**:
  - `bao-mat/*`
  - `fail2ban/*`
  - `bao-mat/nftables/*`
- **Runtime state chinh**:
  - `.htaccess`
  - OLS config
  - `/etc/fail2ban/*`
  - `/etc/nftables.conf` hoac `/etc/sysconfig/nftables.conf`
  - `/sys/fs/bpf/*`
- **Service / reload**:
  - fail2ban
  - nftables/systemd anti-ddos units
  - OLS
- **Logs / verify**:
  - current bans/rules
  - `systemctl status`
  - SSH access con hay khong
- **Rollback**:
  - mo lockout truoc, roi revert hardening/firewall rules

### SSH / SFTP / Login path

- **Entry chính**:
  - `wptt-ssh-main`
- **Script con thường gặp**:
  - `ssh/*`
  - `domain/wptt-khoi-tao-user`
  - `bao-mat/thong-bao-login-ssh`
- **Runtime state chinh**:
  - `/etc/ssh/sshd_config`
  - `/root/.bashrc`
  - `/usr/local/lsws/<domain>/.bashrc`
- **Service / reload**:
  - `sshd`
- **Logs / verify**:
  - `sshd -t`
  - SSH session moi
  - SFTP login thu
- **Rollback**:
  - revert `sshd_config`/shell hooks, restart `sshd`

### PHP

- **Entry chính**:
  - `wptt-php-ini-main`
- **Script con thường gặp**:
  - `php/*`
- **Runtime state chinh**:
  - php.ini cua tung version
  - OLS PHP handler mapping
  - `/etc/wptt/vhost/.<domain>.conf`
- **Service / reload**:
  - OLS reload/restart
  - php service neu distro/runtime dung service tach rieng
- **Logs / verify**:
  - `php -v`
  - phpinfo / upload / extension test
- **Rollback**:
  - tra lai php version/ini cu, restore php.ini neu co

### Webserver Configuration

- **Entry chính**:
  - `wptt-cau-hinh-websever-main`
- **Script con thường gặp**:
  - `cau-hinh/*`
- **Runtime state chinh**:
  - `/usr/local/lsws/conf/httpd_config.conf`
  - `/usr/local/lsws/conf/vhosts/<domain>/<domain>.conf`
  - config MariaDB/Redis/fail2ban/cron neu edit qua menu nay
- **Service / reload**:
  - OLS
  - MariaDB/Redis/fail2ban theo file duoc sua
- **Logs / verify**:
  - syntax/config test neu co
  - service status
  - curl/functionality test
- **Rollback**:
  - khoi phuc file config backup va restart service lien quan

### Cron / Systemd / Root helpers

- **Entry chính**:
  - mot so menu main
  - hoac helper root-level khong di qua menu chinh
- **Script con thường gặp**:
  - `/etc/cron.d/*`
  - `/etc/systemd/system/*`
  - root helpers nhu `kernel`, `oomscore`, `monit`, `disk_*`
- **Runtime state chinh**:
  - cron files
  - systemd units/drop-ins
  - `/etc/sysctl.d/*`, `/etc/security/limits.conf`, `rc.local`
- **Service / reload**:
  - `systemctl daemon-reload`
  - service restart theo helper
- **Logs / verify**:
  - `systemctl status`
  - cron execution logs
- **Rollback**:
  - remove cron/unit/drop-in, restore system tuning files

---

## 3) Fast trace by artefact

| Neu thay doi o runtime | Thuong quay nguoc ve docs/source nao |
|---|---|
| `/etc/wptt/vhost/.<domain>.conf` | `02-MODULES-INDEX.md`, `domain/*`, `db/*`, `php/*`, `backup-restore/*` |
| `/usr/local/lsws/conf/httpd_config.conf` | `01`, `02` section `Webserver Configuration`, `Domain / Vhost`, `cau-hinh/*` |
| `/usr/local/lsws/conf/vhosts/<domain>/<domain>.conf` | `02` section `Domain / Vhost`, `PHP`, `Webserver Configuration` |
| `/usr/local/lsws/<domain>/html/.htaccess` | `02` section `Security`, `Cache`, `Webserver Configuration` |
| `/etc/ssh/sshd_config` | `02` section `SSH`, `11-SHELL-HOOKS-INVENTORY.md` |
| `/root/.bashrc` / per-site `.bashrc` | `11-SHELL-HOOKS-INVENTORY.md`, `SSH`, `Domain / Vhost` |
| `/etc/cron.d/*` | `09-CRON-INVENTORY.md`, module docs lien quan |
| `/etc/systemd/system/*` | `10-SYSTEMD-INVENTORY.md`, `12-ROOT-HELPERS-INVENTORY.md` |
| `/etc/sysctl.d/*`, `/etc/security/limits.conf`, `rc.local` | `12-ROOT-HELPERS-INVENTORY.md`, `kernel` helper |

---

## 4) Khi nao phai roi docs va doc source ngay

- Khi docs da ghi ro `runtime dependency` hoac `template source does not include implementation`.
- Khi bug nam trong wrapper runtime duoc `wptangtoc` dispatch truc tiep nhung repo khong co body script.
- Khi symptom va state runtime mau thuan voi template docs.
- Khi bug co dau hieu la `menu dispatch dung, nhung runtime script sai/khong ton tai`.

Neu roi docs de doc source, van phai giu format phan tich:

- entrypoint
- script con
- state
- verify
- rollback
