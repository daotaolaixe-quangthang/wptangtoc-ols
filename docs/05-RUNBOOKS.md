# Runbooks — WPTangToc OLS (Maintain & Rollback)

Mục tiêu: khi conversation chat yêu cầu “bật/tắt/sửa” các tính năng rủi ro cao, AI/maintainer luôn làm theo cùng một khung:

**Pre-check → Change → Verify → Rollback**

## 0) Rollback checklist theo impact layer (map nhanh)

- **Firewall / anti-DDoS**: lockout risk rất cao → rollback-first phải giữ được SSH.
- **Kernel/eBPF (XDP)**: detach khỏi NIC + remove pinned maps/program + stop service.
- **System services (systemd)**: stop/disable + remove unit/drop-in + daemon-reload.
- **Cron/automation**: remove `/etc/cron.d/*` + restart `cron/crond` + remove wrapper `/etc/wptt-auto/*`.
- **OLS global**: restore `/usr/local/lsws/conf/httpd_config.conf` + restart OLS.
- **OLS per-vhost**: revert `/usr/local/lsws/conf/vhosts/<domain>/<domain>.conf` + restart OLS.
- **`.htaccess`**: remove marker blocks + clear cache.
- **Filesystem hardening** (`chmod/chown/chattr`): undo `chattr +i` trước khi sửa, rollback perms/owners.
- **Cloud integration** (Cloudflare/rclone/Telegram): revoke/disable jobs first, never leak secrets.

---

## 1) Firewall / anti-DDoS (lockout risk cao)

- **Pre-check**:
  - xác định SSH port thật (không assume 22)
  - xác định IP quản trị đang dùng (để allowlist nếu có)
  - backup file config trước khi sửa
  - nếu có console/VNC: đảm bảo có đường vào dự phòng trước khi apply ruleset lớn
- **Change**:
  - chỉ thay 1 lớp mỗi lần (CSF hoặc nftables hoặc SYNPROXY hoặc XDP), tránh “combo”
- **Verify**:
  - kiểm tra còn SSH access (mở session thứ 2 trước khi đóng session hiện tại)
  - `systemctl status` service liên quan
- **Rollback-first (khi nghi lockout)**:
  - vào console/VNC rescue (nếu cần) → tạm mở SSH port / flush ruleset / disable firewall layer vừa bật
  - chỉ khi SSH đã ổn định mới revert dần từng lớp
- **Rollback**:
  - ưu tiên rollback ở layer firewall trước (mở lại SSH port / disable rule)
  - sau đó mới revert config file và restart service

---

## 2) Kernel / eBPF (XDP) (lockout risk + khó debug)

- **Context**: XDP attach vào NIC + BPF pinned trong `/sys/fs/bpf/*` + systemd unit.
- **Pre-check**:
  - xác định interface thật: `ip link` (đừng assume `eth0`)
  - kiểm tra đang attach: `ip link show dev <iface> | grep -i xdp` hoặc `bpftool net list dev <iface>`
  - backup trạng thái: lưu output `bpftool net list dev <iface>` vào log (copy/paste)
- **Change**:
  - chỉ thay 1 thứ mỗi lần (attach/detach/service) để dễ rollback
- **Verify**:
  - `bpftool net list dev <iface>` reflect đúng (attached/detached)
  - service status: `systemctl status ddos-blocker-xdp`
- **Rollback**:
  - `systemctl stop ddos-blocker-xdp && systemctl disable ddos-blocker-xdp`
  - detach: `bpftool net detach xdp dev <iface>`
  - remove pinned files (nếu dùng): `/sys/fs/bpf/xdp_ddos_protection_prog`, `/sys/fs/bpf/log_blacklist`
  - remove unit: `/etc/systemd/system/ddos-blocker-xdp.service` + `systemctl daemon-reload`
  - clean cron/crontab helpers nếu có (vd cleanup_maps)

---

## 3) System services (systemd) — unit/drop-in

