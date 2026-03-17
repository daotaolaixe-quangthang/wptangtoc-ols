# WPTangToc OLS — AI Agent Guide (Rules, Skills, Maintain Playbook)

## Mục tiêu

Tài liệu này “nói trực tiếp với AI Agent / người maintain” để:

- Luôn bám đúng **luồng kiến trúc**.
- Tránh sửa sai lớp (OLS config vs `.htaccess` vs filesystem vs system firewall).
- Chốt **rules**: thay đổi phải có mapping, phải có rollback, phải cập nhật docs.

---

## 1) Ground rules (bắt buộc)

### Không phá vỡ runtime assumptions

- WPTangToc OLS sau cài đặt chạy dựa vào `/etc/wptt/*` và OLS ở `/usr/local/lsws/*`.
- Repo `tool-wptangtoc-ols/*` chủ yếu là template; có thể **không trùng** với runtime nếu installer remote đã cập nhật.

**Rule**: Khi sửa logic vận hành thực tế, phải xác định mục tiêu:

- Sửa template trong repo để lần cài sau đúng?
- Hay sửa runtime trên VPS hiện tại?

(Trong project repo này, ưu tiên là **template + tài liệu**; runtime là bối cảnh triển khai.)

### Mọi thay đổi phải có 4 phần

- **Context**: module nào, entrypoint nào.
- **Impact layer**: OLS global / `.htaccess` / filesystem / system services / cron / cloud.
- **State changes**: ghi vào `.wptt.conf` hay `vhost/.$domain.conf` hay marker `.htaccess`?
- **Rollback plan**: revert file nào, restart service nào.

### Thay đổi firewall / anti-DDOS phải “safe-by-default”

Các script security có thể:

- **Mask/disable iptables/firewalld** và bật `nftables`.
- **Flush ruleset** (`nft flush ruleset`) → rủi ro tự khóa SSH nếu sai.
- Chỉnh `fail2ban` banaction.
- Đụng sysctl conntrack, SYNPROXY, XDP.

**Rule**:

- Bất kỳ thay đổi nào liên quan firewall/anti-ddos phải:
  - có bước **pre-check** port SSH hiện tại (không assume 22),
  - ghi rõ **file config** đụng tới (`/etc/nftables.conf` hoặc `/etc/sysconfig/nftables.conf`, `/etc/fail2ban/jail.local`, `/etc/csf/csf.conf`),
  - có **rollback script/path** (ví dụ `remove-nftables.sh`),
  - mô tả “**risk of lockout**” trong docs.

### XDP (eBPF) — rules bắt buộc khi chạm vào (rủi ro cao)

XDP là lớp **kernel/eBPF**. Sai 1 bước có thể chặn nhầm traffic hoặc làm mất kết nối quản trị.

**Rule**:

- Trước khi attach/detach XDP:
  - xác định đúng **NIC** theo default route (không assume `eth0`)
  - whitelist **IP quản trị** vào map whitelist (nếu có cơ chế whitelist)
  - có lệnh verify ngay sau khi attach: `bpftool net list dev <IFACE>`
- Khi script có tác động SELinux:
  - mọi thay đổi như `setenforce 0` / sửa `/etc/selinux/config` phải được ghi rõ trong docs (state change) và có rollback.
- Khi script ghi ban timestamp vào BPF map:
  - ghi rõ “clock domain” (monotonic vs realtime) và cơ chế cleanup để tránh map phình/không tự unban.
- Khi triển khai “auto enable/disable”:
  - phải nêu rõ side effect nếu có **reboot** (không được coi là hành vi “ngầm định”).

### Secrets handling (Cloudflare / Drive / Telegram)

Các tính năng có thể lưu secrets ở:

- `/etc/wptt/cloudflare-cdn-cache/.wptt-$domain.conf` (Cloudflare Global API Key)
- `/root/.config/rclone/rclone.conf` (Drive token)
- `/etc/wptt/.wptt.conf` (Telegram token/chat_id)

**Rule**:

- Không log secrets ra stdout/log.
- Docs phải đánh dấu rõ “**secret location**” + permission expectations.

### Không hard-code text UI

- Text hiển thị nên nằm ở `/etc/wptt/lang/<locale>.sh` (hoặc template lang trong repo).
- Menu main/module dùng biến ngôn ngữ thay vì string trần (trừ trường hợp đặc biệt).

---

## 2) Conventions cần giữ ổn định

### Menu framework

- Menu chính: `wptangtoc` (options + run_action + exec)
- Menu module: `wptt-*-main` (options + run_action + callback)
- Back-to-main: `wptt-callback-menu-chon` dùng `exec /usr/bin/wptangtoc 1`

**Rule**: Không đổi kiểu index (1-based/0-based) lẫn lộn. Nếu đổi, phải update đồng bộ:

- `options[]`
- `run_action(index)` mappings
- `wptt-callback-menu-chon` input handling

### State & markers

Mặc định:

- Global config: `/etc/wptt/.wptt.conf`
- Per-site config: `/etc/wptt/vhost/.$domain.conf`
- Per-site `.htaccess` marker: `#begin-...` / `#end-...`
- Audit log: `/var/log/wptangtoc-ols.log`

**Rule**: Nếu thêm tính năng bật/tắt, phải có **marker/state** để kiểm tra idempotent.

---

## 3) Skills/Capabilities mà Agent cần (tài liệu hoá)

Khi maintain WPTangToc OLS, AI Agent cần biết/nhớ các khả năng sau:

