# Bug Triage Index -- WPTangToc OLS

Muc tieu: cho AI Agent moi vao project co duong vao nhanh khi gap bug, khong phai doc lai toan repo truoc khi khoanh vung.

Nguyen tac:

- Khoanh bug theo `impact layer` truoc, khong khoanh theo ten script.
- Uu tien `runtime truth` (`/etc/wptt/*`, `/usr/local/lsws/*`, `/etc/systemd/system/*`) neu dang debug production.
- Moi triage phai co 4 dau ra: `entrypoint/module`, `state bi tac dong`, `verify`, `rollback`.

---

## 1) Thu tu triage mac dinh

1. Xac dinh bug thuoc nhom nao:
   - menu/UI dispatch
   - WordPress/site-level
   - OLS/vhost/config
   - PHP/runtime
   - DB
   - cache/performance
   - SSL/domain
   - SSH/login/SFTP
   - firewall/fail2ban/nftables/XDP
   - cron/systemd/background jobs
2. Doc file docs chinh:
   - `00-OVERVIEW.md`
   - `01-ARCHITECTURE-FLOW.md`
   - section lien quan trong `02-MODULES-INDEX.md`
3. Neu bug co side effect runtime:
   - doc them `05-RUNBOOKS.md`
   - doc inventory `09/10/11/12` neu dung cron/systemd/shell hooks/root helpers
4. Moi sang source:
   - tim `wptt-*-main` lien quan
   - tim script con duoc `exec`
   - tim file state/config bi sua

---

## 2) Triage theo nhom su co

### A. Menu loi, khong vao dung chuc nang, loop sai man hinh

- **Doc truoc**:
  - `01-ARCHITECTURE-FLOW.md`
  - `02-MODULES-INDEX.md`
- **Khoanh source**:
  - `tool-wptangtoc-ols/wptangtoc`
  - `tool-wptangtoc-ols/wptt-*-main`
- **Can kiem**:
  - mapping `case`/`options[]`
  - script duoc `exec` co ton tai o runtime khong
  - co sai ten script/menu index khong
- **Verify**:
  - vao lai menu, chon dung option, xac nhan dispatch toi script mong doi
- **Rollback**:
  - revert mapping menu/dispatched path

### B. Website down, redirect sai, loi rewrite, loi static asset

- **Doc truoc**:
  - `02-MODULES-INDEX.md` sections `Domain / Vhost`, `Webserver Configuration`, `Security`
  - `05-RUNBOOKS.md`
- **Khoanh runtime/state**:
  - `/usr/local/lsws/conf/httpd_config.conf`
  - `/usr/local/lsws/conf/vhosts/<domain>/<domain>.conf`
  - `/usr/local/lsws/<domain>/html/.htaccess`
- **Can kiem**:
  - domain co map dung vao vhost/docroot khong
  - `.htaccess` co marker WPTT/bao mat/cache rule moi them khong
  - OLS reload/restart da an config moi chua
- **Verify**:
  - `curl -I` host header dung
  - log OLS/error log theo domain
- **Rollback**:
  - revert vhost/global config hoac `.htaccess` marker moi nhat

### C. Loi PHP, site chay sai version, extension thieu, upload fail

- **Doc truoc**:
  - `02-MODULES-INDEX.md` section `PHP`
  - `05-RUNBOOKS.md`
- **Khoanh runtime/state**:
  - config OLS php handler
  - `php.ini` cua version dang dung
  - state file `/etc/wptt/vhost/.<domain>.conf`
- **Can kiem**:
  - domain dang tro toi `lsphpXX` nao
  - CLI PHP va OLS PHP co lech nhau khong
  - co script restore php.ini da chay gan day khong
- **Verify**:
  - `php -v`, phpinfo theo domain, upload test, extension list
- **Rollback**:
  - tra lai php version/ini cu, restart OLS/PHP service lien quan

### D. Loi DB, mat ket noi DB, restore sai, wipe nham

- **Doc truoc**:
  - `02-MODULES-INDEX.md` sections `Database`, `Backup & Restore`
  - `05-RUNBOOKS.md`
- **Khoanh runtime/state**:
  - `/etc/wptt/vhost/.<domain>.conf`
  - `wp-config.php`
  - MariaDB config/service
- **Can kiem**:
  - DB creds trong state file va `wp-config.php` co khop khong
  - service ten `mysql` hay `mariadb` tren runtime
  - script restore co `drop/create` DB khong
- **Verify**:
  - login DB bang creds that
  - `wp db check`
- **Rollback**:
  - restore tu backup, revert config, dong remote exposure neu co

