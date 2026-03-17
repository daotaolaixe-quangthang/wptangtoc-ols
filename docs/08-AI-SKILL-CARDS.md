# AI Skill Cards — WPTangToc OLS (portable for Claude/Codex/Gemini)

Mục tiêu: biến “rules + flow” thành các **SOP (standard operating procedures)** để AI Agent làm đúng ngay, ít token sai.

Mỗi skill card có chung format:

- **Trigger**: khi nào dùng
- **Inputs needed**: cần user cung cấp gì (tối thiểu)
- **Allowed reads**: file allowlist để tránh đọc lan man
- **Steps**: thao tác theo flow (token‑efficient)
- **Output contract**: Context → Impact layer → State changes → Rollback → Verify
- **DoD (Definition of Done)**: tiêu chí hoàn thành
- **Rollback hooks**: gợi ý rollback‑first nếu rủi ro cao

> Luôn đọc `AGENTS.md` + `docs/README.md` trước nếu chưa rõ scope.

---

## Skill 01 — Triage đúng scope (template vs runtime)

- **Trigger**: user nói “maintain/fix bug/đang lỗi production/không hoạt động”.
- **Inputs needed**:
  - đây là task cho **repo template** hay **VPS runtime**?
  - nếu runtime: user có cung cấp nội dung file `/etc/wptt/*` liên quan không?
- **Allowed reads**:
  - `AGENTS.md`
  - `docs/00-OVERVIEW.md` (phần nguyên tắc repo vs runtime)
- **Steps**:
  - xác định target: template/docs vs runtime debug
  - nếu user chỉ có repo: giải thích giới hạn và làm theo template
  - nếu user cung cấp runtime snippets: bám runtime làm root-cause, đồng thời note chỗ template cần cập nhật
- **Output contract**: nêu rõ “đang làm template hay runtime”
- **DoD**:
  - không nhầm đường dẫn `/etc/wptt/*` với `tool-wptangtoc-ols/*`
  - không đề xuất thay đổi production khi chỉ có template (trừ khi user yêu cầu)

---

## Skill 02 — Docs-only completion (chốt module inventory)

- **Trigger**: user yêu cầu “hoàn thiện tài liệu module X / đọc code rồi điền docs”.
- **Inputs needed**:
  - module/entrypoint cần làm (`wptt-xxx-main` hoặc script con)
- **Allowed reads**:
  - `docs/02-MODULES-INDEX.md` (đúng section)
  - `tool-wptangtoc-ols/<module>/*` (chỉ scripts xuất hiện trong section)
  - `docs/01-ARCHITECTURE-FLOW.md` (nếu cần impact layers/state conventions)
  - `docs/09-CRON-INVENTORY.md`, `docs/10-SYSTEMD-INVENTORY.md` (nếu module tạo artefact runtime)
  - `docs/11-SHELL-HOOKS-INVENTORY.md`, `docs/12-ROOT-HELPERS-INVENTORY.md` (nếu module đụng shell hooks/helper root)
- **Steps**:
  - đọc section module hiện có → xác định coverage còn thiếu hoặc chỗ docs chưa decision-complete
  - đọc scripts liên quan (entrypoint + sub-scripts)
  - chuẩn hoá mô tả theo: Context / Impact layer / State changes / Rollback / Verify / Pitfalls
  - cập nhật link nếu thêm file docs mới (docs index)
- **Output contract**:
  - nêu rõ state files/paths và rollback cụ thể
- **DoD**:
  - section module không còn khoảng mù cho phần vừa làm
  - có rollback + verify
  - không lộ secrets

---

## Skill 03 — Bugfix 1 module (code + docs sync)

- **Trigger**: user báo “script X lỗi / hành vi sai / edge case”.
- **Inputs needed**:
  - script path cụ thể
  - expected vs actual behaviour
  - môi trường OS (Ubuntu vs RHEL-like) nếu bug phụ thuộc distro
- **Allowed reads**:
  - `docs/02-MODULES-INDEX.md` (section module)
  - script bị lỗi + helper scripts mà nó source/exec
- **Steps**:
  - tái hiện logic bằng cách đọc code (không đoán)
  - xác định impact layer + state changes
  - đề xuất patch nhỏ nhất (idempotent, giữ conventions)
  - cập nhật `docs/02-MODULES-INDEX.md` nếu behaviour thay đổi hoặc docs đang sai
- **Output contract**: kèm rollback & verify
- **DoD**:
  - patch không phá pattern menu/index/exec
  - docs phản ánh đúng behaviour mới

---

## Skill 04 — Firewall/anti‑DDoS change (safe-by-default)

- **Trigger**: bật/tắt/tune CSF/nftables/SYNPROXY/XDP/modsecurity related networking rules.
- **Inputs needed**:
  - SSH port hiện tại
  - IP quản trị (whitelist/allowlist)
  - mục tiêu: enable/disable/adjust
- **Allowed reads**:
  - `docs/03-AI-AGENT-GUIDE.md` (firewall rules)
  - `docs/05-RUNBOOKS.md` (rollback-first)
  - `docs/02-MODULES-INDEX.md` (Security section liên quan)
  - `docs/10-SYSTEMD-INVENTORY.md`
  - `docs/11-SHELL-HOOKS-INVENTORY.md` (nếu đổi SSH/login hooks)
  - scripts trong `tool-wptangtoc-ols/bao-mat/**` liên quan
- **Steps**:
  - pre-check: SSH port, current firewall layer active
  - plan rollback-first: mở SSH trước/disable rule trước
  - chỉ thay đổi 1 layer mỗi lần
  - verify: còn SSH access + service status
