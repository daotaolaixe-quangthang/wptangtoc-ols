# WPTangToc OLS Premium — Phân tích chi tiết theo docs + source module

> Phạm vi phân tích: repo `E:\2WEBApp\wptangtoc-ols` (template source hiện có), đối chiếu thêm logic runtime được docs nhắc tới (`/etc/wptt/*`).

## 1) Mục tiêu tài liệu

Tài liệu này tổng hợp **đầy đủ và cặn kẽ** về nhóm chức năng **Premium / Add-ons** của WPTangToc OLS theo 3 lớp:

1. **Lớp docs**: mô tả chính thức trong `docs/*`.
2. **Lớp menu/dispatch shell**: mapping từ menu premium đến script thực thi.
3. **Lớp module/source**: script nào có trong repo, script nào là runtime dependency.

---

## 2) Nguồn dữ liệu đã đối chiếu

## 2.1. Docs chính

- `docs/02-MODULES-INDEX.md`
- `docs/03-AI-AGENT-GUIDE.md`
- `docs/04-ROADMAP-PHASE-TASKS.md`

## 2.2. Source/menu premium chính

- `tool-wptangtoc-ols/wptt-add-one-main`
- `tool-wptangtoc-ols/add-one/add-premium`
- `tool-wptangtoc-ols/add-one/activate-key`
- `tool-wptangtoc-ols/add-one/thiet-lap-downtimes`
- `tool-wptangtoc-ols/add-one/thiet-lap-check-ssl`
- `tool-wptangtoc-ols/add-one/thiet-lap-check-domain-het-han`
- `tool-wptangtoc-ols/add-one/thiet-lap-auto-htaccess-optimize`
- `tool-wptangtoc-ols/wptt-reset`
- `tool-wptangtoc-ols/wptt-htaccess-reset`
- `tool-wptangtoc-ols/cloudflare-cdn-ip-show-real`
- `tool-wptangtoc-ols/ip-scan-block`

---

## 3) Kiến trúc premium ở mức tổng quan

Premium/Add-ons được tổ chức theo pattern:

- **Entrypoint menu**: `wptt-add-one-main`
- **Dispatch theo index**: `run_action(index)`
- **Gọi script runtime** trong `/etc/wptt/*` bằng `exec`
- **Chuẩn tham số**:
  - tham số thứ nhất thường là `98` (đánh dấu gọi từ menu add-on)
  - một số action có thêm tham số mode (ví dụ `backup`, `restore`, `custom-x-powered-by`)

### 3.1. Điều kiện premium (license gating)

Nhiều module kiểm tra premium theo một hoặc nhiều dấu hiệu:

1. Có key trong `/etc/wptt/.wptt.conf` (`key_activate`).
2. Có file premium runtime được tải về sau activate key, ví dụ:
   - `/etc/wptt/add-one/check-ssl-han-su-dung.sh`
   - `/etc/wptt/add-one/check-domain-het-han.sh`
   - `/etc/wptt/add-one/check.sh`

Nếu chưa đạt điều kiện, script thường:

- báo cần mua/kích hoạt Premium,
- chuyển lại menu `wptt-add-one-main` (khi `$1 == 98`),
- hoặc gọi luồng activate key.

---

## 4) Mapping menu shell thực tế (đầy đủ)

Nguồn mapping: `tool-wptangtoc-ols/wptt-add-one-main`.

