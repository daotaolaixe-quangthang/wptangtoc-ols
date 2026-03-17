# WPTangToc OLS — Modules Index (Inventory)

## Mục tiêu

- Tạo “mục lục module” để maintain nhanh.
- Mỗi module có: **entrypoint**, **script con**, **state**, **side-effects**, **rollback**.

> Khi đọc sâu từng phần, chúng ta sẽ điền dần vào các mục bên dưới.

---

## A) Entrypoints (repo)

- **Bootstrap installer**: `wptangtoc-ols`
- **Main menu**: `tool-wptangtoc-ols/wptangtoc`
- **Single-site backup tool**: `wptt`
- **All-sites backup tool**: `wptt-all`
- **Module menu framework**: `tool-wptangtoc-ols/wptt-callback-menu-chon`
- **Header/banner**: `tool-wptangtoc-ols/wptt-header-menu`

---

## B) Main menu → module mains (runtime: `/etc/wptt/*`)

Các module main thường có file tương ứng trong repo: `tool-wptangtoc-ols/wptt-*-main`

### WordPress

- **Entrypoint**: `wptt-wordpress-main`
- **Sub-scripts (repo)**: `tool-wptangtoc-ols/wordpress/*`
  - Update & reinstall:
    - `wptt-update-wordpress-main`: menu update (core/plugins/themes/reinstall/all/downgrade/check version).
    - `update-core`, `update-plugin`, `update-theme`, `update-full`
    - `ghi-de-wordpress-core`, `wptt-update-reinstall-wordpres`
    - `ha-cap-phien-ban-wordpress-core`, `ha-cap-phien-ban-plugins`
    - `kiem-tra-version-wp`
  - Ops & hardening:
    - `wp-cron-job`, `tat-wp-cron-job`
    - `wp-debug`, `bat-wp-debug`
    - `thay-salt`, `passwd-wp`
    - `plugin-xml-rpc-wptangtoc` (disable XML-RPC)
    - `plugin-heartbeat-wptangtoc` (heartbeat tuning)
    - `bao-tri-wordpress` (maintenance mode)
    - `bao-mat-2lop`, `tat-bao-mat-2lop` (wp-admin extra auth realm)
  - Data tools:
    - `transient` (xoá transient)
    - `check-autoload`
    - `query-truyvan` (search/replace DB)
    - `thay-url-option` (home/siteurl)
    - `thay-doi-tien-to` (DB prefix)
  - Performance / media:
    - `image` (optimize images)
    - `wptt-render-thumbnail` (regen thumbnails)
    - `wptt-rocket` (clear/preload rocket)
  - Security checks:
    - `check-decode-base64` (scan base64)
    - `check-hook-speed` (hook speed)
    - `xoa-binh-luan-spam`
  - Advanced:
    - `wordpress-multisite`
    - `unix-stocket-wpconfig` (DB unix socket config)
- **Runtime target paths (sau khi cài)**:
  - Scripts: `/etc/wptt/wordpress/*`
  - Per-site docroot: `/usr/local/lsws/$domain/html`
  - Per-site state: `/etc/wptt/vhost/.$domain.conf`
  - WP CLI: `/usr/local/bin/wp` (được menu chính đảm bảo tồn tại)
- **State**:
  - global: `/etc/wptt/.wptt.conf` (ngôn ngữ, php defaults, flags chung)
  - per-site: `/etc/wptt/vhost/.$domain.conf` (lock_down, php version per domain, 2FA creds…)
  - cron state: `/etc/cron.d/wp-cron-job-$domain.cron` (+ symlink ubuntu)
  - OLS vhost conf: `/usr/local/lsws/conf/vhosts/$domain/$domain.conf` (wp-admin realm, contexts)

#### WordPress guard — phát hiện WordPress tồn tại (`wptt-wordpress-main`)

- Trước khi hiển thị menu quản trị, script scan tất cả vhost:
  - nếu **không có site nào có `wp-load.php`** → cảnh báo và **quay về menu chính**.

#### WordPress flow — menu chính (`wptt-wordpress-main`)

- Menu WordPress là `options[]` + `run_action(index)` (0-based), gọi các script con ở `/etc/wptt/wordpress/*`.
- Pattern giống các module khác: load `.wptt.conf`, load `lang/$ngon_ngu.sh`, rồi `exec` script con.

#### WordPress flow — Update Core (`update-core`)

- Chọn 1 site hoặc “Tất cả website”.
- Guards:
  - Domain phải tồn tại `/etc/wptt/vhost/.$NAME.conf`.
  - Phải là WordPress (`/usr/local/lsws/$NAME/html/wp-load.php`).
- Thực thi:
  - Load PHP CLI theo domain: `. /etc/wptt/php/php-cli-domain-config $NAME`
  - Chạy WP-CLI bằng đúng PHP version:
    - `wp core update`
    - `wp core update-db`
- Ghi log: `/var/log/wptangtoc-ols.log`

#### WordPress flow — Update “all components” (`wptt-update-wordpress-main`)

- Là menu con chuyên cập nhật:
  - core/plugins/themes/reinstall/all, downgrade core/plugins, check WP version.
- Cơ chế dispatch vẫn là `run_action(index)` → `exec /etc/wptt/wordpress/<script>`.

#### WordPress flow — Tối ưu WP-Cron (`wp-cron-job`)

- Mục tiêu: disable WP cron trigger theo traffic, thay bằng system cron chạy định kỳ.
- Hiển thị trạng thái theo từng domain dựa vào file:
  - `/etc/cron.d/wp-cron-job-$domain.cron`
- Khi bật:
  - Validate domain tồn tại + là WordPress.
  - Hỏi chu kỳ phút (default 10).
  - Sửa `wp-config.php`:
    - remove dòng cũ `DISABLE_WP_CRON`
    - chèn `define( 'DISABLE_WP_CRON', true );` sau `<?php`
  - Tạo cron:
    - gọi `wget` vào `https://127.0.0.1/wp-cron.php?doing_wp_cron`
    - set `Host: $domain` (tự detect `www.` theo `wp option get home`)
    - user-agent `WP Cron Optimize by WPTangToc OLS`
    - `--no-check-certificate` (lưu ý security trade-off)
  - Restart service cron:
    - Ubuntu: restart `cron.service` + tạo symlink tên file an toàn (thay `.` thành `_`)
    - RHEL: restart `crond.service`

#### WordPress flow — Bảo mật 2 lớp wp-admin (`bao-mat-2lop`)

- Đây là **layer auth bổ sung** ở OLS vhost (realm/htpasswd), không phải TOTP.
- State detection:
  - nếu tồn tại `/usr/local/lsws/$domain/passwd/.mk` → coi như đã bật.
- Khi bật:
  - Tạo `id_ols_admin` + password ngẫu nhiên, hash bằng `htpasswd.php` (admin_php).
  - Ghi `id:hash` vào `/usr/local/lsws/$domain/passwd/.mk` (chmod 400, owner `nobody:nogroup` hoặc `nobody:nobody` tuỳ OS).
  - Append realm + context `exp:wp-login.php` vào vhost conf `/usr/local/lsws/conf/vhosts/$domain/$domain.conf` để bảo vệ màn hình login/wp-admin theo rule OLS.
  - Lưu lại credential plaintext vào `/etc/wptt/vhost/.$domain.conf`:
    - `id_dang_nhap_wordpress_2=...`
    - `password_dang_nhap_wordpress_2=...`
  - Restart `lsws`.
- Caveat bảo mật:
  - plaintext credential trong `/etc/wptt/vhost/.$domain.conf` (đã chmod 700) cần được nhắc rõ trong tài liệu vận hành.

### Domain / Vhost

- **Entrypoint**: `wptt-domain-main`
- **Sub-scripts (repo)**: `tool-wptangtoc-ols/domain/*`
  - `wptt-themwebsite`: thêm domain/vhost mới (core)
  - `wptt-xoa-website`: xoá domain/vhost (core)
  - `wptt-list-domain`: liệt kê domain theo `/etc/wptt/vhost`
  - `wptt-alias-domain` / `wptt-xoa-alias-domain`: parked/alias domain → trỏ về domain chính
  - `wptt-chuyen-huong` / `wptt-xoa-domain-chuyen-huong`: redirect domain
  - `wptt-bat-tat-domain`: enable/disable domain
  - `wptt-thay-doi-user`, `wptt-khoi-tao-user`: quản lý user per-site
  - `wptt-subfolder-website`: khởi tạo site dạng subfolder
  - `real-time-check`, `real-time-check-domain`: giám sát/kiểm tra domain
  - `punycode.php`: chuyển unicode domain → punycode bằng `idn_to_ascii()`
- **Runtime target paths (sau khi cài)**:
  - Scripts: `/etc/wptt/domain/*`
  - Per-site state: `/etc/wptt/vhost/.$domain.conf`
  - Docroot: `/usr/local/lsws/$domain/html`
  - OLS vhost config: `/usr/local/lsws/conf/vhosts/$domain/$domain.conf`
  - OLS global config: `/usr/local/lsws/conf/httpd_config.conf`
  - Self-signed SSL store: `/etc/wptt-ssl-tu-ky/$domain/*`
  - Optional Let’s Encrypt store: `/etc/letsencrypt/live/$domain/*`

#### Domain flow — Thêm website (`wptt-themwebsite`)

- **Input**:
  - Nhận `$NAME` từ arg hoặc prompt (domain/subdomain).
  - Chuẩn hoá:
    - loại `http(s)://`, bỏ `www.`, lower-case
    - validate phải có dấu `.`
    - convert punycode bằng `php /etc/wptt/domain/punycode.php`
- **Guards**:
  - Nếu tồn tại `/etc/wptt/vhost/.$NAME.conf` → báo domain đã tồn tại.
  - Nếu tồn tại redirect config `/etc/wptt/chuyen-huong/.$NAME.conf` → yêu cầu xử lý redirect trước.
- **Per-site UNIX user**:
  - Tạo user từ domain: `USER=${NAME//[-._]/wp}` (giới hạn 32 ký tự).
  - `useradd -p -m -d /usr/local/lsws/$NAME $USER`
  - Set shell nologin: `usermod $USER -s /sbin/nologin`
  - Thiết lập **SFTP chroot** cho user bằng cách append block `Match User $USER` vào `/etc/ssh/sshd_config` rồi restart `sshd`.
- **OLS config**:
  - Append `virtualhost $NAME { ... user/group $USER ... }` vào `httpd_config.conf`.
  - Thêm map vào listener http: `sed -i "/listener http/a map $NAME $NAME"`.
  - Tạo vhost conf file: `/usr/local/lsws/conf/vhosts/$NAME/$NAME.conf` với:
    - `docRoot /usr/local/lsws/$NAME/html`
    - `vhDomain $NAME`
    - `autoLoadHtaccess 1`
    - `extprocessor` LSAPI dùng socket `/tmp/lshttpd/<NAMEPHP>.sock`
    - `path /usr/local/lsws/lsphp74/bin/lsphp` (sau đó được sed theo `php_version_check`)
    - `vhssl` mặc định dùng **self-signed**
    - module cache block (disabled by default `enableCache 0`)
  - Generate self-signed cert ở `/etc/wptt-ssl-tu-ky/$NAME/*`.
  - Nếu phát hiện đã có Let’s Encrypt cert (case add lại domain) → thay vhssl sang LE.
- **Filesystem**:
  - Tạo thư mục: `html`, `luucache`, `backup-website` và `/usr/local/backup-website/$NAME`.
  - Khởi tạo `.htaccess` “WordPress default rewrite” nếu chưa có.
  - Chown docroot/backup cho `$USER:$USER`; chown `/usr/local/lsws` về `root:root` (hardening).
- **Database provisioning**:
  - Tạo random `DB_Name_web`, `DB_User_web`, `DB_Password_web`.
  - Dùng `mariadb --defaults-extra-file=<tmp.cnf>` (root creds từ `.wptt.conf`) để create DB + user + grant.
  - Ghi state vào `/etc/wptt/vhost/.$NAME.conf` (perm `root:root` + `chmod 700`):
    - `DB_Name_web`, `DB_User_web`, `DB_Password_web`
    - `Duong_Dan_thu_muc_web`
    - `User_name_vhost`
    - `phien_ban_php_domain`
- **Symlink convenience**:
  - `/wptangtoc-ols/$NAME` → symlink tới docroot và backup path.
  - `/home/$NAME` → symlink tới `/usr/local/lsws/$NAME` (để user nhìn thấy qua chroot).
- **Service**:
  - restart `lsws`.
  - nếu số site > 10 hoặc RAM < 1GB, gọi optimize multi-web config.

#### Domain flow — Xoá website (`wptt-xoa-website`)

- **Guards**:
  - Domain phải tồn tại `/etc/wptt/vhost/.$NAME.conf`.
  - Nếu xoá domain chính (`$Website_chinh`), bắt buộc chọn domain khác thay thế làm website chính.
- **Side effects lớn** (cần tài liệu rollback riêng):
  - Có logic đổi SSL đại diện cho listener https trong `httpd_config.conf` sang cert của domain mới.
  - Có logic di chuyển/tái cấu hình phpMyAdmin realm nếu tồn tại.
  - Xoá database của site bằng root creds (`database_admin_username/password`).

#### Domain flow — Alias/Parked domain (`wptt-alias-domain`)

- Chọn domain gốc (đã tồn tại), nhập `NAME2` alias.
- Cập nhật:
  - `httpd_config.conf`: thêm `, $NAME2` vào map.
  - vhost conf: thêm/append `vhAliases $NAME2`.
- Nếu domain chính có Let’s Encrypt:
  - kiểm tra DNS trỏ về IP server trước khi xin cert.
  - xin cert bằng `certbot --webroot --expand` cho domain + aliases.

#### Ưu / Nhược (tóm tắt)

- **Ưu**:
  - One-command provisioning: vhost + user isolation + db creds + htaccess + ssl tự ký.
  - Có IDN support (punycode) cho domain unicode.
  - State rõ ràng (per-site) ở `/etc/wptt/vhost/.$domain.conf`.
- **Nhược / rủi ro**:
  - Can thiệp trực tiếp `httpd_config.conf` bằng append/sed → dễ lệch nếu user custom.
  - SSH/SFTP chroot jail sửa trực tiếp `/etc/ssh/sshd_config` (lockout risk nếu config sai).\
    - **Marker block (idempotent)**: các script đều dùng block:\
      - `#begin-WPTT_JAIL_<user>`\
      - `Match User <user>`\
      - `ChrootDirectory /usr/local/lsws/<domain>`\
      - `ForceCommand internal-sftp` (tuỳ script)\
      - `#end-WPTT_JAIL_<user>`\
    - **Idempotency**: trước khi append block mới, script luôn xoá block cũ theo regex range:\
      - `sed -i -e "/^#begin-WPTT_JAIL_$USER/,/^#end-WPTT_JAIL_$USER$/d" /etc/ssh/sshd_config || true`\
    - **Cleanup/legacy fallback** (code cũ): `wptt-khoi-tao-user` còn xoá thêm dạng cũ:\
      - `sed -i "/# WPTT_JAIL_$User_name_vhost/,+6d" /etc/ssh/sshd_config || true`\
    - **Verify**: sau thay đổi user/xoá domain/tắt login, nên check:\
      - `sshd -t` (syntax OK) rồi mới `systemctl restart sshd`\
      - file không còn block trùng user (`grep -n "^#begin-WPTT_JAIL_<user>$" -n /etc/ssh/sshd_config`)\
  - Vhost conf mặc định path `lsphp74` rồi sed theo `php_version_check` → cần hiểu rõ nguồn `php_version_check`.
  - Xoá website domain chính có nhiều side-effects (SSL listener, phpMyAdmin).

### SSL

- **Entrypoint**: `wptt-ssl-main`
- **Sub-scripts (repo)**: `tool-wptangtoc-ols/ssl/*`
  - `wptt-caissl`: cài SSL miễn phí Lets Encrypt (HTTP-01, multi-check DNS/online).
  - `wptt-gia-han-ssl`: gia hạn thủ công tất cả cert Lets Encrypt (chạy `certbot renew`).
  - `wptt-mo-rong-ssl-free`: mở rộng phạm vi SSL free (multi-domain / alias / multisite subdomain).
  - `wptt-xoa-ssl`: xoá cert SSL miễn phí LetsEncrypt + gỡ redirect markers trong `.htaccess`.
  - `wptt-caissl-traphi` / `wptt-caissl-traphi-xoa` / `wptt-config-ssl-tra-phi`: SSL trả phí.
  - `wptt-kiem-tra-ssl`: kiểm tra chứng chỉ SSL hiện tại (đọc local cert, check expiry).
  - `wptt-renew-chuyen-huong`: design lại rule chuyển hướng HTTP→HTTPS + www/non-www trong `.htaccess`.
  - `cloudflare-api-dns-main` → `cloudflare-api-dns` / `cloudflare-api-dns-xoa`: thiết lập/xoá Cloudflare API DNS (DNS-01 flow).
  - `wptt-caissl-cloudlflare-api-dns`: cài SSL bằng Cloudflare DNS API nếu domain đã cấu hình.
  - Auto/cron helpers:
    - `wptt-cai-ssl-crond-applying-tro-ip`: bật cron “auto cài SSL khi DNS trỏ đúng IP”.
    - `crond-ssl-auto/check-tro-ip-vs-run`: cron worker (check DNS rồi gọi `wptt-caissl`, xong auto remove cron).
    - `cai-ssl-all-free`: cài SSL cho 1 site hoặc “tất cả website chưa có SSL”.
    - `auto-reset-ssl`: auto restart OLS nếu detect cert vừa renew trong `/etc/letsencrypt/archive`.
    - `check-ssl-cert-con-kha-dung`: helper kiểm tra expiry từ local cert (LE hoặc trả phí).
- **Runtime target paths (sau khi cài)**:
  - Scripts: `/etc/wptt/ssl/*`
  - LE certs: `/etc/letsencrypt/live/$domain/*`
  - Self-signed certs (mặc định khi thêm domain): `/etc/wptt-ssl-tu-ky/$domain/*`
  - `.htaccess` per-site: `/usr/local/lsws/$domain/html/.htaccess` (thêm block chuyển hướng với marker)

#### SSL flow — Cài Lets Encrypt (`wptt-caissl`)

