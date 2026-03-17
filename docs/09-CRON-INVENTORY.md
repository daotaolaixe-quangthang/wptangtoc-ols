# Cron inventory (runtime artefacts) — WPTangToc OLS

Mục tiêu: đây là **nguồn sự thật** để debug nhanh “vì sao VPS tự chạy/tự xoá/tự restart…”.

> Lưu ý context: repo này là **template**. Ở runtime, cron thực nằm tại `/etc/cron.d/*` (hoặc `crontab -l`).

## Quy ước chung

- **RHEL-like** dùng `crond.service`; **Ubuntu/Debian** dùng `cron.service`. Nhiều script có nhánh:
  - Ubuntu: tạo file `/etc/cron.d/<name>.cron` + tạo symlink `..._cron` + `systemctl restart cron.service`
  - RHEL: `systemctl restart crond.service`
- **Per-domain** thường convert dấu `.` thành `_` khi đặt tên symlink cron trên Ubuntu: `NAME_CRON_ubuntu=${NAME//[.]/_}`.
- **Rollback pattern** (gần như luôn đúng):
  - Xoá file `/etc/cron.d/<...>.cron` (+ xoá symlink `_cron` nếu Ubuntu)
  - Restart cron daemon (`cron` hoặc `crond`)
  - (Nếu cron gọi script trong `/etc/wptt-auto/*`) xoá thêm wrapper file đó nếu cần.

---

## Inventory — `/etc/cron.d/*` do WPTT OLS tạo

### Update / Maintenance

| Cron file (runtime) | Tạo bởi (template script) | Schedule | Thực thi | State/side effects | Rollback (tối thiểu) |
|---|---|---|---|---|---|
| `/etc/cron.d/wptangtoc-ols.cron` | `tool-wptangtoc-ols/update/wptt-auto-update` | user chọn: \(0 <gio> * * <thu>\) hoặc monthly | `/etc/wptt/wptt-update-wptangtoc-ols` | update toolset (runtime `/etc/wptt/*` + wrappers `/usr/bin/*` tuỳ flow update) | `rm -f /etc/cron.d/wptangtoc-ols.cron` (+ `wptangtoc-ols_cron` trên Ubuntu) → restart cron |

### Alerts / monitoring / watchdog

| Cron file (runtime) | Tạo bởi (template script) | Schedule | Thực thi | State/side effects | Rollback (tối thiểu) |
|---|---|---|---|---|---|
| `/etc/cron.d/reboot-check-service.cron` | `tool-wptangtoc-ols/service/reboot-app/wptt-auto-reboot-thiet-lap` | `*/<phut> * * * *` | `/etc/wptt/service/reboot-app/wptt-auto-reboot` | auto reboot service + Telegram alert (cần `telegram_api`,`telegram_id`) | `rm -f /etc/cron.d/reboot-check-service.cron` (+ `reboot-check-service_cron` Ubuntu) → restart cron |
| `/etc/cron.d/tai-nguyen-check.cron` | `tool-wptangtoc-ols/bao-mat/wptt-thong-bao-het-tai-nguyen-thiet-lap` | `*/<phut> * * * *` | `/etc/wptt/bao-mat/wptt-thong-bao-het-tai-nguyen <ip> <threshold>` | resource alert qua Telegram | `rm -f /etc/cron.d/tai-nguyen-check.cron` (+ `tai-nguyen-check_cron`) → restart cron |
| `/etc/cron.d/downtimes-check.cron` | `tool-wptangtoc-ols/bao-mat/wptt-canh-bao-downtime-thiet-lap` *(legacy core; hiện redirect sang Add-ons)* | `*/<phut> * * * *` | `/etc/wptt/bao-mat/wptt-canh-bao-downtime` | downtime check (Telegram) | `rm -f /etc/cron.d/downtimes-check.cron` (+ `downtimes-check_cron`) → restart cron |

### SSL automation

