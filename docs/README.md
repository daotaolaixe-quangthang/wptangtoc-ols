# Docs Index — WPTangToc OLS

Mục tiêu của thư mục `docs/` là để **con người và AI Agent**:

- hiểu dự án nhanh (context + assumptions),
- maintain đúng luồng (đúng impact layer),
- khi làm thay đổi luôn có: **Context → Impact layer → State changes → Rollback**.

## Bắt đầu từ đây (đọc theo thứ tự)

- `00-OVERVIEW.md`: project context + nguyên tắc vận hành (repo template vs runtime).
- `01-ARCHITECTURE-FLOW.md`: kiến trúc 3 lớp + pattern menu + nơi lưu state + inventory bootstrap.
- `02-MODULES-INDEX.md`: inventory từng module theo entrypoint/script/state/side-effects/rollback.
- `03-AI-AGENT-GUIDE.md`: rules/skills/conventions bắt buộc cho AI Agent & maintainer.

## Các file “operational”

- `04-ROADMAP-PHASE-TASKS.md`: roadmap + phase tasks để hoàn thiện docs/source reading + maintain backlog.
- `05-RUNBOOKS.md`: playbook ngắn “pre-check → change → verify → rollback” cho các impact layer rủi ro cao.
- `06-AI-PROMPT-TEMPLATES.md`: prompt templates (token‑efficient, đúng context).
- `07-AI-TUTORIAL-STEP-BY-STEP.md`: tutorial step-by-step để AI làm đúng flow.
- `08-AI-SKILL-CARDS.md`: skill cards (SOP portable) để AI làm đúng ngay và ít token sai.
- `09-CRON-INVENTORY.md`: inventory cron runtime artefacts (`/etc/cron.d/*` + rollback nhanh).
- `10-SYSTEMD-INVENTORY.md`: inventory systemd runtime artefacts (`/etc/systemd/system/*` + rollback nhanh).
- `11-SHELL-HOOKS-INVENTORY.md`: inventory shell hooks runtime (`/root/.bashrc`, per-site `.bashrc`, `sshd_config` jail markers).
- `12-ROOT-HELPERS-INVENTORY.md`: inventory helper scripts root-level có side effects runtime.
- `13-BUG-TRIAGE-INDEX.md`: đường vào nhanh để AI Agent khoanh bug theo impact layer và chọn đúng docs/source cần đọc.
- `14-SOURCE-TO-RUNTIME-TRACE.md`: map nhanh từ menu/script sang runtime files, services, verify, rollback.
- `15-KNOWN-RISKS-PATTERNS.md`: các pattern lỗi/rủi ro lặp lại để review bugfix và thay đổi an toàn hơn.
- `16-PLATFORM-AGNOSTIC-CAPABILITIES.md`: tách các capability lõi của hệ thống ra khỏi PHP/WordPress/OLS để có thể tái sử dụng cho stack khác.
- `17-PORTING-MAP-PHP-TO-OTHER-STACKS.md`: bản đồ port logic từ WPTangToc OLS sang Node.js hoặc stack production khác.
- `18-DESIGN-PATTERNS-EXTRACTED.md`: các pattern thiết kế và vận hành có thể copy như blueprint cho control plane mới.

## Quy ước cập nhật tài liệu

- Khi bạn thêm/sửa 1 tính năng:
  - update `02-MODULES-INDEX.md` tại đúng module tương ứng
  - nếu thay đổi flow/assumptions chung → update `01-ARCHITECTURE-FLOW.md`
  - nếu có rule/pitfall mới → update `03-AI-AGENT-GUIDE.md`
  - nếu thay đổi làm phát sinh task mới → thêm vào `04-ROADMAP-PHASE-TASKS.md`