- **Chọn site**:
  - Từ `vhost` và (nếu có) từ thư mục `chuyen-huong`.
  - Bắt buộc docroot tồn tại: `/usr/local/lsws/$NAME/html`.
  - Bỏ qua domain bị “khoá” (vhost conf `.conf.bkwptt`).
- **Tương thích LockDown**:
  - Nếu per-site config có `lock_down`, script xoá `.well-known` cũ và bỏ `chattr +i` trên `.well-known` trước khi xin cert.
- **DNS / Online checks**:
  - Kiểm tra site online qua `curl -Is http://$NAME`.
  - Kiểm tra DNS A/AAAA từ nhiều nguồn:
    - `host`, `nslookup`, Google DNS API, Cloudflare DNS API.
  - So sánh với IP server:
    - IPv4: từ `ip a` + icanhazip/checkip.
    - IPv6: nếu OLS listener dùng IPv6, kiểm tra tương tự.
  - Nếu tất cả fail:
    - Thử phát hiện Cloudflare CDN qua `https://$NAME/cdn-cgi/trace`.
    - Nếu dùng Cloudflare CDN: hướng dẫn dùng SSL Cloudflare (Full/Flexible) thay vì LE.
    - Nếu không phải CDN: báo lỗi “chưa trỏ DNS” + nameserver quản lý.
- **Cloudflare DNS API integration**:
  - Nếu tồn tại `/etc/wptt/.cloudflare/wptt-$NAME.ini`:
    - Gọi `wptt-caissl-cloudlflare-api-dns $NAME 98` để dùng DNS-01 thay HTTP-01.
- **Certbot**:
- **Certbot (thực thi chính)**:
  - Nạp danh sách domain/subdomain vào `/tmp/ssl-$NAME.txt`:
    - luôn có `$NAME`
    - có thể add `www.$NAME` nếu detect trỏ DNS cùng IP
    - auto add multisite subdomain (nếu `wp-config.php` có `MULTISITE`): `wp site list --field=url ...`
  - Chuẩn hoá `docRoot`:
    - đọc từ `/usr/local/lsws/conf/vhosts/$NAME/$NAME.conf` (`docRoot ...`)
  - Chạy:
    - `certbot certonly --non-interactive --agree-tos --register-unsafely-without-email --expand --webroot -w <docRoot> -d <all_domains...>`
- **After actions**:
  - **OLS vhost ssl**:
    - Xoá block `vhssl { ... }` cũ (nếu có) rồi append block mới trỏ về:
      - `/etc/letsencrypt/live/$NAME/privkey.pem`
      - `/etc/letsencrypt/live/$NAME/cert.pem`
      - `/etc/letsencrypt/live/$NAME/chain.pem`
  - **Domain chính (`$Website_chinh`)**:
    - Xoá các dòng `keyFile/certFile/CACertFile` trong `/usr/local/lsws/conf/httpd_config.conf` rồi inject lại dưới listener `https` để trỏ về LE cert (thay self-signed).
  - Restart OLS: `/usr/local/lsws/bin/lswsctrl restart`
  - Tương thích CSF:
    - nếu có `/etc/csf/csf.conf` và `TCP_IN` chưa có `443` thì script prepend `443,` rồi `csf -x; csf -e`
  - Tương thích “vhost chuyển htaccess”:
    - nếu per-site state có `vhost_chuyen_htaccess` thì gọi `/etc/wptt/wptt-vhost-chuyen-ve-htaccess $NAME`
  - Optional add-on: nếu `.wptt.conf` có `download_api=1` và có `/etc/wptt/add-one/check.sh` thì gọi check downtime (Telegram/email tuỳ config).
  - Ghi log vào `/var/log/wptangtoc-ols.log`.

#### SSL flow — Gia hạn thủ công (`wptt-gia-han-ssl`)

- Scan tất cả vhost conf trong `/etc/wptt/vhost` và **lọc domain có `/etc/letsencrypt/live/$NAME/cert.pem`**.
- Hỏi confirm 1 lần trước khi gia hạn all.
- Tạm thời tắt một số lớp firewall nếu đang dùng:
  - CSF với `CC_ALLOW_FILTER` → `csf -x` / `csf -e`.
  - nftables với dấu `ipvietnam` trong config → `systemctl stop/start nftables`.
- Chạy `certbot renew --quiet` → restart `lsws`.

#### SSL flow — Renew chuyển hướng HTTP→HTTPS (`wptt-renew-chuyen-huong`)

- Có thể chạy trên:
  - 1 domain (qua chọn tên),
  - hoặc “Tất cả website” (loop qua `vhost`).
- Guard:
  - Domain phải tồn tại trong `vhost` hoặc `chuyen-huong`.
  - Bắt buộc đã có SSL (LE hoặc SSL thư mục `/usr/local/lsws/$NAME/ssl` hoặc đang dùng Cloudflare SSL).
- **Trên WordPress site**:
  - Dùng `wp option get home` để xem đang dùng www hay không.
  - Cập nhật `home`/`siteurl` sang `https://...` (www hoặc non-www).
- **.htaccess rewrite**:
  - Xoá toàn bộ block cũ bằng marker:
    - `#begin-chuyen-huong-http-to-https-wptangtoc-ols` → `#end-...`
    - `#begin-chuyen-huong-www-to-non-www-wptangtoc-ols` → `#end-...`
    - `#begin-chuyen-huong-non-www-to-www-wptangtoc-ols` → `#end-...`
  - Thêm mới:
    - block **HTTP→HTTPS** (dùng `%{HTTP_HOST}` + `REQUEST_URI`).
    - block `www→non-www` hoặc `non-www→www` tuỳ theo DNS & WP `home`.
  - Marker đảm bảo có thể idempotent/bật lại sau này.
- Restart `lsws` và ghi log.

#### SSL flow — Cloudflare API DNS (`cloudflare-api-dns-main`)

- Sub-menu 2 lựa chọn:
  - Thiết lập Cloudflare API DNS: lưu token/zone để các script SSL dùng DNS-01.
  - Xoá thiết lập Cloudflare API DNS.
- Sau khi thiết lập, `wptt-caissl` sẽ tự động chuyển sang flow dùng `wptt-caissl-cloudlflare-api-dns` nếu detect file config Cloudflare cho domain.

#### SSL flow — Mở rộng (alias) chứng chỉ LetsEncrypt (`wptt-mo-rong-ssl-free`)

- **Context**: đã có LE cert ở `/etc/letsencrypt/live/$NAME/cert.pem`, giờ muốn expand thêm subdomain/alias vào cùng 1 cert.
- **Impact layers**: certbot + OLS reload + (optional) Cloudflare DNS API.
- **Flow**:
  - Guard:
    - domain tồn tại trong `vhost`
    - docroot tồn tại
    - đã có LE cert cho `$NAME`
    - nếu không có Cloudflare DNS API config → yêu cầu `$NAME` trỏ DNS về IP server
  - Lấy danh sách SAN hiện tại bằng:
    - `openssl x509 -in /etc/letsencrypt/live/$NAME/cert.pem -noout -text | ...`
  - Nhập thêm subdomain (loop), sau đó auto-add:
    - multisite subdomain (nếu WP multisite)
    - `vhAliases` trong vhost conf (nếu có)
  - Nếu không có Cloudflare DNS API config:
    - lọc bỏ các subdomain chưa trỏ đúng IP (remove khỏi `/tmp/ssl-$NAME.txt`)
  - Chạy `certbot certonly --expand --webroot ... -d ...` (hoặc `--dns-cloudflare` nếu có `/etc/wptt/.cloudflare/wptt-$NAME.ini`)
  - Restart OLS.
- **State**:
  - LE cert updated tại `/etc/letsencrypt/live/$NAME/*`
  - temporary list: `/tmp/ssl-$NAME.txt`
- **Rollback**:
  - Rollback “remove SAN” không có script riêng; thực tế rollback là:
    - re-issue cert chỉ với tập domain mong muốn (chạy lại `wptt-mo-rong-ssl-free` nhưng nhập danh sách khác), hoặc
    - xoá cert và cài lại (dùng `wptt-xoa-ssl` rồi `wptt-caissl`).
- **Pitfalls**:
  - Script có logic filter DNS theo IP: nếu DNS propagation chậm, domain có thể bị loại khỏi list và không được add vào cert.

#### SSL flow — Xoá SSL LetsEncrypt (`wptt-xoa-ssl`)

- **Context**: revoke + remove LE cert của 1 domain (có thể là domain trong `vhost` hoặc `chuyen-huong`).
- **Impact layers**: certbot + filesystem `/etc/letsencrypt/*` + OLS config + `.htaccess`.
- **State changes**:
  - `certbot revoke --non-interactive --agree-tos --cert-path <cert.pem>`
  - remove:
    - `/etc/letsencrypt/live/$domain` (và `www.$domain` nếu có)
    - `/etc/letsencrypt/renewal/$domain.conf`
    - `/etc/letsencrypt/archive/$domain`
  - Nếu là domain chính (`$Website_chinh`):
    - generate self-signed mới trong `/etc/wptt-ssl-tu-ky/$domain/`
    - cập nhật `/usr/local/lsws/conf/httpd_config.conf` để trỏ `keyFile/certFile` về self-signed, rồi restart OLS
  - Gỡ redirect markers trong `/usr/local/lsws/$domain/html/.htaccess`:
    - `#begin-chuyen-huong-http-to-https-wptangtoc-ols` ... `#end-...`
    - `#begin-chuyen-huong-www-to-non-www-wptangtoc-ols` ... `#end-...`
    - `#begin-chuyen-huong-non-www-to-www-wptangtoc-ols` ... `#end-...`
  - Restart OLS.
- **Rollback**:
  - Cài lại SSL bằng `wptt-caissl` (khuyến nghị).
- **Pitfalls**:
  - Đây là thao tác phá huỷ cert: nếu revoke nhầm, chỉ có thể issue cert mới (không “undo”).

#### SSL flow — Kiểm tra SSL (`wptt-kiem-tra-ssl` + `check-ssl-cert-con-kha-dung`)

- **Context**: kiểm tra “còn hạn” bằng cách đọc local cert và so sánh expiry.
- **Detect cert type** (trong helper):
  - Nếu vhost conf có `certFile` chứa `letsencrypt` → dùng `/etc/letsencrypt/live/$NAME/cert.pem`
  - Nếu vhost conf có `certFile` chứa `lsws` → dùng `/usr/local/lsws/$NAME/ssl/cert.crt` (ssl trả phí)
- **Output**:
  - set biến `ssl_xac_thuc_kha_dung=true|false` rồi script in trạng thái theo màu.
- **Pitfalls**:
  - Helper có nhánh test `[ $(cat ... ) ]` dễ lỗi khi output rỗng (shell test). Khi gặp false-negative, cần verify thủ công bằng `openssl x509 -enddate`.

#### SSL auto/cron — Auto cài SSL khi DNS trỏ đúng IP (`wptt-cai-ssl-crond-applying-tro-ip` + `crond-ssl-auto/check-tro-ip-vs-run`)

- **Mục tiêu**: khi bạn **chưa trỏ DNS** xong, có thể bật cron để hệ thống tự “canh” DNS rồi cài SSL ngay khi trỏ đúng.
- **State**:
  - cron file (per-domain): `/etc/cron.d/cai-ssl-auto-$domain-tu-dong.cron`
  - Ubuntu symlink: `/etc/cron.d/cai-ssl-auto-${domain//./_}-tu-dong_cron`
  - cron command: `*/2 * * * * root /etc/wptt/ssl/crond-ssl-auto/check-tro-ip-vs-run $domain`
- **Worker flow (`check-tro-ip-vs-run`)**:
  - Nếu domain đã có SSL (`/etc/letsencrypt/live/$domain` hoặc `/usr/local/lsws/$domain/ssl`) → remove cron và exit.
  - Check DNS A record bằng `host`/`nslookup`, so với IP server (local + public via icanhazip/checkip).
  - Nếu match → gọi `/etc/wptt/ssl/wptt-caissl $domain`, rồi **auto remove cron** (self-clean).
- **Rollback**:
  - Xoá cron file (và symlink Ubuntu) + restart `cron`/`crond`.
- **Pitfalls**:
  - Cron chạy mỗi 2 phút: nếu DNS đang “flap” hoặc cache DNS chưa ổn định, có thể trigger cài SSL sớm; nên bật khi bạn chắc chắn chuẩn bị trỏ DNS.

#### SSL auto — Cài SSL cho “tất cả website chưa có SSL” (`cai-ssl-all-free`)

- **Context**: loop qua `/etc/wptt/vhost` và gọi `cai-ssl-all-free <domain>` nếu domain chưa có `/etc/letsencrypt/live/$domain`.
- **State**:
  - Không tạo state riêng; gọi thẳng `wptt-caissl` để tạo LE cert + update OLS config.
- **Pitfalls**:
  - Script check DNS match rất “cứng”; nếu 1 số site chưa trỏ DNS, chúng sẽ bị skip.

#### SSL auto — Auto restart OLS sau renew (`auto-reset-ssl`)

- **Context**: nếu detect có file trong `/etc/letsencrypt/archive` vừa được sửa trong ~27h và “newer” hơn `/usr/local/lsws/cgid` thì restart OLS.
- **State**: không.
- **Rollback**: không cần (read-only + restart).


### Database

- **Entrypoint**: `wptt-db-main`
- **Sub-scripts**: `tool-wptangtoc-ols/db/*`
- **Impact layers (DB module hay đụng tới)**:
  - **MariaDB config**: `/etc/mysql/my.cnf` (Ubuntu) hoặc `/etc/my.cnf.d/server.cnf` (RHEL-like).
  - **Firewall**: `firewalld`, CSF (`/etc/csf/csf.conf`), `nftables` config file.
  - **Runtime secrets/state**:
    - DB root/admin creds: `/etc/wptt/.wptt.conf` (`database_admin_username`, `database_admin_password`)
    - DB per-site creds: `/etc/wptt/vhost/.$domain.conf` (`DB_Name_web`, `DB_User_web`, `DB_Password_web`)
  - **WordPress config**: `wp-config.php` (khi script tự update bằng `wp-cli`).
  - **Cron**: auto-restart/auto-optimize (tạo file trong `/etc/cron.d/*`).

#### Menu dispatch (`wptt-db-main`)

- **Các action chính (0-based index)**:
  - Tạo DB/user: `db/wptt-them-database`
  - Xoá DB: `db/wptt-xoa-database`
  - Backup DB: `db/wptt-saoluu-database`
  - Restore DB: `db/wptt-nhapdatabase`
  - Danh sách DB: `db/wptt-thongtin-db` (liệt kê databases)
  - Optimize/repair all DB: `/etc/wptt/wordpress/all-database`, `/etc/wptt/wordpress/all-database-sua-chua`
  - Wipe DB (drop+create lại DB của 1 site): `db/wptt-wipe-database`
  - Đổi password DB user của site: `db/wptt-thay-doi-passwd-database`
  - “Kết nối DB với WordPress”: `db/wptt-ket-noi` (sync DB creds → `wp-config.php`)
  - Xem thông tin tài khoản DB: `db/wptt-thongtin-db-tk`
  - Format nén DB backup: `/etc/wptt/backup-restore/wptt-sql-gzip-config`
  - Xem dung lượng DB (WP-only): `db/wptt-dung-luong-database`
  - Auto optimize DB (cron): `db/wptt-tu-dong-hoa-toi-uu-database`
  - Đổi storage engine: `db/chuyen-doi-dinh-dang-luu-tru-storage-engine-innodb-myisam-aria` (ALTER TABLE ENGINE bằng `wp db query`)
  - Remote DB (mở truy cập từ xa + mở port): `db/wptt-remote-database`
  - Auto restart MariaDB: `db/wptt-auto-restart-mysql`
  - Rebuild MariaDB: `db/wptt-rebuild-mariadb`
  - Đổi phiên bản MariaDB: `db/wptt-thay-doi-phien-ban-mariadb`
  - MariaDB CLI (root hoặc per-site): `db/cli-mysql-thuc-thi`

#### Tạo DB/user cho 1 site (`db/wptt-them-database`)

- **Context**: cấp DB/user mới cho 1 domain đã tồn tại trong `/etc/wptt/vhost/.$domain.conf`.
- **State changes**:
  - DB layer:
    - `DROP DATABASE IF EXISTS <generated_db>`
    - `CREATE DATABASE IF NOT EXISTS <generated_db>`
    - `DROP USER IF EXISTS '<generated_user>'@'localhost'`
    - `CREATE USER ... IDENTIFIED BY <generated_password>`
    - `GRANT ALL PRIVILEGES ON <db>.* ... WITH GRANT OPTION`
  - State file:
    - overwrite các dòng `DB_Name_web`, `DB_User_web`, `DB_Password_web` trong `/etc/wptt/vhost/.$domain.conf`
- **Rollback**:
  - Nếu tạo nhầm: dùng `db/wptt-xoa-database` để drop DB (lưu ý: script này chỉ drop **database**, không xoá user theo site state).
  - Với user đã tạo: cần xoá thủ công user tương ứng bằng MariaDB CLI (hoặc dùng flow domain delete nếu xóa cả site).

#### Xoá database (drop DB) (`db/wptt-xoa-database`)

- **Context**: xoá vĩnh viễn 1 database theo danh sách `SHOW DATABASES` (lọc bằng pattern `_dbname`).
- **State changes**:
  - DB layer: `DROP DATABASE <selected>`
  - Audit log: append `/var/log/wptangtoc-ols.log`
- **Rollback**:
  - Không rollback dữ liệu (trừ khi có backup). Khuyến nghị: backup trước bằng `db/wptt-saoluu-database`.

#### Wipe/format trắng DB của 1 site (`db/wptt-wipe-database`)

- **Context**: “làm trắng” DB cho 1 domain bằng cách drop DB hiện tại và tạo lại DB cùng tên.
- **State changes**:
  - DB layer:
    - `DROP DATABASE $DB_Name_web`
    - `CREATE DATABASE IF NOT EXISTS $DB_Name_web`
  - Audit log: append `/var/log/wptangtoc-ols.log`
- **Rollback**:
  - Không rollback dữ liệu; chỉ có thể restore từ backup (dùng `db/wptt-nhapdatabase`).

#### Đổi password DB user của 1 site (`db/wptt-thay-doi-passwd-database`)

