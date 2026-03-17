# Roadmap & Phase Tasks — WPTangToc OLS docs

Tài liệu này là “bảng điều hướng” để:

- biết còn thiếu gì trong docs/source reading,
- ưu tiên theo rủi ro vận hành (lockout/downtime/data loss),
- giúp AI Agent trong conversation chat có thể **pick task tiếp theo** mà không bị lạc ngữ cảnh.

## Phase 0 — Nguyên tắc “đóng gói change”

Mọi thay đổi (code hoặc docs) phải ghi rõ:

- **Context**: module/entrypoint nào
- **Impact layer**: chạm lớp nào (OLS global / vhost / `.htaccess` / filesystem / systemd / cron / kernel eBPF / cloud)
- **State changes**: file/state nào thay đổi
- **Rollback**: revert thế nào + restart service nào

## Phase 1 — Hoàn thiện inventory artefacts runtime (high value)

Mục tiêu: biến kiến thức “rải rác trong module docs” thành **bảng inventory chuẩn**.

- [ ] **Cron inventory**: bảng “`/etc/cron.d/<file>` → module tạo → mục đích → rollback (xoá file + restart cron)”  
      Nguồn: `tool-wptangtoc-ols/**` (grep `/etc/cron.d/`).
- [ ] **Systemd inventory**: bảng “unit/service → module tạo → enable/disable path → rollback”  
      Nguồn: `tool-wptangtoc-ols/**` (grep `/etc/systemd/system/`, `systemctl enable/disable`).
- [ ] **Shell hooks inventory**:
  - `/root/.bashrc` (đã xác minh có hook `wptt-status` + update check)  
  - các hook per-site SFTP jail trong `/etc/ssh/sshd_config` markers `#begin-WPTT_JAIL_*`.

## Phase 2 — Runbook rollback (production-safe)

Mục tiêu: có playbook ngắn cho những thay đổi dễ lockout/downtime.

- [ ] Firewall / anti-DDoS changes: CSF / nftables / SYNPROXY / XDP (lockout risk + rollback-first).
- [ ] OLS global config changes: ModSecurity, listener cert, WebAdmin port open/close.
- [ ] DB high-impact: remote DB exposure, wipe/drop, version change/rebuild.
- [ ] Backup/restore high-impact: restore overwrites docroot + drop/create DB.

## Phase 3 — “Conversation-ready” entrypoints

Mục tiêu: AI Agent chỉ cần hỏi 2-3 câu là đủ làm task đúng luồng.

- [ ] Chuẩn hoá mỗi module trong `02-MODULES-INDEX.md` có:
  - pre-check checklist (đặc biệt firewall/db/restore),
  - verify steps (post-change),
  - rollback steps (đã có nhưng cần “ngắn & chắc”).

## Phase 4 — Enforcement (tuỳ chọn)

- [x] Tạo `AGENTS.md` ở root (AI must-read).
- [x] Tạo `.cursor/rules/*` để enforce:
  - repo là template; runtime là reference
  - mọi thay đổi phải update docs
  - firewall/anti-ddos phải safe-by-default

