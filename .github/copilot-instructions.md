# Copilot Instructions — `{{workspace-root}}`

> [AUTO-LOAD] GitHub Copilot đọc tự động trong VS Code. Giữ NGẮN — chỉ stack, conventions, patterns.

## Bối cảnh

`{{workspace-root}}` = knowledge hub điều phối AI cho phát triển các hệ thống của bạn.
Không phải runtime app. Tài liệu nguồn: `ai-context/project-context.md`.

## Stack

- Hub: Markdown + YAML (`SKILL.md`), JSON (`plugin.json`/`settings.json`), Python (catalog scripts), PowerShell + Bash.
- Downstream targets: tech stack tuỳ hệ thống của bạn (ví dụ Go, Node, React/TypeScript, .NET gateway REST↔XML/SOAP).

## Conventions

- File naming: kebab-case mô tả rõ mục đích. Code: Go → snake_case; C#/Java → PascalCase; JS/TS/Python/shell → kebab-case.
- Comment: tối thiểu, 1 dòng, chỉ giải thích WHY khi không hiển nhiên.
- Giữ file < ~200 dòng; tách theo concern. YAGNI · KISS · DRY.

## Patterns bắt buộc

- DB **READ-ONLY** — không sinh code/lệnh ghi-xóa DB tự chạy.
- KHÔNG auto `git commit`/`git push`.
- Skill dùng chung project ở `ai-context/skills/` (Copilot đọc qua junction `.github/skills/`); mỗi skill = 1 folder + `SKILL.md` (frontmatter `name` + `description`).
- Thuật ngữ domain: `docs/glossary.md`. Quyết định kiến trúc: `docs/decisions/`.

## Universal Project Structure (Multi-AI)

Áp dụng cho mọi project theo chuẩn thư mục dùng chung đa tool.

- Root chuẩn: `.github/copilot-instructions.md`, `CLAUDE.md`, `GEMINI.md`, `AGENTS.md`, `note.md`.
- Google Antigravity: dùng `AGENTS.md` + `GEMINI.md` + thư mục `.agent/` (`rules/`, `workflows/`).
- AI context chuẩn: `ai-context/project-context.md`, `ai-context/working/sprint-current.md`, `ai-context/working/issues-open.md`, `ai-context/memory/`, `ai-context/prompts/`.
- Docs chuẩn (single source of truth): `docs/INDEX.md`, `business/`, `architecture/`, `data-models/`, `api-contracts/`, `security/`, `deployment/`, `diagrams/`, `decisions/`, `glossary.md`.
- Services chuẩn: nhiều repo độc lập (`repo_1/`, `repo_2/`, `repo_3/`...) và `scripts/` dùng chung.

## Copilot Default Workflow

- Trước khi viết code, luôn đọc theo thứ tự:
	1. `ai-context/project-context.md` (đọc đầu tiên)
	2. `docs/INDEX.md` rồi mở tài liệu liên quan trong `docs/`
	3. `ai-context/working/sprint-current.md`
	4. `ai-context/working/issues-open.md`
- Nếu code mâu thuẫn docs, ưu tiên docs.
- Ghi bài học mới vào `ai-context/memory/`.
- Khi phù hợp, dùng template trong `ai-context/prompts/` như `feature.md`, `review.md`, `debug.md`, `refactor.md`.