- **Pre-check**:
  - `systemctl status <unit>` + `systemctl is-enabled <unit>`
  - kiểm tra file origin:
    - unit: `/etc/systemd/system/<unit>.service`
    - drop-in: `/etc/systemd/system/<svc>.service.d/*.conf`
- **Verify**:
  - `systemctl daemon-reload` không báo lỗi
  - restart target service OK
- **Rollback (unit)**:
  - `systemctl stop <unit> && systemctl disable <unit>`
  - `rm -f /etc/systemd/system/<unit>.service`
  - `systemctl daemon-reload`
- **Rollback (drop-in)**:
  - `rm -f /etc/systemd/system/<svc>.service.d/<file>.conf`
  - `systemctl daemon-reload && systemctl restart <svc>`

> Inventory units/drop-ins: `docs/10-SYSTEMD-INVENTORY.md`.

---

## 4) Cron / automation (`/etc/cron.d/*` + wrappers)

- **Pre-check**:
  - xác định artefacts hiện có:
    - `/etc/cron.d/*` (cron file)
    - `/etc/wptt-auto/*` (wrapper scripts)
    - `crontab -l` (một số module ghi vào user crontab)
- **Verify**:
  - cron daemon restarted đúng: Ubuntu `cron`, RHEL `crond`
  - job không còn chạy lặp (check file tồn tại / log)
- **Rollback**:
  - remove cron file(s) đúng tên (ưu tiên theo inventory)
  - remove symlink `_cron` trên Ubuntu nếu có
  - remove wrappers trong `/etc/wptt-auto/*` nếu cron gọi wrapper
  - `systemctl restart cron.service` hoặc `systemctl restart crond.service`

> Inventory cron: `docs/09-CRON-INVENTORY.md`.

---

## 5) OLS global config (ModSecurity / listener cert / WebAdmin)

- **Pre-check**:
  - backup `/usr/local/lsws/conf/httpd_config.conf`
  - biết rõ port WebAdmin hiện tại và rule firewall đang mở
- **Change**:
  - thay đổi phải idempotent (marker hoặc block dễ remove)
- **Verify**:
  - `lswsctrl restart` xong phải check site response
- **Rollback**:
  - restore file backup + restart OLS

### Rollback checklist (OLS global)

- Restore `httpd_config.conf` từ backup
- `lswsctrl restart`
- Verify tối thiểu:
  - `lswsctrl status`
  - `curl -Is http://127.0.0.1 | head -n1` (trên server)
  - WebAdmin port (nếu liên quan)

---

## 6) OLS per-vhost config (domain-level)

- **Pre-check**:
  - backup: `/usr/local/lsws/conf/vhosts/<domain>/<domain>.conf`
  - xác định cert path đang dùng (LE vs self-signed) trước khi sửa
- **Verify**:
  - `lswsctrl restart` không lỗi
  - site response OK
- **Rollback**:
  - restore vhost conf backup
  - `lswsctrl restart`

---

## 7) `.htaccess` per-site rules (rewrite/security/redirect)

- **Pre-check**:
  - backup `/usr/local/lsws/<domain>/html/.htaccess`
  - xác định marker blocks (vd `#begin-...`/`#end-...`) để rollback nhanh
- **Verify**:
  - `curl -I http://<domain>` / `https://<domain>` (đúng redirect/không loop)
  - check 403/404 rules không block nhầm admin/health endpoints
- **Rollback**:
  - remove/revert marker blocks trong `.htaccess`
  - clear cache (LSCache/Redis/page-html) nếu rule liên quan cache/redirect

---

## 8) Filesystem hardening (chmod/chown/chattr)

- **Pre-check**:
  - ghi lại ownership/perms trước khi đổi (đặc biệt docroot, vhost conf)
  - nếu có LockDown: kiểm tra file đang immutable: `lsattr <file>`
- **Verify**:
  - service vẫn đọc được file (OLS/PHP), không 403 do perms
- **Rollback**:
  - nếu bị `chattr +i`: `chattr -i` trước khi restore
  - revert perms/owner về giá trị cũ (ưu tiên theo backup/ghi chú)

