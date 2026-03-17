# Systemd inventory (runtime artefacts) — WPTangToc OLS

Mục tiêu: liệt kê **unit files** và **drop-in overrides** mà WPTT OLS có thể tạo ra ở runtime (để debug/rollback nhanh).

> Lưu ý: repo là template. Runtime units nằm ở `/etc/systemd/system/*` (hoặc `/usr/lib/systemd/system/*` tuỳ distro).

## Quy ước rollback (an toàn)

- Unit mới: `systemctl stop <unit>; systemctl disable <unit>; rm -f /etc/systemd/system/<unit>.service; systemctl daemon-reload`
- Drop-in override: xoá file trong `/etc/systemd/system/<svc>.service.d/*.conf` → `systemctl daemon-reload` → restart service
- Anti-DDoS: luôn cảnh báo **risk lockout**, ưu tiên rollback layer “mạng/firewall” trước khi chỉnh sâu.

---

## Inventory — units tạo mới (`/etc/systemd/system/*.service`)

| Unit (runtime) | Tạo bởi (template script) | ExecStart | Enable/Start | Files/state liên quan | Rollback (tối thiểu) |
|---|---|---|---|---|---|
| `ddos-blocker-xdp.service` | `tool-wptangtoc-ols/bao-mat/nftables/xdp/install-xdp-ddos-protection` | `/usr/local/bin/start_blocker.sh` (tail error.log → pipe vào `/usr/local/bin/anti`) | `systemctl start ddos-blocker-xdp` + `enable` | pinned BPF: `/sys/fs/bpf/xdp_ddos_protection_prog`, `/sys/fs/bpf/log_blacklist`; helper: `/etc/xdp-protection/cleanup_maps` + crontab `*/5` | chạy `tool-wptangtoc-ols/bao-mat/nftables/xdp/remove-xdp-ddos-protection` hoặc stop/disable + xoá unit + detach XDP + xoá pinned files + daemon-reload |
| `ddos-blocker-nftables.service` | `tool-wptangtoc-ols/bao-mat/nftables/nftables-ddos-bao-mat-rewrite.sh` (+ `...rewrite-quoc-te.sh`) | `/usr/local/bin/anti` | `start` + `enable` | nftables config: `/etc/nftables.conf` (Ubuntu) hoặc `/etc/sysconfig/nftables.conf`; script còn **mask fail2ban/iptables** | `tool-wptangtoc-ols/bao-mat/nftables/remove-nftables-ddos-bao-mat.sh` hoặc stop/disable + remove unit + daemon-reload; unmask fail2ban/iptables |
| `layer7-ddos-blocker-nftables.service` | `tool-wptangtoc-ols/bao-mat/nftables/nftables-ddos-bao-mat-layer7-trang-chu.sh` và `...layer7-no-static-file.sh` | `/usr/local/bin/anti-website-layer-7` | `start` + `enable` | whitelist file: `/etc/go_blocker/whitelist.txt`; phụ thuộc nftables ruleset đã có `blackaction` | stop/disable + remove unit + daemon-reload; xoá binary nếu cần |
| `layer7-lsws-litmit-ddos-blocker-nftables.service` | `tool-wptangtoc-ols/bao-mat/nftables/block-nftables-lsws-limit-ruleset` | `/usr/local/bin/block-nftables-lsws-limit-ruleset` | `start` + `enable` | binary build từ go; theo dõi lsws ruleset/log | stop/disable + remove unit + daemon-reload |
| `ddos-blocker-xdp-access.service` | `tool-wptangtoc-ols/bao-mat/nftables/xdp/remove-xdp-ddos-protection` chỉ **nhắc tới khi cleanup**; không tìm thấy create-path hay install-path trong template source hiện tại | *(legacy runtime artefact; not verified in current template source)* | *(legacy / not verified)* | coi là unit cũ còn sót trên một số runtime hoặc do installer/script ngoài template tạo ra | nếu tồn tại: stop/disable + rm unit + daemon-reload; giữ trạng thái docs là legacy-intentional, không suy đoán hành vi |

---

## Inventory — systemd drop-in overrides (`/etc/systemd/system/<svc>.service.d/*.conf`)

| Drop-in path (runtime) | Tạo bởi (template script) | Tác dụng | Rollback (tối thiểu) |
|---|---|---|---|
| `/etc/systemd/system/mariadb.service.d/jemalloc.conf` | `tool-wptangtoc-ols/db/lemalloc` | `Environment=LD_PRELOAD=<libjemalloc.so.X>` để MariaDB dùng jemalloc | `rm -f .../jemalloc.conf` → `systemctl daemon-reload` → `systemctl restart mariadb` |
| `/etc/systemd/system/<service>.service.d/oom-protect.conf` | `tool-wptangtoc-ols/oomscore` | `OOMScoreAdjust=<negative>` để hạn chế OOM killer kill dịch vụ “lõi” (`sshd`, `mariadb`, `lshttpd`, `redis`, `valkey`, `memcached`) | xoá file drop-in của service → daemon-reload → restart service (hoặc reboot) |

---

## Unit/system boot compatibility (rc.local)

Script `tool-wptangtoc-ols/kernel` có logic tạo `/usr/lib/systemd/system/rc-local.service` và `/etc/rc.d/rc.local` (khi thiếu), và remove `/etc/systemd/system/rc-local.service` nếu tồn tại bản “non-standard”.

Đây là artefact hệ thống (không phải module web) và cần cẩn trọng khi rollback:

- Nếu muốn rollback thay đổi kernel/limits: revert `/etc/security/limits.conf`, `/etc/sysctl.d/101-sysctl.conf`, `/etc/rc.local` và chạy lại `sysctl --system`.
- `rc-local.service` ở đây là compatibility unit cấp hệ thống; template source hiện tại tạo dưới `/usr/lib/systemd/system/rc-local.service`, không phải custom WPTT-only unit trong `/etc/systemd/system/`.