| Cron file (runtime) | Tạo bởi (template script) | Schedule | Thực thi | State/side effects | Rollback (tối thiểu) |
|---|---|---|---|---|---|
| `/etc/cron.d/cai-ssl-auto-<domain>-tu-dong.cron` | `tool-wptangtoc-ols/ssl/wptt-cai-ssl-crond-applying-tro-ip` | `*/2 * * * *` | `/etc/wptt/ssl/crond-ssl-auto/check-tro-ip-vs-run <domain>` | worker sẽ tự remove cron khi xong; cài SSL khi DNS trỏ đúng | `rm -f /etc/cron.d/cai-ssl-auto-<domain>-tu-dong.cron` (+ ubuntu symlink) → restart cron |

### WordPress

| Cron file (runtime) | Tạo bởi (template script) | Schedule | Thực thi | State/side effects | Rollback (tối thiểu) |
|---|---|---|---|---|---|
| `/etc/cron.d/wp-cron-job-<domain>.cron` | `tool-wptangtoc-ols/wordpress/wp-cron-job` | `*/<phut> * * * *` | `wget ... https://127.0.0.1/wp-cron.php?doing_wp_cron --header="Host: <domain or www.domain>" ...` | sửa `wp-config.php` add `DISABLE_WP_CRON=true` | `rm -f /etc/cron.d/wp-cron-job-<domain>.cron` (+ ubuntu symlink) → restart cron; revert `DISABLE_WP_CRON` nếu muốn |
| `/etc/cron.d/wp-delete-setup-bao-mat-<domain>.cron` | `tool-wptangtoc-ols/wptt-install-wordpress2` | `*/3 * * * *` | `/etc/wptt/wordpress/delete-auth-setup-wordpress-config <domain>` | cleanup auth setup tạm thời sau install | `rm -f /etc/cron.d/wp-delete-setup-bao-mat-<domain>.cron` (+ ubuntu symlink) → restart cron |

### Cache (HTML page cache disk limit)

| Cron file (runtime) | Tạo bởi (template script) | Schedule | Thực thi | State/side effects | Rollback (tối thiểu) |
|---|---|---|---|---|---|
| `/etc/cron.d/delete-cache-page-<domain>.cron` | `tool-wptangtoc-ols/cache/wptt-setup-gioi-han-dung-luong-page-cache-html` | `<rand_minute> 0,1,2,...,23 * * *` *(mỗi giờ)* | `/etc/wptt/cache/delete-cache-html-page-gioi-han-dung-luong-o-cung <domain> <MB>` | giới hạn dung lượng page cache bằng auto delete | `rm -f /etc/cron.d/delete-cache-page-<domain>.cron` (+ ubuntu symlink) → restart cron |

### Backup / Restore (local + cloud)

