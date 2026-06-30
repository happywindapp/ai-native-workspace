# Data Models — Hub artifacts

> Hub không có database runtime. "Data model" ở đây = **cấu trúc dữ liệu của các artifact**
> (memory, config, catalog). Schema DB của hệ thống thật nằm ở downstream repo + skill
> phù hợp với domain của bạn, và `docs/data-models/` của từng project.

## 1. Memory store (file-based)

Vị trí: `~/.claude/projects/<workspace>/memory/`. Mỗi memory = 1 file `.md`:

```markdown
---
name: <kebab-slug>
description: <1 dòng — dùng để recall>
metadata:
  type: user | feedback | project | reference
---
<nội dung; feedback/project kèm **Why:** + **How to apply:**; link [[other-memory]]>
```

- Index: `MEMORY.md` — 1 dòng/memory (`- [Title](file.md) — hook`), không chứa nội dung.

## 2. Skill catalog

- Sinh bởi `~/.claude/scripts/generate_catalogs.py --skills`.
- Khoá định danh: `name` (skill) / `plugin:skill` (scoped). Field match: `description`.

## 3. Environment config (`server-config.md`)

> ⚠️ File gốc là **SENSITIVE** (credentials UAT) — KHÔNG commit, KHÔNG nhân bản giá trị vào đây.
> Dưới đây chỉ liệt kê **inventory khóa** (không kèm giá trị bí mật):

Inventory biến môi trường (tên khóa, không secret) gom theo tier sinh từ `template.config.json` —
xem [server-config.example.md](../../server-config.example.md) (khối `gen:env`) và
[Service Registry](../services-registry.md).

- URL service trong cluster thường là K8s service DNS (chỉ resolve nội bộ). `{{host}}` là endpoint cần VPN của bạn.

## Chưa áp dụng

- ERD / migration của runtime DB: **N/A** cho hub. Lập khi tài liệu hoá downstream service.