- **Output contract**:
  - 반드시 có “risk lockout” + rollback-first
- **DoD**:
  - có rollback sequence rõ (ưu tiên layer firewall)
  - không flush ruleset/close SSH mà không có escape hatch

---

## Skill 05 — Secrets handling (document without leaking)

- **Trigger**: bất cứ khi nào task đụng Telegram/Cloudflare/rclone/passwords.
- **Inputs needed**: none (nhưng có thể hỏi nơi user lưu secrets)
- **Allowed reads**:
  - `docs/03-AI-AGENT-GUIDE.md` (secrets section)
  - module docs tương ứng trong `docs/02-MODULES-INDEX.md`
  - `docs/11-SHELL-HOOKS-INVENTORY.md` nếu secret/alert đi qua shell hooks
- **Steps**:
  - chỉ ghi location + permission expectations
  - không in token/password, không copy vào docs
  - nếu cần example: dùng placeholder `<TOKEN>` `<CHAT_ID>`
- **DoD**:
  - không có secret literal trong commit/docs

---

## Skill 06 — Database destructive ops (wipe/drop/restore/remote)

- **Trigger**: wipe/drop DB, restore site, remote DB exposure, change password.
- **Inputs needed**:
  - domain + xác nhận backup hiện có hay chưa
  - mục tiêu cụ thể (wipe vs restore vs open remote)
- **Allowed reads**:
  - `docs/02-MODULES-INDEX.md` sections: Database + Backup & Restore
  - scripts trong `tool-wptangtoc-ols/db/*` và/hoặc `tool-wptangtoc-ols/backup-restore/*`
  - `docs/05-RUNBOOKS.md`
  - `docs/09-CRON-INVENTORY.md` nếu task có auto backup / retention
- **Steps**:
  - bắt buộc cảnh báo data loss
  - backup-first hoặc ghi rõ “không có rollback dữ liệu”
  - nếu remote DB: safe-by-default (allowlist IP), rollback-first (đóng port trước)
- **DoD**:
  - có rollback thực tế (restore từ backup / close port / revert config)

---

## Skill 07 — Shell hooks / login-path changes

- **Trigger**: task đụng `.bashrc`, alert login SSH, SFTP jail, `sshd_config`, user shell riêng domain.
- **Inputs needed**:
  - user/domain bị ảnh hưởng
  - đây là root shell, per-site shell hay SSH/SFTP path
- **Allowed reads**:
  - `docs/11-SHELL-HOOKS-INVENTORY.md`
  - `docs/02-MODULES-INDEX.md` sections: SSH, Domain / Vhost
  - `docs/05-RUNBOOKS.md`
  - scripts trong `tool-wptangtoc-ols/domain/*`, `tool-wptangtoc-ols/ssh/*`, `tool-wptangtoc-ols/bao-mat/*`
- **Steps**:
  - xác định artefact đích: `/root/.bashrc`, `/usr/local/lsws/<domain>/.bashrc`, hay `/etc/ssh/sshd_config`
  - backup file trước khi sửa
  - nếu sửa SSH config: luôn `sshd -t` trước restart
  - nếu sửa shell hook: verify bằng shell interactive mới
- **Output contract**:
  - nêu rõ hook path, state changes, verify, rollback
- **DoD**:
  - không làm hỏng login path
  - nếu đụng `sshd_config`, có verify syntax và rollback rõ

---

## Skill 08 — Root helpers / installer artefacts

- **Trigger**: task đụng helper root-level như `kernel`, `oomscore`, `monit`, `disk*`, `proxy-setup`, `api-preload-wptangtoc`.
- **Inputs needed**:
  - helper script cụ thể
  - đây là phân tích docs hay debug runtime artefact
- **Allowed reads**:
  - `docs/12-ROOT-HELPERS-INVENTORY.md`
  - `docs/01-ARCHITECTURE-FLOW.md`
  - `docs/09-CRON-INVENTORY.md`, `docs/10-SYSTEMD-INVENTORY.md` nếu helper tạo cron/unit
  - script helper tương ứng trong root `tool-wptangtoc-ols/*`
- **Steps**:
  - xác định helper có side effect ở layer nào
  - map helper → runtime files changed → verify → rollback
  - nếu helper không đi qua menu main, phải ghi rõ điều đó trong output
- **DoD**:
  - không nhầm helper root-level với module menu chuẩn
  - có đủ state/rollback để maintainer debug runtime artefact

---

## Skill 09 — Task/Phase execution loop (plan approval workflow)

- **Trigger**: mọi task/phase > nhỏ lẻ.
- **Inputs needed**:
  - user goal + constraints + priority
- **Allowed reads**:
  - `AGENTS.md`
  - `docs/07-AI-TUTORIAL-STEP-BY-STEP.md` (Step 0)
  - `docs/04-ROADMAP-PHASE-TASKS.md` nếu task nằm trong docs/backlog
  - `docs/13-BUG-TRIAGE-INDEX.md`, `docs/14-SOURCE-TO-RUNTIME-TRACE.md`, `docs/15-KNOWN-RISKS-PATTERNS.md` nếu task là fix bug / review / maintain
  - `docs/16-PLATFORM-AGNOSTIC-CAPABILITIES.md`, `docs/17-PORTING-MAP-PHP-TO-OTHER-STACKS.md`, `docs/18-DESIGN-PATTERNS-EXTRACTED.md` nếu task là học logic để clone/port sang stack khác
- **Steps**:
  - propose plan (scoped allowlist, risks, rollback, verify)
  - wait for approval
  - execute
  - user review → fix → re-approve
  - final report
- **DoD**:
  - user approved plan and final outcome
  - report includes files touched + rollback/verify summary