- **Bash scripting**: `case`, `select`, `exec`, quoting, idempotency.
- **Linux services**: `systemctl`, restart/reload, logs.
- **OpenLiteSpeed**: cấu trúc config, vhost/docroot, restart safe.
- **WordPress ops**: wp-cli, wp-config.php parsing, backup/restore.
- **Security ops**:
  - ModSecurity/CRS tuning,
  - `.htaccess` rulesets (8G, bot rules),
  - filesystem hardening (chmod/chattr),
  - system firewall (CSF/nftables/XDP),
  - alerting (Telegram).
- **Backup**: zip/tar, mysqldump/mariadb, rclone.

### Minimum knowledge bắt buộc cho cụm rủi ro cao

- **Firewall / fail2ban / nftables / XDP**:
  - biết phân biệt firewall backend đang active là gì
  - luôn đọc SSH port thật từ `sshd_config`, không assume `22`
  - hiểu chỗ state nằm ở `httpd_config.conf`, `/etc/fail2ban/*`, `/etc/csf/*`, `/etc/nftables.conf` hoặc `/etc/sysconfig/nftables.conf`, `/sys/fs/bpf/*`
- **DB destructive ops**:
  - phân biệt create/drop/wipe/restore
  - biết state per-site nằm ở `/etc/wptt/vhost/.$domain.conf`
  - hiểu script có thể đụng DB grants, MariaDB config, firewall và fail2ban cùng lúc
- **Restore**:
  - restore thường overwrite docroot và drop/create DB
  - không có “undo” thực sự nếu chưa có backup khác
- **SSH / sshd_config / chroot**:
  - hiểu `Match User`, `Subsystem sftp internal-sftp`, marker `#begin-WPTT_JAIL_*`
  - luôn verify `sshd -t` trước khi restart nếu sửa trực tiếp config
- **OLS config vs `.htaccess`**:
  - global config ở `/usr/local/lsws/conf/httpd_config.conf`
  - per-vhost ở `/usr/local/lsws/conf/vhosts/<domain>/<domain>.conf`
  - per-site rule/redirect/cache marker ở `/usr/local/lsws/<domain>/html/.htaccess`
- **PHP multi-version**:
  - biết `lsphpXX` trên OLS, symlink CLI và `php-cli-domain-config`
  - phân biệt PHP per-domain với PHP mặc định toàn server

### Pitfalls cụ thể cần nhớ

- **fail2ban vs firewall backend mismatch**:
  - một số script đổi `banaction` theo `nftables` hoặc `firewalld`; rollback phải đổi cả backend liên quan
- **Tên service DB không đồng nhất**:
  - source có chỗ dùng `mysql.service`, chỗ khác dùng `mariadb.service`
  - docs phải ghi rõ đây là bẫy compatibility, verify theo distro/runtime thật
- **Repo template có thể lệch runtime**:
  - runtime `/etc/wptt/*` là nguồn sự thật khi debug production
- **Hook thông báo SSH login chỉ chạy qua interactive shell**:
  - vì dựa vào `.bashrc` và biến `SSH_CLIENT`, không phải PAM/auditd
- **Menu model dùng `exec`**:
  - backtracking/debug stack menu khác shell app truyền thống; script con sẽ thay thế process hiện tại
- **Premium/Add-ons có dependency runtime ngoài repo**:
  - ví dụ `add-one/check.sh` hoặc logic API/remote files
  - docs phải ghi rõ “runtime dependency”, không đoán behaviour nếu source hiện tại không chứa file đó

---

## 4) Maintain workflow (chuẩn)

### Khi sửa/ thêm 1 tính năng

- Xác định:
  - module main nào (`wptt-xxx-main`)
  - script con nào (trong `tool-wptangtoc-ols/<module>/...`)
  - state ở đâu (`.wptt.conf`, `vhost`, `.htaccess`, OLS config)
- Thiết kế:
  - idempotent (chạy nhiều lần không hỏng)
  - safe defaults
  - có confirm khi tác động lớn (firewall/hardening/global config)
- Cập nhật:
  - menu `options[]` + `run_action`
  - lang variables (nếu có UI text)
  - docs module tương ứng trong `docs/02-MODULES-INDEX.md`
  - flow nếu thay đổi luồng trong `docs/01-ARCHITECTURE-FLOW.md`

### Khi debug production issue

- Xác định lớp lỗi:
  - OLS global? `.htaccess`? PHP? DB? filesystem permissions? firewall?
- Tìm marker/state:
  - `.wptt.conf`, `vhost/.$domain.conf`, `.htaccess` markers, log `/var/log/wptangtoc-ols.log`
- Tách lỗi:
  - config sai vs tool missing vs network blocked vs permission denied.

---

## 5) “Chốt cố định” rules cho hội thoại AI (conversation rules)

Khi user yêu cầu thay đổi:

- AI phải trả lời bằng:
  - “Sửa ở module nào, file nào”
  - “Tác động lớp nào”
  - “Rollback thế nào”
  - “Cập nhật docs ở đâu”

Nếu user chỉ hỏi “tại sao/hoạt động thế nào”:

- Ưu tiên vẽ flow, chỉ ra entrypoint, state, side effects.

---

## 6) AGENTS.md / Cursor rules

Nếu muốn enforce mạnh hơn trong IDE:

- [x] `AGENTS.md` ở root (AI must-read) — đã tạo.
- [x] `.cursor/rules/*.md` để ràng buộc — đã tạo bộ rules tối thiểu.

Đã có đủ inventory nền để dùng `AGENTS.md` và `.cursor/rules/*` làm lớp guard chính.

## 7) Links (điểm vào khi maintain)

- Docs index: `docs/README.md`
- Roadmap/phase tasks: `docs/04-ROADMAP-PHASE-TASKS.md`
- Runbooks rollback: `docs/05-RUNBOOKS.md`

