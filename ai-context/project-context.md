# Project Context

> Mọi AI Agent ĐỌC FILE NÀY ĐẦU TIÊN. Nội dung rút từ mã nguồn/cấu hình thực tế của workspace của bạn.

## Elevator pitch

`{{workspace-root}}` là **AI-Native development knowledge hub**: nơi tập trung
skill/plugin, rule điều phối AI, ngữ cảnh chung và memory để **lái các AI agent
(Claude Code, Copilot, Gemini CLI, GPT Codex, Google Antigravity)** làm việc trên các hệ thống của bạn.

Hub này **không tự chạy nghiệp vụ** — mã nguồn các hệ thống thật nằm ở root khác
(xem [Downstream targets](#downstream-targets)). Đây là "buồng lái" để điều khiển AI.

## Kiến trúc high-level

- **Mô hình:** workspace tổng hợp (aggregator), KHÔNG phải microservices runtime.
- **3 trụ cột:**
  | Trụ cột | Vai trò | Vị trí |
  |---------|---------|--------|
  | Skill/Plugin libraries | Kho năng lực AI nạp động (markdown + script) | 3 git repo vendored trong hub |
  | AI config & rules | Luật điều phối agent, naming, workflow | `~/.claude/*` + `CLAUDE.md`/`GEMINI.md`/`.github/` |
  | Shared context | Memory, plans, reports, server-config | `ai-context/`, `plans/`, `docs/`, `server-config.md` |

- **Skill libraries (git repo vendored — tuỳ chọn, clone riêng):**
  | Repo | Nội dung | Quy mô |
  |------|----------|--------|
  | `<skill-repo-a>` | Skill tham chiếu + document skills (docx/pdf/pptx/xlsx) | _N_ SKILL.md |
  | `<skill-repo-b>` | Plugin theo vai trò (sales, finance, data, engineering...) | _N_ SKILL.md |
  | `<skill-repo-c>` | Skill engineering/productivity | _N_ SKILL.md |

- **Cơ chế hoạt động:** skill nạp động theo `description` → catalog sinh bởi
  `~/.claude/scripts/generate_catalogs.py` → agent kích hoạt khi liên quan. Hành vi tự động
  (before/after) cấu hình qua **hooks** trong `settings.json`.

## Downstream targets

Hub điều khiển AI làm việc trên các hệ thống thật của bạn (root riêng, có skill chuyên trích xuất kiến thức):

| Hệ thống | Root | Tech stack chính | Skill liên quan |
|----------|------|------------------|-----------------|
| `{{service-a}}` | `path/to/your-repo-a` | (ví dụ) Go, Node, React/TS | skill phù hợp với domain của bạn |
| `{{service-b}}` | `path/to/your-repo-b` | (ví dụ) Go, Node, React/TS | skill phù hợp với domain của bạn |
| `{{gateway-service}}` | `path/to/your-gateway` | (ví dụ) Go, REST↔XML/SOAP | skill phù hợp với domain của bạn |
| `{{core-system}}` | (back-office) | (ví dụ) SOAP, SQL Server | skill phù hợp với domain của bạn |

## Ranh giới & nguyên tắc

- Hub chỉ tổng hợp; KHÔNG sửa chéo vào 3 repo vendored (chúng là upstream của bên thứ ba).
- Khi làm việc trên downstream target → dùng **work-context path của project đó**, không phải CWD (xem `~/.claude/rules/orchestration-protocol.md`).
- DB luôn **READ-ONLY**. Git commit/push do user tự chạy.

## Trỏ tới chi tiết

- Workflow điều phối AI: `docs/business/`
- Cấu trúc hub & pipeline catalog: `docs/architecture/`
- Schema skill/plugin & inventory cấu hình: `docs/api-contracts/`, `docs/data-models/`
- Thuật ngữ: `docs/glossary.md`
