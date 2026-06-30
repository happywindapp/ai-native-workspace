# Deployment — Hub `{{workspace-root}}`

> Hub không deploy lên server. "Deployment" ở đây = **bảo trì bộ công cụ AI** + **cấu hình môi trường agent**.
> Deploy hệ thống thật (các service, CI/CD, env dev/uat/prod) → downstream, không thuộc hub.

## 1. Cập nhật AI CLI

`scripts/update-ai-clis.ps1` — cập nhật 4 CLI cùng lúc:

| CLI | Cách update | Nguồn |
|-----|-------------|-------|
| `claude` (Claude Code) | native installer: `claude update` | tự update |
| `codex` | `npm install -g @openai/codex@latest` | npm global |
| `gemini` | `npm install -g @google/gemini-cli@latest` | npm global |
| `copilot` | `npm install -g @github/copilot@latest` | npm global |

```powershell
pwsh ./scripts/update-ai-clis.ps1            # update thật
pwsh ./scripts/update-ai-clis.ps1 -DryRun    # xem trước, không cài
```

CLI chưa cài → `skipped`; npm lỗi → `error` (exit 1). Cuối chạy in bảng Before/After/Status.

## 2. Sinh skills catalog

- `python ~/.claude/scripts/generate_catalogs.py --skills` — sinh catalog để agent kích hoạt skill theo `description`.
- Script lỗi → sửa & chạy lại tới khi pass (luật vận hành).

## 3. Biến môi trường agent (KHÔNG commit secrets — Quy tắc #3)

Inject qua `~/.claude/settings.json` → `env`. Danh sách biến gom **theo tier** sinh từ
`template.config.json` — xem [server-config.example.md](../../server-config.example.md) (khối `gen:env`)
và [Service Registry](../services-registry.md). Giá trị thật sống trong `server-config.md` (gitignored).

## 4. Cài dev tool

- Mọi tool dưới `C:\_devtools\<tool>\`, per-user (không admin), `PrependPath=0`, tự quản PATH ở User scope.
- Backup PATH ra file timestamp trong thư mục tool trước khi sửa env.