| Action index (0-based) | Menu label | Target script | Args thực thi |
|---:|---|---|---|
| 0 | Active Key | `/etc/wptt/add-one/activate-key` | `98` |
| 1 | Bật/tắt kiểm tra uptime và downtime website API | `/etc/wptt/add-one/thiet-lap-downtimes` | `98` |
| 2 | Bật/tắt kiểm tra thông báo hạn SSL | `/etc/wptt/add-one/thiet-lap-check-ssl` | `98` |
| 3 | Bật/tắt kiểm tra thông báo hạn domain | `/etc/wptt/add-one/thiet-lap-check-domain-het-han` | `98` |
| 4 | Sao lưu Website UPloads lên telegram Cloud Free không giới hạn | `/etc/wptt/add-one/add-premium` | `98 backup` |
| 5 | Download file backup từ telegram Cloud Free không giới hạn | `/etc/wptt/add-one/add-premium` | `98 restore` |
| 6 | Thiết lập tự động backup uploads lên telegram Cloud Free không giới hạn | `/etc/wptt/add-one/add-premium` | `98 auto-backup-setup` |
| 7 | Huỷ thiết lập tự động backup uploads lên telegram Cloud Free không giới hạn | `/etc/wptt/add-one/add-premium` | `98 tat-auto-backup` |
| 8 | Quét lỗ hỏng bảo mật WordPress | `/etc/wptt/add-one/add-premium` | `98 quet-bao-mat-wordpress` |
| 9 | Factory reset máy chủ LiteSpeed Webserver | `/etc/wptt/wptt-reset` | `98 premium` |
| 10 | Factory reset htaccess website | `/etc/wptt/wptt-htaccess-reset` | `98 premium` |
| 11 | Bật/Tắt Tăng tốc LiteSpeed Webserver Htaccess Cache | `/etc/wptt/add-one/thiet-lap-auto-htaccess-optimize` | `98` |
| 12 | Hiện thị IP Log Real bypass Cloudflare CDN | `/etc/wptt/cloudflare-cdn-ip-show-real` | `98 premium` |
| 13 | Tuỳ biến x-powered-by HTTP | `/etc/wptt/add-one/add-premium` | `98 custom-x-powered-by` |
| 14 | Chặn Truy cập trực tiếp http://IP | `/etc/wptt/ip-scan-block` | `98 premium` |

---

## 5) Inventory premium theo feature (docs + source + runtime)

| Feature | Docs mô tả | Module/entrypoint trong repo | Mức hiện diện source |
|---|---|---|---|
| Activate key/license | `docs/02-MODULES-INDEX.md` | `add-one/activate-key` | Có source rõ |
| Downtime API check | `docs/02-MODULES-INDEX.md`, `docs/03-AI-AGENT-GUIDE.md` | `add-one/thiet-lap-downtimes` + runtime `add-one/check.sh` | Có wrapper/source setup; checker chi tiết phụ thuộc runtime |
| SSL expiry check | `docs/02-MODULES-INDEX.md` | `add-one/thiet-lap-check-ssl` + runtime `check-ssl-han-su-dung.sh` | Có wrapper/source setup; worker chính là runtime |
| Domain expiry check | `docs/02-MODULES-INDEX.md` | `add-one/thiet-lap-check-domain-het-han` + runtime `check-domain-het-han.sh` | Có wrapper/source setup; worker chính là runtime |
| Backup uploads Telegram | `docs/02-MODULES-INDEX.md` | `add-one/add-premium backup` | Có dispatcher; xử lý chuyên sâu qua script runtime |
| Restore uploads Telegram | `docs/02-MODULES-INDEX.md` | `add-one/add-premium restore` | Có dispatcher; xử lý chuyên sâu qua script runtime |
| Auto backup setup uploads | `docs/02-MODULES-INDEX.md` | `add-one/add-premium auto-backup-setup` | Có dispatcher/module setup |
| Disable auto backup uploads | `docs/02-MODULES-INDEX.md` | `add-one/add-premium tat-auto-backup` | Có dispatcher/module setup |
| Security scan WordPress | `docs/02-MODULES-INDEX.md` | `add-one/add-premium quet-bao-mat-wordpress` | Dispatcher có; core scan ở script runtime |
| Auto htaccess optimize | `docs/02-MODULES-INDEX.md` | `add-one/thiet-lap-auto-htaccess-optimize` | Có source setup/cron; worker optimize runtime |
| Factory reset webserver | `docs/02-MODULES-INDEX.md` | `wptt-reset premium` | Có source đầy đủ, tác động lớn |
| Factory reset htaccess | `docs/02-MODULES-INDEX.md` | `wptt-htaccess-reset premium` | Có source đầy đủ |
| Cloudflare real IP | `docs/02-MODULES-INDEX.md` | `cloudflare-cdn-ip-show-real premium` | Có source trực tiếp |
| Custom X-Powered-By | `docs/02-MODULES-INDEX.md` | `add-one/add-premium custom-x-powered-by` | Dispatcher có; implementation ở script runtime |
| Block direct IP access | `docs/02-MODULES-INDEX.md` | `ip-scan-block premium` | Có source trực tiếp |

---

## 6) Phân tích chi tiết từng module cốt lõi

## 6.1 `add-one/activate-key`

