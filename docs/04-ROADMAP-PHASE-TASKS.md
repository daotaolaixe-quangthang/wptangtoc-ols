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

- [x] **Cron inventory**: đã chuẩn hoá trong `docs/09-CRON-INVENTORY.md`.
  - Việc còn lại: rà thêm coverage cho helper dùng `crontab -l` và cron tự xoá setup.
- [x] **Systemd inventory**: đã chuẩn hoá trong `docs/10-SYSTEMD-INVENTORY.md`.
  - Việc còn lại: verify các unit legacy/unknown như `ddos-blocker-xdp-access.service`.
- [x] **Shell hooks inventory**:
  - `/root/.bashrc` (đã xác minh có hook `wptt-status` + update check)  
  - các hook per-site SFTP jail trong `/etc/ssh/sshd_config` markers `#begin-WPTT_JAIL_*`.
- [x] **Root helpers inventory**:
  - `kernel`, `oomscore`, `api-preload-wptangtoc`, `delete-preload-api`, `proxy-setup`, `disk*`, `monit`
  - chuẩn hoá trong `docs/12-ROOT-HELPERS-INVENTORY.md`

## Phase 2 — Runbook rollback (production-safe)

Mục tiêu: có playbook ngắn cho những thay đổi dễ lockout/downtime.

- [x] Firewall / anti-DDoS changes: CSF / nftables / SYNPROXY / XDP (lockout risk + rollback-first).
- [x] OLS global config changes: ModSecurity, listener cert, WebAdmin port open/close.
- [x] DB high-impact: remote DB exposure, wipe/drop, version change/rebuild.
- [x] Backup/restore high-impact: restore overwrites docroot + drop/create DB.
- [x] SSH / sshd_config changes: port, timeout, chroot/SFTP user, login alert hook.
- [x] PHP / php.ini changes: per-domain, all-server, editor path, OLS restart.

## Phase 3 — “Conversation-ready” entrypoints

Mục tiêu: AI Agent chỉ cần hỏi 2-3 câu là đủ làm task đúng luồng.

- [x] Chuẩn hoá các module còn thiếu coverage chính:
  - SSH
  - IP / Fail2ban
  - PHP
  - Webserver Configuration
  - Add-ons / Premium
- [~] Chuẩn hoá mỗi module trong `02-MODULES-INDEX.md` có:
  - pre-check checklist (đặc biệt firewall/db/restore),
  - verify steps (post-change),
  - rollback steps ngắn gọn, truy được từ source.
  - Trạng thái hiện tại: các cụm rủi ro cao đã có; các module ít rủi ro hơn sẽ được làm dần nếu phát sinh maintain tiếp theo.

## Phase 4 — Enforcement (tuỳ chọn)

- [x] Tạo `AGENTS.md` ở root (AI must-read).
- [x] Tạo `.cursor/rules/*` để enforce:
  - repo là template; runtime là reference
  - mọi thay đổi phải update docs
  - firewall/anti-ddos phải safe-by-default

## Phase 5 — Cleanup TODO/gap

- [x] Đóng TODO trong `docs/01-ARCHITECTURE-FLOW.md` về rollback story.
- [x] Đóng TODO trong `docs/03-AI-AGENT-GUIDE.md` về minimum knowledge và pitfalls.
- [x] Đồng bộ `docs/README.md` với inventory mới.
- [~] Giữ lại các mục “legacy/unknown” chỉ khi không xác minh được từ template source hiện tại.

