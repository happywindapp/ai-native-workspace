# Architecture — Knowledge Hub

> Cấu trúc tĩnh của `{{workspace-root}}` và cách nó điều khiển AI làm việc trên downstream targets.
> Rút từ cấu trúc thư mục thực tế + cấu hình `~/.claude/`.

## Thành phần

| Lớp | Thành phần | Vai trò |
|-----|-----------|---------|
| Config & rules | `~/.claude/CLAUDE.md`, `~/.claude/rules/*`, `settings.json`, hooks | Luật điều phối agent, env, hành vi tự động |
| Local AI config | `CLAUDE.md`, `GEMINI.md`, `.github/copilot-instructions.md` | Auto-load cho từng AI tool tại hub |
| Skill libraries | vendored skill/plugin repos (tuỳ chọn, clone riêng) | Kho năng lực AI nạp động (vendored git repo) |
| Shared context | `ai-context/`, `docs/`, `plans/`, `server-config.md` | Memory, tài liệu, kế hoạch, cấu hình môi trường |
| Agents | `~/.claude/agents/*` (planner, researcher, tester, code-reviewer...) | Subagent chuyên trách trong workflow |

## Cách skill được nạp

```text
SKILL.md (frontmatter: name + description)
      │  generate_catalogs.py --skills
      ▼
skills catalog  ──►  Agent đọc description  ──►  kích hoạt skill khi liên quan
```

## Sơ đồ tổng thể

```mermaid
flowchart TB
    User([Kỹ sư])

    subgraph Hub["{{workspace-root}} — AI-Native Knowledge Hub"]
        AIcfg["AI config\nCLAUDE/GEMINI/copilot"]
        Skills["Skill libraries\n(3 vendored repos)"]
        Ctx["Shared context\nai-context / docs / plans"]
        Cfg["server-config.md\n(SENSITIVE)"]
    end

    subgraph Global["~/.claude"]
        Rules["rules/* + CLAUDE.md"]
        Agents["agents/* (planner, tester...)"]
        Hooks["settings.json + hooks"]
        Catalog["scripts/generate_catalogs.py"]
    end

    subgraph Targets["Downstream targets (root riêng)"]
        SvcA["{{service-a}}\npath/to/your-repo-a"]
        SvcB["{{service-b}}\npath/to/your-repo-b"]
        Gw["{{gateway-service}}\npath/to/your-gateway"]
    end

    User -->|prompt| AIcfg
    AIcfg --> Rules
    Rules --> Agents
    Catalog --> Skills
    Agents -->|đọc| Skills
    Agents -->|đọc| Ctx
    Agents -->|làm việc trên| Targets
    Cfg -.endpoint/cred.-> Targets
    Hooks -.inject context.-> AIcfg
```

## Luồng điều phối 1 task (sequence)

```mermaid
sequenceDiagram
    actor U as Kỹ sư
    participant CC as Claude Code (hub)
    participant PL as planner
    participant IM as implement/fullstack-dev
    participant TS as tester
    participant CR as code-reviewer
    participant T as Downstream repo

    U->>CC: yêu cầu task
    CC->>PL: tạo plan + TODO (plans/)
    PL-->>CC: plan
    CC->>IM: implement theo plan
    IM->>T: sửa file thật + compile
    CC->>TS: chạy test
    TS-->>CC: pass/fail
    CC->>CR: review code sạch
    CR-->>CC: nhận xét
    CC-->>U: kết quả + lệnh git (user tự commit)
```

## Nguyên tắc kiến trúc

- Hub **không sửa chéo** 3 repo vendored (upstream bên thứ ba) — chỉ tham chiếu/kích hoạt.
- Mỗi downstream target có git root + work-context riêng; subagent luôn nhận đúng path.
- Quyết định kiến trúc đã chốt → ghi ADR trong `docs/decisions/`.
