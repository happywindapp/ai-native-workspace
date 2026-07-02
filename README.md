# ai-native-workspace

Khung thư mục (scaffold) để **nhiều AI coding agent** — Claude Code, Gemini CLI, GitHub Copilot, GPT Codex, **Google Antigravity** —
cùng làm việc trên một hệ **microservices** với **một nguồn tài liệu duy nhất** (`docs/` là single source of truth).

Mỗi AI tự đọc file config của nó khi khởi động (`CLAUDE.md`, `GEMINI.md`, `AGENTS.md`, `.github/copilot-instructions.md`, `.agent/`),
rồi cùng tra cứu chung `docs/` + `ai-context/`. Khai báo service **một chỗ** trong `template.config.json`, chạy 1 script là điền xong toàn bộ.

> **Google Antigravity** tự nạp `AGENTS.md` + `GEMINI.md` (root) và thư mục `.agent/` (`rules/`, `workflows/`). Xem [`.agent/README.md`](.agent/README.md).

## Vì sao

- **Hết lệch ngữ cảnh giữa các AI** — tất cả đọc cùng `docs/` + `ai-context/`, không mỗi tool một kiểu.
- **DRY** — tên/stack/endpoint service khai 1 lần ở manifest; docs khác tham chiếu hoặc sinh tự động.
- **An toàn để public** — secrets tách ra `server-config.md` (gitignored); template chỉ chứa placeholder.

## Cấu trúc

```text
ai-native-workspace/
├── CLAUDE.md · GEMINI.md · AGENTS.md       # [auto-load] config từng AI tool
├── .github/copilot-instructions.md         # [auto-load] Copilot
├── .agent/                                 # [auto-load] Google Antigravity
│   ├── rules/                              # rules always-on (context · guardrails · clean-code)
│   └── workflows/                          # workflows: feature/debug/review/refactor
├── template.config.json                    # ★ NGUỒN KHAI BÁO DUY NHẤT (services/scalars)
├── server-config.example.md                # mẫu env → copy thành server-config.md (gitignored)
├── ai-context/
│   ├── project-context.md                  # đọc đầu tiên: mục tiêu + kiến trúc
│   ├── working/{sprint-current,issues-open}.md
│   ├── memory/                             # bài học AI tự ghi
│   ├── prompts/                            # 6 prompt vận hành + feature/debug/review/refactor
│   └── skills/                             # ★ SKILLS DÙNG CHUNG mọi AI (chuẩn mở SKILL.md)
├── docs/                                   # SINGLE SOURCE OF TRUTH
│   ├── INDEX.md · glossary.md · services-registry.md
│   └── business/ architecture/ data-models/ api-contracts/
│       security/ deployment/ diagrams/ decisions/
├── scripts/
│   ├── apply-template.ps1                  # điền template từ manifest
│   └── link-shared-skills.ps1              # junction skills → mọi AI tool (chạy 1 lần/clone)
└── note.md
```

## Quick start

1. **Điền 1 file** — mở [`template.config.json`](template.config.json):
   - `scalars`: `workspace-root`, `company`, hệ ngoài (`core-system`, `exchange`, `depository`)…
   - `services[]`: mỗi service `{ key, name, tier, stack, repo, baseUrlEnv }` — tên service **chỉ khai ở đây**.
   - `datastores[]`, `messaging[]`.

2. **Chạy 1 lệnh:**

   ```bash
   # PowerShell 7 (cross-platform)
   pwsh scripts/apply-template.ps1
   pwsh scripts/apply-template.ps1 -DryRun        # xem trước, không ghi

   # Windows PowerShell 5.1 (không có pwsh)
   powershell -ExecutionPolicy Bypass -File scripts/apply-template.ps1
   ```

   Script thay token `{{...}}` trong mọi `.md` và **tự sinh** bảng Service Registry + env theo tier.

3. **Link skills dùng chung** (1 lần sau mỗi clone):

   ```bash
   pwsh scripts/link-shared-skills.ps1
   ```

4. **Rà soát & commit.** Có thể xoá `template.config.json` + `scripts/apply-template.ps1` nếu không cần điền lại.

## Cơ chế

| Loại | Ví dụ | Script xử lý? |
|------|-------|---------------|
| Token dự án | `{{workspace-root}}`, `{{service-a}}`, `{{oms-service}}` | ✅ Thay từ manifest |
| Token lúc-dùng | `{{repo}}`, `{{host}}`, `{{tên feature}}` | ❌ Giữ nguyên (điền khi chạy prompt/copy config) |
| Bảng tự sinh | giữa `<!-- BEGIN gen:NAME -->` … `<!-- END -->` | ✅ Sinh lại từ `services`, đừng sửa tay |

Tên service sống **chỉ trong `services[]`**; `tokenMap` nối token inline tới một field service qua `@svc:<key>:<field>`.

## Service tiers

Service phân theo tầng vai trò — map được mọi hệ microservice, không kẹt số lượng:

