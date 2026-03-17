# Known Risks Patterns -- WPTangToc OLS

Muc tieu: cho AI Agent moi co danh sach cac mau loi/pattern de san bug tiem an, review thay doi, va tranh pha vo runtime.

Khong phai bug list co san. Day la danh sach cac pattern nguy hiem lap lai trong loai project shell-installer/runtime nay.

---

## 1) Runtime drift vs template drift

- **Pattern**:
  - repo la template, nhung production chay tren `/etc/wptt/*`
- **Rui ro**:
  - sua repo dung, nhung bug production van con vi runtime wrapper da lech
- **Dau hieu**:
  - menu dispatch dung, nhung behaviour tren server khac docs/source
- **Kiem tra**:
  - so sanh runtime script voi template script
- **Safe action**:
  - coi runtime la source of truth khi debug production

## 2) `exec` menu model lam debug nham stack

- **Pattern**:
  - menu chinh/menu con `exec` sang script tiep theo
- **Rui ro**:
  - agent tuong la shell function call thong thuong va debug sai luong quay lui
- **Dau hieu**:
  - process hien tai bi thay the, quay lai menu bang cach chay lai script khac
- **Safe action**:
  - trace theo dispatch path, khong suy nghi theo call stack thong thuong

## 3) Destructive DB ops khong co undo that

- **Pattern**:
  - wipe/drop/create/restore DB trong shell scripts
- **Rui ro**:
  - mat du lieu that neu khong backup truoc
- **Dau hieu**:
  - `DROP DATABASE`, recreate user, overwrite `wp-config.php`
- **Safe action**:
  - backup-first hoac ghi ro `no data rollback`

## 4) Service name mismatch theo distro/runtime

- **Pattern**:
  - co cho dung `mysql.service`, co cho dung `mariadb.service`
- **Rui ro**:
  - verify/restart nham service, ket luan sai la fix that bai
- **Dau hieu**:
  - script/docs va runtime goi ten service khac nhau
- **Safe action**:
  - check service name that tren runtime truoc khi restart/rollback

## 5) Firewall backend mismatch

- **Pattern**:
  - fail2ban, nftables, CSF, XDP co the cung ton tai dau vet
- **Rui ro**:
  - unban o 1 layer nhung van bi chan o layer khac
- **Dau hieu**:
  - da go block o fail2ban nhung traffic van mat
- **Safe action**:
  - xac dinh backend active, rollback theo thu tu lockout-first

## 6) SSH/login hook dua vao shell interactive

- **Pattern**:
  - login alert va mot so shell hooks dua vao `.bashrc`, `SSH_CLIENT`
- **Rui ro**:
  - agent tuong day la PAM/auditd level hook, dan toi verify sai
- **Dau hieu**:
  - chi trigger khi vao interactive shell
- **Safe action**:
  - verify bang session SSH moi, khong verify bang non-interactive command

## 7) `sed -i` / text rewrite lam hu config im lang

- **Pattern**:
  - shell scripts sua truc tiep config bang `sed`, `awk`, append markers
- **Rui ro**:
  - duplicate blocks, regex match nham, syntax config vo
- **Dau hieu**:
  - config co marker lap, line bi chen sai vi tri, service restart loi
- **Safe action**:
  - truoc khi sua: backup file
  - sau khi sua: syntax test + diff + service verify

## 8) `.htaccess` va OLS config cung canh tranh layer

- **Pattern**:
  - redirect/cache/hardening co the nam mot phan trong vhost config, mot phan trong `.htaccess`
- **Rui ro**:
  - fix 1 layer nhung bug o layer kia
- **Dau hieu**:
  - site van redirect/403 du da sua `.htaccess`
- **Safe action**:
  - kiem ca global OLS, per-vhost va `.htaccess`

## 9) PHP CLI va PHP web khong cung version

- **Pattern**:
  - OLS dung `lsphpXX`, CLI co the tro version khac
- **Rui ro**:
  - agent test bang `php -v` thay on, nhung web van loi
- **Dau hieu**:
  - extension co tren CLI nhung thieu tren web, hoac nguoc lai
- **Safe action**:
  - verify ca CLI va web handler theo domain

## 10) Root helpers tao artefacts ngoai menu main

- **Pattern**:
  - helper nhu `kernel`, `oomscore`, `monit`, `disk_*` tao system state nhung khong luon di qua menu chinh
- **Rui ro**:
  - agent tim trong `wptt-*-main` khong thay va bo sot nguon gay loi
- **Dau hieu**:
  - thay cron/unit/drop-in/sysctl thay doi nhung khong truy duoc tu module docs chinh
- **Safe action**:
  - doc `12-ROOT-HELPERS-INVENTORY.md`

## 11) Legacy runtime artefacts con sot

- **Pattern**:
  - mot so unit/hook/script chi duoc thay trong cleanup path hoac runtime server cu
- **Rui ro**:
  - agent tu suy doan chuc nang khong co bang chung
- **Dau hieu**:
  - docs ghi `legacy`, `runtime artifact`, `not verified in current template source`
- **Safe action**:
  - giu wording bao thu, khong fabricate implementation

## 12) Premium/add-on route phu thuoc runtime dependency ngoai repo

- **Pattern**:
  - add-on/premium goi toi file runtime, API, script khong co trong template source
- **Rui ro**:
  - doc source khong du de ket luan full behaviour
- **Dau hieu**:
  - dispatch co, nhung body script khong co trong repo
- **Safe action**:
  - danh dau runtime dependency, verify bang runtime that

---

## 2) Cach dung file nay khi review/fix bug

1. Tim pattern giong symptom hien tai.
2. Doi chieu voi:
   - `13-BUG-TRIAGE-INDEX.md`
   - `14-SOURCE-TO-RUNTIME-TRACE.md`
   - module docs trong `02-MODULES-INDEX.md`
3. Neu pattern la `high impact`:
   - doc them `05-RUNBOOKS.md`
4. Trong output, phai goi ten pattern da nghi ngo.

Vi du:

- "Nghiem trong vi day la pattern `firewall backend mismatch`."
- "Canh bao: day la pattern `PHP CLI vs PHP web mismatch`."
- "Khong thay source create-path; xu ly nhu `legacy runtime artefact`."
