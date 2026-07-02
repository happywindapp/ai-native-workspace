---
name: example-domain-skill
description: Skill mẫu theo chuẩn mở Agent Skills. Thay bằng kiến thức domain của project — mô tả rõ KHI NÀO kích hoạt (trigger keywords, tên nghiệp vụ, tên service) để agent tự chọn đúng lúc.
---

# Example Domain Skill

Skill = kiến thức chuyên sâu mà agent chỉ nạp khi task liên quan (tránh phình context).

## Viết gì vào đây

- Nghiệp vụ/flow đặc thù của project mà agent không suy ra được từ code.
- Bước-theo-bước cho task lặp lại (deploy, migration, debug pattern).
- Trỏ về single source of truth: chi tiết dài để trong `references/`, tra `docs/` qua `docs/INDEX.md`.

## Cấu trúc tuỳ chọn

```text
example-domain-skill/
├── SKILL.md          # bắt buộc — frontmatter name + description, hướng dẫn ngắn
├── references/       # tài liệu dài, agent đọc khi cần
└── scripts/          # script deterministic cho bước lặp lại
```
