# AI Tutorial — Làm việc với WPTangToc OLS (đúng context, ít token)

Tài liệu này hướng dẫn cách dùng AI Agent để maintain/hoàn thiện docs mà không bị “đọc lan man”.

---

## Step 0 — Task/Phase workflow (bắt buộc khi làm việc theo plan)

Áp dụng khi user giao **task/phase** (không phải câu hỏi thông tin đơn lẻ).

1) **Nắm rõ yêu cầu user**
   - restate ngắn: goal, constraints, phạm vi (template vs runtime)
2) **Phân tích và đưa Plan**
   - plan phải mô tả: Context, Impact layer, State changes dự kiến, Rollback approach, Verify approach
   - plan phải “token‑efficient”: file allowlist + stop condition (đọc tới đâu là đủ)
3) **User duyệt Plan**
   - không implement trước khi plan được duyệt (trừ hotfix nhỏ được user cho phép)
4) **Thực thi theo Plan đã duyệt**
   - giữ đúng scope/allowlist đã cam kết
5) **User review**
   - user kiểm tra kết quả (docs/code/behaviour)
6) **AI Agent fix / iterate**
   - fix đúng issue user phản hồi, không mở rộng scope trừ khi được yêu cầu
7) **User duyệt lại**
8) **Done + Report**
   - báo cáo cuối phải có: thay đổi chính, files touched, state/rollback/verify summary, follow‑ups (nếu có)

---

## Step 1 — Chọn đúng scope (template vs runtime)

- Nếu task là **tài liệu + template code**: làm trong repo `E:\2WEBApp\wptangtoc-ols`
- Nếu task là **debug production**: đọc runtime `/etc/wptt/*` (ngoài scope repo)

> Câu hỏi check nhanh: “Mình đang sửa cho lần cài sau hay sửa cho VPS đang chạy?”

---

## Step 2 — Chọn đúng “điểm vào” trong docs (không đọc cả repo)

- “Dự án này là gì?” → `docs/00-OVERVIEW.md`
- “Luồng chạy thế nào?” → `docs/01-ARCHITECTURE-FLOW.md`
- “Module này làm gì?” → `docs/02-MODULES-INDEX.md` (đúng section)
- “Rules/skills bắt buộc?” → `docs/03-AI-AGENT-GUIDE.md`
- “Rollback nhanh?” → `docs/05-RUNBOOKS.md`
- “Còn thiếu task gì?” → `docs/04-ROADMAP-PHASE-TASKS.md`

---

## Step 3 — Đọc code theo “inventory dẫn đường”

Nguyên tắc:

- Đọc `docs/02-MODULES-INDEX.md` trước để biết:
  - entrypoint nào
  - script con nào
  - state ở đâu
  - rollback pattern
- Sau đó chỉ đọc đúng các script con liên quan (không scan hết module).

---

## Step 4 — Output phải theo format chuẩn

AI Agent phải trả lời:

- Context
- Impact layer
- State changes
- Rollback
- Verify

Nếu thiếu thông tin:

- ghi “assumptions tối thiểu”
- chọn đường an toàn (safe-by-default)

---

## Step 5 — Khi nào phải dừng và xin “ràng buộc” thêm

AI Agent nên dừng đọc thêm và hỏi (hoặc tự chốt assumptions) khi:

- task có nhiều module có thể liên quan (ví dụ cache + firewall + OLS global)
- có rủi ro lockout/downtime/data loss mà prompt chưa nói rõ priority

---

## Step 6 — Thói quen tiết kiệm token

- luôn nêu **file allowlist** trong prompt
- luôn nêu “đọc tối đa 2 file docs trước”
- ưu tiên “đọc đúng section” thay vì đọc cả file dài

Mẫu prompt có sẵn ở: `docs/06-AI-PROMPT-TEMPLATES.md`.

