# Root helpers inventory — WPTangToc OLS

Mục tiêu: map các helper script root-level có side effects runtime nhưng không nằm trọn trong flow module inventory.

> Chỉ inventory các helper có tác động thực sự tới runtime. Bỏ qua script purely UI/search/text-only.

## `kernel`

- **Script path**: `tool-wptangtoc-ols/kernel`
- **Được gọi từ đâu**:
  - installer OS-specific
- **Impact layers**:
  - kernel/sysctl
  - filesystem/system boot compatibility
- **State changes**:
  - `/etc/security/limits.conf`
  - `/etc/security/limits.d/20-nproc.conf`
  - `/etc/sysctl.d/101-sysctl.conf`
  - `/etc/rc.d/rc.local`
  - `/usr/lib/systemd/system/rc-local.service`
  - `/etc/rc.local` symlink
- **Rollback**:
  - restore backup các file trên
  - `sysctl --system`
  - `systemctl daemon-reload`

## `oomscore`

- **Script path**: `tool-wptangtoc-ols/oomscore`
- **Được gọi từ đâu**:
  - installer / bootstrap optimization
- **Impact layers**:
  - systemd drop-in overrides
- **State changes**:
  - `/etc/systemd/system/<service>.service.d/oom-protect.conf`
  - `systemctl daemon-reload`
- **Rollback**:
  - xoá drop-in override từng service
  - `systemctl daemon-reload`
  - restart service hoặc reboot

## `api-preload-wptangtoc`

- **Script path**: `tool-wptangtoc-ols/api-preload-wptangtoc`
- **Được gọi từ đâu**:
  - flow preload/API helper
- **Impact layers**:
  - cron/automation
- **State changes**:
  - root crontab entries cho `wptt-preload-cache2`, `preload-wptangtoc-api`, `php-preload`
  - `/etc/cron.d/<domain>-api-wptt-pre.cron`
  - `/etc/cron.d/<domain>-api-wptt-pre-du-phong.cron`
  - Ubuntu symlink `_cron`
- **Rollback**:
  - gọi `delete-preload-api <domain>` hoặc remove đúng cron entries/files

## `delete-preload-api`

- **Script path**: `tool-wptangtoc-ols/delete-preload-api`
- **Được gọi từ đâu**:
  - cron self-cleanup do `api-preload-wptangtoc` tạo
- **Impact layers**:
  - cron/automation cleanup
- **State changes**:
  - remove root crontab entries chứa domain
  - remove `/etc/cron.d/*api-wptt-pre*`
  - restart `cron`/`crond`
- **Rollback**:
  - muốn bật lại thì chạy lại `api-preload-wptangtoc <domain>`

## `proxy-setup`

- **Script path**: `tool-wptangtoc-ols/proxy-setup`
- **Được gọi từ đâu**:
  - helper root-level, không đi qua main menu chuẩn
- **Impact layers**:
  - OLS global config
  - `.htaccess`
- **State changes**:
  - append `extprocessor <domain>proxy` vào `/usr/local/lsws/conf/httpd_config.conf`
  - rewrite `.htaccess` của `/usr/local/lsws/<domain>/html/.htaccess` với marker `#begin-proxy-wptangtoc`
  - restart OLS
- **Rollback**:
  - remove extprocessor block
  - remove proxy marker block trong `.htaccess`
  - restart OLS

## `disk_noatime`

- **Script path**: `tool-wptangtoc-ols/disk_noatime`
- **Được gọi từ đâu**:
  - installer optimization
- **Impact layers**:
  - filesystem mount options
- **State changes**:
  - backup `/etc/fstab` thành `/etc/fstab.wptt.bak`
  - append `noatime` cho mount `/`
  - `mount -o remount /`
- **Rollback**:
  - restore `/etc/fstab` backup
  - remount hoặc reboot

## `disk_fast_commit_ext4`

- **Script path**: `tool-wptangtoc-ols/disk_fast_commit_ext4`
- **Được gọi từ đâu**:
  - installer optimization, hiện có note “chưa ổn định để run” trong installer
- **Impact layers**:
  - filesystem/ext4 features
- **State changes**:
  - `tune2fs -O fast_commit <root_partition>`
- **Rollback**:
  - không có rollback nhẹ nhàng trong script
  - cần xử lý ở mức ext4 feature management; docs phải coi đây là helper high-risk/optional

## `disk-optimize`

- **Script path**: `tool-wptangtoc-ols/disk-optimize`
- **Được gọi từ đâu**:
  - installer optimization
- **Impact layers**:
  - filesystem mount options
  - bootloader args khi root là XFS
- **State changes**:
  - backup `/etc/fstab` thành `/etc/fstab.wptt.bak`
  - append `noatime`
  - với XFS có thể append `logbsize=256k`
  - `mount -o remount /`
  - `grubby --update-kernel=ALL --args="rootflags=logbsize=256k,noatime"` khi root là XFS
- **Rollback**:
  - restore `/etc/fstab`
  - remove bootloader arg nếu đã thêm
  - remount hoặc reboot

## `monit`

- **Script path**: `tool-wptangtoc-ols/monit`
- **Được gọi từ đâu**:
  - installer
  - cache modules như Redis/Memcached
- **Impact layers**:
  - OS packages
  - system services
  - service monitoring config
- **State changes**:
  - install package `monit`
  - core config: `/etc/monitrc` hoặc `/etc/monit/monitrc`
  - rules: `/etc/monit.d/*` hoặc `/etc/monit/conf.d/*`
  - enable/restart `monit`
- **Rollback**:
  - remove WPTT-created rules
  - disable/restart monit
  - nếu cần gỡ hẳn: uninstall package và restore core config

## Ghi chú vận hành

- Nhiều helper được installer gọi trực tiếp, không phải lúc nào cũng có menu main tương ứng.
- Khi debug production, phải kiểm tra helper đã từng được chạy hay chưa bằng cách nhìn vào runtime artefacts thực tế, không chỉ nhìn source template.
