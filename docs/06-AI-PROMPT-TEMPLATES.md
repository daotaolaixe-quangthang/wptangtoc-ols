# AI Prompt Templates — WPTangToc OLS (token‑efficient)

Mục tiêu: giúp bạn viết prompt ngắn nhưng “đóng khung” đủ để AI Agent luôn:

- đúng context (template vs runtime),
- đúng flow (impact layer + state + rollback),
- ít đọc thừa (token discipline),
- output đúng format.

> Bạn có thể copy/paste các mẫu dưới đây, chỉ cần thay `<...>`.

---

## Template 0 — Universal wrapper (dán trước mọi request)

```text
Bạn đang làm việc trong repo template `E:\2WEBApp\wptangtoc-ols` (KHÔNG phải runtime `/etc/wptt`).

Goal (1 câu): <...>

Allowed reads (hard limit): 
- docs/ (chỉ các file mình nêu)
- tool-wptangtoc-ols/<module>/... (chỉ các file liên quan)
Không được scan toàn repo nếu chưa cần.

Mandatory output format:
Context → Impact layer → State changes → Rollback → Verify

Safety rules:
- Firewall/anti-ddos: safe-by-default + risk lockout + rollback-first + pre-check SSH port
- Secrets: chỉ nêu location, không in giá trị
- Data loss ops (wipe/drop/restore): phải cảnh báo + yêu cầu backup

Deliverables:
1) <...>
2) <...>
```

---

## Template 1 — Docs-only (bổ sung/chốt tài liệu)

```text
Goal: cập nhật docs cho module <MODULE>.

Read:
- docs/02-MODULES-INDEX.md (chỉ section <heading>)
- tool-wptangtoc-ols/<module>/* (chỉ các script liên quan được nhắc trong section)

Update:
- docs/02-MODULES-INDEX.md: phải có Context/Impact/State/Rollback/Verify

Do NOT:
- sửa code trừ khi phát hiện bug nghiêm trọng và có rollback rõ
```

---

## Template 2 — Bugfix script (1 module)

```text
Goal: fix bug <symptom> trong <script>.

Read:
- docs/02-MODULES-INDEX.md section <module>
- tool-wptangtoc-ols/<module>/<script>
- các helper mà script source/include (nếu có)

Constraints:
- giữ conventions: state file paths, markers, menu index 0-based, exec pattern
- thay đổi phải idempotent

Deliverables:
- patch code
- patch docs/02-MODULES-INDEX.md (nếu thay đổi behaviour)
```

---

## Template 3 — Firewall / anti‑DDoS (high risk)

```text
Goal: <enable/disable/tune> <feature> (CSF/nftables/SYNPROXY/XDP).

Read (minimum):
- docs/03-AI-AGENT-GUIDE.md (firewall rules)
- docs/05-RUNBOOKS.md (rollback-first)
- docs/02-MODULES-INDEX.md section Security/<feature>
- tool-wptangtoc-ols/bao-mat/<...> scripts involved

Mandatory output:
- pre-check SSH port + current access plan
- risk lockout
- rollback-first exact steps
```

---

## Template 4 — Database destructive ops

```text
Goal: <wipe/drop/restore/remote-access> for domain <domain>.

Read:
- docs/02-MODULES-INDEX.md section Database + Backup/Restore
- tool-wptangtoc-ols/db/<script>
- tool-wptangtoc-ols/backup-restore/<script> (nếu liên quan)

Mandatory:
- state files impacted: /etc/wptt/.wptt.conf, /etc/wptt/vhost/.$domain.conf, wp-config.php
- backup requirement + rollback plan
```