- **Context**: rotate password cho DB user của 1 site (không cho đổi root).
- **Impact layer**: DB + per-site state + (nếu WordPress) `wp-config.php`.
- **State changes**:
  - DB layer:
    - `SET PASSWORD FOR '$DB_User_web'@'localhost' = PASSWORD('<new>')`
    - nếu user có remote grant: set password cho host `'%'` tương tự
    - `FLUSH PRIVILEGES`
  - Per-site state:
    - overwrite `DB_Password_web=...` trong `/etc/wptt/vhost/.$domain.conf`
  - Services:
    - restart `mariadb`
    - restart OLS (`lswsctrl restart`)
  - WordPress (nếu có `wp-config.php`):
    - chạy `wp config set DB_PASSWORD ... --allow-root --path=/usr/local/lsws/$domain/html`
- **Rollback**:
  - Không có “rollback password” tự động; rollback thực tế là đổi lại password (bằng CLI hoặc chạy lại script để generate password mới rồi cập nhật WP config).
- **Pitfalls**:
  - Nếu site không phải WordPress (không có `wp-config.php`) thì script không tự update app config → ứng dụng sẽ lỗi kết nối DB cho đến khi bạn cập nhật secret tương ứng.

#### Kết nối DB creds vào WordPress (`db/wptt-ket-noi`)

- **Context**: đồng bộ DB creds từ `/etc/wptt/vhost/.$domain.conf` vào `wp-config.php` (DB_HOST/DB_NAME/DB_USER/DB_PASSWORD).
- **Impact layers**: filesystem (wp-config.php) + wp-cli + (tuỳ site) `chattr` nếu LockDown.
- **Flow**:
  - Chọn 1 site hoặc “Tất cả website” (loop qua `vhost`, chỉ xử lý site có `wp-config.php`).
  - Nếu `lock_down`:
    - `chattr -i` `wp-config.php` trước khi sửa, xong `chattr +i` lại.
  - Set constants bằng `wp-cli`:
    - `wp config set DB_HOST 'localhost' --type=constant ...`
    - `wp config set DB_NAME "$DB_Name_web" ...`
    - `wp config set DB_USER "$DB_User_web" ...`
    - `wp config set DB_PASSWORD "$DB_Password_web" ...`
  - Flush rewrite:
    - `wp rewrite flush`
  - Nếu có subfolder sites (dir `/etc/wptt/$domain-wptt/`):
    - loop các `.conf` subfolder, tìm `.../subfolder/wp-config.php` và set constants tương tự.
  - Restart OLS.
- **Rollback**:
  - Không có rollback tự động; rollback là set lại constants về giá trị cũ (nếu bạn còn lưu) hoặc restore `wp-config.php` từ backup.
- **Pitfalls**:
  - Script assume DB_HOST luôn `localhost` (không dành cho remote DB host).
  - “Tất cả website” sẽ restart OLS sau loop; chạy giờ thấp điểm.

#### Đổi storage engine (InnoDB/MyISAM/Aria) (`db/chuyen-doi-dinh-dang-luu-tru-storage-engine-innodb-myisam-aria`)

- **Context**: chuyển engine các table trong DB của WordPress site sang engine mong muốn.
- **Guards**:
  - chỉ chạy trên WordPress (`wp-load.php`).
- **Flow**:
  - Chọn site hoặc “Tất cả website”.
  - Load PHP CLI đúng version theo domain: `/etc/wptt/php/php-cli-domain-config $domain`.
  - Lấy danh sách tables cần đổi:
    - `wp db query "SHOW TABLE STATUS WHERE Engine != '<ENGINE>'" ...` → lấy cột table name.
  - Loop:
    - `wp db query "ALTER TABLE <table> ENGINE=<ENGINE>" ...`
  - Ghi audit log `/var/log/wptangtoc-ols.log`.
- **Rollback**:
  - Rollback là chạy lại script và chọn engine cũ (không có snapshot).
- **Pitfalls / rủi ro**:
  - ALTER TABLE có thể lock table lâu trên site lớn → nên chạy giờ thấp điểm và backup trước.

#### Xem thông tin tài khoản DB (root/per-site/subfolder) (`db/wptt-thongtin-db-tk`)

- **Context**: in `DB_Name_web/DB_User_web/DB_Password_web` từ state file; có option `root` để in `database_admin_*` trong `.wptt.conf`.
- **State changes**: không (read-only) ngoại trừ ghi audit log `/var/log/wptangtoc-ols.log`.

#### Xem dung lượng DB (WordPress-only) (`db/wptt-dung-luong-database`)

- **Context**: dùng `wp db size --tables` (yêu cầu site có `wp-load.php`).
- **Impact**: read-only.

#### Auto optimize database (cron) (`db/wptt-tu-dong-hoa-toi-uu-database`)

- **Context**: thiết lập lịch auto optimize DB (daily/weekly/monthly).
- **State changes**:
  - cron file: `/etc/cron.d/database-toi-uu-hoa-all.cron`
  - Ubuntu symlink (nếu có): `database-toi-uu-hoa-all_cron`
  - cron target: `/etc/wptt/wordpress/all-database`
- **Rollback**:
  - xoá cron file và restart `cron`/`crond`.

#### Auto restart MariaDB (cron) (`db/wptt-auto-restart-mysql` + `db/restart-auto-mariadb.sh`)

- **Context**: watchdog đơn giản: cứ 5 phút check nếu DB service down thì restart.
- **State changes**:
  - cron file: `/etc/cron.d/auto-restart-mariadb.cron` (*/5)
  - Ubuntu symlink: `/etc/cron.d/auto-restart-mariadb_cron`
  - worker: `/etc/wptt/db/restart-auto-mariadb.sh`
- **Rollback**:
  - xoá cron file và restart `cron`/`crond`.
- **Pitfalls**:
  - Worker đang check `mysql.service` nhưng restart `mariadb.service` → tuỳ distro có thể lệch tên unit. Nếu watchdog không hoạt động như mong đợi, cần chuẩn hoá unit name.

#### Remote Database (mở truy cập DB từ xa + mở port) (`db/wptt-remote-database`)

- **Context**: bật remote access cho **1 site user** (`'$DB_User_web'@'%'`) và đồng thời mở port MariaDB ra ngoài.
- **Impact layers**: MariaDB config + firewall + fail2ban + DB grants.
- **State changes (khi bật)**:
  - MariaDB config:
    - set `port=<chosen_port>`
    - set `bind-address=0.0.0.0` (listen all interfaces)
    - file path:
      - Ubuntu: `/etc/mysql/my.cnf`
      - RHEL-like: `/etc/my.cnf.d/server.cnf`
  - Firewall:
    - `firewall-cmd --permanent --add-port=<port>/tcp` (nếu `firewalld` active)
    - CSF: prepend port vào `TCP_IN` trong `/etc/csf/csf.conf`
    - nftables: insert rule `tcp dport <port> accept #port remote mariadb` rồi restart nftables
  - Fail2ban:
    - sửa `/etc/fail2ban/jail.local` (thay `3306` → `<port>`), restart fail2ban
  - DB grants:
    - `CREATE USER IF NOT EXISTS '$DB_User_web'@'%' IDENTIFIED BY '$DB_Password_web'`
    - `GRANT ALL PRIVILEGES ON $DB_Name_web.* TO '$DB_User_web'@'%' WITH GRANT OPTION`
  - Audit log: append `/var/log/wptangtoc-ols.log`
- **State changes (khi tắt)**:
  - drop user remote: `DROP USER IF EXISTS '$DB_User_web'@'%'`
  - nếu không còn site nào dùng remote:
    - xoá `bind-address` và `port=` khỏi MariaDB config, đóng port trên firewall/csf/nftables
    - restart `mysql.service` (script dùng tên này) + restart fail2ban
    - revert `jail.local` bằng cách replace `<port>` → `3306`
- **Rollback**:
  - Khuyến nghị: chạy lại `wptt-remote-database` và chọn “tắt” cho site đó.
  - Nếu bị lockout/overexpose: đóng port ở firewall layer trước (firewalld/CSF/nftables), rồi revert `bind-address/port` và restart MariaDB.
- **Security / lockout risk**:
  - Đây là thay đổi **rủi ro cao**: mở DB ra Internet. Mặc định chỉ nên allowlist IP quản trị/ứng dụng (firewall) và ưu tiên dùng VPN/LAN.
  - Khi đổi port MariaDB khỏi 3306, các tool/monitoring khác có thể hỏng (cần cập nhật đồng bộ).

#### MariaDB CLI interactive (`db/cli-mysql-thuc-thi`)

- **Context**: mở MariaDB shell dưới quyền `root` (from `.wptt.conf`) hoặc user per-site (from `vhost/.$domain.conf`).
- **Security**:
  - Script tạo temp `*.cnf` (chmod 600) để tránh lộ password qua argv/process list.

### Backup & Restore

- **Entrypoint**: `wptt-backup-restore-main`
- **Sub-scripts (repo)**: `tool-wptangtoc-ols/backup-restore/*`
  - Core:
    - `wptt-saoluu`: sao lưu website (files + DB) theo domain hoặc all.
    - `wptt-khoiphuc`: khôi phục website từ file backup (files + DB).
  - Auto backup (cron):
    - `wptt-auto-backup`: thiết lập tự động backup website (files + DB).
    - `wptt-tat-auto-backup`: tắt auto backup website.
    - `wptt-auto-backup-database`: thiết lập tự động backup database.
    - `wptt-tat-auto-backup-database`: tắt auto backup database.
  - Retention / cleanup (cron):
    - `wptt-auto-delete-backup`: thiết lập auto-delete local backup theo số ngày.
    - `wptt-tat-delete-backup`: tắt auto-delete local backup.
    - `wptt-thiet-lap-auto-delete-google-driver-backup`, `wptt-auto-delete-google-driver-backup`, `wptt-tat-auto-delete-backup-google-driver`: auto delete backup trên cloud (rclone: Google Drive/OneDrive).
  - Manual cleanup:
    - `wptt-xoa-file-backup`: xoá file backup local.
    - `wptt-xoa-file-backup-google-driver`: xoá file backup trên cloud.
  - Cloud storage (rclone):
    - `wptt-rclone`: thiết lập rclone Google Drive (remote name: `wptangtoc:`).
    - `wptt-rclone-one-driver`: thiết lập OneDrive (remote vẫn dùng `wptangtoc:` nhưng backend khác).
    - `wptt-huy-rclone`: huỷ đăng ký drive.
    - `wptt-download-rclone`: download backup từ cloud về local.
  - Formats:
    - `wptt-sql-gzip-config`: chọn format nén DB (`.sql`, `.sql.gz`, `.sql.zst`).
    - `dinh-dang-backup-ma-nguon-option`: chọn format nén mã nguồn (`.zip`, `.tar.gz`, `.tar.zst`).
  - Incremental:
    - `wptt-saoluu-luy-tien-tung-phan`, `wptt-khoiphuc-luy-tien-tung-phan` (backup “luỹ tiến”: core + uploads theo tháng).
  - Utilities:
    - `wptt-check-disk-dieu-kien-backup`: check disk trước khi backup.
- **Integrations**:
  - **rclone**: Google Drive / OneDrive, remote name `wptangtoc:`, default folder `wptangtoc_ols_backup`.
  - (Tuỳ bản/add-on) Telegram upload backup (liên quan `wptt-telegram`).
- **Runtime target paths (sau khi cài)**:
  - Scripts: `/etc/wptt/backup-restore/*`
  - Local backup store (root): `/usr/local/backup-website/$domain/`
  - Local backup store (user): `/usr/local/lsws/$domain/backup-website/`
  - Auto scripts: `/etc/wptt-auto/*`
  - Cron entries (per-site):
    - Backup website: `/etc/cron.d/backup$domain.cron`
    - Backup DB: `/etc/cron.d/backup-database$domain.cron`
    - Delete expired: `/etc/cron.d/delete$domain.cron`
  - Cron entries (all websites):
    - `/etc/cron.d/backup-all-website-wptt-ols.cron`
    - `/etc/cron.d/backup-all-database-wptt-ols.cron`
    - `/etc/cron.d/delete-backup-all-website-wptt-ols-local.cron`
  - Ubuntu compatibility symlinks:
    - `/etc/cron.d/<name>_cron` (thay `.` thành `_`) + restart `cron.service`

#### Backup flow — Sao lưu thủ công (`wptt-saoluu`)

- **Chọn scope**:
  - 1 domain (qua menu) hoặc `Tất cả website`.
- **All websites**:
  - Nếu rclone đã cấu hình (`~/.config/rclone/rclone.conf` có remote `wptangtoc`):
    - hỏi có upload backup lên cloud không (`ggdriver=1998`).
    - hỏi có xoá local sau khi upload không (`ggdriver2=1998`).
  - Tạo `START_TIMESTAMP_FILE` để xác định website nào backup xong.
  - Loop qua `/etc/wptt/vhost/*` → gọi lại chính `wptt-saoluu $domain $ggdriver $ggdriver2`.
  - In summary “hoàn tất backup” và dung lượng trước/sau.
- **Single domain**:
  - **Locking chống đụng độ**:
    - lock file per-domain: `/tmp/wptt_lock_sao_luu_khoi_phuc_${NAME}.lock`
    - dùng `flock -n` FD 200, nếu bận thì báo lỗi và quay menu.
  - Load secrets/state:
    - `. /etc/wptt/.wptt.conf`
    - `. /etc/wptt/vhost/.$NAME.conf` (DB creds)
  - Check disk trước khi backup:
    - dựa theo format nén mã nguồn (`dinh_dang_nen_ma_nguon`: zip/tar.gz/tar.zst)
    - gọi `wptt-check-disk-dieu-kien-backup`.
  - Dump database:
    - dùng temp cnf (chmod 600) với `DB_User_web/DB_Password_web`.
    - `max_allowed_packet=1G`, charset `utf8mb4`.
  - Backup mã nguồn:
    - format mặc định `.zip`, hoặc `.tar.gz` / `.tar.zst` nếu đã set.
  - Output location:
    - root store: `/usr/local/backup-website/$NAME/`
    - có thể thêm user store: `/usr/local/lsws/$NAME/backup-website/` (phục vụ restore theo quyền).

#### Restore flow — Khôi phục thủ công (`wptt-khoiphuc`)

- Guards:
  - Domain phải tồn tại `/etc/wptt/vhost/.$NAME.conf`.
  - Phải có file backup ở:
    - `/usr/local/backup-website/$NAME/` hoặc `/usr/local/lsws/$NAME/backup-website/`.
- Chọn file backup mã nguồn:
  - hỗ trợ `.zip`, `.tar.gz`, `.tar.zst` (lọc loại incremental `-wptt-luy-tuyen`).
  - nếu trùng file ở root store và user store → hỏi chọn “file root” hay “file user”.
- Chọn file backup database:
  - hỗ trợ `.sql`, `.sql.gz`, `.sql.zst`.
  - nếu trùng file ở root store và user store → hỏi chọn “file root” hay “file user”.
- (Chi tiết restore giải nén + import DB nằm phần sau file; sẽ bổ sung khi đọc tiếp range còn lại.)

#### Auto-backup flow — thiết lập cron (`wptt-auto-backup`, `wptt-auto-backup-database`)

- Hiển thị trạng thái auto-backup theo sự tồn tại của:
  - `/etc/cron.d/backup$domain.cron`
  - `/etc/cron.d/backup-database$domain.cron`
- Cho phép chọn:
  - 1 domain hoặc “Tất cả website” (tạo cron riêng cho all).
- Tham số schedule:
  - hỏi giờ `0..23`.
  - chọn theo tuần/ngày/tháng qua `wptt-times-tuan` (có `*` hàng ngày, hoặc `ngày_cua_thang`).
- Upload drive integration:
  - kiểm tra file `/etc/wptt-auto/$domain-auto-backup*` có “1998” để biểu diễn mode upload/xoá local (logic hiển thị trong status).
- Ubuntu:
  - tạo symlink cron file (đổi `.` thành `_`) và restart `cron.service`.
- RHEL:
  - restart `crond.service`.

#### Retention flow — auto delete local backups (`wptt-auto-delete-backup`)

- User nhập số ngày giữ backup → quy đổi `phut = ngay * 60 * 24`.
- Tạo script delete:
  - per-domain: `/etc/wptt-auto/$domain-delete-backup` (chmod 740)
    - `ionice -c 3 nice -n 19 find /usr/local/backup-website/$domain -maxdepth 1 -type f -mmin +$phut -delete`
  - all websites: `/etc/wptt-auto/all-delete-backup-het-han-local`
    - `find /usr/local/backup-website -maxdepth 2 ... -delete`
- Tạo cron chạy 2h sáng:
  - per-domain: `/etc/cron.d/delete$domain.cron`
  - all: `/etc/cron.d/delete-backup-all-website-wptt-ols-local.cron`
- Restart cron service theo OS.

#### Retention flow — auto delete cloud backups (rclone) (`wptt-thiet-lap-auto-delete-google-driver-backup` + `wptt-auto-delete-google-driver-backup` + `wptt-tat-auto-delete-backup-google-driver`)

- **Context**: tự động xoá file backup quá hạn trên cloud (`wptangtoc:wptangtoc_ols_backup/<domain>/`) theo “min-age N ngày”.
- **Deps**:
  - phải có rclone remote `wptangtoc:` trong `~/.config/rclone/rclone.conf`.
- **State changes (setup)**:
  - Tạo script runner:
    - per-domain: `/etc/wptt-auto/<domain>-delete-backup-google-driver` (chmod 740)
    - all websites: `/etc/wptt-auto/delete-backup-google-driver-all-website` (chmod 740)
  - Tạo cron chạy 3h sáng:
    - per-domain: `/etc/cron.d/delete-google-driver-<domain>.cron`
    - all: `/etc/cron.d/delete-google-driver-all-website-wptt-ols.cron`
  - Ubuntu symlink: `/etc/cron.d/<name>_cron` (thay `.` thành `_`) + restart `cron.service`; RHEL restart `crond.service`.
