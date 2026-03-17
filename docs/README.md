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

## Quy ước cập nhật tài liệu

- Khi bạn thêm/sửa 1 tính năng:
  - update `02-MODULES-INDEX.md` tại đúng module tương ứng
  - nếu thay đổi flow/assumptions chung → update `01-ARCHITECTURE-FLOW.md`
  - nếu có rule/pitfall mới → update `03-AI-AGENT-GUIDE.md`
  - nếu thay đổi làm phát sinh task mới → thêm vào `04-ROADMAP-PHASE-TASKS.md`