### Vai trò

- Kích hoạt key premium cho VPS.
- Tải gói premium từ endpoint key server.

### Luồng chính

1. Kiểm tra key cũ (`key_activate`, `date_key`) trong `.wptt.conf`.
2. Nếu người dùng muốn kích hoạt lại, tiếp tục nhập key mới.
3. Tải `https://key.wptangtoc.com/?<key>` về `premium.zip`.
4. Giải nén vào `/etc/wptt/add-one` và phân quyền `chmod -R 700`.
5. Ghi lại `key_activate`, `date_key` vào `/etc/wptt/.wptt.conf`.
6. Tạo cron xác thực/key refresh định kỳ (`/etc/cron.d/key-activate-wptangtoc.crond`).

### State/side effects quan trọng

- `/etc/wptt/.wptt.conf`
- `/etc/wptt/add-one/*` (payload premium runtime)
- `/etc/cron.d/key-activate-wptangtoc.crond`

### Rủi ro

- Phụ thuộc endpoint remote key server.
- Payload runtime premium không nằm hoàn toàn trong repo template.

---

## 6.2 `add-one/add-premium` (dispatcher mode)

### Vai trò

- Dispatcher cho nhiều tính năng premium theo `$2`.

### Guard premium

- Kiểm tra tồn tại `/etc/wptt/add-one/check-ssl-han-su-dung.sh`.
- Kiểm tra `key_activate` trong `.wptt.conf`.

### Các mode

- `backup` → gọi `sao-luu-telegram.sh`
- `restore` → gọi `restore-telegram.sh`
- `auto-backup-setup` → gọi `thiet-lap-auto-backup-telegram`
- `tat-auto-backup` → gọi `huy-thiet-lap-auto-backup-telegram`
- `quet-bao-mat-wordpress` → gọi `quet-bao-mat-wordpress.sh`
- `custom-x-powered-by` → gọi `Powered-By-WPTangToc`

### Dependency đáng chú ý

- Telegram credentials (`telegram_api`, `telegram_id`) bắt buộc cho các mode backup/restore/auto-backup.
- `jq` được cài tự động nếu thiếu (`dnf install jq -y`).

---

## 6.3 `add-one/thiet-lap-downtimes`

### Vai trò

- Bật/tắt giám sát downtime qua API premium.

### Luồng bật

1. Guard premium: yêu cầu có `/etc/wptt/add-one/check.sh`.
2. Chọn kênh thông báo: Telegram / Email / cả hai.
3. Nếu Telegram chưa cấu hình → gọi `bao-mat/wptt-telegram`.
4. Gọi checker: `/etc/wptt/add-one/check.sh ...`.
5. Ghi state:
   - `download_api=1`
   - `email_check_downtime=<email>` (nếu có)

### Luồng tắt

- Xoá `download_api` khỏi `.wptt.conf`.
- Gọi `huy-checkdowntime.sh`.

### State

- `/etc/wptt/.wptt.conf`
- Runtime checker scripts trong `/etc/wptt/add-one/*`

---

## 6.4 `add-one/thiet-lap-check-ssl`

### Vai trò

- Bật/tắt cảnh báo SSL sắp hết hạn.

### Luồng bật

1. Guard premium: cần `check-ssl-han-su-dung.sh`.
2. Yêu cầu Telegram đã setup.
3. Tạo cron:
   - `/etc/cron.d/check-ssl-han-su-dung-wptangtoc.crond`
   - lịch: `0 8 * * *`
4. Ubuntu tạo symlink `_crond`, restart cron.

### Luồng tắt

- Xoá cron file + symlink tương ứng.

---

## 6.5 `add-one/thiet-lap-check-domain-het-han`

### Vai trò

- Bật/tắt cảnh báo domain sắp hết hạn.

### Luồng bật

1. Guard premium: cần `check-domain-het-han.sh`.
2. Yêu cầu Telegram setup.
3. Tạo cron:
   - `/etc/cron.d/check-domain-han-su-dung-wptangtoc.crond`
   - lịch: `0 9 * * *`

### Luồng tắt

- Xoá cron file + symlink tương ứng.

---

## 6.6 `add-one/thiet-lap-auto-htaccess-optimize`

### Vai trò

- Bật/tắt automation tối ưu htaccess cache (premium optimization).

### Luồng bật

