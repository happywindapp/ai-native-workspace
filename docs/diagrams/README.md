# Diagrams — Hub `{{workspace-root}}`

Lưu **cả file gốc** (draw.io / mermaid) **lẫn ảnh export** để AI đọc được. Ưu tiên mermaid (`.md`/`.mmd`) — AI đọc trực tiếp text. Với draw.io: lưu `.drawio` + export `.png/.svg` cùng tên.

## Topology hub (rút từ `ai-context/project-context.md`)

```mermaid
flowchart TB
    subgraph HUB["{{workspace-root}} — AI-Native Knowledge Hub"]
        AICFG["AI config & rules<br/>CLAUDE.md / GEMINI.md / .github/<br/>~/.claude/rules/*"]
        CTX["Shared context<br/>ai-context/ · plans/ · docs/<br/>memory/ · server-config.md"]
        subgraph SKILLS["Skill / Plugin libraries (vendored — tuỳ chọn)"]
            S1["skill-library A"]
            S2["skill-library B"]
            S3["…"]
        end
    end

    AGENTS["AI Agents<br/>Claude Code · Copilot · Gemini CLI"]
    subgraph DOWN["Downstream targets (root riêng)"]
        SVCA["{{service-a}} monorepo<br/>path/to/your-repo-a"]
        SVCB["{{service-b}}<br/>path/to/your-repo-b"]
    end

    AGENTS -->|đọc rule + nạp skill| HUB
    HUB -->|điều phối, work-context path| DOWN
    SKILLS -.->|kích hoạt theo description| AGENTS
```

## Gợi ý bổ sung

- Sequence diagram cho luồng nghiệp vụ thật → thuộc downstream (xem skill phù hợp với domain của bạn), không vẽ ở hub.