- **Worker logic (`wptt-auto-delete-google-driver-backup`)**:
  - Liệt kê file theo tuổi bằng:
    - `rclone ls wptangtoc:wptangtoc_ols_backup/<domain> --min-age ${N}d`
  - Filter extension (theo code):
    - `.zip`, `.tar.gz`, `.tar.zst`, `.sql`, `.sql.gz`, `.sql.zst`
  - Delete:
    - Google Drive: `rclone --drive-use-trash=false delete ...` (xoá thẳng, không vào trash)
    - OneDrive: `rclone --onedrive-hard-delete delete ...` (Business/SharePoint), Personal thì delete bình thường
- **Rollback (disable)**:
  - Xoá cron + xoá script `/etc/wptt-auto/*delete-backup-google-driver*`, rồi restart `cron`/`crond`.
- **Pitfalls**:
  - Đây là “delete vĩnh viễn” (drive-use-trash=false / hard-delete) → cần cảnh báo rõ rủi ro mất dữ liệu.

#### Backup formats — DB compression (`wptt-sql-gzip-config`)

- **State**: ghi `sql_gz=<mode>` vào `/etc/wptt/.wptt.conf`:
  - không có `sql_gz` → `.sql`
  - `sql_gz=1` → `.sql.gz` (cài `gzip` nếu thiếu)
  - `sql_gz=2` → `.sql.zst` (cài `zstd` nếu thiếu)
- **Impact**: ảnh hưởng định dạng dump DB trong `wptt-saoluu` / `wptt-saoluu-luy-tien-tung-phan` và import trong restore scripts.
- **Rollback**: chạy lại menu và chọn `.sql` (remove `sql_gz` line).

#### Backup formats — source archive (`dinh-dang-backup-ma-nguon-option`)

- **State**: ghi `dinh_dang_nen_ma_nguon=<mode>` vào `/etc/wptt/.wptt.conf`:
  - không có `dinh_dang_nen_ma_nguon` → `.zip`
  - `dinh_dang_nen_ma_nguon=1` → `.tar.zst` (cài `tar`, `zstd`)
  - `dinh_dang_nen_ma_nguon=2` → `.tar.gz` (cài `tar`, `pigz`, `gzip`)
- **Rollback**: chọn `.zip` (remove line).

#### Backup incremental — luỹ tiến từng phần (`wptt-saoluu-luy-tien-tung-phan`)

- **Context**: tối ưu dung lượng backup bằng cách:
  - tạo “core zip” loại trừ uploads/cache/backups nặng
  - tạo 1 file “uploads full theo tháng” (mỗi tháng 1 lần)
  - tạo “uploads tháng hiện hữu” và merge vào core zip để backup incremental
- **State / artefacts**:
  - core: `<domain><time>-wptt-luy-tien-tung-phan-core.zip`
  - uploads monthly full: `<domain>-<mm_YYYY>-wptt-luy-tien-wp-uploads-dir.zip`
  - DB dump: `.sql` / `.sql.gz` / `.sql.zst` theo `sql_gz`
- **Cloud upload**:
  - có thể upload lên `wptangtoc:wptangtoc_ols_backup/<domain>/` bằng rclone, và tuỳ chọn xoá local sau khi upload.
- **Rollback**:
  - restore bằng `wptt-khoiphuc-luy-tien-tung-phan` (khuyến nghị) hoặc dùng backup thường nếu có.

#### Restore incremental — luỹ tiến từng phần (`wptt-khoiphuc-luy-tien-tung-phan`)

- **Context**: khôi phục từ “core zip” + “uploads full theo tháng” + DB dump (đúng bộ file).
- **Guards**:
  - Domain phải tồn tại trong `/etc/wptt/vhost/.$domain.conf`
  - Phải có ít nhất:
    - `*-wptt-luy-tien-tung-phan-core.zip`
    - `*-wptt-luy-tien-wp-uploads-dir.zip` tương ứng theo tháng/năm của bản core
    - DB dump `.sql|.sql.gz|.sql.zst`
- **Flow (high level)**:
  - Chọn file core zip (root store hoặc user store; nếu trùng timestamp sẽ hỏi chọn).
  - Chọn file DB dump (root/user tương tự).
  - Drop + create DB (root/admin creds) rồi import DB (zcat/zstd -d hoặc plain).
  - Remove docroot hiện tại rồi unzip core zip + unzip uploads-dir zip, sau đó unzip phần uploads tháng hiện hữu (nếu có).
  - Fix permissions theo `lock_down`:
    - lock_down: chmod 404/515 + re-apply chattr lock
    - không lock_down: chmod 644/755
  - Update `wp-config.php` bằng `wp config set` (DB_HOST/DB_NAME/DB_USER/DB_PASSWORD), flush rewrite.
  - Clear cache (`wptt-xoacache`) + remove `/usr/local/lsws/$domain/luucache` nếu có, restart OLS.
- **Rollback**:
  - Không rollback tự động; rollback là restore lại từ bộ backup khác.

#### Caveats / rủi ro vận hành

- Backup/restore phụ thuộc DB creds trong `/etc/wptt/vhost/.$domain.conf` (root-only).
- Cơ chế lock per-domain dùng chung cho **cả backup và restore**:
  - `/tmp/wptt_lock_sao_luu_khoi_phuc_${NAME}.lock` + `flock` FD 200 (có trong `wptt-saoluu` và `wptt-khoiphuc`).
- Cron tạo file trong `/etc/cron.d/` + script trong `/etc/wptt-auto/` → khi uninstall/cleanup cần xóa đồng bộ.
- Rclone setup dùng `expect` và remote name cố định `wptangtoc:`; token nằm trong `/root/.config/rclone/rclone.conf` (secrets).

### Services

- **Entrypoint**: `wptt-service-main`
- **Sub-scripts**: `tool-wptangtoc-ols/service/*`
- **Targets**: lsws, mariadb, php services, watchdog

#### Menu dispatch (`wptt-service-main`)

- Gồm các nhánh chính (runtime: `/etc/wptt/service/*`):
  - **Restart services**: `/etc/wptt/service/wptt-reboot-main`
  - **Start services**: `/etc/wptt/service/wptt-start-main`
  - **Stop services**: `/etc/wptt/service/wptt-stop-main`
  - **Status overview**: `/etc/wptt/service/wptt-status`
  - **Auto reboot + alert khi service down**: `/etc/wptt/service/reboot-app/wptt-auto-reboot-thiet-lap`
  - **Check versions**: `/etc/wptt/update/wptt-kiem-tra-version`
  - **Reboot server**: `/etc/wptt/service/reboot-server`

#### Start/Stop/Restart services (`service/wptt-start-main`, `service/wptt-stop-main`, `service/wptt-reboot-main`)

- **Chức năng**: quản lý các service phổ biến bằng `systemctl start|stop|restart` hoặc command tương ứng.
- **Service targets** (tuỳ OS/cài đặt):
  - Web/DB core: `lsws`, `mariadb`
  - Security: `nftables`, `firewalld`, `fail2ban`, `csf`
  - Cache: `memcached`, `lsmcd`, `redis` / `redis-server`
  - Monitoring: `monit`
  - Access: `sshd`
- **OS nuance**:
  - Redis service name khác nhau: Ubuntu thường `redis-server`, RHEL thường `redis`.
- **Safety**:
  - `wptt-stop-main` có confirm khi dừng `sshd` (rủi ro lockout).
  - Các thao tác “All services” có thể tác động firewall/ssh nếu thao tác sai → docs vận hành phải cảnh báo.
- **Rollback**:
  - Dừng nhầm → dùng `wptt-start-main`.
  - Restart nhầm → restart lại service đúng hoặc dùng `status` để verify.

#### Status overview (`service/wptt-status`)

- In bảng status bằng `systemctl is-active` cho OLS, DB, Redis, Memcached, SSH, cron, fail2ban, firewall...
- **Pitfall**:
  - Script có nhánh kiểm tra LiteSpeed bằng `lshttpd.service` (khác `lsws.service`), nên runtime có thể hiển thị khác tuỳ bản cài.

#### Watchdog: auto reboot + Telegram alert (`service/reboot-app/wptt-auto-reboot-thiet-lap` + `service/reboot-app/wptt-auto-reboot`)

- **Mục tiêu**: nếu `lsws` hoặc `mariadb` không active → tự restart và gửi Telegram.
- **State**:
  - cron file: `/etc/cron.d/reboot-check-service.cron`
  - Ubuntu symlink: `/etc/cron.d/reboot-check-service_cron`
- **Deps**:
  - Telegram setup (`telegram_api`, `telegram_id` trong `/etc/wptt/.wptt.conf`).
- **Rollback**:
  - Xoá cron file + restart `cron`/`crond`.

### Cache

- **Entrypoint**: `wptt-cache-main`
- **Sub-scripts (repo)**: `tool-wptangtoc-ols/cache/*`
  - `wptt-xoacache`: dọn cache tổng hợp theo website WordPress (OPcache + WP object cache + Redis/Memcached + plugin cache + Cloudflare).
  - `wptt-xoa-cache-redis`: flush Redis/Valkey toàn hệ thống.
  - `wptt-redis`: bật/tắt object cache Redis/Valkey (cài đặt service + php extensions).
  - `wptt-memcached`: cài/gỡ Memcached (unix socket) + PHP extensions + monit integration.
  - `wptt-kich-hoat-lsmemcaced`: cài LSMCD (LiteSpeed Memcached) từ source + enable service.
  - `wptt-bat-lsmemcached`, `wptt-tat-lsmemcached`: bật/tắt service `lsmcd`.
  - `clear-opcache`: “xoá opcache” (thực tế: restart OLS).
  - `wptt-opcache`: bật/tắt OPcache bằng cách chỉnh `opcache.revalidate_freq`.
  - `lscache` / `huy-lscache`: bật/tắt LSCache bằng `.htaccess` (khi không dùng plugin LSCache).
  - `cache-wptangtoc-page-html`: bật rewrite/page cache HTML theo cơ chế “wptangtoc plugin cache”.
  - `sitemap-yoast-seo-cache`: add OLS context cho `sitemap_index.xml` + extra headers.
  - `wptt-setup-gioi-han-dung-luong-page-cache-html`: thiết lập cron giới hạn dung lượng cache HTML (MB).
  - `delete-cache-html-page-gioi-han-dung-luong-o-cung`: worker xoá cache dirs khi vượt ngưỡng.
  - Cloudflare CDN purge API:
    - `cloudflare-api-cdn-cache-thiet-lap`: thiết lập/xoá cấu hình API purge theo domain.
    - `cloudflare-cdn-cache-xoa`: thực thi purge (được gọi từ xoá cache tổng hợp).
  - Misc:
    - `xoa-cache-full`: xoá cache cho tất cả website (loop gọi `wptt-xoacache`).
    - `php-preload`, `php-cache-preload-wptangtoc.sh`: preload cache theo plugin `wptangtoc` (WP eval + warmup URL).
    - `memcached.php`: helper flush memcached.
- **Runtime target paths (sau khi cài)**:
  - Scripts: `/etc/wptt/cache/*`
  - Per-site docroot: `/usr/local/lsws/$domain/html`
  - PHP ini per version:
    - Ubuntu: `/usr/local/lsws/lsphpXX/etc/php/<x.y>/litespeed/php.ini`
    - RHEL: `/usr/local/lsws/lsphpXX/etc/php.ini`
  - Cloudflare purge configs:
    - `/etc/wptt/cloudflare-cdn-cache/.wptt-$domain.conf` (chmod 400, dir chmod 700)

#### Cache flow — Xoá cache tổng hợp theo website (`wptt-xoacache`)

- Chọn 1 website WordPress hoặc “Tất cả website”.
- Guards:
  - domain tồn tại `/etc/wptt/vhost/.$NAME.conf`
  - phải là WordPress (`wp-load.php`)
- Core actions (theo code):
  - Load PHP CLI theo domain: `/etc/wptt/php/php-cli-domain-config $NAME`
  - Lấy danh sách plugin active (wp-cli) để chọn strategy purge.
  - **OPcache reset**: `wp eval 'opcache_reset();'`
  - **WP object cache flush**: `wp cache flush`
  - **Redis FLUSHALL** nếu config tồn tại và `redis-cli ping = PONG`
  - **Memcached flush_all** qua unix socket `/var/run/memcached/memcached.sock` bằng `socat`
  - Purge plugin cache theo plugin active:
    - LiteSpeed Cache: `wp litespeed-purge all` + optional purge object
    - Swift Performance (lite/pro): `wp sp_clear_all_cache`
    - WP Rocket: `rocket_clean_domain()`
    - W3 Total Cache: `w3tc_pgcache_flush()`
    - Cache Enabler: `Cache_Enabler::clear_total_cache()`
    - Autoptimize: `autoptimizeCache::clearall()`
    - FlyingPress: `wp flying-press purge-everything`
    - WP-Optimize: `purge_page_cache()`
    - Breeze: `do_action('breeze_clear_all_cache')`
    - FlyingProxy: `FlyingProxy\Purge::invalidate()`
  - **Cloudflare purge**:
    - Nếu có plugin cloudflare hoặc có config `/etc/wptt/.cloudflare/wptt-$NAME.ini` hoặc `/etc/wptt/cloudflare-cdn-cache/.wptt-$NAME.conf`
    - Nếu detect Cloudflare trace, gọi `cloudflare-cdn-cache-xoa $NAME`
- Ghi audit log: `/var/log/wptangtoc-ols.log`

#### Cache flow — Thiết lập Cloudflare CDN purge API (`cloudflare-api-cdn-cache-thiet-lap`)

- Lưu Cloudflare **email + Global API key** vào:
  - `/etc/wptt/cloudflare-cdn-cache/.wptt-$NAME.conf`
  - Permissions: dir `700`, file `400`
- Xác thực zone id bằng Cloudflare API `/zones?name=...` (parse JSON bằng python3).
- Nếu đã setup → cho phép xoá config (tắt tích hợp).

#### Cache flow — Flush Redis (`wptt-xoa-cache-redis`)

- Hỗ trợ Redis/Valkey:
  - OS 10 dùng `valkey`, các bản khác dùng `redis`
- Xác định path config theo OS:
  - Alma/Rocky/Oracle/RHEL 8: `/etc/<service>.conf`
  - Các bản khác: `/etc/<service>/<service>.conf`
- Nếu service ping không PONG → báo lỗi.
- Thực thi `flushall` qua `<service>-cli`.

#### Cache flow — Bật/tắt OPcache (`wptt-opcache`)

- Duyệt các `lsphpXX` đã cài trong `/usr/local/lsws/`.
- Xác định php.ini theo OS (Ubuntu vs RHEL).
- Bật/tắt dựa trên marker:
  - Nếu có `opcache.revalidate_freq=0` → coi như đang tắt → bật bằng `opcache.revalidate_freq=100`
  - Nếu không có marker → tắt bằng `opcache.revalidate_freq=0`
- Restart `lsws`.

#### Cache flow — Cài/gỡ Memcached (`wptt-memcached`)

- **Impact layers**: OS packages + systemd service + memcached unix socket + PHP extensions + (optional) SELinux + monit.
- **State / artefacts**:
  - Config path:
    - Ubuntu: `/etc/memcached.conf`
    - RHEL-like: `/etc/sysconfig/memcached`
  - Socket:
    - `/var/run/memcached/memcached.sock`
  - Service:
    - `memcached.service` (enable/start/stop/disable)
  - Monit rule:
    - `/etc/monit/conf.d/memcached` (Ubuntu/Debian) hoặc `/etc/monit.d/memcached` (RHEL-like)
- **Flow (enable/install)**:
  - Guard: RAM < 1024MB thì từ chối (khuyến nghị tối thiểu 2GB).
  - Install `memcached` + cài PHP extension cho tất cả `lsphpXX` hiện có.
  - Tạo `/var/run/memcached` + set owner (Ubuntu: `memcache`, RHEL: `memcached`).
  - Config unix socket:
    - Ubuntu: ghi full `/etc/memcached.conf` với `-s /var/run/memcached/memcached.sock -a 0766 ...`
    - RHEL: append `OPTIONS="-s '/var/run/memcached/memcached.sock' -a 0766"`
  - RHEL: set SELinux permissive cho `memcached_t` (`semanage permissive -a memcached_t`).
  - Gọi `/etc/wptt/monit` (để setup monit) rồi enable/start `memcached`, restart `lsws`.
- **Flow (disable/uninstall)**:
  - stop/disable service + remove packages + remove memcached config file.
  - Remove PHP memcached extensions (nếu không dùng `lsmcd`).
  - Remove monit rule file `memcached` và reload monit nếu active.
  - restart `lsws`.
- **Rollback**:
  - Bật lại bằng `wptt-memcached` (reinstall).
- **Pitfalls**:
  - Script dùng `yum/dnf` ngay cả trên Ubuntu (khả năng môi trường runtime có wrapper tương thích). Nếu hệ không có `yum`, flow sẽ fail → cần xác nhận distro/manager khi vận hành.
  - Socket permission `0766` khá rộng; nếu multi-user, cần review security boundary.

#### Cache flow — LSMCD (LiteSpeed Memcached) (`wptt-kich-hoat-lsmemcaced`, `wptt-bat-lsmemcached`, `wptt-tat-lsmemcached`)

- **Impact layers**: compile from source + systemd service + unix socket.
- **State / artefacts**:
  - Install dir: `/usr/local/lsmcd/`
  - Config: `/usr/local/lsmcd/conf/node.conf`
  - Socket: `/tmp/lsmcd.sock` (UDS)
  - Service: `lsmcd` (enable/start/stop/disable)
