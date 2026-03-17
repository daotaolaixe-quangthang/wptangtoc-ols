# Shell hooks inventory (runtime artefacts) — WPTangToc OLS

Mục tiêu: chuẩn hoá các artefacts runtime không phải `cron`/`systemd` nhưng vẫn ảnh hưởng trực tiếp tới UX, alerts, login shell và SSH/SFTP.

> Lưu ý: đây là inventory cho runtime sau cài đặt. Repo là template; runtime thực tế nằm ở `/root/.bashrc`, `/usr/local/lsws/<domain>/.bashrc`, `/etc/ssh/sshd_config`.

## 1) `/root/.bashrc`

- **Nguồn tạo**:
  - installer OS-specific: `wptangtoc-ols-ubuntu`, `wptangtoc-ols-almalinux*`
  - một số flow migrate/rebuild như `chuyen-vps`
- **Hook chính đã xác minh**:
  - `[[ $- != *i* ]] && return`
  - `alias 1='wptangtoc'`
  - `alias 11='wptangtoc'`
  - `alias 00='. /etc/wptt/search-wptangtoc'`
  - `alias 99='. /etc/wptt/wptt-update'`
  - `alias 999='. /etc/wptt/wptt-update2 999'`
  - `. /etc/wptt/wptt-status`
  - `. /etc/wptt/wptt-check`
  - `cd /wptangtoc-ols`
- **Side effects**:
  - vào shell root interactive sẽ hiện dashboard/status
  - alert login SSH qua Telegram phụ thuộc một phần vào hook shell này
  - alias vận hành nhanh phụ thuộc `.bashrc`, không phải binary thật
- **Rollback tối thiểu**:
  - backup rồi remove đúng các dòng WPTT append vào `/root/.bashrc`
  - mở shell mới để verify hook không còn chạy
- **Pitfalls**:
  - đây là hook interactive shell, không bắt mọi kiểu login non-interactive
  - nếu xoá bừa `. /etc/wptt/wptt-status` có thể làm mất login alert/dashboard nhưng không ảnh hưởng service nền

## 2) Per-site `.bashrc`

- **Path runtime**:
  - `/usr/local/lsws/<domain>/.bashrc`
- **Nguồn tạo**:
  - installer user-mode hoặc flow tạo/chuyển domain
  - `domain/thay-doi-domain`
- **Hook điển hình**:
  - `[[ $- != *i* ]] && return`
  - `alias 1='wptangtoc-user'`
  - `alias 11='wptangtoc-user'`
  - `. /etc/wptt-user/wptt-status`
- **Mục đích**:
  - cung cấp shell UX riêng cho user/domain
- **Rollback tối thiểu**:
  - backup rồi remove block WPTT, hoặc xoá file `.bashrc` per-site nếu chỉ dùng cho WPTT user

## 3) SSH/SFTP jail markers trong `/etc/ssh/sshd_config`

- **Nguồn tạo**:
  - `domain/wptt-khoi-tao-user`
  - installer OS-specific khi bật user/domain riêng
- **Pattern marker**:
  - `#begin-WPTT_JAIL_<user>`
  - `Match User <user>`
  - `ChrootDirectory ...`
  - `ForceCommand internal-sftp`
  - `#end-WPTT_JAIL_<user>`
- **State changes liên quan**:
  - sửa trực tiếp `/etc/ssh/sshd_config`
  - thêm `Subsystem sftp internal-sftp` nếu cần
- **Verify**:
  - `sshd -t`
  - chỉ restart `sshd` sau khi syntax OK
  - grep lại marker block đúng user
- **Rollback tối thiểu**:
  - remove block theo marker:
    - `sed -i -e "/^#begin-WPTT_JAIL_<user>/,/^#end-WPTT_JAIL_<user>$/d" /etc/ssh/sshd_config`
  - `sshd -t`
  - `systemctl restart sshd`
- **Pitfalls**:
  - sai syntax có thể lockout SSH/SFTP
  - một số script có nhánh cleanup legacy theo pattern khác (`# WPTT_JAIL_<user>`)

## 4) Hook login alert SSH

- **Script liên quan**:
  - `bao-mat/thong-bao-login-ssh`
  - `. /etc/wptt/wptt-status`
- **Đặc điểm**:
  - đây là shell-driven hook, không phải PAM/auditd
  - thường dựa vào shell root interactive và biến `SSH_CLIENT`
- **Ý nghĩa vận hành**:
  - nếu user nói “không thấy alert login SSH”, phải kiểm tra cả `.bashrc` lẫn config Telegram trong `.wptt.conf`
- **Rollback tối thiểu**:
  - remove flag/line tương ứng trong `.wptt.conf`
  - remove source hook nếu muốn tắt hoàn toàn đường alert shell-based

## 5) Checklist khi đụng shell hooks

- **Pre-check**:
  - xác định hook nằm ở `/root/.bashrc`, per-site `.bashrc` hay `/etc/ssh/sshd_config`
  - backup file trước khi sửa
- **Verify**:
  - shell mới có còn source hook không
  - `sshd -t` nếu sửa SSH config
- **Rollback**:
  - restore file backup
  - restart `sshd` nếu có sửa `sshd_config`
