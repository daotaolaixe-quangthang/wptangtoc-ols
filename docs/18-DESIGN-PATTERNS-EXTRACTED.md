# Design Patterns Extracted -- Reusable Control-Plane Logic

Muc tieu: rut ra cac pattern thiet ke co gia tri tai su dung tu `wptangtoc-ols`, de AI Agent co the hoc va ap dung cho he thong clone khac.

File nay la "bo nao" de copy logic. Khong mo ta syntax stack cu the.

---

## 1) Pattern: Menu dispatcher -> module main -> action script

- **Mo ta**:
  - 1 entry chung dan vao cac module chinh, moi module lai dispatch tiep toi action script
- **Gia tri**:
  - de tach control plane, de khoanh bug, de mo rong module
- **Tai su dung**:
  - giu pattern nay cho CLI admin tool, web control panel, hoac API orchestration
- **Khong nen copy may moc**:
  - implementation `exec` shell cu the

## 2) Pattern: Impact-layer-first thinking

- **Mo ta**:
  - moi thay doi duoc nhin qua lop tac dong: app, config, DB, filesystem, firewall, cron, systemd
- **Gia tri**:
  - giam sua nham tang, giam debug lan man
- **Tai su dung**:
  - bat ky stack production nao cung nen co impact layer taxonomy

## 3) Pattern: Central per-app state file

- **Mo ta**:
  - per-site state duoc tap trung vao file nhu `/etc/wptt/vhost/.<domain>.conf`
- **Gia tri**:
  - debug nhanh, provision nhanh, rollback co diem bam
- **Tai su dung**:
  - clone sang stack khac bang per-app manifest/state file
- **Can than**:
  - state file phai duoc coi la source of truth co versioning/backup

## 4) Pattern: Marker-based idempotent edits

- **Mo ta**:
  - config duoc them/sua dua vao marker de tranh chen lap vo toi va
- **Gia tri**:
  - shell automation an toan hon khi chay lai
- **Tai su dung**:
  - voi config templates, generated blocks, managed sections
- **Can than**:
  - marker xau + regex xau van co the pha config; luon can diff + syntax test

## 5) Pattern: Runtime artefact inventories

- **Mo ta**:
  - docs rieng cho cron, systemd, shell hooks, root helpers
- **Gia tri**:
  - debug production nhanh hon vi agent khong bo sot artefact ngoai source chinh
- **Tai su dung**:
  - bat ky he thong co automation nen co inventory artefacts runtime

## 6) Pattern: Verify and rollback as first-class outputs

- **Mo ta**:
  - tai lieu va flow deu ep co `verify` va `rollback`
- **Gia tri**:
  - production-safe by default
- **Tai su dung**:
  - trong design clone, moi action module phai tra ve:
    - state changes
    - verify
    - rollback

## 7) Pattern: Backup-first for destructive operations

- **Mo ta**:
  - DB wipe/restore, clone, migration deu duoc coi la destructive/high-risk
- **Gia tri**:
  - dat nguong ky luat cho automation
- **Tai su dung**:
  - app nao co data cung phai co policy nay, khong phu thuoc PHP hay Node

## 8) Pattern: Runtime truth beats template truth

- **Mo ta**:
  - khi debug production, runtime wrappers/files/config la su that uu tien
- **Gia tri**:
  - tranh fix nham repo ma bo sot drift tren server
- **Tai su dung**:
  - voi moi control plane co installer/template + deployed runtime

## 9) Pattern: Split global state vs per-app state

- **Mo ta**:
  - global tuning va per-site tuning duoc tach ro
- **Gia tri**:
  - giam side effect cheo, de rollback
- **Tai su dung**:
  - clone stack nen tach:
    - global proxy/runtime defaults
    - per-app overrides

## 10) Pattern: Multi-layer security model

- **Mo ta**:
  - security khong nam o 1 cho, ma nam o app layer, webserver/proxy layer, host/network layer
- **Gia tri**:
  - bug va lockout duoc nhin dung lop
- **Tai su dung**:
  - stack moi van nen giu 3 lop nay du implementation khac nhau

## 11) Pattern: Operator-path aware design

- **Mo ta**:
  - docs co shell hooks, SSH/SFTP, admin paths
- **Gia tri**:
  - he thong van hanh khong chi co app path, ma con co operator path
- **Tai su dung**:
  - clone he thong can tinh ca deploy access, emergency access, support workflows

## 12) Pattern: Docs as operational memory, khong chi la mo ta

- **Mo ta**:
  - docs khong dung o overview; co inventory, runbook, triage, trace, risk patterns
- **Gia tri**:
  - AI Agent moi vao co the hoat dong nhu maintainer co bo nho he thong
- **Tai su dung**:
  - clone system nen co docs bo tri giong vay tu dau

---

## 2) Mau abstraction ma AI Agent nen copy

Thay vi copy module `PHP`, `SSL`, `Backup` y nguyen, agent nen copy abstraction sau:

- **Provisioning**
- **Runtime binding**
- **Routing binding**
- **Data operations**
- **Security controls**
- **Performance controls**
- **Scheduled operations**
- **Operator access**
- **Observability and rollback**

Sau do moi map abstraction do sang stack moi.

---

## 3) Anti-patterns can tranh khi clone

- copy syntax OLS/.htaccess sang stack moi
- copy ten script thay vi copy capability
- bo qua inventories runtime vi nghi source la du
- bo qua rollback/verify vi "sau nay bo sung"
- gop global va per-app state vao 1 cho
- coi CLI test la du, khong kiem app runtime truth

---

## 4) Dinh nghia "copy duoc logic" mot cach dung

AI Agent duoc coi la "copy duoc logic" khi:

- xac dinh duoc capability goc
- map duoc source of truth moi
- giu duoc verify/rollback discipline
- khong mang theo phu thuoc stack cu khong can thiet
- viet lai implementation nhung giu nguyen operational semantics

Neu chi doi ten file va doi lenh shell, do khong phai copy logic.