- **Flow (install)**:
  - Guard: RAM < 1024MB thì từ chối.
  - Cài PHP memcached extension cho các `lsphpXX`.
  - Install build deps, clone `https://github.com/litespeedtech/lsmcd.git`, build + `make install`.
  - Ghi `node.conf` (CACHED.ADDR=UDS:///tmp/lsmcd.sock), set `CachedProcCnt` theo CPU cores.
  - enable/start `lsmcd`, restart `lsws`.
- **Flow (toggle)**:
  - `wptt-bat-lsmemcached`: `systemctl start/enable lsmcd` + `lsmcdctrl start` + restart `lsws`.
  - `wptt-tat-lsmemcached`: `systemctl stop/disable lsmcd` + restart `lsws`.
- **Rollback**:
  - Disable bằng `wptt-tat-lsmemcached`.
- **Pitfalls**:
  - Script `wptt-bat-lsmemcached`/`wptt-tat-lsmemcached` có check điều kiện `-d /usr/local/lsmcd` ngược logic (đọc code cho thấy có thể in thông báo sai). Khi vận hành cần verify bằng `systemctl status lsmcd`.

#### Cache flow — “Clear OPcache” (`clear-opcache`)

- **Thực tế**: script chỉ restart OLS (`lswsctrl restart`) → không phải opcache_reset theo từng PHP pool.
- **Impact**: restart webserver (có thể gián đoạn ngắn).
- **Rollback**: không cần.

#### Cache flow — LSCache bằng `.htaccess` (`lscache` / `huy-lscache`)

- **Context**: chỉ dùng khi **không** dùng plugin LiteSpeed Cache; script sẽ từ chối nếu detect plugin cache khác để tránh xung đột.
- **State changes (enable)**:
  - prepend block `<IfModule LiteSpeed> CacheEnable public / ... </IfModule>` vào `/usr/local/lsws/$domain/html/.htaccess`
  - set `define('WP_CACHE', true);` trong `wp-config.php` (xoá dòng cũ rồi insert lại)
  - restart `lsws`
- **State changes (disable)**:
  - remove block trong `.htaccess` bằng `sed '/IfModule LiteSpeed/,+6d' ...`
  - restart `lsws`
- **Rollback**:
  - Bật/tắt lại bằng các script tương ứng.
- **Pitfalls**:
  - `sed` remove theo range `+6` khá “mỏng”; nếu block bị user chỉnh tay, disable có thể xoá lệch.

#### Cache flow — Page cache HTML (WPTangToc plugin cache) (`cache-wptangtoc-page-html`)

- **Guards**:
  - yêu cầu plugin file tồn tại: `/usr/local/lsws/$domain/html/wp-content/plugins/wptangtoc/class/Cache.php`
- **Impact layers**: OLS vhost config + `.htaccess` markers + redirect workflow.
- **State changes**:
  - OLS vhost config:
    - remove context cũ `context exp:^.*(html)$ { ... }` rồi append context mới vào `/usr/local/lsws/conf/vhosts/$domain/$domain.conf`
    - context này inject extra headers (cache hit marker `x-wptangtoc-cache HIT`, security headers).
  - `.htaccess`:
    - remove marker block `#begin-cache-wptangtoc ... #end-cache-wptangtoc`
    - prepend rewrite rules trỏ tới `wp-content/cache/wptangtoc/<host>/<uri>/index.html` (khi không dùng CSSAllLazy*)
  - Gọi `ssl/wptt-renew-chuyen-huong` và nếu site đang dùng vhost rewrite (`vhost_chuyen_htaccess`) thì gọi `wptt-htaccess-tat-chuyen-doi-vhost`
  - restart `lsws`
- **Rollback**:
  - Không có script “disable page cache html” riêng tại đây; rollback thực tế là xoá marker trong `.htaccess` + remove context trong vhost conf (có thể làm thủ công hoặc reinstall plugin/config).
- **Pitfalls**:
  - Đây là “rewrite ở 2 nơi”: vừa OLS vhost context vừa `.htaccess` rewrite → nếu site có rewrite custom, cần backup `.htaccess` và vhost conf trước.

#### Cache flow — Yoast sitemap cache context (`sitemap-yoast-seo-cache`)

- **State changes**:
  - Append context `context exp:^.*(sitemap_index.xml)$ { ... }` vào `/usr/local/lsws/conf/vhosts/$domain/$domain.conf`
  - restart `lsws`
- **Rollback**:
  - Remove context block bằng `sed` range tương ứng và restart `lsws`.

#### Cache flow — Giới hạn dung lượng cache HTML (cron) (`wptt-setup-gioi-han-dung-luong-page-cache-html` + `delete-cache-html-page-gioi-han-dung-luong-o-cung`)

- **Mục tiêu**: nếu cache dirs vượt ngưỡng MB thì xoá toàn bộ cache dirs (hard reset) để tránh đầy disk.
- **State**:
  - cron file per-domain: `/etc/cron.d/delete-cache-page-$domain.cron`
  - Ubuntu symlink: `/etc/cron.d/delete-cache-page-${domain//./_}_cron`
  - cron target:
    - `/etc/wptt/cache/delete-cache-html-page-gioi-han-dung-luong-o-cung $domain $MAX_MB`
- **Worker behaviour**:
  - Check size (MB) của:
    - `/usr/local/lsws/$domain/luucache`
    - `/usr/local/lsws/$domain/html/wp-content/cache`
  - Nếu vượt ngưỡng:
    - `rm -rf` target dirs
    - restart `lsws`
    - gọi `wptt-xoacache $domain` để clear các cache layer khác
- **Rollback**:
  - Remove cron file + restart `cron`/`crond`.
- **Pitfalls**:
  - Worker xoá nguyên thư mục cache khi vượt ngưỡng (không LRU/partial cleanup) → có thể gây “cold cache” spike.

#### Cache flow — Xoá cache ALL sites (`xoa-cache-full`)

- Loop qua `/etc/wptt/vhost/*` và gọi `wptt-xoacache $domain` cho những site là WordPress.
- **Impact**: có thể flush Redis/Memcached và purge Cloudflare tuỳ từng site (giống `wptt-xoacache`).

#### Cache flow — Preload cache (WPTangToc plugin) (`php-cache-preload-wptangtoc.sh` + `php-preload`)

- **Context**: preload dựa trên plugin file `/wp-content/plugins/wptangtoc/class/PreloadAllPHP.php`.
- **Flow**:
  - Load PHP CLI đúng version theo domain: `/etc/wptt/php/php-cli-domain-config $domain`
  - `wp eval 'WPTangToc\PreloadAllPHP::preload_cache();'`
  - Warmup 1 URL `https://$domain/?wptangtoc_cache=<random>` với UA “WPTangToc OLS preload cache”
  - Helper `php-preload` sẽ ping URL trong `wp-content/preload-wptangtoc-url.txt` nếu file “idle” > 110s.
- **State**:
  - `/usr/local/lsws/$domain/html/wp-content/preload-wptangtoc-url.txt` (do plugin tạo)

#### Caveats / rủi ro vận hành

- `wptt-xoacache` có thể gọi **FLUSHALL Redis** (toàn bộ), cần cảnh báo rõ vì có thể ảnh hưởng ứng dụng khác trên server.
- Cloudflare purge thiết lập đang dùng **Global API Key** (nhạy cảm), lưu dưới `/etc/wptt/cloudflare-cdn-cache/*`.
- Nhiều purge dựa vào plugin active và `wp eval` → phụ thuộc WP CLI và PHP CLI theo domain.

### Security

- **Entrypoint**: `wptt-bao-mat-main`
- **Sub-scripts**: `tool-wptangtoc-ols/bao-mat/*`
- **Layers**:
  - OLS global config (ModSecurity)
  - `.htaccess` rulesets (8G firewall, bot rules, hotlinking…)
  - filesystem hardening (LockDown chmod/chattr)
  - alerts (Telegram, SSH login, downtime)
  - system firewall/anti-ddos (CSF/nftables/XDP)

#### Web Application Firewall — ModSecurity (OLS global)

- **Scripts**:
  - Bật: `bao-mat/wptt-modsecurity`
  - Tắt: `bao-mat/wptt-tat-modsecurity`
- **Detect state**:
  - Nếu tồn tại `/usr/local/lsws/modsec/owasp/` → coi như đã bật (script sẽ từ chối bật lại).
- **Flow — bật (`wptt-modsecurity`)**:
  - Confirm “Đồng ý/Không đồng ý”.
  - Ghi audit log: `/var/log/wptangtoc-ols.log`
  - Tải OWASP Core Rule Set (CRS) từ GitHub:
    - download zip `coreruleset` version `v4.12.0`
    - unzip vào `/usr/local/lsws/modsec/owasp/`
    - rename thành `/usr/local/lsws/modsec/owasp/crs30` (tên legacy)
  - Tạo file config:
    - copy `crs-setup.conf.example` → `crs-setup.conf`
    - enable “exclusion rules”:
      - `REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf`
      - `RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf`
    - generate `/usr/local/lsws/modsec/owasp/crs30/owasp-master.conf` gồm danh sách `include` các rules chính.
  - Enable ModSecurity trong OLS **global** bằng cách append vào:
    - `/usr/local/lsws/conf/httpd_config.conf`
    - block dạng:
      - `module mod_security { modsecurity on; ... modsecurity_rules_file ... }`
  - Restart OLS: `/usr/local/lsws/bin/lswsctrl restart`
- **Flow — tắt (`wptt-tat-modsecurity`)**:
  - Confirm.
  - Xoá thư mục `/usr/local/lsws/modsec/owasp/`
  - Xoá block `module mod_security` trong `httpd_config.conf` bằng:
    - `sed -i '/mod_security/,+6d' /usr/local/lsws/conf/httpd_config.conf`
  - Restart OLS.
- **State / side-effects**:
  - Filesystem:
    - `/usr/local/lsws/modsec/owasp/` (CRS + owasp-master.conf)
  - OLS config:
    - `/usr/local/lsws/conf/httpd_config.conf` bị append block module.
  - Performance:
    - WAF có thể giảm hiệu năng đáng kể (đặc biệt trên VPS nhỏ).
- **Rollback**:
  - Chạy `wptt-tat-modsecurity` (khuyến nghị).
  - Nếu rollback thủ công:
    - remove `/usr/local/lsws/modsec/owasp/`
    - xoá block mod_security trong `httpd_config.conf`
    - restart OLS
- **Known issues / pitfalls**:
  - “Enable” dùng append `>>` vào `httpd_config.conf`: nếu file đã được chỉnh tay hoặc có nhiều block tương tự, thao tác “disable” bằng `sed '/mod_security/,+6d'` có thể xoá sai đoạn → nên backup `httpd_config.conf` trước thay đổi khi vận hành production.

#### Alerts / Monitoring (Telegram + cron + add-ons)

Các cảnh báo hiện có chia thành:

- **Core (miễn phí)**: setup Telegram + cảnh báo SSH login + cảnh báo sắp hết tài nguyên (CPU/RAM).
- **Add-ons (Premium/API)**: downtime check (và một số check khác) chuyển sang nhánh `add-one/*`.

##### Telegram setup (`bao-mat/wptt-telegram`)

- **Mục tiêu**: thiết lập token bot + chat_id/group_id để các module khác gửi cảnh báo.
- **State**: lưu vào `/etc/wptt/.wptt.conf`:
  - `telegram_api=<bot_token>`
  - `telegram_id=<chat_id_or_group_id>`
- **Flow**:
  - Nếu đã tồn tại 2 biến → hỏi confirm “thiết lập lại”, sau đó xoá dòng cũ và ghi lại.
  - Gửi thử 1 tin nhắn kiểm tra qua endpoint worker `.../bot<TOKEN>/sendMessage`.
- **Rollback**:
  - Xoá 2 dòng `telegram_api=` và `telegram_id=` khỏi `/etc/wptt/.wptt.conf`.
- **Security considerations**:
  - Đây là **secret** (bot token). Không log/commit/paste ra ngoài. Permissions của `.wptt.conf` phải chặt.

##### Alert SSH login (`bao-mat/thong-bao-login-ssh`)

- **State**: flag trong `/etc/wptt/.wptt.conf`:
  - `thong_bao_login_ssh=1` (bật), tắt thì xoá dòng.
- **Deps**:
  - yêu cầu đã setup Telegram (`telegram_api`, `telegram_id`), nếu thiếu sẽ gọi `wptt-telegram`.
- **Hook thực tế (runtime)**:
  - Installer thêm vào `/root/.bashrc`:
    - `. /etc/wptt/wptt-status`
  - `wptt-status` sẽ gửi Telegram **ngầm trong background** khi detect:
    - `thong_bao_login_ssh=1`
    - biến môi trường `SSH_CLIENT` có giá trị (lấy client IP)
- **Caveats / limitations**:
  - Đây là “hook theo interactive shell” (bashrc), không phải PAM/auditd → có thể **không bắt** mọi kiểu login (non-interactive/automation).
  - Nếu `telegram_api/telegram_id` trống, `wptt-status` vẫn cố gửi request → cần đảm bảo đã setup Telegram trước khi bật flag.

##### Alert tài nguyên (CPU/RAM) (`bao-mat/wptt-thong-bao-het-tai-nguyen-thiet-lap` + `bao-mat/wptt-thong-bao-het-tai-nguyen`)

- **Mục tiêu**: khi CPU hoặc RAM vượt ngưỡng thì gửi Telegram.
- **State**:
  - cron file: `/etc/cron.d/tai-nguyen-check.cron`
  - Ubuntu symlink cron: `/etc/cron.d/tai-nguyen-check_cron`
- **Flow**:
  - Setup chọn threshold (80/90/100) + interval phút.
  - Cron gọi:
    - `/etc/wptt/bao-mat/wptt-thong-bao-het-tai-nguyen <server_ip> <threshold>`
  - Worker script lấy CPU/RAM rồi gửi message Telegram kèm list domain từ `/etc/wptt/vhost/*`.
- **Rollback**:
  - Xoá cron file + restart `cron`/`crond`.

##### Downtime check (đã chuyển sang Add-ons / API)

- **Core scripts (legacy)**:
  - `bao-mat/wptt-canh-bao-downtime-thiet-lap` hiện điều hướng sang add-on.
  - `bao-mat/wptt-canh-bao-downtime` là legacy checker theo `curl` domain list.
- **Đường chính hiện tại**: `add-one/thiet-lap-downtimes`
  - **State**: set `download_api=1` trong `/etc/wptt/.wptt.conf`, có thể set thêm `email_check_downtime=...`
  - Cơ chế kiểm tra thực tế chạy qua `/etc/wptt/add-one/check.sh` (Premium).
#### System firewall — CSF (`csf-main`)

- **Menu**: `tool-wptangtoc-ols/bao-mat/csf-main`
  - Cài/gỡ CSF:
    - `/etc/wptt/bao-mat/csf/wptt-cai-dat-csf`
    - `/etc/wptt/bao-mat/csf/wptt-huy-cai-dat-csf`
  - Chặn quốc gia / allowlist quốc gia:
    - `chan-ip-quoc-gia`, `chan-ip-quoc-gia-chi-cho-phep`
    - `huy-bo-chan-ip-quoc-gia`, `huy-bo-chan-ip-quoc-gia-chi-cho-phep`
  - Chế độ chống DDOS (CSF mod):
    - `mod-csf-chong-ddos`, `tat-mod-csf-chong-ddos`
- **State**:
  - CSF config thường ở `/etc/csf/csf.conf` (được các module khác tham chiếu khi renew SSL).
  - Khi bật chặn quốc gia, có thể dùng `CC_ALLOW_FILTER` (đã thấy logic tạm disable CSF khi renew certbot).

#### System firewall — nftables (cài & gỡ)

##### Install nftables (`bao-mat/nftables/install-nftables.sh`)

- **Tác động lớn**:
  - Disable/mask `iptables` và `firewalld`.
  - Cài `nftables`, enable + start service.
  - Flush ruleset (`nft flush ruleset`).
  - Ghi config cơ bản từ `nftables-co-ban.conf` vào:
    - Ubuntu: `/etc/nftables.conf`
    - RHEL: `/etc/sysconfig/nftables.conf`
  - Mở port cần thiết bằng cách insert rule vào chain input:
    - SSH port đọc từ `sshd_config` (default 22)
    - WebAdmin port (OLS admin console) nếu chưa disable webconsole
    - MariaDB remote port nếu cấu hình có `port=...`
  - Chỉnh `fail2ban` banaction sang `nftables-allports` trong `/etc/fail2ban/jail.local`, rồi restart `fail2ban`.
- **Config base**: `bao-mat/nftables/nftables-co-ban.conf`
  - policy drop input, allow loopback, established/related, icmp echo-request, allow tcp 80/443, udp 443 (QUIC).
  - có table `ip blackblock` với set `blackaction` để drop IPs.

##### Remove nftables (`bao-mat/nftables/remove-nftables.sh`)

- Flush ruleset, stop/disable nftables.
- Cài lại và bật `firewalld`, mở http/https/443udp + ssh + webgui + mariadb remote (nếu có).
- Xoá file nftables config path.
- Chuyển `fail2ban` banaction về `firewallcmd-allports`, restart fail2ban.

#### Anti-DDoS — SYNPROXY / sysctl tuning

- **SYNPROXY config**: `bao-mat/nftables/synproxy-nftables`
  - Thêm table `netdev synproxy_filter` hook ingress (device hardcode `"eth0"`).
  - Dùng `notrack` cho SYN vào port 80/443 để giảm tải conntrack.
  - Sửa rule established để accept cả `untracked`.
- **Layer3 tuning**: `bao-mat/nftables/synflood-layer3-bao-mat.sh`
  - `modprobe nf_synproxy_core`
  - set sysctl liên quan conntrack/syncookies/hashsize/conntrack_max.

#### XDP (advanced)

- **Repo path**: `tool-wptangtoc-ols/bao-mat/nftables/xdp/*`
- **Mục tiêu**: chống DDoS tầng kernel bằng XDP (drop sớm) + cơ chế “log-driven block” đọc OLS log để thêm IP vào BPF map.
- **Entrypoints / scripts chính**:
  - Cài đặt & bật:
    - `install-xdp-ddos-protection`
    - `start-boot.sh` (loader attach lại XDP khi cần)
    - `auto-enable-xdp` (bật/tắt theo CPU, có nhánh reboot khi disable)
  - Gỡ bỏ:
    - `remove-xdp-ddos-protection`
  - Ops (map management):
    - `block-ip` (chặn thủ công vào `log_blacklist`)
    - `unban-ip`, `unban-all-xdp`
    - `list-block-ip-xdp` (đếm entry trong `log_blacklist` + `rl_blacklist`)
    - `bypass-ip.sh` (thêm IP vào `whitelist_map`)
- **Core program (XDP C)**: `xdp_ddos_protection.c`
  - Maps:
    - `whitelist_map` (max 256) → allowlist IP
    - `log_blacklist` (max 131072) → drop nếu `now < ban_until`
    - `rl_counters` (max 262144) → rate-limit counters
    - `rl_blacklist` (max 131072) → drop nếu `now < ban_until`
  - Rate-limit defaults:
    - window: 1s, threshold: 50 packets, ban: 30 phút
  - Optimization:
    - TCP ACK (không SYN) được pass nhanh (ưu tiên traffic đã established)
- **Runtime state / artefacts (khi đã bật)**:
  - **BPF pin** (state cốt lõi):
    - program pinned: `/sys/fs/bpf/xdp_ddos_protection_prog`
    - maps pinned (tối thiểu):
      - `/sys/fs/bpf/log_blacklist`
      - `/sys/fs/bpf/rl_blacklist` (được dọn bởi cleanup job)
      - `whitelist_map` (map name trong kernel, update bằng `bypass-ip.sh`)
  - **Systemd**:
    - `/etc/systemd/system/ddos-blocker-xdp.service`
    - runner: `/usr/local/bin/start_blocker.sh`
    - binary: `/usr/local/bin/anti`
  - **Cron/cleanup**:
    - `/etc/xdp-protection/cleanup_maps` (Go tool build từ `cleanup_maps.go`)
    - crontab: `*/5 * * * * /etc/xdp-protection/cleanup_maps > /var/log/bpf_cleanup.log 2>&1`
- **External deps**:
  - `bpftool`, `clang`, `llvm`, `libbpf-devel`, `kernel-devel`, `golang`, `jq`, `sysstat`, `bc`
- **Caveats / rủi ro vận hành (bắt buộc ghi rõ khi dùng)**:
  - **Interface hardcode**: nhiều script mặc định `eth0`. Nếu VPS dùng `ens3/eno1/...` thì attach/detach sai → phải xác định đúng NIC theo default route trước khi bật.
  - **SELinux**: `install-xdp-ddos-protection` có nhánh **tắt SELinux** (`setenforce 0` + sửa `/etc/selinux/config`) → tác động hệ thống lớn, cần policy/rollback rõ.
  - **Clock domain**: XDP dùng `bpf_ktime_get_ns()` (monotonic); tool cleanup cố đồng bộ thời gian boot để xoá entry hết hạn. Các thao tác block thủ công dùng realtime timestamp có thể lệch nếu không dọn map.
  - **Lockout risk**: nếu cấu hình/whitelist sai có thể chặn nhầm (bao gồm IP quản trị). Luôn whitelist IP quản trị trước khi enable.
- **Rollback / remove checklist**:
  - Stop/disable services:
    - `systemctl stop ddos-blocker-xdp && systemctl disable ddos-blocker-xdp`
  - Detach XDP khỏi NIC:
    - `bpftool net detach xdp dev <IFACE>`
  - Xoá pinned objects:
    - `rm -f /sys/fs/bpf/xdp_ddos_protection_prog /sys/fs/bpf/log_blacklist (/sys/fs/bpf/rl_blacklist nếu có)`
  - Xoá service file + daemon reload:
    - `rm -f /etc/systemd/system/ddos-blocker-xdp.service && systemctl daemon-reload`
  - Gỡ cron cleanup + xoá `/etc/xdp-protection`
  - Nếu `bpftool map list` vẫn còn map: **reboot** để clean hẳn (script remove có nhắc).

### IP / Fail2ban

- **Entrypoint**: `wptt-khoa-ip-main`
- **Scripts liên quan**:
  - `fail2ban/wptt-khoa-ip`
  - `fail2ban/wptt-mokhoaip`
  - `fail2ban/wptt-list`
  - `fail2ban/unban-all`
- **Impact layers**:
  - OLS global config
  - fail2ban jails
  - system firewall (CSF / nftables / firewalld)
  - audit log

#### Menu dispatch (`wptt-khoa-ip-main`)

- Menu có 4 nhánh:
  - block IP
  - unblock IP
  - list blocked IP
  - unban all
- Từ `run_action(index)`:
  - `0` → `/etc/wptt/fail2ban/wptt-khoa-ip`
  - `1` → `/etc/wptt/fail2ban/wptt-mokhoaip`
  - `2` → `/etc/wptt/fail2ban/wptt-list`
  - `3` → `/etc/wptt/fail2ban/unban-all`

#### Block IP (`fail2ban/wptt-khoa-ip`)

- **Flow chính**:
  - đọc IP nhập vào, validate IPv4/IPv6
  - append deny vào `/usr/local/lsws/conf/httpd_config.conf`
  - `fail2ban-client set sshd banip <ip>`
  - nếu có `csf` → `csf -d <ip>`
  - nếu `nftables` active → call `/etc/wptt/bao-mat/nftables/block-ip-nftables.sh <ip>`
  - nếu `firewalld` active → add rich rule reject tương ứng family ipv4/ipv6
  - restart OLS
  - ghi log `/var/log/wptangtoc-ols.log`
- **State changes**:
  - `/usr/local/lsws/conf/httpd_config.conf`
  - backend firewall active hiện tại
  - fail2ban runtime state
  - `/var/log/wptangtoc-ols.log`
- **Verify**:
  - grep deny trong `httpd_config.conf`
  - `fail2ban-client status sshd`
  - kiểm tra rule ở CSF / nftables / firewalld nếu backend tương ứng đang bật
- **Rollback**:
  - dùng `fail2ban/wptt-mokhoaip`
  - hoặc remove deny trong `httpd_config.conf`, unban fail2ban, remove rule firewall tương ứng

#### Unblock IP (`fail2ban/wptt-mokhoaip`)

- **Flow chính**:
  - unban IP ở tất cả jail fail2ban đang active
  - remove deny khỏi `httpd_config.conf`
  - `csf -dr <ip>` nếu có CSF
  - remove rich rule firewalld
  - call `bao-mat/nftables/unban.sh <ip>` nếu nftables active
  - restart OLS
  - ghi log `/var/log/wptangtoc-ols.log`
- **State changes**:
  - giống chiều ngược lại của block IP
- **Verify**:
  - grep không còn IP trong `httpd_config.conf`
  - `fail2ban-client status <jail>` không còn IP
  - rule backend firewall không còn
- **Rollback**:
  - chạy lại block IP nếu cần re-ban

#### List blocked IP (`fail2ban/wptt-list`)

- **Flow chính**:
  - tổng hợp từ:
    - `/var/log/fail2ban.log*`
    - `/etc/csf/csf.deny`
    - deny trong `httpd_config.conf`
    - rich rules của `firewalld`
  - merge, dedupe rồi in ra
- **Impact**: read-only
- **Pitfalls**:
  - list là aggregation runtime, không phải source of truth duy nhất cho mọi backend

#### Unban all (`fail2ban/unban-all`)

- **Flow chính**:
  - iterate toàn bộ jail fail2ban và unban từng IP
  - reset `accessControl` trong `httpd_config.conf`
  - restart OLS
  - restart `nftables` nếu active
  - flush deny của CSF (`csf -tf`, `csf -df`)
  - remove toàn bộ rich rules reject của firewalld
- **State changes**:
  - `/usr/local/lsws/conf/httpd_config.conf`
  - toàn bộ runtime block state của fail2ban / firewall
- **Verify**:
  - `fail2ban-client status` không còn banned list
  - `httpd_config.conf` về `allow ALL` hoặc allowlist Cloudflare tương thích proxy mode
- **Rollback**:
  - không có rollback granular; cần re-block thủ công các IP thực sự cần chặn

#### Caveats / rủi ro vận hành

- Block/unblock IP không chỉ đụng 1 backend; script cố đồng bộ nhiều lớp cùng lúc.
- Nếu backend firewall và fail2ban không cùng hướng, có thể xảy ra trạng thái “đã unban ở jail nhưng vẫn còn rule firewall”.
- `unban-all` là thao tác lớn, có thể xoá cả deny hợp lệ đã cấu hình trước đó ở OLS/firewall.

### SSH

- **Entrypoint**: `wptt-ssh-main`
- **Scripts liên quan**:
  - `ssh/wptt-passwd`
  - `ssh/wptt-ssh-port`
  - `ssh/wptt-set-timeout-ssh`
  - `bao-mat/thong-bao-login-ssh`
  - `domain/wptt-khoi-tao-user`
- **Impact layers**:
  - `sshd_config`
  - firewall / fail2ban / SELinux port mapping
  - `.wptt.conf`
  - shell hooks (`/root/.bashrc`)

#### Menu dispatch (`wptt-ssh-main`)

- Menu có 5 nhánh:
  - đổi password SSH user
  - đổi SSH port
  - setup alert login SSH qua Telegram
  - bật/tắt username riêng domain / SFTP user
  - đổi timeout SSH

#### Đổi password SSH user (`ssh/wptt-passwd`)

- **Flow chính**:
  - liệt kê user shell từ `/etc/passwd`
  - gọi `passwd <user>`
  - so md5 `/etc/shadow` trước/sau để detect thay đổi
  - ghi audit log vào `/var/log/wptangtoc-ols.log`
- **State changes**:
  - `/etc/shadow`
  - `/var/log/wptangtoc-ols.log`
- **Verify**:
  - `passwd` thành công
  - checksum `/etc/shadow` đổi
- **Rollback**:
  - không có rollback tự động; chỉ có thể đổi lại password lần nữa

#### Đổi SSH port (`ssh/wptt-ssh-port`)

- **Flow chính**:
  - đọc port hiện tại từ `/etc/ssh/sshd_config`, fallback `22`
  - guard tránh trùng port với `80`, `443`, port WebAdmin OLS, MariaDB remote port
  - rewrite `Port <new>` trong `/etc/ssh/sshd_config`
  - `semanage port -a -t ssh_port_t -p tcp <new>` nếu có SELinux tools
  - update firewall backend active:
    - firewalld add new/remove old
    - CSF update `TCP_IN`
    - nftables patch config rồi restart
  - rewrite `/etc/fail2ban/jail.d/sshd.local`
  - reload/restart `sshd`, reload fail2ban
  - update `port_ssh=<new>` trong `/etc/wptt/.wptt.conf`
- **State changes**:
  - `/etc/ssh/sshd_config`
  - `/etc/fail2ban/jail.d/sshd.local`
  - `/etc/wptt/.wptt.conf`
  - active firewall backend config
- **Verify**:
  - `sshd -t`
  - mở session SSH thứ 2 trên port mới trước khi đóng session cũ
  - `fail2ban-client status sshd`
- **Rollback**:
  - revert port cũ trong `sshd_config`
  - restore firewall rules cũ
  - restore `sshd.local`
  - restart `sshd` và reload fail2ban

#### Alert login SSH qua Telegram (`bao-mat/thong-bao-login-ssh`)

- **Context**:
  - đường alert này không chỉ là cron/service; nó dựa vào shell hook runtime
- **State changes**:
  - flag `thong_bao_login_ssh=1` trong `/etc/wptt/.wptt.conf`
  - hook `. /etc/wptt/wptt-status` trong `/root/.bashrc`
- **Verify**:
  - shell root interactive mới phải source hook
  - có Telegram token/chat ID trong `.wptt.conf`
- **Rollback**:
  - remove flag trong `.wptt.conf`
  - remove shell hook nếu muốn tắt hẳn đường alert này
- **Pitfalls**:
  - chỉ bắt tốt interactive SSH login; non-interactive automation có thể không trigger

#### Bật/tắt username riêng domain / SFTP user (`domain/wptt-khoi-tao-user`)

- **Flow chính**:
  - tạo user per-site
  - append/remove block `Match User` trong `/etc/ssh/sshd_config`
  - setup chroot/internal-sftp
  - restart `sshd` sau khi verify syntax
- **State changes**:
  - `/etc/ssh/sshd_config`
  - user/group hệ thống
  - state file per-site `/etc/wptt/vhost/.$domain.conf`
- **Verify**:
  - `sshd -t`
  - grep marker `#begin-WPTT_JAIL_<user>`
  - test login SFTP user
- **Rollback**:
  - remove block marker trong `sshd_config`
  - restart `sshd`
  - disable/remove user theo flow domain nếu cần

#### SSH timeout (`ssh/wptt-set-timeout-ssh`)

- **Flow chính**:
  - toggle giữa:
    - mặc định comment `#ClientAliveInterval 0` / `#ClientAliveCountMax 3`
    - timeout bật `ClientAliveInterval 28800` / `ClientAliveCountMax 3`
  - restart `sshd`
- **State changes**:
  - `/etc/ssh/sshd_config`
- **Verify**:
  - grep `ClientAliveInterval`
  - `sshd -t`
- **Rollback**:
  - đổi lại về trạng thái comment mặc định

#### Caveats / rủi ro vận hành

- SSH port đổi là thao tác lockout-risk cao vì đụng đồng thời `sshd`, firewall, fail2ban, SELinux.
- SFTP jail sửa trực tiếp `sshd_config`; luôn cần `sshd -t` trước restart.
- Alert login SSH là shell hook, không phải PAM; đừng kỳ vọng coverage như audit subsystem.

### PHP

- **Entrypoint**: `wptt-php-ini-main`
- **Scripts liên quan**:
  - `php/wptt-sua-phpini`
  - `php/wptt-php-ini-uploads`
  - `php/wptt-php-max-input-time`
  - `php/wptt-php-max-input-vars`
  - `php/wptt-domain-php`
  - `php/wptt-php-version-all-server`
  - `php/wptt-php-version-domain`
  - `php/php-extension-them`
  - `php/php-extension-xoa`
  - `php/wptt-php-max-execution-time`
  - `php/wptt-php-memory-limit-ram-php`
  - `php/wptt-khoi-phuc-sua-phpini`
- **Impact layers**:
  - OLS global / per-vhost config
  - PHP packages/extensions
  - `.wptt.conf` và `vhost/.$domain.conf`
  - CLI PHP symlink toàn server

#### Menu dispatch (`wptt-php-ini-main`)

- Menu gom hai nhóm thao tác:
  - sửa giá trị php.ini / extensions
  - đổi PHP version per-domain hoặc all-server

#### Sửa php.ini (`php/wptt-sua-phpini`)

- **Flow chính**:
  - detect `LSPHP` qua `php/wptt-php-version-domain` và `php/tenmien-php`
  - resolve đúng path `php.ini` theo distro:
    - Ubuntu: `/usr/local/lsws/<lsphp>/etc/php/<ver>/litespeed/php.ini`
    - RHEL-like: `/usr/local/lsws/<lsphp>/etc/php.ini`
  - mở file bằng editor cấu hình (`nano` fallback)
  - nếu checksum đổi → restart OLS
- **State changes**:
  - php.ini của version được chọn
  - `/var/log/wptangtoc-ols.log`
- **Verify**:
  - file tồn tại
  - OLS restart OK
- **Rollback**:
  - restore php.ini từ backup/editor history nếu có
  - hoặc dùng `php/wptt-khoi-phuc-sua-phpini`

#### Upload size / max_input_time / max_input_vars / max_execution_time / memory_limit

- **Scripts**:
  - `wptt-php-ini-uploads`
  - `wptt-php-max-input-time`
  - `wptt-php-max-input-vars`
  - `wptt-php-max-execution-time`
  - `wptt-php-memory-limit-ram-php`
- **Impact**:
  - edit trực tiếp php.ini của version đang chọn
  - restart OLS để apply
- **Rollback**:
  - set lại giá trị cũ
  - hoặc restore php.ini

#### Đổi PHP version theo domain (`php/wptt-domain-php`)

- **Flow chính**:
  - chọn website, đọc state file `/etc/wptt/vhost/.$NAME.conf`
  - quét repo LiteSpeed để list `lsphpXX` khả dụng
  - guard tương thích tối thiểu với WordPress core version
  - nếu thiếu package → cài thêm PHP + extensions cần thiết
  - patch `/usr/local/lsws/conf/vhosts/$NAME/$NAME.conf` sang `lsphpXX`
  - update `phien_ban_php_domain=<x.y>` trong `/etc/wptt/vhost/.$NAME.conf`
  - nếu có file manager thì đảm bảo zip extension tồn tại
  - restart OLS
- **State changes**:
  - `/usr/local/lsws/conf/vhosts/$NAME/$NAME.conf`
  - `/etc/wptt/vhost/.$NAME.conf`
  - package/extensions của `lsphpXX`
  - php.ini mặc định của version mới được cài có thể bị tune thêm
- **Verify**:
  - grep `lsphpXX` trong vhost conf
  - `php/wptt-php-version-domain`
  - test WP CLI/site response
- **Rollback**:
  - đổi domain về version cũ bằng cùng script

#### Đổi PHP version toàn server (`php/wptt-php-version-all-server`)

- **Flow chính**:
  - quét repo LiteSpeed và cài package/extensions nếu thiếu
  - patch `httpd_config.conf` sang `lsphpXX`
  - patch toàn bộ vhost conf sang version mới
  - update `php_version_check=<x.y>` trong `/etc/wptt/.wptt.conf`
  - update `phien_ban_php_domain=<x.y>` trong từng `vhost/.$domain.conf`
  - đổi symlink CLI:
    - `/usr/bin/php`
    - `/usr/local/lsws/fcgi-bin/lsphp5`
    - `/usr/local/lsws/fcgi-bin/lsphpXX`
  - restart OLS
- **State changes**:
  - `/usr/local/lsws/conf/httpd_config.conf`
  - `/usr/local/lsws/conf/vhosts/*/*.conf`
  - `/etc/wptt/.wptt.conf`
  - `/etc/wptt/vhost/.*.conf`
  - symlink CLI PHP
- **Verify**:
  - `php -v`
  - grep version mới trong OLS conf/vhost conf
  - test vài site tiêu biểu
- **Rollback**:
  - chạy lại script với version cũ

#### Kiểm tra PHP version (`php/wptt-php-version-domain`)

- **Mục đích**:
  - đọc PHP version effective theo domain để các script khác dùng đúng path/config
- **Impact**: read-only

#### Cài / xoá PHP extensions (`php/php-extension-them`, `php/php-extension-xoa`)

- **Impact**:
  - package manager + php module dirs của `lsphpXX`
- **Verify**:
  - `/usr/local/lsws/lsphpXX/bin/php -m`
- **Rollback**:
  - cài lại hoặc gỡ extension tương ứng

#### Restore php.ini (`php/wptt-khoi-phuc-sua-phpini`)

- **Mục đích**:
  - phục hồi php.ini về bản backup/template của flow PHP hiện có
- **Verify**:
  - diff/grep giá trị mục tiêu
  - restart OLS thành công

#### Caveats / rủi ro vận hành

- PHP all-server change là global-impact: đụng OLS global, mọi vhost, CLI symlink.
- Script có logic cài package từ repo chính hoặc fallback repo EOL; docs phải coi đây là state change mức package system.
- Nhiều flow PHP còn tune opcache/JIT và php.ini mặc định ngay khi cài version mới.

### Webserver Configuration

- **Entrypoint**: `wptt-cau-hinh-websever-main`
- **Scripts liên quan**:
  - `cau-hinh/wptt-sua-websever`
  - `cau-hinh/wptt-cau-hinh-vhost`
  - `cau-hinh/wptt-sua-mariadb`
  - `cau-hinh/wptt-cau-hinh-htaccess`
  - `cau-hinh/wptt-cron`
  - `cau-hinh/wptt-cau-hinh-redis`
  - `cau-hinh/wptt-cau-hinh-fail2ban`
  - `cau-hinh/wptt-hostname`
  - `cau-hinh/wptt-editor-cau-hinh`
- **Impact layers**:
  - OLS global config
  - OLS per-vhost config
  - `.htaccess`
  - MariaDB / Redis / fail2ban configs
  - cron
  - hostname/system identity

#### Menu dispatch (`wptt-cau-hinh-websever-main`)

- Menu này là “generic config editor hub”.
- Hầu hết action mở editor hoặc gọi helper sửa file config trực tiếp rồi quay lại menu.
- Đây là cụm high-risk vì người dùng có thể sửa tay các file nền tảng.

#### Sửa OLS global config (`cau-hinh/wptt-sua-websever`)

- **Context**:
  - sửa trực tiếp file cấu hình OLS global
- **State changes**:
  - chủ yếu ở `/usr/local/lsws/conf/httpd_config.conf`
- **Verify**:
  - restart OLS thành công
  - site vẫn response
- **Rollback**:
  - restore file backup trước khi edit

#### Sửa vhost config (`cau-hinh/wptt-cau-hinh-vhost`)

- **State changes**:
  - `/usr/local/lsws/conf/vhosts/<domain>/<domain>.conf`
- **Verify**:
  - grep context/listener/handler mục tiêu
  - restart OLS
- **Rollback**:
  - restore backup vhost conf

#### Sửa MariaDB config (`cau-hinh/wptt-sua-mariadb`)

- **State changes**:
  - distro-specific MariaDB config file
- **Verify**:
  - `mariadb` hoặc `systemctl status mariadb`
- **Rollback**:
  - restore config backup rồi restart DB

#### Sửa `.htaccess` template/path (`cau-hinh/wptt-cau-hinh-htaccess`)

- **State changes**:
  - `.htaccess` của website được chọn
- **Verify**:
  - redirect/rule không loop
  - admin vẫn truy cập được
- **Rollback**:
  - restore `.htaccess` backup hoặc remove marker block

#### Sửa cron config (`cau-hinh/wptt-cron`)

- **State changes**:
  - `/etc/cron.d/*` hoặc user crontab tuỳ flow cụ thể
- **Verify**:
  - `crontab -l`
  - restart `cron` / `crond` nếu cần
- **Rollback**:
  - remove cron entry/file đã thêm

#### Sửa Redis config (`cau-hinh/wptt-cau-hinh-redis`)

- **State changes**:
  - redis/valkey config runtime
- **Verify**:
  - service active
  - socket/port đúng
- **Rollback**:
  - restore config và restart service

#### Sửa Fail2ban config (`cau-hinh/wptt-cau-hinh-fail2ban`)

- **State changes**:
  - `/etc/fail2ban/jail.local` hoặc file liên quan
- **Verify**:
  - `fail2ban-client status`
- **Rollback**:
  - restore config backup rồi restart fail2ban

#### Đổi hostname (`cau-hinh/wptt-hostname`)

- **Impact**:
  - system identity / prompt / mail/reporting side effects
- **Rollback**:
  - set hostname cũ

#### Generic config editor (`cau-hinh/wptt-editor-cau-hinh`)

- **Context**:
  - editor tổng quát cho các file cấu hình
- **Pitfalls**:
  - vì là generic editor, mọi verify/rollback phụ thuộc loại file được chọn

#### Caveats / rủi ro vận hành

- Đây là cụm “đi thẳng vào file cấu hình”, nên docs phải coi là high-trust/high-risk.
- Bất kỳ thay đổi nào ở đây đều cần backup file trước khi mở editor.
- Nếu user chỉ nói “sửa cấu hình webserver”, AI phải xác định đúng file/layer trước khi hướng dẫn.

### Update / Maintenance

- **Entrypoint**: `wptt-update-main`
- **Sub-scripts (repo)**:
  - Core update:
    - `wptt-update` (update WPTangToc OLS stable)
    - `wptt-update2` (reinstall/switch branch stable/beta, có backup để rollback)
    - `wptt-update-wptangtoc-ols` (entry dùng cho auto-update cron)
  - System / components:
    - `update/wptt-update-he-thong` (update OS packages + refresh WPTT + reboot)
    - `update/wptt-update-phpmyadmin` (update phpMyAdmin)
    - `update/wptt-wp-cli` (install/update WP-CLI)
    - `update/wptt-kiem-tra-version` (in version inventory)
  - Automation / rollback:
    - `update/wptt-auto-update` (setup cron auto update WPTT)
    - `update/wptt-tat-auto-update` (remove cron)
    - `update/wptt-rollback-update-wptangtoc` (rollback từ backup zip)

#### Update menu dispatch (`wptt-update-main`)

- Menu gọi các script ở `/etc/wptt/` và `/etc/wptt/update/`.
- Có 2 “nhánh update WPTT”:
  - `wptt-update`: update stable theo version check.
  - `wptt-update2`: chuyển nhánh `chinhthuc`/`beta` hoặc reinstall, có logic backup/rollback.

#### Update WPTangToc OLS (stable) (`wptt-update`)

- **Flow**:
  - Fetch version mới từ `https://wptangtoc.com/share/version-wptangtoc-ols.txt`
  - Nếu khác version hiện tại (`version_wptangtoc_ols` trong `.wptt.conf`) → confirm update.
  - Download zip + checksum (`checksum.txt`), verify md5.
  - **Backup để rollback**:
    - tạo zip: `/etc/wptt/wptt-backup-system/wptt_backup_<time>_version:<old>.zip`
    - exclude `.wptt.conf` và `vhost/*` (giữ state runtime)
    - thêm `/usr/bin/wptangtoc` và `/usr/bin/wptt` vào zip.
    - giữ tối đa 10 bản zip.
  - Copy `tool-wptangtoc-ols/*` → `/etc/wptt/` và refresh `/usr/bin/wptangtoc`, `/usr/bin/wptt`.
  - Chạy script phụ remote `https://wptangtoc.com/share/update` nếu có marker `UPDATE_DONE="YES"`.
  - Update `version_wptangtoc_ols` trong `/etc/wptt/.wptt.conf`, ghi audit log.
- **State**:
  - backups: `/etc/wptt/wptt-backup-system/*.zip`
  - version: `version_wptangtoc_ols=...` trong `/etc/wptt/.wptt.conf`
- **Rollback**:
  - dùng `update/wptt-rollback-update-wptangtoc` (khôi phục từ zip backup).
- **Pitfalls**:
  - Update là “overwrite runtime scripts” (trừ `.wptt.conf`/`vhost/*`) → cần tách rõ “template vs runtime state”.
  - Remote script `update` chạy tự động: phải coi là **impact layer system** và ghi rõ khi audit.

#### Auto update WPTangToc OLS (cron) (`update/wptt-auto-update` + `update/wptt-tat-auto-update`)

- **State**:
  - cron: `/etc/cron.d/wptangtoc-ols.cron`
  - Ubuntu symlink: `/etc/cron.d/wptangtoc-ols_cron`
- **Cron target**:
  - chạy `/etc/wptt/wptt-update-wptangtoc-ols` theo lịch.
- **Rollback**:
  - `wptt-tat-auto-update` xoá cron file + restart `cron`/`crond`.

#### Rollback update (`update/wptt-rollback-update-wptangtoc`)

- Chọn 1 file zip trong `/etc/wptt/wptt-backup-system/` → unzip vào `/etc/wptt` rồi refresh `/usr/bin/wptangtoc`, `/usr/bin/wptt`.
- Update `version_wptangtoc_ols` trong `.wptt.conf`.

#### Update toàn bộ hệ thống (OS packages) (`update/wptt-update-he-thong`)

- **Impact**: rất lớn (dnf/yum update + restart services + **reboot server**).
- **Flow**:
  - refresh WPTT scripts từ zip + chạy script remote `update`
  - `dnf update -y` (có nhánh exclude kernel/grub nếu `/boot` nằm partition khác `/`)
  - restart `sssd`, restart OLS
  - reinstall WP-CLI (download phar)
  - reboot
- **Rollback**:
  - không có rollback OS packages “1 bước” trong script; rollback ở mức WPTT có thể dùng backup zip, nhưng OS/kernel rollback cần playbook riêng.

#### Update phpMyAdmin (`update/wptt-update-phpmyadmin`)

- Guard:
  - yêu cầu phpMyAdmin đang được kích hoạt (dir `/usr/local/lsws/$Website_chinh/html/phpmyadmin` + biến login).
- Update strategy:
  - replace directory phpMyAdmin theo version hardcode (hiện là `5.2.2`)
  - regenerate `config.inc.php` và tạo blowfish secret
  - chown `nobody:nogroup|nobody:nobody`, chmod tempdir.

#### Update WP-CLI (`update/wptt-wp-cli`)

- Nếu chưa có `wp` → download `wp-cli.phar` vào `/usr/local/bin/wp` (fallback `/usr/bin/wp`).
- Nếu đã có → `wp cli update`.

### WebAdmin / Web tools

- **WebAdmin**: `wptt-webadmin-main`
- **phpMyAdmin**: `wptt-phpmyadmin-main`
- **File manager**: `wptt-quan-ly-files-main`

#### OLS WebAdmin GUI (`wptt-webadmin-main` + `webadmin/*`)

- **Impact layer**: OLS admin config + firewall ports + credentials.
- **State**:
  - enable/disable marker:
    - `/usr/local/lsws/conf/disablewebconsole` (tồn tại = webadmin bị disable)
  - port/config:
    - `/usr/local/lsws/admin/conf/admin_config.conf` (listener `adminListener`)
    - `port_webgui_openlitespeed=...` trong `/etc/wptt/.wptt.conf`
  - credentials:
    - `/usr/local/lsws/admin/conf/htpasswd`
    - plaintext trong `/etc/wptt/.wptt.conf`:
      - `Ten_dang_nhap_ols_webgui=...`
      - `Password_OLS_webgui=...`
- **Flow (enable)**: `webadmin/wptt-mo-webgui`
  - Remove `/usr/local/lsws/conf/disablewebconsole` → restart OLS.
  - Optional đổi port (default 19019) + kiểm tra tránh trùng SSH/MariaDB remote port.
  - Rewrite listener `adminListener` trong `admin_config.conf`.
  - Mở port trên:
    - `firewalld` (nếu active)
    - `CSF` (`/etc/csf/csf.conf` TCP_IN)
    - `nftables` (insert rule + restart) nếu active.
  - Regenerate username/password → ghi vào `htpasswd` và lưu plaintext vào `.wptt.conf`.
- **Flow (disable)**: `webadmin/wptt-dong-webgui`
  - `touch /usr/local/lsws/conf/disablewebconsole` + restart OLS.
  - Remove port khỏi `firewalld`/CSF/nftables (nếu đang dùng).
- **Flow (rotate creds)**: `webadmin/wptt-thay-doi-password-user`
  - Generate new random username/password, update `htpasswd`, update `.wptt.conf`.
- **Rollback**:
  - Disable webadmin (khuyến nghị) để đóng port + giảm attack surface.
  - Nếu đổi port sai: chạy lại “đổi port” về port mong muốn (hoặc disable).
- **Pitfalls / security**:
  - WebAdmin mở 1 port public → tăng bề mặt tấn công, nên chỉ bật khi cần và giới hạn firewall/allowlist.
  - Plaintext creds trong `.wptt.conf` là **secret**; phải bảo vệ permission và không log.

#### phpMyAdmin (`wptt-phpmyadmin-main` + `phpmyadmin/*` + `update/wptt-update-phpmyadmin`)

- **Impact layer**: filesystem + OLS vhost config + realm/htpasswd + plaintext creds.
- **Install flow**: `phpmyadmin/wptt-mo-phpmyadmin`
  - Download phpMyAdmin (version hardcode 5.2.2) vào:
    - `/usr/local/lsws/$Website_chinh/html/phpmyadmin`
    - tempdir `/usr/local/lsws/phpmyadmin`
  - Generate blowfish secret trong `config.inc.php`.
  - Tạo realm + context `/phpmyadmin/` trong:
    - `/usr/local/lsws/conf/vhosts/$Website_chinh/$Website_chinh.conf`
    - htpasswd: `/usr/local/lsws/$Website_chinh/passwd/.phpmyadmin` (chmod 400)
  - Lưu plaintext creds vào `/etc/wptt/.wptt.conf`:
    - `id_dang_nhap_phpmyadmin=...`
    - `password_dang_nhap_phpmyadmin=...`
  - Restart OLS.
- **Uninstall flow**: `phpmyadmin/wptt-xoa-phpmyadmin`
  - Remove dir + remove realm/context khỏi vhost conf + xoá htpasswd + xoá creds trong `.wptt.conf`, restart OLS.
- **Info/rotate creds**:
  - `phpmyadmin/wptt-thongtin-phpmyadmin` (in lại URL + creds + DB creds từng site)
  - `phpmyadmin/wptt-phpmyadmin-doi-password` (regen htpasswd + update `.wptt.conf`)
- **Update**: `update/wptt-update-phpmyadmin`
  - Guard: phải đang kích hoạt, rồi replace directory, regen config + blowfish secret.
- **Rollback**:
  - Có thể uninstall rồi cài lại.
  - Nếu update lỗi: restore từ backup filesystem (không có auto-backup trong script update).
- **Pitfalls / security**:
  - Plaintext creds trong `.wptt.conf` + DB creds nằm trong `vhost/.$domain.conf` → tuyệt đối không log.
  - phpMyAdmin là target phổ biến → nên hạn chế public exposure, cân nhắc disable khi không dùng.

#### File Manager per-site (`wptt-quan-ly-files-main` + `trinh-quan-ly-files/*`)

- **Impact layer**: filesystem per-site + OLS vhost realm/context + per-site secrets.
- **Install flow**: `trinh-quan-ly-files/kich-hoat-quan-ly-files`
  - Chọn domain chưa bật; download `quan-ly-files.zip` vào:
    - `/usr/local/lsws/$domain/html/quan-ly-files/`
  - Generate 2 lớp credential:
    - **App credential** (id/password) lưu trong `/etc/wptt/vhost/.$domain.conf`:
      - `id_quan_ly_files=...`
      - `password_quan_ly_file=...`
      - được “mã hoá”/ghi vào json dưới `/usr/local/lsws/$domain/passwd/.user-giatuan.json` và `.password-giatuan.json`
    - **HTTP basic (cap 2)**: realm htpasswd `/usr/local/lsws/$domain/passwd/.files`
      - plaintext lưu trong `/etc/wptt/vhost/.$domain.conf`:
        - `id_dang_nhap_quan_ly_file_cap_2=...`
        - `password_dang_nhap_quan_ly_file_cap_2=...`
  - Append realm + context `/quan-ly-files/` vào vhost conf:
    - `/usr/local/lsws/conf/vhosts/$domain/$domain.conf`
  - Fix php zip extension cho php version domain nếu thiếu.
  - Restart OLS.
- **Uninstall flow**: `trinh-quan-ly-files/huy-kich-hoat-quan-ly-files`
  - Remove realm/context khỏi vhost conf + xoá thư mục `quan-ly-files` + xoá htpasswd + xoá secrets khỏi `vhost/.$domain.conf`.
- **Info/rotate creds**:
  - `xem-lai-thong-tin-quan-ly-files` (verify json vs state rồi in lại)
  - `thay-doi-tai-khoan-password-quan-ly-files` (regen cả app creds + cap2)
- **Rollback**:
  - Uninstall và reinstall (đơn giản, idempotent theo folder tồn tại).
- **Pitfalls / security**:
  - FileManager là “rủi ro rất cao” vì cho phép thao tác file trực tiếp → chỉ bật khi cần, xong phải tắt.
  - Lưu plaintext creds trong `vhost/.$domain.conf` (chmod 700) cần nêu rõ trong docs vận hành.

### System resources

- **Swap**: `wptt-swap-main`
- **Disk**: `wptt-disk-main`
- **Logs**: `wptt-logs-main`
- **Resource monitor**: `wptt-tai-nguyen-main`

#### Disk (`wptt-disk-main` + `disk/*`)

- **Chức năng**: kiểm tra dung lượng ổ đĩa và truy vết file/thư mục lớn.
- **Sub-scripts**:
  - `disk/wptt-disk-website`: thống kê dung lượng theo website
  - `disk/wptt-larg-file-all-dung-luong-lon`: file lớn toàn hệ thống
  - `disk/wptt-larg-file-website-dung-luong-lon`: file lớn theo 1 domain
  - `disk/wptt-larg-thu-muc-all-dung-luong-lon`: thư mục lớn toàn hệ thống
  - `disk/wptt-larg-thu-muc-website-dung-luong-lon`: thư mục lớn theo 1 domain
- **Impact**: chủ yếu read-only (df/du/ls), không thay đổi state hệ thống.

#### Swap (`wptt-swap-main` + `swap/*` + `zram`)

- **Tạo swap file**: `swap/wptt-tao-swap`
  - **State changes**:
    - tạo `/var/swap.1` (dd + mkswap + swapon)
    - sửa `/etc/fstab`:
      - comment các dòng swap cũ
      - append `'/var/swap.1 none swap defaults 0 0'`
  - **Guard**:
    - nếu đang dùng zram (`/etc/systemd/zram-generator.conf`) → từ chối tạo swap truyền thống.
- **Xoá swap file**: `swap/wptt-xoa-swap`
  - `swapoff -a`, xoá `/var/swap.1`, xoá dòng `swap.1` trong `/etc/fstab`
  - guard tương tự nếu đang dùng zram.
- **ZRAM**:
  - entry trong menu swap gọi `/etc/wptt/zram` (repo: `tool-wptangtoc-ols/zram`).
  - **Detect state**:
    - Ubuntu/Debian: tồn tại `/etc/default/zramswap`
    - RHEL-like: tồn tại `/etc/systemd/zram-generator.conf`
  - **Flow (enable)**:
    - Tính size:
      - RAM \(\le 8GB\): ZRAM = 50% RAM
      - RAM \(> 8GB\): cap 4096MB
    - Ubuntu/Debian:
      - `apt-get install zram-tools`
      - config `/etc/default/zramswap` (ALGO=zstd, SIZE=<MB>, PRIORITY=100)
      - `systemctl restart zramswap && systemctl enable zramswap`
    - RHEL-like:
      - `yum install zram-generator`
      - chọn compression algo theo `/sys/block/zram0/comp_algorithm` (ưu tiên `zstd`, fallback `lz4`, rồi `lzo`)
      - config `/etc/systemd/zram-generator.conf` (`[zram0] ... fs-type=swap`)
      - `systemctl enable systemd-zram-setup@zram0.service && systemctl start ...`
    - Kernel tuning:
      - tạo `/etc/sysctl.d/99-wptt-zram.conf`:
        - `vm.page-cluster = 0`
        - `vm.swappiness = 100`
      - có nhánh sửa thêm `tuned.conf` và `/etc/sysctl.d/101-sysctl.conf` để set `vm.swappiness=100`
      - apply `sysctl -p ...`
    - Dọn swap đĩa cũ (state change lớn):
      - `swapoff -a`
      - xoá `/var/swap.1`, `/swapfile`, `/swap.img`
      - comment/remove swap entries trong `/etc/fstab`
  - **Flow (disable/uninstall)**:
    - Stop/disable service:
      - Ubuntu: `zramswap`
      - RHEL: `systemd-zram-setup@zram0.service`
    - Remove package:
      - Ubuntu: purge `zram-tools`
      - RHEL: remove `zram-generator`
    - Remove configs:
      - `/etc/default/zramswap` hoặc `/etc/systemd/zram-generator.conf`
      - `/etc/sysctl.d/99-wptt-zram.conf`
    - Restore `vm.swappiness=10` (tuned + `101-sysctl.conf` nếu có), apply sysctl
    - **Khởi tạo lại swap đĩa 1GB**:
      - tạo `/var/swap.1` + `mkswap` + `swapon` + append vào `/etc/fstab`
- **Rollback**:
  - Swap: chạy “xóa swap” để revert.
  - Nếu fstab bị sai: restore `/etc/fstab` từ backup hệ thống (không có backup tự động trong script).

#### Resource monitor / Network / Bandwidth (`wptt-tai-nguyen-main` + `tai-nguyen-server/*`)

- **Chức năng**: kiểm tra CPU/RAM/disk/network, xem monitor (gotop/top), và quản lý băng thông.
- **Sub-scripts**:
  - CPU/RAM/Disk/ảo hoá/network:
    - `tai-nguyen-server/wptt-kiem-tra-cpu`
    - `tai-nguyen-server/wptt-kiem-tra-ram`
    - `tai-nguyen-server/wptt-kiem-tra-disk`
    - `tai-nguyen-server/wptt-kiem-tra-ao-hoa-vps`
    - `tai-nguyen-server/wptt-kiem-tra-mang-internet`, `wptt-kiem-tra-ping-internet` (+ `.py`)
  - Monitor:
    - `logs/wptt-xem-tien-trinh` (có logic auto-install `gotop` rpm nếu thiếu)
    - `tai-nguyen-server/wptt-top`
  - Bandwidth menu:
    - `tai-nguyen-server/wptt-bang-thong-main` → dispatch tới `bang-thong/*`
      - `install-tool-bang-thong-do-luong`, `xem-bang-thong-live-realtime`, `xem-bang-thong-5-phut`, `xem-bang-thong-gio`, `xem-bang-thong-gio-do-hoa`, `xem-bang-thong-ngay`, `xem-bang-thong-thang`, `xem-bang-thong-nam`
- **Impact**:
  - Một số nhánh có thể cài thêm tool (ví dụ `gotop` rpm) → là state change (packages).

#### Logs (`wptt-logs-main` + `logs/*`)

- **Chức năng**: xem/bật/tắt/xoá logs ở OLS server-level và per-domain.
- **State / impact layers**:
  - OLS global config:
    - `logs/wptt-bat-tat-access-log-server` append/remove block `accesslog` trong `/usr/local/lsws/conf/httpd_config.conf`, tạo/truncate `/usr/local/lsws/logs/access.log`, restart OLS.
  - OLS vhost config (per-domain):
    - `logs/wptt-bat-logs-domain`: add `errorlog` + `accesslog` blocks vào `/usr/local/lsws/conf/vhosts/$domain/$domain.conf`, tạo `/usr/local/lsws/$domain/logs/`, set `bat_log_domain=1` trong `/etc/wptt/vhost/.$domain.conf`.
    - `logs/wptt-tat-logs-domain`: remove blocks + xoá `/usr/local/lsws/$domain/logs/`, remove `bat_log_domain` flag.
  - Log rotation/cleanup:
    - `logs/wptt-xoa-logs` truncate nhiều log (per-domain + server logs + `/var/log/wptangtoc-ols.log`) và xoá archives.
- **Rollback**:
  - Bật/tắt logs là reversible bằng các script toggle tương ứng; “xoá logs” không rollback (data mất).

### Add-ons / Premium

- **Entrypoint**: `wptt-add-one-main`
- **Scripts liên quan**:
  - `add-one/activate-key`
  - `add-one/thiet-lap-downtimes`
  - `add-one/thiet-lap-check-ssl`
  - `add-one/thiet-lap-check-domain-het-han`
  - `add-one/add-premium`
  - `add-one/thiet-lap-auto-htaccess-optimize`
  - root helpers được gọi gián tiếp: `wptt-reset`, `wptt-htaccess-reset`, `cloudflare-cdn-ip-show-real`, `ip-scan-block`
- **Impact layers**:
  - cron/automation
  - `.htaccess`
  - cloud/API integration
  - runtime premium dependency ngoài repo hiện tại

#### Menu dispatch (`wptt-add-one-main`)

- Menu gom cả activate key, add-on monitoring, backup uploads qua Telegram, premium security/optimization và một số factory-reset helper.
- Nhiều nhánh gọi `add-one/add-premium <mode>` thay vì script tách riêng.

#### Activate key (`add-one/activate-key`)

- **State changes**:
  - key/license runtime trong `.wptt.conf` hoặc file premium runtime tương ứng
- **Verify**:
  - menu premium/action premium không còn báo chưa kích hoạt
- **Rollback**:
  - remove/deactivate key theo flow premium runtime

#### Downtime API check (`add-one/thiet-lap-downtimes`)

- **Context**:
  - đây là đường chính hiện tại cho downtime check premium/API
- **State changes**:
  - cron/service/runtime dependency premium như `/etc/wptt/add-one/check.sh`
- **Pitfalls**:
  - template source hiện tại không chứa toàn bộ implementation check runtime
- **Rollback**:
  - remove automation premium tương ứng

#### SSL expiry check / Domain expiry check

- **Scripts**:
  - `add-one/thiet-lap-check-ssl`
  - `add-one/thiet-lap-check-domain-het-han`
- **Impact**:
  - monitoring/alerts
  - thường là cron/API-based flow
- **Rollback**:
  - disable/remove setup premium tương ứng

#### Telegram uploads backup (`add-one/add-premium` với mode backup/restore/auto-backup-setup/tat-auto-backup)

- **State changes**:
  - premium backup automation
  - Telegram/cloud secret usage
- **Verify**:
  - job tồn tại và có thể upload/restore
- **Rollback**:
  - remove automation premium; giữ nguyên nguyên tắc không log secrets

#### Premium security scan (`add-one/add-premium quet-bao-mat-wordpress`)

- **Impact**:
  - chủ yếu read/analyze, nhưng có thể phát sinh report hoặc remote API usage
- **Pitfalls**:
  - behaviour chi tiết phụ thuộc runtime premium code không nằm trọn trong repo template

#### Factory reset helpers (`wptt-reset`, `wptt-htaccess-reset`)

- **Impact**:
  - high-risk; reset về trạng thái mặc định server hoặc `.htaccess`
- **Rollback**:
  - chỉ an toàn khi đã có backup config/site trước đó

#### Auto htaccess optimize (`add-one/thiet-lap-auto-htaccess-optimize`)

- **Impact**:
  - `.htaccess` per-site / automation
- **Rollback**:
  - remove automation và revert marker blocks tối ưu hóa nếu có

#### Real IP bypass Cloudflare (`cloudflare-cdn-ip-show-real`)

- **Impact**:
  - OLS / reverse proxy / trusted IP handling
- **Rollback**:
  - revert trusted proxy/real IP config đã thêm

#### Custom X-Powered-By (`add-one/add-premium custom-x-powered-by`)

- **Impact**:
  - header exposure / branding
- **Rollback**:
  - revert header customization

#### Block direct IP access (`ip-scan-block`)

- **Impact**:
  - OLS rules / firewall / anti-scan behavior
- **Rollback**:
  - remove block rule, verify domain access qua host header vẫn đúng

#### Caveats / rủi ro vận hành

- Premium/add-on là cụm dễ lệch nhất giữa repo template và runtime thật.
- Nếu không thấy source implementation đầy đủ trong repo, docs phải ghi rõ “runtime dependency” thay vì đoán.
- Các factory reset helper phải được xem là destructive/high-impact.

### Others

- **Duplicate / clone website**: `wptt-sao-chep-website`
- **Move website**: `wptt-chuyen-web-main`
- **Preload cache**: `wptt-preload-cache2`
- **Community / support**: `wptt-feedback`, `wptt-donate`, `wptt-nhom-fb`
- **Language**: `lang-main`
- **Docs search**: `search-wptangtoc-huong-dan`
- **Add-ons**: `wptt-add-one-main`

#### Duplicate / clone website (nội bộ) (`wptt-sao-chep-website`)

- **Mục tiêu**: nhân bản 1 website (code + DB + vhost config) sang domain mới trong cùng server.
- **Impact layers**:
  - OLS global/vhost config (`/usr/local/lsws/conf/httpd_config.conf`, `/usr/local/lsws/conf/vhosts/$domain/$domain.conf`)
  - filesystem (`/usr/local/lsws/$domain/*`, symlinks `/wptangtoc-ols/*`, `/home/$domain`)
  - DB (tạo DB/user mới, dump/import)
  - SSH chroot (append `Match User` vào `/etc/ssh/sshd_config`)
- **State changes (high level)**:
  - Tạo user per-site (sau đó set shell `/sbin/nologin`).
  - Tạo vhost OLS mới + map listener.
  - Tạo SSL tự ký: `/etc/wptt-ssl-tu-ky/$domain/*`.
  - Tạo DB mới + import từ site nguồn, update wp-config bằng `wp config set ...`.
  - Copy code (cp -r) + search-replace URL (`wp search-replace //old //new`).
  - Tạo state file mới: `/etc/wptt/vhost/.$domain.conf` (root-only) chứa DB creds + user + php version.
- **Rollback**:
  - Nếu clone sai → xoá website đích bằng `domain/wptt-xoa-website <domain>` (xoá vhost/files/user/db theo flow module Domain).

#### Move / migrate websites (`wptt-chuyen-web-main` + `chuyen-web/*`)

- **Menu actions**:
  - `chuyen-web/wptt-chuyen-website`: migrate từ hosting/vps ngoài về WPTangToc OLS bằng cách download `giatuan-wptangtoc.zip` + `giatuan-wptangtoc.sql` từ `http://<domain>/...` rồi import.
  - `chuyen-web/wptt-chuyen-website-all`: migrate ALL sites theo danh sách `danh-sach-website-wptangtoc-ols.txt`.
  - `chuyen-web/rsync-move`: rsync toàn bộ `/usr/local/lsws/<domain>` (multi-sites), có dump DB `.sql` vào docroot trước khi rsync.
  - `chuyen-web/rsync-move-only`: rsync 1 site, có option xoá site nguồn sau khi chuyển.
  - `chuyen-web/get-rsync-move`: phía server đích “nhận” sites sau khi rsync (tạo domain nếu thiếu, import DB từ `.sql` trong docroot, set perms, restart OLS).
  - Advanced variants:
    - `rsync-move-vs-mydumper-only` / `rsync-move-vs-mydumper-all` (dùng mydumper để dump nhanh/parallel).
- **State / artefacts**:
  - Temporary dump files trong docroot:
    - `/usr/local/lsws/$domain/html/giatuan-wptangtoc.sql` hoặc `giatuan-wptangtoc-mydumper-sql/`
    - `/usr/local/lsws/$domain/html/danh-sach-website-wptangtoc-ols.txt`
    - `subfolder-wptt.txt` (nếu dùng subfolder websites)
- **Rollback**:
  - Nếu import sai: xoá website vừa migrate và tạo lại (ưu tiên restore bằng backup/restore module nếu có backup).
  - Với rsync move: cần rollback ở 2 phía (nguồn/đích) theo trạng thái đã xoá source hay chưa.
- **Pitfalls / security**:
  - `wptt-chuyen-website` download qua `http://<domain>/...` (no TLS) → chỉ dùng trong môi trường tin cậy/đã xác thực.
  - Có thao tác `DROP DATABASE`/recreate DB → rủi ro mất dữ liệu nếu nhầm domain/state.

#### Preload cache (`wptt-preload-cache2`)

- **Mục tiêu**: crawl sitemap và “warm cache” theo plugin/page-cache đang dùng.
- **Flow (high level)**:
  - Chọn domain hoặc “Tất cả website”.
  - Detect sitemap từ các path phổ biến (`/sitemap_index.xml`, `/sitemap.xml`, `/wp-sitemap.xml`, ...).
  - Check site reachable (HTTP 200) bằng `curl --connect-to ...127.0.0.1...` để tránh dependency DNS external.
  - Guard: nếu CPU/RAM > 85% thì từ chối chạy.
  - Chạy `/etc/wptt/wptt-preload-cache-all.sh` với flags (mobile/webp) nếu detect trong htaccess/vhost context.
- **State**:
  - Chủ yếu tạo cache ở layer plugin/server; không có state file riêng của WPTT ngoài audit log.

#### Community / support entrypoints (`wptt-feedback`, `wptt-donate`, `wptt-nhom-fb`)

- **Mục tiêu**: các entry phụ từ menu chính dùng để mở luồng feedback, donate, hoặc community/support channel.
- **Entrypoint / scripts**:
  - `wptangtoc` dispatch trực tiếp tới:
    - `/etc/wptt/wptt-feedback`
    - `/etc/wptt/wptt-donate`
    - `/etc/wptt/wptt-nhom-fb`
- **Impact layers**:
  - thường là output/UI hoặc mở thông tin hướng dẫn liên hệ
  - có thể gọi browser/text client hoặc hiển thị link/channel; template source hiện tại không cho thấy thay đổi runtime state
- **State changes**:
  - không thấy state file riêng trong template source hiện tại
- **Verify**:
  - chạy entry từ menu chính, xác nhận script tồn tại ở runtime và chỉ hiển thị/điều hướng đúng nội dung mong đợi
- **Rollback**:
  - hầu như không cần rollback cấu hình; nếu runtime wrapper bị lỗi thì khôi phục script wrapper trong `/etc/wptt/`
- **Known issues / pitfalls**:
  - đây là nhóm runtime wrappers; repo template hiện tại chỉ cho thấy dispatch từ `wptangtoc`, không chứa implementation chi tiết của các wrapper này

#### Language selector (`lang-main`)

- **Mục tiêu**: đổi ngôn ngữ hiển thị cho CLI/tooling ở runtime.
- **Entrypoint / scripts**:
  - `wptangtoc` dispatch tới `/etc/wptt/lang-main`
- **Impact layers**:
  - shell UI / locale markers / text rendering ở runtime
- **State changes**:
  - template source hiện tại không cho thấy file state cụ thể; nhiều khả năng script runtime ghi marker cấu hình ngôn ngữ riêng
- **Verify**:
  - chạy menu đổi ngôn ngữ, thoát ra và vào lại menu chính để xem text/UI có đổi hay không
- **Rollback**:
  - đổi lại ngôn ngữ cũ qua cùng menu, hoặc khôi phục marker/runtime file do `lang-main` quản lý
- **Known issues / pitfalls**:
  - vì repo template không chứa implementation của `lang-main`, docs chỉ coi đây là runtime artifact được `wptangtoc` gọi trực tiếp

#### Docs search (`search-wptangtoc-huong-dan`)

- **Mục tiêu**: tìm nhanh tài liệu/hướng dẫn từ menu CLI.
- **Entrypoint / scripts**:
  - `wptangtoc` dispatch tới `/etc/wptt/search-wptangtoc-huong-dan`
- **Impact layers**:
  - shell UI / search helper
- **State changes**:
  - không thấy state file hay service/runtime mutation trong template source hiện tại
- **Verify**:
  - chạy menu search, xác nhận script mở đúng luồng tra cứu docs/hướng dẫn
- **Rollback**:
  - nếu helper lỗi thì khôi phục runtime wrapper `/etc/wptt/search-wptangtoc-huong-dan`
- **Known issues / pitfalls**:
  - đây là low-priority utility; không nên giả định có side effect cấu hình nếu chưa xác minh từ runtime thật

#### Emulation / staging (`gia-lap-website/*`)

- **Mục tiêu**: tạo site “giả lập” dưới domain phục vụ `wptangtoc-ols.com` để test/staging.
- **Dependencies**:
  - Cloudflare DNS automation (`gia-lap-website/cloudflare-dns`, `cf-dns.sh`, `auth.json`) → secrets/API token.
- **Flows**:
  - `gia-lap-website-wordpress-moi-hoan-toan`: tạo subdomain random + add DNS record + tạo site + cài WP mới.
  - `gia-lap-website-co-du-lieu`: clone site có sẵn sang subdomain staging (gọi `wptt-sao-chep-website`) + set `blog_public=0` + add `X-Robots-Tag noindex`.
- **State**:
  - flag `domain_gia_lap=1` trong `/etc/wptt/vhost/.<domain>.conf`.
- **Rollback**:
  - Xoá staging site bằng `domain/wptt-xoa-website <staging-domain>`.
