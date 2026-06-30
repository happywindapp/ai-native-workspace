# Security — Hub `{{workspace-root}}`

> Hub không có auth flow/người dùng cuối. "Security" ở đây = **guardrails vận hành AI** + **xử lý secrets**.
> Nguồn sự thật: `~/.claude/CLAUDE.md`, `~/.claude/rules/*`, `~/.claude/.ckignore`. Auth/permission của hệ thống thật → downstream (skill phù hợp với domain của bạn).

## 1. Guardrails cứng (agent BẮT BUỘC tuân thủ)

| Luật | Nội dung | Nguồn |
|------|----------|-------|
| **DB READ-ONLY** | Chỉ `SELECT`/`SHOW`/`EXPLAIN`/metadata. Mọi INSERT/UPDATE/DELETE/DDL → viết SQL cho user TỰ chạy, kể cả khi user ra lệnh trực tiếp | `~/.claude/CLAUDE.md` |
| **Git thủ công** | KHÔNG tự `git commit`/`git push`. Chuẩn bị xong → đưa lệnh/message cho user | `~/.claude/CLAUDE.md` |
| **Editing scope** | Chỉ đụng dòng cần cho task; không drive-by reformat/cleanup | `~/.claude/CLAUDE.md` |
| **Cài tool per-user** | Dưới `C:\_devtools\<tool>\`, không admin; backup PATH trước khi sửa env | `~/.claude/CLAUDE.md` |

## 2. Xử lý secrets (Quy tắc #3 — inventory, KHÔNG copy giá trị)

**File nhạy cảm — chỉ tham chiếu, KHÔNG đưa giá trị vào docs/commit:**

| File | Chứa gì | Trạng thái |
|------|---------|-----------|
| `server-config.md` | Credentials UAT/local: PostgreSQL, MySQL, Redis, MongoDB, message queue, gRPC, UAT endpoints của các service | Tự đánh dấu SENSITIVE + "KHÔNG commit" ở đầu file |
| `~/.claude/settings.json` → `env` | Cùng tập biến (`PG_*`, `MYSQL_*`, `REDIS_*`, `MONGO_*`, `*_URL`...) inject vào shell agent | Local, không commit |

**Quy tắc khi tài liệu hoá:** liệt kê TÊN biến/endpoint, KHÔNG bao giờ chép password/token. Khi cần kết nối → trỏ tới `server-config.md`.

## 3. Context-safety (`.ckignore`)

- Chặn đọc thư mục nặng (`node_modules`, `.git`, `vendor`, `target`, `dist`, `.venv`...) để không bơm rác vào context LLM.
- `.env`/`.env.*` được phép đọc (`!` prefix) — nhưng vẫn áp Quy tắc #3 khi trích xuất.

## 4. N/A cho hub (áp dụng ở downstream, không ở đây)

- Auth flow (OAuth/JWT/session), permission matrix role×resource, rate limiting → thuộc hệ thống thật (các service downstream), không thuộc hub này.
