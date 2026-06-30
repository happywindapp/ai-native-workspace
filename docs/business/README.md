# Business — Workflow điều phối AI

> "Nghiệp vụ" của hub này = các **workflow vận hành AI agent** (rút từ `~/.claude/rules/*`).
> Nghiệp vụ thật (ví dụ đặt lệnh, reconciliation, settlement) nằm ở downstream targets — xem
> skill phù hợp với domain của bạn.

## 1. Primary workflow (vòng đời 1 task code)

```
planner → researcher (parallel) → implement → code-simplifier → tester → code-reviewer → docs-manager
```

- **planner**: tạo plan + TODO trong `plans/` trước khi code.
- **researcher** (nhiều agent song song): nghiên cứu chủ đề kỹ thuật, báo cáo về planner.
- **implement**: sửa file thật (không mock), compile sau mỗi thay đổi.
- **tester**: chạy test trên code đã simplify; KHÔNG bỏ qua test fail.
- **code-reviewer**: review code sạch đã test.
- **docs-manager**: cập nhật `docs/` nếu có thay đổi.

## 2. Skill activation

- Catalog sinh bởi `~/.claude/scripts/generate_catalogs.py --skills`.
- Agent đọc catalog → kích hoạt skill liên quan theo `description`. Script lỗi → sửa & chạy lại tới khi pass.

## 3. Memory protocol

- File-based tại `~/.claude/projects/<workspace>/memory/`, index ở `MEMORY.md` (1 dòng/memory).
- Loại: `user` / `feedback` / `project` / `reference`. Tự lưu khi có thông tin durable, không hỏi trước.

## 4. Orchestration protocol

- Khi spawn subagent: luôn truyền **work-context path** (git root của project đang sửa), reports path, plans path.
- Nếu CWD khác project đang sửa → dùng path của project, không phải CWD.
- Sequential chaining (có phụ thuộc) vs Parallel (độc lập, không tranh chấp file).

## 5. Guardrails (luật cứng)

| Luật | Nội dung |
|------|----------|
| DB READ-ONLY | Chỉ `SELECT`/metadata; lệnh ghi-xóa → viết SQL cho user tự chạy |
| Git thủ công | Không tự `commit`/`push`; chuẩn bị xong đưa lệnh cho user |
| Editing scope | Chỉ đụng dòng cần; không drive-by cleanup |
| Comments | Mặc định không; khi cần chỉ 1 dòng WHY |
| Docs review | `.docx/.xlsx/.pdf/.pptx` phải xem cả ảnh/diagram, không tin text-only |