### E. SSH, SFTP, jail, login alert bi hu

- **Doc truoc**:
  - `02-MODULES-INDEX.md` section `SSH`
  - `11-SHELL-HOOKS-INVENTORY.md`
  - `05-RUNBOOKS.md`
- **Khoanh runtime/state**:
  - `/etc/ssh/sshd_config`
  - `/root/.bashrc`
  - `/usr/local/lsws/<domain>/.bashrc`
- **Can kiem**:
  - `Match User` blocks, jail markers, shell path
  - login alert co dua vao `.bashrc` va `SSH_CLIENT` khong
  - co `sshd -t` pass truoc restart khong
- **Verify**:
  - SSH session moi
  - SFTP/chroot login thu
- **Rollback**:
  - backup `sshd_config`/`.bashrc` truoc do, restart `sshd`

### F. Block nham IP, lockout, firewall conflict

- **Doc truoc**:
  - `02-MODULES-INDEX.md` sections `IP / Fail2ban`, `Security`
  - `03-AI-AGENT-GUIDE.md`
  - `05-RUNBOOKS.md`
  - `10-SYSTEMD-INVENTORY.md`
- **Khoanh runtime/state**:
  - `/etc/fail2ban/*`
  - `/etc/nftables.conf` hoac `/etc/sysconfig/nftables.conf`
  - `/sys/fs/bpf/*`
  - unit anti-ddos trong systemd
- **Can kiem**:
  - backend active la fail2ban, nftables, CSF hay XDP
  - SSH port thuc te
  - co unit/background blocker dang chay khong
- **Verify**:
  - van con SSH access
  - IP test con bi block khong
  - `systemctl status` + ruleset/list current bans
- **Rollback**:
  - uu tien mo lockout/firewall truoc, roi moi cleanup layer sau

### G. Cron/task nen chay sai, backup/checkers khong chay

- **Doc truoc**:
  - `09-CRON-INVENTORY.md`
  - `02-MODULES-INDEX.md` section lien quan
- **Khoanh runtime/state**:
  - `/etc/cron.d/*`
  - cron script runtime trong `/etc/wptt/*`
- **Can kiem**:
  - file cron ton tai, executable, path dung
  - helper script runtime co ton tai khong
- **Verify**:
  - dry-run script
  - log output/artefacts moi tao
- **Rollback**:
  - xoa cron entry moi, khoi phuc file cu

### H. Service nen/systemd artefact gay loi sau reboot

- **Doc truoc**:
  - `10-SYSTEMD-INVENTORY.md`
  - `12-ROOT-HELPERS-INVENTORY.md`
- **Khoanh runtime/state**:
  - `/etc/systemd/system/*`
  - drop-in overrides
  - root helper da tao artefact
- **Can kiem**:
  - unit do duoc tao boi script nao
  - unit la template-supported hay legacy runtime artefact
- **Verify**:
  - `systemctl status`
  - `daemon-reload` + reboot compatibility logic neu lien quan
- **Rollback**:
  - stop/disable + remove unit/drop-in + daemon-reload

---

## 3) Bug signatures -> duong vao nhanh

| Dau hieu bug | Doc truoc | Runtime truth can mo |
|---|---|---|
| Menu chon sai script | `01`, `02` | `wptangtoc`, `wptt-*-main` |
| Site 403/500 sau hardening | `02`, `05` | `.htaccess`, vhost config, OLS logs |
| WP khong ket noi DB | `02`, `05` | `wp-config.php`, `/etc/wptt/vhost/.<domain>.conf`, DB service |
| SSH vao duoc, nhung SFTP/jail loi | `02`, `11`, `05` | `/etc/ssh/sshd_config`, per-user shell files |
| IP bi block nham | `02`, `03`, `05`, `10` | fail2ban/nftables/XDP state |
| Sau reboot mat tweak kernel/oom/systemd | `10`, `12` | systemd units/drop-ins, `kernel`, `oomscore` artefacts |
| Backup/check SSL/check domain het han khong chay | `09`, `02` | `/etc/cron.d/*`, runtime scripts |
| Doi PHP xong site loi | `02`, `05` | php.ini, php handler, state file domain |

---

## 4) Cac cau hoi AI Agent phai tra loi truoc khi sua bug

- Bug nay nam o `impact layer` nao?
- Runtime file/service/state nao dang la source of truth?
- Script nao tao ra thay doi do?
- Verify thanh cong bang dau hieu nao?
- Rollback toi thieu la gi neu bug nang hon?

Neu chua tra loi duoc 5 cau hoi nay, agent chua nen sua.
