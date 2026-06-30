# GEMINI.md — AI-Native Knowledge Hub (`{{workspace-root}}`)

> [AUTO-LOAD] Gemini CLI (v0.1.6+) đọc tự động. Nội dung đồng bộ `CLAUDE.md` — giữ ngắn.

## Workspace là gì

`{{workspace-root}}` là **knowledge hub điều phối AI** cho phát triển các hệ thống của bạn,
KHÔNG phải runtime microservices. Đọc `ai-context/project-context.md` đầu tiên.

## Tech stack (hub)

- Skills: Markdown + YAML frontmatter (`SKILL.md`).
- Config/manifest: JSON (`plugin.json`, `settings.json`).
- Scripts: Python (catalog), PowerShell + Bash.
- Downstream targets thật: tech stack tuỳ hệ thống của bạn (ví dụ Go, Node, React/TS, .NET/SOAP gateway).

## Naming & clean code

- File kebab-case mô tả rõ mục đích. Code theo convention ngôn ngữ (Go snake_case, C# PascalCase).
- YAGNI · KISS · DRY. Diff hẹp. Comment tối thiểu (mặc định không comment). File < ~200 dòng.

## Guardrails

- DB **READ-ONLY** (chỉ SELECT/metadata).
- KHÔNG tự `git commit`/`git push` — đưa lệnh cho user.
- Skill mới → `~/.claude/skills/`. Cài tool → `C:\_devtools\`.

## Điều hướng

1. `ai-context/project-context.md` 2. `docs/INDEX.md` 3. `ai-context/working/sprint-current.md` 4. `ai-context/memory/`

