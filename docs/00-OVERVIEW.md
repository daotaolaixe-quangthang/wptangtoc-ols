# WPTangToc OLS — Overview (Project Context)

## Mục tiêu tài liệu

Tài liệu này là **nguồn sự thật duy nhất (single source of truth)** cho việc:

- **Hiểu dự án nhanh** (con người và AI Agent).
- **Maintain/Update đúng luồng** theo kiến trúc hiện tại.
- **Chốt rules + conventions** để tránh “sửa sai chỗ / sai lớp”.

Tài liệu sẽ được hoàn thiện dần khi đọc toàn bộ source code.

## Dự án này giải quyết bài toán gì?

WPTangToc OLS là bộ script giúp:

- Cài đặt và vận hành **OpenLiteSpeed + LSPHP + MariaDB** (và các tool phụ trợ).
- Tối ưu và quản trị máy chủ tập trung cho **WordPress** (domain/vhost, SSL, cache, backup, bảo mật, monitoring, update…).
- Cung cấp một **CLI menu** tiếng Việt (có đa ngôn ngữ) để thao tác nhanh, giảm cấu hình thủ công.

## Đối tượng sử dụng

- Sysadmin/DevOps vận hành VPS chạy WordPress.
- Người dùng WordPress muốn tự host, cần công cụ “đủ tính năng” nhưng dễ dùng.

## Nguyên tắc vận hành (trọng yếu)

- Repo này chứa **bootstrap + template scripts**.
- Khi cài đặt xong, hệ thống runtime quan trọng nằm ở:
  - `/etc/wptt/*` (config + scripts được cài ra)
  - `/usr/local/lsws/*` (OpenLiteSpeed + vhost document root)
  - Một số binary/command như `wptangtoc`, `wp`, `rclone`, `zip`, `mysqldump`…

> **Note cho AI/maintainer**: Khi debug production, ưu tiên đọc **bản đang chạy** ở `/etc/wptt/*`.
> Repo chỉ là “nguồn template” (có thể lệch version với runtime nếu installer remote cập nhật).

## Cách cài đặt (tham chiếu nhanh)

Theo `README.md`:

- Cài chuẩn:
  - `curl -sO https://wptangtoc.com/share/wptangtoc-ols && bash wptangtoc-ols`
- Cài kèm WordPress:
  - `curl -sO https://wptangtoc.com/share/wptangtoc-ols && bash wptangtoc-ols wp`

## Những file/entrypoint quan trọng trong repo

- `README.md`: hướng dẫn cài, tính năng, OS support.
- `wptangtoc-ols`: bootstrap cài đặt theo OS + chặn panel xung đột.
- `wptt`: backup 1 site WordPress (zip + mysqldump).
- `wptt-all`: backup hàng loạt tất cả site trên server WPTT OLS.
- `tool-wptangtoc-ols/wptangtoc`: menu chính (CLI control panel).
- `tool-wptangtoc-ols/wptt-*-main`: menu module (Domain/SSL/Cache/Backup/Security…).
- `tool-wptangtoc-ols/wptt-callback-menu-chon`: framework menu cho module con.
- `tool-wptangtoc-ols/wptt-header-menu`: banner + health check (lsws + mariadb).

## Glossary (tạm)

- **OLS**: OpenLiteSpeed.
- **LSAPI/LSPHP**: PHP handler của LiteSpeed.
- **WPTT**: tên hệ thống script và thư mục `/etc/wptt`.
- **Module main**: script kiểu `wptt-xxx-main` (định nghĩa menu `options[]` và `run_action()`).
- **Script con**: script thực thi tác vụ, thường ở `/etc/wptt/<module>/...` sau cài đặt.

## Liên kết nội bộ (docs)

- Docs index (điểm vào): `docs/README.md`
- Kiến trúc & flow: `docs/01-ARCHITECTURE-FLOW.md`
- Danh mục module: `docs/02-MODULES-INDEX.md`
- Hướng dẫn AI Agent / maintain: `docs/03-AI-AGENT-GUIDE.md`
- Roadmap / phase tasks: `docs/04-ROADMAP-PHASE-TASKS.md`
- Runbooks (pre-check/verify/rollback): `docs/05-RUNBOOKS.md`