| Tier | Vai trò |
|------|---------|
| `frontend` | Web/mobile app người dùng cuối |
| `bff` | Backend-for-frontend / API gateway / edge |
| `gateway` | Reverse proxy / ingress (nếu tách khỏi bff) |
| `domain` | Service nghiệp vụ (mỗi bounded context 1 service) |
| `integration` | Adapter hệ ngoài (bank, SOAP, message protocol…) |
| `infra` | CI/CD, observability, shared contracts (proto/OpenAPI) |

## Skills dùng chung (chuẩn mở SKILL.md)

Skill viết **1 lần** ở `ai-context/skills/` — mọi AI tool cùng đọc qua junction (chuẩn mở [Agent Skills](https://developers.openai.com/codex/skills), Anthropic công bố 12/2025, Codex/Copilot/Gemini/Antigravity đều adopt):

| Junction (gitignored) | Tool đọc |
| --- | --- |
| `.agents/skills/` | GPT Codex CLI, Copilot CLI |
| `.claude/skills/` | Claude Code |
| `.github/skills/` | Copilot (VS Code agent mode) |
| `.agent/skills/` | Google Antigravity |
| `.gemini/skills/` | Gemini CLI |

- Junction KHÔNG commit (gitignore) → clone mới chạy lại `scripts/link-shared-skills.ps1`. Chỉ `ai-context/skills/` vào git.
- Skill phải "thuần chuẩn mở" (frontmatter chỉ `name` + `description`) để chạy mọi tool. Chi tiết: [`ai-context/skills/README.md`](ai-context/skills/README.md).
- Caveat: tính năng server-side (Copilot cloud agent trên github.com) không thấy junction — cần thì commit bản thật riêng vào `.github/skills/`.

## Workflow đa-AI (prompts)

`ai-context/prompts/` chứa các prompt copy-paste theo vòng đời:

| # | Prompt | Khi nào |
|---|--------|---------|
| 1 | `init-ai-native-workspace` | Dựng khung (1 lần) |
| 2 | `autofill-workspace-from-codebase` | Điền docs từ code (bootstrap) |
| 3 | `system-documentation` | Sinh tài liệu hệ thống đa repo |
| 4 | `sync-docs-from-context` | Re-sync docs khi drift (event-driven) |
| 5 | `remember-standard-structure-to-memory` | Lưu cấu trúc vào memory mỗi AI (1 lần/AI) |
| 6 | `onboard-read-context` | Lưu protocol đọc-context (1 lần/AI → tự áp dụng mỗi phiên) |

### Sau khi apply template vào 1 project — prompt nào chạy lại?

**KHÔNG chạy lại cả 6 mỗi project.** Run-matrix:

| Prompt | Chạy lại trong project? | Vì sao |
|--------|-------------------------|--------|
| 1 `init` | ❌ Không | Khung đã dựng khi copy scaffold + `apply-template.ps1`. Chạy lại chỉ tạo trùng. |
| 2 `autofill` | 🟡 Tùy chọn | Phần lõi thường đã điền tay (project-context, glossary, architecture, api-contracts, data-models, services-registry từ code/README thật). Còn thiếu thì điền nốt `docs/business/`, `docs/security/`, `docs/deployment/` + merge `FE/README.md` — làm trực tiếp được, khỏi "gọi prompt". |
| 3 `system-documentation` | 🟡 Độc lập | Chỉ chạy khi muốn **1 file tổng hợp** `SYSTEM_DOCUMENTATION.md` (TOC + build/run + E2E + C4/sequence). Khác với `docs/` phân mảnh. |
| 4 `sync-docs` | ❌ Chưa cần | Chạy **định kỳ** khi code drift khỏi docs. Docs vừa tạo chưa lệch → để dành. |
| 5 `remember-structure` | ❌ Không (per-project) | Setup **1-lần / mỗi AI tool ở mức GLOBAL**, không per-project. Claude Code đã có sẵn trong `~/.claude/CLAUDE.md` (mục *"Standard Multi-AI Project Structure"*). Chỉ chạy khi onboard tool mới (Gemini/Copilot/Codex). |
| 6 `onboard-read-context` | ❌ Không (per-project) | Cũng **1-lần ở mức GLOBAL**. Protocol "đọc `project-context.md` trước khi code" đã ở global config → AI tự áp dụng mỗi session. |

**Tóm lại:** sau khi apply template, mặc định **bỏ qua 1/4/5/6**; chỉ cân nhắc hoàn tất phần thiếu của **2** (làm tay) và optional **3**. Prompt **5 & 6 là 1-lần-cho-mỗi-tool, KHÔNG per-project** — với Claude đã nằm sẵn trong `~/.claude/`.

## Quy ước & an toàn

- Tài liệu sống ở `docs/`, **không nhân bản** trong từng repo.
- `server-config.md` (giá trị thật) **không bao giờ commit** — đã có trong [`.gitignore`](.gitignore). Dùng `server-config.example.md` làm mẫu.
- Vendored skill/plugin repos đã gitignore — clone riêng hoặc dùng git submodule.
- DB read-only cho agent; agent không tự `git commit`/`push` (xem guardrails trong `CLAUDE.md`).
