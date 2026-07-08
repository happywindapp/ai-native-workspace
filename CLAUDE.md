# CLAUDE.md — AI-Native Knowledge Hub (`{{workspace-root}}`)

> [AUTO-LOAD] Claude Code đọc khi khởi động từ `{{workspace-root}}`.
> Luật chi tiết sống ở `~/.claude/CLAUDE.md` + `~/.claude/rules/*` (đã auto-load). File này tóm tắt + trỏ đường.

## Workspace là gì

`{{workspace-root}}` là **knowledge hub điều phối AI** cho việc phát triển các hệ thống của bạn,
KHÔNG phải runtime microservices. Đọc `ai-context/project-context.md` để nắm toàn cảnh.

## Tech stack (của hub)

- **Skills:** Markdown + YAML frontmatter (`SKILL.md`).
- **Manifest/config:** JSON (`plugin.json`, `settings.json`).
- **Scripts:** Python (catalog: `~/.claude/scripts/generate_catalogs.py`), PowerShell + Bash.
- **Downstream targets** (project thật, root riêng): tech stack tuỳ hệ thống của bạn (ví dụ Go, Node, React/TS, .NET/SOAP gateway).

## Naming convention

- File: **kebab-case** + tên mô tả rõ mục đích (để LLM hiểu ngay qua Grep/Glob).
- Code theo ngôn ngữ: Go/Rust → snake_case; C#/Java → PascalCase; JS/TS/Python/shell → kebab-case.

## Clean code rules (BẮT BUỘC)

- **YAGNI · KISS · DRY.** Diff hẹp, chỉ đụng dòng cần cho task. Không drive-by reformat.
- **Comment tối thiểu:** mặc định KHÔNG comment; khi cần chỉ 1 dòng giải thích WHY.
- Giữ file < ~200 dòng; tách module theo concern.
- Check module/skill đã có trước khi tạo mới.

## Guardrails vận hành (từ `~/.claude/`)

- **DB READ-ONLY:** chỉ `SELECT`/metadata. Mọi INSERT/UPDATE/DELETE/DDL → viết SQL, để user tự chạy.
- **Git:** KHÔNG tự `git commit`/`git push`. Chuẩn bị xong → đưa lệnh/message cho user.
- **PR description:** dùng chuẩn trong `ai-context/prompts/pr-description.md` (khác nhau cho fix bug vs feature mới).
- **Skill:** skill cá nhân xuyên-project → `~/.claude/skills/`; skill dùng chung đa-AI của project → `ai-context/skills/` (chuẩn mở SKILL.md, junction `.claude/skills/` tạo bởi `scripts/link-shared-skills.ps1`).
- **Cài tool:** dưới `C:\_devtools\<tool>\`, per-user (không admin).
- **Orchestration:** khi làm trên downstream target, dùng work-context path của project đó.

## Bản đồ điều hướng

1. `ai-context/project-context.md` — toàn cảnh hub (đọc đầu tiên).
2. `docs/INDEX.md` — mục lục tài liệu.
3. `ai-context/working/sprint-current.md` — việc đang làm.
4. `ai-context/memory/` + `MEMORY.md` index — bài học đã rút ra.
5. `.agent/` — rules + workflows cho Google Antigravity (đọc `AGENTS.md`+`GEMINI.md`+`.agent/`).
6. `ai-context/skills/` — skills dùng chung mọi AI tool (junction tạo bởi `scripts/link-shared-skills.ps1`).