| Cron file (runtime) | Tạo bởi (template script) | Schedule | Thực thi | State/side effects | Rollback (tối thiểu) |
|---|---|---|---|---|---|
| `/etc/cron.d/backup<domain>.cron` | `tool-wptangtoc-ols/backup-restore/wptt-auto-backup` | `<rand_minute> <gio> * * <thu>` hoặc monthly | `/etc/wptt-auto/<domain>-auto-backup` (wrapper gọi `wptt-saoluu` hoặc incremental) | tạo wrapper `/etc/wptt-auto/<domain>-auto-backup` (chmod 740) | `rm -f /etc/cron.d/backup<domain>.cron` (+ ubuntu symlink) + `rm -f /etc/wptt-auto/<domain>-auto-backup` → restart cron |
| `/etc/cron.d/backup-all-website-wptt-ols.cron` | `tool-wptangtoc-ols/backup-restore/wptt-auto-backup` | tương tự trên | `/etc/wptt-auto/all-website-wptt-ols-auto-backup` | backup all websites | `rm -f /etc/cron.d/backup-all-website-wptt-ols.cron` (+ symlink) + remove wrapper nếu cần → restart cron |
| `/etc/cron.d/backup-database<domain>.cron` | `tool-wptangtoc-ols/backup-restore/wptt-auto-backup-database` | `<rand_minute> <gio> * * <thu>` hoặc monthly | `/etc/wptt-auto/<domain>-auto-backup-database` (wrapper gọi `/etc/wptt/db/wptt-saoluu-database`) | tạo wrapper `/etc/wptt-auto/<domain>-auto-backup-database` (chmod 740) | `rm -f /etc/cron.d/backup-database<domain>.cron` (+ ubuntu symlink) + `rm -f /etc/wptt-auto/<domain>-auto-backup-database` → restart cron |
| `/etc/cron.d/backup-all-database-wptt-ols.cron` | `tool-wptangtoc-ols/backup-restore/wptt-auto-backup-database` | tương tự trên | `/etc/wptt-auto/all-database-wptt-ols-auto-backup` | backup DB all websites | `rm -f /etc/cron.d/backup-all-database-wptt-ols.cron` (+ symlink) + remove wrapper nếu cần → restart cron |
| `/etc/cron.d/delete<domain>.cron` | `tool-wptangtoc-ols/backup-restore/wptt-auto-delete-backup` | `0 2 * * *` | `/etc/wptt-auto/<domain>-delete-backup` (wrapper find `-mmin +<minutes> -delete`) | auto delete local backups hết hạn | `rm -f /etc/cron.d/delete<domain>.cron` (+ ubuntu symlink) + `rm -f /etc/wptt-auto/<domain>-delete-backup` → restart cron |
| `/etc/cron.d/delete-backup-all-website-wptt-ols-local.cron` | `tool-wptangtoc-ols/backup-restore/wptt-auto-delete-backup` | `0 2 * * *` | `/etc/wptt-auto/all-delete-backup-het-han-local` | auto delete local backups all websites | `rm -f /etc/cron.d/delete-backup-all-website-wptt-ols-local.cron` (+ symlink) + remove wrapper → restart cron |
| `/etc/cron.d/delete-google-driver-<domain>.cron` | `tool-wptangtoc-ols/backup-restore/wptt-thiet-lap-auto-delete-google-driver-backup` | `0 3 * * *` | `/etc/wptt-auto/<domain>-delete-backup-google-driver` | auto delete remote backups trên Drive | `rm -f /etc/cron.d/delete-google-driver-<domain>.cron` (+ ubuntu symlink) + `rm -f /etc/wptt-auto/<domain>-delete-backup-google-driver` → restart cron |
| `/etc/cron.d/delete-google-driver-all-website-wptt-ols.cron` | `tool-wptangtoc-ols/backup-restore/wptt-thiet-lap-auto-delete-google-driver-backup` | `0 3 * * *` | `/etc/wptt-auto/delete-backup-google-driver-all-website` | auto delete remote backups all websites | `rm -f /etc/cron.d/delete-google-driver-all-website-wptt-ols.cron` (+ symlink) + remove wrapper → restart cron |

### API preload (cron.d + user crontab)

| Artefact | Tạo bởi (template script) | Schedule | Thực thi | Notes | Rollback (tối thiểu) |
|---|---|---|---|---|---|
| `/etc/cron.d/<domain>-api-wptt-pre.cron` | `tool-wptangtoc-ols/api-preload-wptangtoc` | `0 0 <tru_day> <loc_thang> *` | `/etc/wptt/delete-preload-api <domain>` | cron “tự xoá setup” để hạn chế rác | remove 2 cron files + ubuntu symlinks; restart cron |
| `/etc/cron.d/<domain>-api-wptt-pre-du-phong.cron` | `tool-wptangtoc-ols/api-preload-wptangtoc` | `0 12 <tru_day> <loc_thang> *` | `/etc/wptt/delete-preload-api <domain>` | fallback nếu 0:00 fail | như trên |
| `crontab -l` entries (user root) | `tool-wptangtoc-ols/api-preload-wptangtoc` | `0 <H> * * *` + `*/5 * * * *` (+ preload helpers) | `/etc/wptt/wptt-preload-cache2 <domain>`; `/etc/wptt/preload-wptangtoc-api <domain>`; optional `php-cache-preload-wptangtoc.sh` và `php-preload` | **không nằm trong `/etc/cron.d/`** | `crontab -l | sed "/<domain>/d" | crontab -` (script `delete-preload-api` đã làm) |

---

## Domain deletion cleanup (cron artefacts)

Khi xoá domain, script `tool-wptangtoc-ols/domain/wptt-xoa-website` có cleanup các cron liên quan backup/retention theo pattern:

- `/etc/cron.d/backup<domain>.cron`
- `/etc/cron.d/delete<domain>.cron`
- `/etc/cron.d/delete-google-driver-<domain>.cron`

Rollback/verify ở đây là “không còn cron artefacts” sau xoá domain.

