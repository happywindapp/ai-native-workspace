# API Contracts — Hub artifacts

> Hub này KHÔNG expose REST/gRPC API. "Contract" ở đây = **schema của các artifact AI** mà
> mọi tool phải tuân theo. API/contract của hệ thống thật nằm ở downstream repo
> (xem `docs/api-contracts/` trong từng project + skill phù hợp với domain của bạn).

## 1. `SKILL.md` frontmatter (Agent Skills spec)

Mỗi skill = 1 folder + `SKILL.md`. Frontmatter YAML bắt buộc:

```yaml
---
name: my-skill-name          # unique, lowercase, kebab-case
description: >-              # mô tả LÀM GÌ + KHI NÀO dùng (agent match theo field này)
  Clear description of what this skill does and when to use it.
---
```

- Nội dung markdown bên dưới = instructions + examples + guidelines.
- Nguồn spec & template: theo vendored skill library bạn dùng (`<skill-repo>/spec/`, `<skill-repo>/template/`).

## 2. `plugin.json` manifest (Claude plugin)

Mỗi plugin theo cấu trúc:

```text
plugin-name/
├── .claude-plugin/plugin.json   # manifest (name, version, skills, commands)
├── .mcp.json                    # khai báo MCP connector (tool ngoài)
├── commands/                    # slash command kích hoạt tường minh
└── skills/                      # domain knowledge nạp tự động
```

## 3. MCP connector (`.mcp.json`)

- Khai báo MCP server để agent gọi tool ngoài (Slack, Jira, Figma, PDF viewer...).
- Hub hiện kết nối: Figma (claude.ai), PDF viewer (plugin) — xem catalog MCP của session.

## Chưa áp dụng

- OpenAPI/Swagger, gRPC `.proto`: **N/A** cho hub. Tạo khi tài liệu hoá downstream service.