1. Guard premium: cần `auto-optimize-htaccess-litespeed.sh`.
2. Tạo cron:
   - `/etc/cron.d/optimize-htaccess-wptangtoc-ols-premium.cron`
   - lịch: mỗi 3 phút `*/3 * * * *`
3. Chạy optimize ngay: `optimize-htaccess-run-now.sh`.

### Luồng tắt

- Gọi rollback script: `vhost-chuyen-htaccess-ve-mac-dinh.sh`.

---

## 6.7 `wptt-reset` (mode premium)

### Vai trò

- Factory reset cấu hình OpenLiteSpeed toàn server (high-impact).

### Đặc điểm kỹ thuật

- Regenerate `/usr/local/lsws/conf/httpd_config.conf` gần như toàn bộ.
- Regenerate vhost config cho các domain từ state (`/etc/wptt/vhost*`).
- Rebuild self-signed SSL store (`/etc/wptt-ssl-tu-ky/*`).
- Re-apply mapping theo PHP version/domain.
- Chạm vào SSH jail block trong `sshd_config`.
- Restart LSWS.

### Cảnh báo

- Đây là thao tác **destructive/high-risk**.
- Tác động nhiều lớp: webserver global, vhost, ssl, ssh jail, cron phụ trợ.

---

## 6.8 `wptt-htaccess-reset` (mode premium)

### Vai trò

- Reset `.htaccess` WordPress về chuẩn mặc định.

### Luồng chính

1. Chọn 1 site hoặc tất cả WordPress site.
2. Ghi lại `.htaccess` block mặc định WordPress.
3. Nếu có SSL thì renew redirect (`wptt-renew-chuyen-huong`).
4. Flush rewrite bằng WP-CLI.
5. Re-apply một số logic cache/security theo state.
6. Restart LSWS.

### Cảnh báo

- Có thể ghi đè toàn bộ custom rules trong `.htaccess`.

---

## 6.9 `cloudflare-cdn-ip-show-real` (mode premium)

### Vai trò

- Cấu hình OLS để log/nhận diện real client IP sau Cloudflare proxy.

### Luồng chính

1. Ghi `useIpInProxyHeader 2` vào `httpd_config.conf`.
2. Rebuild block `accessControl` với dải IP Cloudflare allowlist.
3. Preserve deny rules hiện có (nếu có).
4. Restart LSWS.

### Lưu ý

- Chỉnh trực tiếp `httpd_config.conf`; cần backup trước khi vận hành production.

---

## 6.10 `ip-scan-block` (mode premium)

### Vai trò

- Chặn truy cập trực tiếp bằng IP server (`http://IP`) để giảm scan bot/host-header abuse.

### Luồng chính

1. Lấy public IPv4 server.
2. Tạo pseudo-site theo IP qua `domain/wptt-themwebsite`.
3. Viết `.htaccess` rule block và `chattr +i` để khóa file.
4. Cleanup/restart logs và restart LSWS.
5. Remove state file vhost IP tạm nếu cần.

### Cảnh báo

- Can thiệp theo cách tạo vhost IP riêng; cần kiểm tra tương thích với hệ thống multi-site hiện hữu.

---

## 7) Runtime dependency vs source trong repo

## 7.1 Có source rõ trong repo

- Menu dispatch: `wptt-add-one-main`
- Các module premium wrappers/setup:
  - `add-one/activate-key`
  - `add-one/add-premium`
  - `add-one/thiet-lap-downtimes`
  - `add-one/thiet-lap-check-ssl`
  - `add-one/thiet-lap-check-domain-het-han`
  - `add-one/thiet-lap-auto-htaccess-optimize`
- Helpers/high-impact:
  - `wptt-reset`
  - `wptt-htaccess-reset`
  - `cloudflare-cdn-ip-show-real`
  - `ip-scan-block`

## 7.2 Phụ thuộc runtime premium payload (không nằm trọn trong template repo)

Ví dụ các script được gọi nhưng thường xuất hiện sau khi activate key/tải payload:

- `/etc/wptt/add-one/check.sh`
- `/etc/wptt/add-one/check-ssl-han-su-dung.sh`
- `/etc/wptt/add-one/check-domain-het-han.sh`
- `/etc/wptt/add-one/quet-bao-mat-wordpress.sh`
- `/etc/wptt/add-one/Powered-By-WPTangToc`
- `/etc/wptt/add-one/sao-luu-telegram.sh`
- `/etc/wptt/add-one/restore-telegram.sh`

