# Platform-Agnostic Capabilities -- WPTangToc OLS Extracted

Muc tieu: tach cac nang luc loi cua `wptangtoc-ols` ra khoi implementation PHP/WordPress/OLS cu the, de AI Agent co the hoc va dung lai cho stack khac.

File nay khong mo ta "script nao chay ra sao" ma mo ta "he thong nay dang giai quyet bai toan van hanh gi".

---

## 1) Capability model tong quat

Tu source va docs hien tai, he thong nay co the duoc tach thanh cac capability doc lap:

1. App provisioning
2. Domain and routing binding
3. TLS lifecycle management
4. Runtime version and execution environment management
5. Data lifecycle: backup / restore / clone / migrate
6. Security hardening and abuse protection
7. Performance and cache optimization
8. Scheduled automation and background supervision
9. Access control and operator workflows
10. Observability, verification, rollback

Neu clone sang stack khac, nen clone theo capability, khong clone theo ten file shell hay ten module PHP.

---

## 2) Capability -> trong WPTangToc OLS dang hien than thanh gi

### 2.1 App provisioning

- **Trong project nay**:
  - tao website, tao docroot, tao user, tao DB, cai WordPress, map vhost
- **Ban chat generic**:
  - tao mot application unit co du state de chay tren production
- **Output generic can co**:
  - app directory
  - app owner / execution user
  - app config
  - domain binding
  - runtime binding
  - initial secrets/config bootstrap

### 2.2 Domain and routing binding

- **Trong project nay**:
  - OLS listener, vhost, domain mapping, redirects, `.htaccess`
- **Ban chat generic**:
  - map incoming traffic vao dung app/service
- **Output generic can co**:
  - reverse proxy/site definition
  - route rules
  - redirect rules
  - static asset policy

### 2.3 TLS lifecycle management

- **Trong project nay**:
  - SSL issue, renew, self-signed fallback, listener/vhost SSL wiring
- **Ban chat generic**:
  - provision, rotate, verify va rollback TLS cho app domain
- **Output generic can co**:
  - cert source
  - key path
  - renew hook
  - verification check

### 2.4 Runtime version and execution environment management

- **Trong project nay**:
  - multi PHP version, php.ini, CLI vs web PHP, per-domain version
- **Ban chat generic**:
  - moi app can duoc gan vao dung runtime version va execution model
- **Output generic can co**:
  - runtime version per app
  - global default runtime
  - app-specific env vars
  - process manager / handler mapping

### 2.5 Data lifecycle

- **Trong project nay**:
  - backup, restore, clone, move, dump/import DB, rsync data
- **Ban chat generic**:
  - sao luu, nhan ban, khoi phuc, di chuyen data va app state
- **Output generic can co**:
  - backup artefacts
  - restore flow
  - migration flow
  - data loss warning + rollback expectations

### 2.6 Security hardening and abuse protection

- **Trong project nay**:
  - file hardening, `.htaccess` rules, ModSecurity-like rules, fail2ban, nftables, XDP, bot blocks
- **Ban chat generic**:
  - giam attack surface o app layer va network layer
- **Output generic can co**:
  - app-layer security rules
  - proxy-layer restrictions
  - host firewall
  - ban/unban workflow
  - anti-abuse automation

### 2.7 Performance and cache optimization

- **Trong project nay**:
  - Redis, object/page cache, preload cache, disk/io/sysctl tuning
- **Ban chat generic**:
  - toi uu response path va tai nguyen server
- **Output generic can co**:
  - caching policy
  - warmup/preload jobs
  - runtime tuning
  - disk and memory tuning

### 2.8 Scheduled automation and background supervision

- **Trong project nay**:
  - cron inventory, systemd units, helper daemons, health checks
- **Ban chat generic**:
  - viec lap lich, monitor, tu dong xu ly su co, keep-alive control plane
- **Output generic can co**:
  - cron/scheduler jobs
  - background services
  - service overrides
  - maintenance tasks

### 2.9 Access control and operator workflows

- **Trong project nay**:
  - SSH user, SFTP jail, login alert, file manager, phpMyAdmin, WebAdmin
- **Ban chat generic**:
  - quan tri vien can duong vao an toan de van hanh app
- **Output generic can co**:
  - admin access paths
  - scoped operator user
  - audit/alert hooks

### 2.10 Observability, verification, rollback

- **Trong project nay**:
  - runbooks, inventories, verify per module, rollback-first voi high-impact ops
- **Ban chat generic**:
  - bat ky control plane production nao cung phai co verify va rollback ro rang
- **Output generic can co**:
  - health checks
  - runtime inventories
  - explicit rollback plan
  - change verification checklist

---

## 3) Capability contract de clone sang stack khac

Neu AI Agent muon port logic sang stack khac, moi capability nen duoc dat theo contract sau:

- **Intent**: no giai quyet bai toan gi
- **Inputs**: app name, domain, runtime version, secrets, backup source, policy
- **State**: file/config/service/database/resource nao la source of truth
- **Apply flow**: tao/sua/cap nhat cai gi
- **Verify**: bieu hien thanh cong la gi
- **Rollback**: quay lai toi thieu nhu the nao
- **Portability note**: phan nao generic, phan nao stack-specific

Khong co contract nay thi "copy logic" se tro thanh copy script be mat.

---

## 4) Nhan loi tu project nay co the hoc duoc

Nhung bai hoc co the tai su dung gan nhu nguyen ven:

- menu/control plane nen map toi state changes ro rang
- moi thay doi production nen co verify + rollback
- state per-app nen co diem tap trung
- inventories cron/systemd/shell hooks giup debug nhanh hon rat nhieu
- destructive ops phai duoc danh dau va can backup-first
- runtime truth phai duoc uu tien hon template docs khi debug server that

Nhung phan khong nen copy nguyen van:

- `.htaccess`-specific hardening
- OLS listener/vhost syntax
- WordPress/wp-cli actions
- PHP version naming va php.ini assumptions

---

## 5) Cach AI Agent nen dung file nay

Khi muon clone sang stack khac:

1. Doc file nay de lay danh sach capability can giu.
2. Doc `17-PORTING-MAP-PHP-TO-OTHER-STACKS.md` de map capability sang stack moi.
3. Doc `18-DESIGN-PATTERNS-EXTRACTED.md` de lay nguyen ly thiet ke va co che van hanh.

Neu mot tinh nang khong fit vao 1 capability ro rang, kha nang cao la dang clone theo implementation thay vi clone theo logic.
