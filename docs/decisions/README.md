# Decisions (ADR) — Hub `{{workspace-root}}`

Architecture Decision Records — quyết định kiến trúc đã chốt. AI đọc để hiểu **TẠI SAO** hub được tổ chức như vậy.

## Quy ước

- Mỗi quyết định = 1 file `NNNN-tieu-de-kebab.md` (số tăng dần).
- Cấu trúc: **Context → Decision → Consequences** (+ Status: proposed/accepted/superseded).

## Quyết định đã chốt (rút từ `~/.claude/CLAUDE.md` + `rules/*` — chưa tách file riêng)

| # | Quyết định | Lý do (Context) | Hệ quả |
|---|-----------|-----------------|--------|
| 1 | **Hub = aggregator, KHÔNG phải microservices runtime** | Tách "buồng lái AI" khỏi code nghiệp vụ; code thật sống ở root riêng (`path/to/your-repo-a`, `path/to/your-repo-b`) | docs hub mô tả workflow/skill/config, không mô tả service runtime |
| 2 | **Skill mới tạo ở global `~/.claude/skills/`, không project-level** | Global tự auto-discover ở mọi project; `description` chống mis-activate | Không dùng `<project>/.claude/skills/` kể cả skill domain-specific |
| 3 | **Memory file-based tại `~/.claude/projects/<workspace>/memory/`** | 1 fact/file + `MEMORY.md` index nạp mỗi session; bền vững xuyên session | Tự lưu khi có info durable, không hỏi trước; loại `user/feedback/project/reference` |
| 4 | **DB READ-ONLY tuyệt đối cho agent** | Tránh mọi rủi ro mất/hỏng data; user là người duy nhất chạy lệnh ghi | Lệnh ghi/xóa → viết SQL cho user tự chạy, kể cả khi user ra lệnh trực tiếp |
| 5 | **Agent không tự `git commit`/`push`** | User giữ quyền kiểm soát cuối + gắn ticket ID riêng | Chuẩn bị xong → đưa lệnh/message; approval không bền qua turn |
| 6 | **Cài dev tool dưới `C:\_devtools\<tool>\`, per-user** | User không có admin; gom tooling 1 chỗ, portable, isolate khỏi Program Files | `InstallAllUsers=0`, `PrependPath=0`, tự quản PATH User scope |
| 7 | **Comment tối thiểu (mặc định không, khi cần 1 dòng WHY)** | Comment dài bloat file + rot nhanh hơn code | Không doc-paragraph, không restate WHAT; chỉ giải thích WHY non-obvious |

> Các mục trên có thể tách thành file `NNNN-*.md` đầy đủ khi cần thảo luận lại. Hiện chỉ index để AI nắm WHY nhanh.