Điểm này khớp với ghi chú trong docs: nhóm Premium/Add-ons có phần implementation runtime ngoài repo template.

---

## 8) Bảng cốt lõi: Feature -> module files liên quan

| Feature | Entrypoint chính | Module file liên quan trực tiếp |
|---|---|---|
| Activate key | `add-one/activate-key` | `add-one/activate-key`, `wptt-add-one-main` |
| Downtime API | `add-one/thiet-lap-downtimes` | `add-one/thiet-lap-downtimes`, `bao-mat/wptt-canh-bao-downtime-thiet-lap`, runtime `add-one/check.sh` |
| SSL expiry | `add-one/thiet-lap-check-ssl` | `add-one/thiet-lap-check-ssl`, runtime `add-one/check-ssl-han-su-dung.sh` |
| Domain expiry | `add-one/thiet-lap-check-domain-het-han` | `add-one/thiet-lap-check-domain-het-han`, runtime `add-one/check-domain-het-han.sh` |
| Backup Telegram | `add-one/add-premium backup` | `add-one/add-premium`, runtime `add-one/sao-luu-telegram.sh` |
| Restore Telegram | `add-one/add-premium restore` | `add-one/add-premium`, runtime `add-one/restore-telegram.sh` |
| Auto backup Telegram | `add-one/add-premium auto-backup-setup` | `add-one/add-premium`, `add-one/thiet-lap-auto-backup-telegram` |
| Disable auto backup Telegram | `add-one/add-premium tat-auto-backup` | `add-one/add-premium`, `add-one/huy-thiet-lap-auto-backup-telegram` |
| Security scan WP | `add-one/add-premium quet-bao-mat-wordpress` | `add-one/add-premium`, runtime `add-one/quet-bao-mat-wordpress.sh` |
| Custom X-Powered-By | `add-one/add-premium custom-x-powered-by` | `add-one/add-premium`, runtime `add-one/Powered-By-WPTangToc` |
| Auto htaccess optimize | `add-one/thiet-lap-auto-htaccess-optimize` | `add-one/thiet-lap-auto-htaccess-optimize`, runtime optimize scripts |
| Factory reset server | `wptt-reset premium` | `wptt-reset`, `wptt-add-one-main` |
| Factory reset htaccess | `wptt-htaccess-reset premium` | `wptt-htaccess-reset`, `wptt-add-one-main` |
| Real IP Cloudflare | `cloudflare-cdn-ip-show-real premium` | `cloudflare-cdn-ip-show-real`, `wptt-add-one-main` |
| Block direct IP | `ip-scan-block premium` | `ip-scan-block`, `wptt-add-one-main` |

---

## 9) Đánh giá rủi ro vận hành (premium nhóm high-impact)

Các tính năng cần cảnh báo khi vận hành production:

1. **`wptt-reset premium`**: reset cấu hình webserver toàn diện (destructive).
2. **`wptt-htaccess-reset premium`**: ghi đè `.htaccess`.
3. **`cloudflare-cdn-ip-show-real premium`**: chỉnh block accessControl global.
4. **`ip-scan-block premium`**: tạo vhost theo IP + khóa `.htaccess` bằng `chattr`.
5. **Các mode add-premium dùng payload runtime**: behavior thực tế phụ thuộc script được tải qua key activation.

Khuyến nghị vận hành:

- Backup trước khi chạy các chức năng high-impact.
- Ghi rõ playbook rollback theo từng tính năng.
- Theo dõi cron file trong `/etc/cron.d/*` khi bật/tắt các monitor premium.

---

## 10) Kết luận ngắn

1. Premium trong repo hiện tại đã có **menu dispatch + nhiều module setup thật**.
2. Một phần lõi premium vẫn là **runtime payload** tải sau activate key, không nằm trọn trong template source.
3. File quan trọng nhất để map tính năng premium là:
   - `tool-wptangtoc-ols/wptt-add-one-main`
   - `tool-wptangtoc-ols/add-one/add-premium`
4. Về tài liệu chính thức, tham chiếu chuẩn vẫn là `docs/02-MODULES-INDEX.md` (kèm bổ sung trong file phân tích này).