---

## 9) Database high-impact (remote DB / wipe/drop / version)

- **Pre-check**:
  - xác định đúng DB/site đang thao tác (state file `/etc/wptt/vhost/.$domain.conf`)
  - backup DB trước khi wipe/drop
- **Change**:
  - thao tác remote DB phải safe-by-default (allowlist IP nếu có)
- **Verify**:
  - login test bằng `mariadb` hoặc app/WP
- **Rollback**:
  - đóng port firewall trước nếu bị overexpose
  - restore DB từ backup nếu dữ liệu bị mất

---

## 10) Restore website (data loss chắc chắn)

- **Pre-check**:
  - xác định file backup đúng (root store vs user store)
  - đảm bảo không có job backup/restore đang chạy (lock file)
- **Change**:
  - restore thường drop/create DB và xoá docroot
- **Verify**:
  - check `wp-config.php` DB creds đúng
  - clear cache + restart OLS
- **Rollback**:
  - restore lại từ bộ backup khác (không có undo)

---

## 11) Cloud integration (Cloudflare / rclone / Telegram)

- **Pre-check**:
  - không in secrets; chỉ xác định location:
    - Telegram: `/etc/wptt/.wptt.conf`
    - Cloudflare API per-domain: `/etc/wptt/cloudflare-cdn-cache/.wptt-<domain>.conf`
    - rclone: `/root/.config/rclone/rclone.conf`
- **Verify**:
  - check action “idempotent” (purge/backup upload/delete) không chạy lặp
- **Rollback**:
  - ưu tiên disable automation (cron/service) trước khi rotate token
  - rotate secret ở đúng file + permission đúng (root-only)

---

## 12) IP / Fail2ban (block / unblock / unban-all)

- **Pre-check**:
  - xác định backend đang active: CSF, nftables hay firewalld
  - grep deny hiện tại trong `/usr/local/lsws/conf/httpd_config.conf`
  - nếu block nhầm IP quản trị, chuẩn bị session/console dự phòng
- **Verify**:
  - `fail2ban-client status`
  - grep rule trong backend firewall active
  - grep deny trong `httpd_config.conf`
- **Rollback**:
  - với 1 IP: dùng flow unblock
  - với trạng thái rối nhiều lớp: unban ở fail2ban trước, rồi remove firewall rule, rồi clean deny trong OLS
  - `unban-all` chỉ dùng khi chấp nhận mất toàn bộ deny runtime

---

## 13) SSH port / sshd changes

- **Pre-check**:
  - đọc port hiện tại từ `/etc/ssh/sshd_config`
  - mở sẵn session SSH thứ 2 hoặc console rescue
  - kiểm tra trùng port với OLS WebAdmin và MariaDB remote
- **Verify**:
  - `sshd -t`
  - SSH login được trên port mới
  - fail2ban/firewall đã sync theo port mới
- **Rollback**:
  - restore port cũ trong `sshd_config`
  - restore fail2ban/firewall config cũ
  - restart `sshd`, reload fail2ban

---

## 14) PHP version / php.ini changes

- **Pre-check**:
  - xác định đang sửa PHP per-domain hay toàn server
  - note version cũ và path php.ini cũ
  - backup file config trước khi edit
- **Verify**:
  - `php -v` cho all-server, hoặc `wptt-php-version-domain` cho 1 site
  - OLS restart OK
  - site/WP CLI không lỗi
- **Rollback**:
  - đổi lại version cũ bằng cùng flow
  - restore php.ini từ backup hoặc dùng script restore php.ini

---

## 15) Config editor / webserver config changes

- **Pre-check**:
  - xác định rõ layer: OLS global, vhost, `.htaccess`, MariaDB, Redis, fail2ban, cron
  - backup file đích trước khi mở editor
- **Verify**:
  - restart/reload đúng service
  - test response hoặc status command tương ứng
- **Rollback**:
  - restore file backup
  - restart/reload service đích

