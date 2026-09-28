# Multi-Agent Software Engineering System — Master Plan (v2)

> Bản v1 được giữ tại `multi-agent-software-engineering-master-plan.v1.md`.

## 0. Thay đổi so với v1

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | Mọi command chạy trong **Docker sandbox** (network off, chỉ mount worktree). Sau mỗi command có **post-exec diff check**. | `python`, `pytest`, `npm test` chạy code tùy ý → File Policy ở tầng tool bị bypass qua shell. |
| 2 | Thêm **read policy** (`read_denied`). | Coder đọc `.env` = secret bị gửi lên LLM provider. |
| 3 | Tuần 1 bắt đầu bằng **spike OpenHands** có tiêu chí go/no-go. Chỉ Coder chạy trên OpenHands. | Chưa xác minh Policy Engine hook được vào tool loop của OpenHands — rủi ro kiến trúc lớn nhất. |
| 4 | `user_ceiling`, `denied`, `max_rework`, budget **không nằm trong output của Architect**. Runtime gắn từ config. Test command do Architect đề xuất phải qua Command Policy. | LLM không được tự đặt giới hạn cho chính nó. |
| 5 | Thêm state `AWAITING_APPROVAL` (duyệt plan, duyệt command, duyệt mở rộng scope). | `plan.md` "dành cho con người" nhưng v1 không có bước user duyệt. |
| 6 | State machine viết lại dạng **bảng transition đầy đủ** (check fail, gate fail, replan, multi-step, budget). | v1 thiếu nhiều nhánh. |
| 7 | `plan_allowed` = path cụ thể (+ `op`); chỉ `user_ceiling` dùng glob. | Kiểm tra "glob ⊆ glob" khó làm đúng; ví dụ v1 mâu thuẫn với validator. |
| 8 | Thêm **Scope Expansion Request**. | Coder thường cần file ngoài plan (`__init__.py`, `conftest.py`). |
| 9 | Gate pseudo-code khớp đầy đủ checklist + kiểm tra nhất quán verdict. | v1 thiếu acceptance, high severity, out-of-scope. |
| 10 | Supervisor ở V1 là **code deterministic** + một LLM classifier nhỏ. | Workflow đã cố định bằng state machine. |
| 11 | **Git worktree + checkpoint commit mỗi step** đưa vào V1. | Rollback, cách ly working tree của user, resume sau crash. |
| 12 | **Baseline checks** trước khi sửa; Gate đánh giá theo delta. | Repo cũ có sẵn lỗi lint/test → loop fail mãi. |
| 13 | **Phát hiện test bị làm yếu** trong diff validation. | Coder tự viết test cho code của mình → nguy cơ False PASS. |
| 14 | **Budget** mỗi task: token, cost, wall-clock, tool calls. | `MAX_REWORK` không đủ để chặn chi phí. |
| 15 | Dùng **LiteLLM** thay vì tự viết 5 provider. | OpenHands đã dùng LiteLLM. |
| 16 | Golden Tasks, event log JSONL, SQLite persistence đưa lên **sớm**. | Cần đo và debug từ đầu. |
| 17 | Thêm test case policy **đặc thù Windows**. | Môi trường chạy là Windows. |
| 18 | Thêm mục **prompt injection** và **Risks / Open questions**. | |

---

## 1. Mục tiêu sản phẩm

Xây một hệ thống Multi-Agent Software Engineering chạy local, nhận yêu cầu từ người dùng và tự điều phối:

```text
USER
 │
 ▼
WORKFLOW ENGINE (deterministic Supervisor)
 │
 ▼
ARCHITECT ──► plan.md + plan.json
 │
 ▼
PLAN VALIDATOR
 │
 ▼
USER APPROVAL  ◄── bắt buộc ở V1
 │
 ▼
PREPARE (worktree + baseline)
 │
 ▼
RESEARCHER ──────── optional
 │
 ▼
┌─► CODER (step N) ──► AUTO CHECK ──► checkpoint commit ──┐
│                          │ fail                          │ còn step
│                          └──► CODER (retry)              │
└──────────────────────────────────────────────────────────┘
 │ hết step
 ▼
REVIEWER ── FAIL ──► CODER (rework) ──► AUTO CHECK ──► REVIEWER
 │ PASS
 ▼
GATE (deterministic → LLM)
 │
 ▼
FINAL REPORT (user tự quyết merge task branch)
```

Triết lý cốt lõi:

> LLM chỉ đề xuất hành động. Runtime mới là thứ quyết định hành động có được phép chạy hay không.

```text
LLM
 ↓
Action proposal
 ↓
Schema validation
 ↓
Policy validation (file / command / budget)
 ↓
Sandboxed execution
 ↓
Post-exec verification (diff check)
```

Hệ quả: **không một giá trị nào do LLM sinh ra được dùng làm giới hạn an toàn** (scope ceiling, denied list, retry limit, budget, command allowlist).

---

## 2. Kiến trúc nền

### Base runtime

- **OpenHands Software Agent SDK**: agent loop, file editor, terminal cho **Coder**.
- **OpenHands Docker workspace**: sandbox thực thi.
- **LiteLLM** (thông qua OpenHands và dùng trực tiếp cho các agent còn lại): lớp gọi model.
- Tham khảo: **ChatDev 2.0** (workflow/routing), **mini-SWE-agent** (agent loop đơn giản), **MetaGPT** (Role + SOP + handoff), **aider** (repo map).

### Agent nào chạy trên đâu

| Agent | Cách chạy | Lý do |
|---|---|---|
| Architect | LLM call → structured output | Không cần tool loop; chỉ đọc context do runtime chuẩn bị. |
| Researcher | LLM call + tool search (read-only) | Không ghi file. |
| Coder | **OpenHands agent loop** trong sandbox | Cần đọc/sửa/chạy lặp lại. |
| Reviewer | LLM call → structured output | Read-only, input do runtime chuẩn bị. |
| Gate (phần semantic) | LLM call → structured output | |
| Classifier | LLM call nhỏ (có thể dùng model local) | |

### Không fork OpenHands

```text
┌────────────────────────────────────────────┐
│                YOUR PROJECT                │
│  Workflow Engine · State · Policy          │
│  Architect · Researcher · Reviewer · Gate  │
│  Context Builder · Budget · UI             │
└───────────────────┬────────────────────────┘
                    │  (policy hook — xem §2.1)
                    ▼
┌────────────────────────────────────────────┐
│        OpenHands SDK / Agent Server        │
│  Coder agent loop · Tools · Events         │
└───────────────────┬────────────────────────┘
                    ▼
┌────────────────────────────────────────────┐
│   Docker sandbox (network off mặc định)    │
│   mount: <worktree> only                   │
└────────────────────────────────────────────┘
```

### 2.1 Spike tuần 1 — tiêu chí go/no-go

Trước khi viết bất kỳ agent nào, phải chứng minh được:

| # | Tiêu chí | Cách kiểm |
|---|---|---|
| G1 | Mọi file write/edit của Coder đi qua Policy Engine của ta **trước** khi thực thi. | Cấu hình cho agent ghi `../outside.txt` → bị chặn, có event DENY. |
| G2 | Mọi command của Coder đi qua Command Policy trước khi thực thi. | Agent chạy lệnh DENY → bị chặn. |
| G3 | Command chạy trong Docker, chỉ thấy worktree, không có network. | `curl`, `ls /host`, đọc file ngoài mount → thất bại. |
| G4 | Chạy được trên máy Windows của team (Docker Desktop + WSL2). | Chạy end-to-end một task đơn giản. |
| G5 | Lấy được event stream (tool call, token usage) để ghi log. | |

Điểm hook cần xác minh (theo thứ tự ưu tiên): custom tool thay thế tool mặc định → security analyzer / confirmation policy của SDK → wrapper ở tầng Agent Server.

**Nếu G1/G2 không đạt:** fallback sang agent loop tự viết theo kiểu mini-SWE-agent (một loop đơn giản, tool do ta định nghĩa), vẫn dùng Docker để sandbox. Quyết định phải được chốt cuối tuần 1.

---

## 3. Scope V1

### Có trong V1

- Nhận task từ user (CLI trước, UI sau).
- Đọc source code, repo map.
- Architect tạo `plan.md` + `plan.json`.
- User duyệt plan.
- Research khi cần.
- Sửa / tạo file trong scope.
- Chạy command trong sandbox.
- Baseline + test + lint + type check.
- Review code, rework có giới hạn.
- Gate.
- File policy (read + write), command policy.
- Scope expansion request có duyệt.
- Git worktree + checkpoint commit trên task branch.
- Budget per task.
- State persistence, resume task.
- Logs / trace, token usage.
- UI.
- Multi-model qua LiteLLM.

### Chưa làm ở V1

- Auto merge vào branch của user, push, PR.
- Deploy production.
- Cloud multi-user, multi-tenant, Kubernetes.
- Distributed / hàng chục agent song song.
- Nhiều task chạy đồng thời trên **cùng một** project.
- Agent tự thay đổi permission, policy, budget.
- Agent tự sửa plan không qua duyệt.
- Supervisor dạng LLM tự do.
- Training model, Vector DB.

---

## 4. Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.12 |
| Agent runtime | OpenHands Software Agent SDK (Coder) |
| Sandbox | Docker (Docker Desktop + WSL2 trên Windows) |
| LLM layer | LiteLLM |
| Backend | FastAPI |
| Schemas | Pydantic (+ xuất JSON Schema) |
| Database | SQLite (WAL mode) |
| ORM | SQLModel |
| Frontend | Next.js + TypeScript (có thể thay bằng Vite + React nếu muốn nhẹ hơn) |
| UI | Tailwind + shadcn/ui |
| Realtime | SSE (đủ cho stream một chiều) hoặc WebSocket |
| Testing | pytest |
| Lint / Type | Ruff / Pyright |
| Git | Git CLI (worktree) |
| Async | asyncio |
| Config | YAML + `.env` |
| Logs | Structured JSON logs (JSONL) |
| Repo map | tree-sitter hoặc universal-ctags + ripgrep |

---

## 5. Kiến trúc tổng thể

```text
                         USER
                          │
                ┌─────────┴─────────┐
                │  CLI  /  Next.js  │
                └─────────┬─────────┘
                       REST / SSE
                          ▼
              ┌────────────────────┐
              │  ORCHESTRATOR API  │  FastAPI
              └─────────┬──────────┘
            ┌───────────┼────────────────┐
            ▼           ▼                ▼
     Workflow Engine  State Manager   Budget Manager
     (state machine)     │
            │          SQLite
            │
   ┌────────┼──────────┬──────────┬────────┐
   ▼        ▼          ▼          ▼        ▼
Classifier Architect Researcher Reviewer  Gate
   │        │          │          │        │
   └────────┴────┬─────┴──────────┴────────┘
                 │  (LLM calls qua LiteLLM, context từ Context Builder)
                 ▼
             Coder (OpenHands)
                 │
          Policy Engine  ── file (read/write) · command · scope
                 │
          Docker sandbox ── worktree của task
                 │
          Post-exec diff check
                 │
          Git (runtime-owned): worktree, checkpoint commit
```

---

## 6. Agent Contract

Mỗi agent phải có:

```text
Agent
├── role
├── objective
├── input schema        (Pydantic)
├── output schema       (Pydantic, validate trước khi dùng)
├── tools
├── permissions         (lấy từ policy config, không phải từ prompt)
├── context policy      (Context Builder chuẩn bị gì)
├── retry policy        (retry khi output sai schema: tối đa 2)
├── budget share        (phần budget tối đa của task)
└── stop condition
```

---

## 7. Supervisor = Workflow Engine (deterministic)

Ở V1, Supervisor **là code**, không phải LLM.

```text
User request
     ↓
Classifier (LLM nhỏ) → { task_type, needs_research, risk }
     ↓
Load state
     ↓
Transition table (§20) quyết định bước tiếp theo
     ↓
Dispatch agent
     ↓
Validate output (schema + policy)
     ↓
Persist state + event
```

Classifier output:

```json
{
  "task_type": "feature | bugfix | refactor | test | simple_edit",
  "needs_research": false,
  "research_questions": [],
  "risk": "low | medium | high"
}
```

Classifier chỉ ảnh hưởng tới các nhánh *được phép* (có research hay không). Nó không bỏ qua được validation, approval hay gate.

---

## 8. Architect Agent

Input (do Context Builder chuẩn bị):

```text
user_request
project_summary        (từ memory/architecture.md)
repo_map               (symbols, không phải toàn bộ code)
relevant_files         (nội dung, đã lọc qua read policy)
constraints
policy_summary         (user_ceiling + denied — để Architect biết giới hạn, KHÔNG để Architect ghi lại)
baseline_summary       (nếu đã có)
```

Output bắt buộc:

```text
plan.md    — cho người đọc / duyệt
plan.json  — contract cho runtime
```

`plan.md` được **render từ `plan.json`** (template), không phải do LLM viết riêng. Như vậy hai file không bao giờ lệch nhau.

---

## 9. `plan.md`

Render từ `plan.json`. Ví dụ:

```markdown
# TASK-001 — Implement JWT Authentication

## Goal
Add authentication to FastAPI.

## Current architecture
...

## Steps
1. [S1] Add auth service — app/auth.py (create)
2. [S2] Add login endpoint — app/routes/auth.py (create), app/main.py (modify)
3. [S3] Add tests — tests/test_auth.py (create)

## Files
| Path | Op |
|---|---|
| app/auth.py | create |
| app/routes/auth.py | create |
| app/main.py | modify |
| tests/test_auth.py | create |

## Acceptance Criteria
- AC1 Login returns token — verify: test `tests/test_auth.py::test_login_ok`
- AC2 Invalid password returns 401 — verify: test `tests/test_auth.py::test_login_bad_password`
- AC3 Protected endpoint validates token (incl. expiry) — verify: test + review

## Commands proposed
- pytest tests/test_auth.py
```

---

## 10. `plan.json`

Chỉ chứa những gì Architect **được phép đề xuất**. Giới hạn an toàn nằm ở `policy snapshot` do runtime gắn vào.

```json
{
  "schema_version": "2.0",
  "task_id": "TASK-001",
  "goal": "Implement JWT authentication",
  "files": [
    { "path": "app/auth.py",        "op": "create" },
    { "path": "app/routes/auth.py", "op": "create" },
    { "path": "app/main.py",        "op": "modify" },
    { "path": "tests/test_auth.py", "op": "create" }
  ],
  "steps": [
    {
      "id": "S1",
      "title": "Add auth service",
      "depends_on": [],
      "files": ["app/auth.py"],
      "acceptance": ["AC3"]
    },
    {
      "id": "S2",
      "title": "Add login endpoint",
      "depends_on": ["S1"],
      "files": ["app/routes/auth.py", "app/main.py"],
      "acceptance": ["AC1", "AC2"]
    },
    {
      "id": "S3",
      "title": "Add tests",
      "depends_on": ["S2"],
      "files": ["tests/test_auth.py"],
      "acceptance": ["AC1", "AC2", "AC3"]
    }
  ],
  "acceptance_criteria": [
    { "id": "AC1", "text": "Login returns token",
      "verify": { "type": "test", "ref": "tests/test_auth.py::test_login_ok" } },
    { "id": "AC2", "text": "Invalid password returns 401",
      "verify": { "type": "test", "ref": "tests/test_auth.py::test_login_bad_password" } },
    { "id": "AC3", "text": "Protected endpoint validates token incl. exp",
      "verify": { "type": "test+review", "ref": "tests/test_auth.py::test_expired_token" } }
  ],
  "proposed_commands": [
    "pytest tests/test_auth.py"
  ]
}
```

Policy snapshot runtime gắn vào run (lưu trong DB, **không** lấy từ output của LLM):

```json
{
  "policy_version": "2026-09-28.1",
  "user_ceiling": ["app/**", "tests/**"],
  "write_denied": [".env", ".env.*", ".git/**", "secrets/**", "config/policies.yaml"],
  "read_denied":  [".env", ".env.*", "secrets/**", "**/*.pem", "**/*.key"],
  "limits": {
    "max_replan": 2,
    "max_check_retry_per_step": 3,
    "max_rework": 3,
    "max_scope_requests": 3
  },
  "budget": {
    "max_tokens": 2000000,
    "max_cost_usd": 5.0,
    "max_wall_clock_min": 60,
    "max_tool_calls": 300
  }
}
```

Nếu `plan.json` có các field `user_ceiling`, `denied`, `max_rework`, `budget`… thì Validator **từ chối** (`PLAN_SCHEMA_INVALID`), vì schema không có các field đó (`extra = "forbid"`).

---

## 11. Plan Validator

```text
Architect → plan.json
    ↓
Pydantic / JSON Schema (extra=forbid)
    ↓
Structural validation
    ↓
Scope validation
    ↓
Command validation
    ↓
PASS → render plan.md → AWAITING_PLAN_APPROVAL
```

Rules:

```text
schema valid
AND steps != empty
AND files != empty
AND step ids unique, depends_on không có vòng
AND mỗi step.files ⊆ plan.files
AND mỗi plan.files là path cụ thể (không glob), đã normalize, nằm trong workspace
AND mỗi plan.files match user_ceiling
AND không file nào match write_denied
AND op=create → file chưa tồn tại; op=modify/delete → file đã tồn tại
AND mỗi acceptance id được tham chiếu tồn tại; mỗi AC được ít nhất một step đảm nhận
AND mỗi proposed_command qua Command Policy với mức ≤ SANDBOXED
```

Error types:

```text
PLAN_SCHEMA_INVALID
PLAN_INCOMPLETE
PLAN_SCOPE_CONFLICT
PLAN_COMMAND_REJECTED
```

Khi fail: gửi lỗi có cấu trúc lại cho Architect, tối đa `max_replan` lần → sau đó `FAILED` và báo user.

---

## 12. File Permission Model

Ba lớp, **nguồn khác nhau**:

| Lớp | Nguồn | Dạng |
|---|---|---|
| `user_ceiling` | User / project config | glob |
| `write_denied`, `read_denied` | Project config + default hệ thống | glob |
| `plan_allowed` | `plan.json` (đã validate + user duyệt) | path cụ thể + op |

**Write** được phép khi:

```text
path ∈ plan_allowed   (khớp chính xác sau normalize)
AND op khớp (create / modify / delete)
AND match(user_ceiling)
AND NOT match(write_denied)
```

**Read** được phép khi:

```text
path nằm trong workspace
AND NOT match(read_denied)
```

Ví dụ:

```text
user_ceiling:  app/**, tests/**
plan_allowed:  app/auth.py (create), app/main.py (modify)

write app/auth.py        → ALLOW
write app/utils.py       → DENY  (không trong plan_allowed → có thể gửi Scope Request)
write docs/readme.md     → DENY  (ngoài user_ceiling → Scope Request cũng bị từ chối)
read  .env               → DENY  (read_denied)
```

### 12.1 Scope Expansion Request

Coder không tự sửa plan. Khi cần file ngoài `plan_allowed`, Coder gọi tool:

```json
{
  "tool": "request_scope",
  "path": "app/__init__.py",
  "op": "create",
  "reason": "Package needs __init__ to import app.auth"
}
```

Runtime xử lý:

```text
path match write_denied      → DENY ngay
path ngoài user_ceiling      → DENY ngay
số request > max_scope_requests → BLOCKED
còn lại                      → AWAITING_APPROVAL (user duyệt)
   approved → plan_version += 1, thêm vào plan_allowed, quay lại CODING
   rejected → trả lỗi cho Coder, quay lại CODING
```

Tuỳ chọn: project config có thể khai báo `auto_approve_scope` cho một số pattern rủi ro thấp (vd `**/__init__.py` trong ceiling). Mặc định tắt.

---

## 13. File Policy Engine

```text
Coder: write_file / read_file(path)
 ↓
Reject nếu path chứa NUL, ADS (":" sau drive letter), device name (CON, NUL, COM1…)
 ↓
Normalize (case-fold trên Windows, "/" thống nhất, bỏ "." và "..")
 ↓
Resolve symlink / junction / reparse point (realpath)
 ↓
Check workspace boundary (sau resolve)
 ↓
Check denied (read_denied / write_denied)
 ↓
Check plan_allowed + op        (write)
 ↓
Check user_ceiling             (write)
 ↓
ALLOW / DENY  → event
```

Phải chống:

- `../outside.txt`
- Absolute path ngoài workspace, `C:\Windows\...`
- UNC `\\server\share`, `\\?\C:\...`, `\\.\`
- Symlink / junction trỏ ra ngoài
- Case: `.ENV`, `.Env`, `SECRETS/x`
- NTFS ADS: `.env::$DATA`, `app.py:hidden`
- 8.3 short name: `SECRET~1`
- Reserved device names: `CON`, `NUL`, `aux.py`
- `.git/config`, `.git/hooks/*`

**File policy chỉ là lớp thứ hai.** Lớp thứ nhất là sandbox (§29.1). Lớp thứ ba là post-exec diff check (§16).

---

## 14. Coder Agent

Input:

```text
task goal
plan (bản đã duyệt, plan_version hiện tại)
current_step
relevant_files       (Context Builder chọn: step.files + import/caller liên quan từ repo map)
research_result      (đánh dấu UNTRUSTED)
check_feedback       (nếu retry sau check fail)
review_feedback      (nếu rework)
```

Tools:

- `read_file`, `search_code` (qua read policy)
- `write_file`, `create_file`, `apply_patch` (qua write policy)
- `run_command` (qua Command Policy, chạy trong sandbox)
- `git_diff` (read-only)
- `request_scope`
- `report_step_done`

Không được phép:

- Sửa `plan.json`, policy config, `user_ceiling`.
- Bất kỳ lệnh git thay đổi trạng thái (`commit`, `checkout`, `reset`, `stash`, `branch`…). **Git do runtime quản lý.**
- Network (trừ command được duyệt riêng).

---

## 15. Researcher Agent

Chỉ chạy khi Classifier trả `needs_research = true`, hoặc Architect/Coder yêu cầu và runtime cho phép.

Output:

```json
{
  "question": "...",
  "findings": [],
  "sources": [],
  "recommendation": "",
  "confidence": "low | medium | high"
}
```

- Researcher không viết source code, không có write tool.
- Output của Researcher là **dữ liệu không tin cậy** (xem §25.1).

---

## 16. Auto Checks

Chạy sau mỗi step (Coder báo `report_step_done`) và sau mỗi lần rework.

```text
Coder done
 ↓
Diff validation (runtime, từ git trong worktree)
   ├─ mọi file thay đổi ∈ plan_allowed?        (không → revert file đó + FAIL)
   ├─ không file nào match write_denied?
   ├─ không file binary / file lớn bất thường?
   └─ test weakening detection
 ↓
Syntax check
 ↓
Lint          (chỉ trên file thay đổi, so với baseline)
 ↓
Type check    (so với baseline)
 ↓
Tests         (targeted trước: test của step / file liên quan → full suite ở cuối)
 ↓
Security rules (secret pattern trong diff, eval/exec, SQL string concat…)
 ↓
PASS → checkpoint commit trên task branch
FAIL → CODING (retry) kèm feedback có cấu trúc
```

### 16.1 Baseline

Ở bước `PREPARING`, trước khi Coder chạm vào code:

```text
ruff check <ceiling paths>   → baseline_lint
pyright                      → baseline_types
pytest                       → baseline_tests
```

Tiêu chí PASS là **không có lỗi mới**, không phải "0 lỗi":

```text
new_lint_errors_on_changed_files == 0
new_type_errors == 0
tests pass trước đây vẫn pass
test mới của plan pass
```

### 16.2 Test weakening detection

Diff check phát hiện và đánh dấu `severity: high` cho Reviewer / Gate:

- Xóa test function hoặc test file có sẵn.
- Thêm `skip`, `xfail`, `pytest.mark.skip`, `if False:`.
- Xóa hoặc nới lỏng `assert` trong test có sẵn.
- Sửa test có sẵn mà file đó không nằm trong plan với `op=modify`.
- Test mới không có assert nào, hoặc chỉ `assert True`.

Test có sẵn **bị sửa** phải được Reviewer xác nhận lý do.

---

## 17. Reviewer Agent

Độc lập với Coder: dùng model/prompt riêng, không thấy lịch sử hội thoại của Coder.

Input:

```text
original requirement
plan (đã duyệt)
full git diff (base → HEAD của task branch)
check results + baseline delta
test weakening flags
policy events (DENY, scope requests)
changed files (nội dung đầy đủ)
```

Read-only.

Output:

```json
{
  "verdict": "PASS | FAIL",
  "acceptance": [
    { "id": "AC3", "satisfied": false, "evidence": "exp claim not checked" }
  ],
  "issues": [
    {
      "id": "REV-001",
      "severity": "high | medium | low",
      "file": "app/auth.py",
      "line": 54,
      "category": "security | correctness | test | style | scope",
      "description": "JWT expiration is not validated",
      "required_fix": "Validate exp claim"
    }
  ]
}
```

Runtime kiểm tra nhất quán:

```text
verdict == PASS AND tồn tại issue severity=high  → coi như FAIL
verdict == PASS AND tồn tại AC satisfied=false  → coi như FAIL
```

---

## 18. Review Loop

```text
REVIEWING
   │
   ├── PASS ───────────────→ GATING
   │
   └── FAIL
         │ rework_count < max_rework ?
         ├── yes → REWORK → CODING (chỉ các issue required_fix) → CHECKING → REVIEWING
         └── no  → BLOCKED
```

- Check fail trong vòng rework được tính vào `max_check_retry_per_step`, không tính vào `rework_count`.
- Nếu `rework_count` hoặc budget vượt giới hạn → `BLOCKED`, báo user kèm review cuối và diff.

---

## 19. Gate

Reviewer hỏi: *Code có lỗi không?*
Gate hỏi: *Task có thực sự hoàn thành chưa?*

### 19.1 Deterministic gate (chạy trước)

```python
def deterministic_gate(run) -> GateResult:
    if run.policy_violations:                       # DENY chưa xử lý, file ngoài scope
        return FAIL("POLICY")
    if run.files_changed - run.plan_allowed:
        return FAIL("OUT_OF_SCOPE")
    if run.incomplete_steps:
        return FAIL("INCOMPLETE_STEPS")
    if run.new_test_failures or run.new_lint_errors or run.new_type_errors:
        return FAIL("CHECKS")
    if run.test_weakening_flags_unresolved:
        return FAIL("TEST_WEAKENING")
    for ac in run.plan.acceptance_criteria:
        if ac.verify.type in ("test", "test+review") and not run.test_passed(ac.verify.ref):
            return FAIL(f"AC_NOT_VERIFIED:{ac.id}")
    if run.review.effective_verdict != "PASS":      # sau kiểm tra nhất quán §17
        return FAIL("REVIEW")
    if run.review.unresolved(severity="high"):
        return FAIL("HIGH_SEVERITY")
    return PASS
```

### 19.2 LLM gate (chỉ chạy khi deterministic PASS)

Đối chiếu yêu cầu gốc với diff + plan + review, trả lời "có phần nào của yêu cầu chưa được làm không".

Output: `{ "complete": bool, "missing": [...] }`.

- `complete = false` và còn `rework_count` → `REWORK` với `missing` làm feedback.
- Hết lượt → `BLOCKED`.

LLM gate **không thể** chuyển FAIL thành PASS. Nó chỉ có thể chặn thêm.

---

## 20. State Machine

### 20.1 States

```text
CREATED
CLASSIFYING
PLANNING
PLAN_VALIDATION
AWAITING_PLAN_APPROVAL
PREPARING            (tạo worktree + baseline)
RESEARCHING
CODING
AWAITING_APPROVAL    (command REQUIRES_APPROVAL, scope request)
CHECKING
REVIEWING
REWORK
GATING
DONE

BLOCKED              (vượt giới hạn / budget — user có thể resume)
FAILED               (lỗi không phục hồi)
CANCELLED            (user dừng)
```

### 20.2 Transition table

| From | Điều kiện | To |
|---|---|---|
| CREATED | — | CLASSIFYING |
| CLASSIFYING | — | PLANNING |
| PLANNING | có output | PLAN_VALIDATION |
| PLAN_VALIDATION | valid | AWAITING_PLAN_APPROVAL |
| PLAN_VALIDATION | invalid, `replan < max_replan` | PLANNING |
| PLAN_VALIDATION | invalid, hết lượt | FAILED |
| AWAITING_PLAN_APPROVAL | user approve | PREPARING |
| AWAITING_PLAN_APPROVAL | user feedback | PLANNING (không tính vào `replan`) |
| AWAITING_PLAN_APPROVAL | user reject | CANCELLED |
| PREPARING | ok, `needs_research` | RESEARCHING |
| PREPARING | ok | CODING (step đầu tiên) |
| PREPARING | lỗi (không phải git repo, Docker lỗi…) | FAILED |
| RESEARCHING | — | CODING |
| CODING | Coder gọi tool cần duyệt | AWAITING_APPROVAL |
| CODING | `report_step_done` | CHECKING |
| AWAITING_APPROVAL | user trả lời | CODING |
| CHECKING | pass, còn step | checkpoint commit → CODING (step kế) |
| CHECKING | pass, hết step | REVIEWING |
| CHECKING | fail, `check_retry < max` | CODING (cùng step, kèm feedback) |
| CHECKING | fail, hết lượt | BLOCKED |
| REVIEWING | effective PASS | GATING |
| REVIEWING | FAIL, `rework < max_rework` | REWORK |
| REVIEWING | FAIL, hết lượt | BLOCKED |
| REWORK | — | CODING (`rework += 1`) |
| GATING | PASS | DONE |
| GATING | FAIL, `rework < max_rework` | REWORK |
| GATING | FAIL, hết lượt | BLOCKED |
| *any active* | vượt budget | BLOCKED |
| *any active* | user stop | CANCELLED |
| *any active* | lỗi không phục hồi | FAILED |
| BLOCKED | user resume (+ budget / hướng dẫn) | CODING hoặc PLANNING |

Mọi transition do Workflow Engine thực hiện, được ghi vào bảng `events` **trong cùng transaction** với việc cập nhật `tasks.status`.

---

## 21. State Schema

```python
class TaskState(BaseModel):
    task_id: str
    run_id: str
    user_request: str
    status: TaskStatus
    current_agent: str | None
    current_step: str | None
    plan_version: int
    policy_version: str

    # counters (giới hạn lấy từ policy snapshot, không lấy từ plan)
    replan_count: int
    check_retry_count: dict[str, int]    # theo step
    rework_count: int
    scope_request_count: int

    # git
    base_commit: str
    task_branch: str
    worktree_path: str
    last_checkpoint: str | None          # commit sha sau step PASS gần nhất

    # results
    baseline: dict | None
    changed_files: list[str]
    test_result: dict | None
    review_result: dict | None
    gate_result: dict | None

    # budget usage
    tokens_used: int
    cost_usd: float
    tool_calls: int
    started_at: datetime

    pending_approval: dict | None
```

---

## 22. LLM không update state trực tiếp

```text
LLM
 ↓
Structured output
 ↓
Schema validator
 ↓
Policy validator
 ↓
Workflow engine (transition table)
 ↓
SQLite (transaction: state + event)
```

---

## 23. Database

V1 dùng SQLite (WAL mode). **Có ngay từ tuần 2**, cùng lúc với state machine.

Tables:

```text
projects
tasks
task_runs
plans            (mỗi plan_version một dòng)
policy_snapshots
steps
agent_runs
tool_calls
approvals
file_changes
check_runs       (baseline + mỗi lần check)
reviews
gate_results
events
model_usage
```

Quan hệ:

```text
Project
 └─ Task
     └─ Run
         ├─ Policy snapshot
         ├─ Plans (versions)
         ├─ Steps
         ├─ Agent Runs ── Tool Calls
         ├─ Approvals
         ├─ Check Runs
         ├─ Reviews
         ├─ Gate Results
         └─ Events
```

Một project chỉ có **tối đa một run đang active** (lock theo `project_id`).

---

## 24. Memory

Không cần Vector DB trong MVP.

### Session Memory
Task hiện tại, plan, review, tool output (đã tóm tắt).

### Project Memory

```text
memory/
├── architecture.md
├── conventions.md
├── decisions.md
└── commands.md      (lệnh test/lint/build của project — nguồn cho Command Policy)
```

- Agent **không tự ghi** vào project memory ở V1. Sau khi task DONE, hệ thống có thể đề xuất cập nhật, user duyệt.
- `memory/` nằm ngoài `user_ceiling` mặc định.

### Long-term Memory
Để phase sau.

---

## 25. Context Builder

Không truyền toàn bộ conversation hay toàn bộ repo cho agent nào.

| Agent | Context |
|---|---|
| Classifier | Requirement, project summary ngắn |
| Architect | Requirement, repo map, relevant files, architecture docs, policy summary, baseline summary |
| Coder | Plan step, file của step + file liên quan (import/caller từ repo map), research, check/review feedback |
| Reviewer | Requirement, plan, full diff, check results, flags |
| Gate (LLM) | Requirement, plan, diff summary, review |

Mọi file đưa vào context đều đi qua **read policy**.

### 25.1 Prompt injection

Nguồn không tin cậy: nội dung file trong repo, output của command, kết quả của Researcher, trang web.

- Bọc nội dung không tin cậy trong delimiter rõ ràng và ghi chú trong system prompt rằng đó là *data*.
- Cơ chế phòng thủ chính **không phải prompt** mà là: sandbox không có network, policy, không có tool git ghi, và user duyệt.
- Coder không bao giờ nhận được `.env` hay secret (read_denied + sandbox không mount secret).

---

## 26. Model Router

Dùng **LiteLLM**; không tự viết từng provider.

```yaml
# config/models.yaml
agents:
  classifier:
    model: ollama/qwen2.5:3b
    fallback: [gemini/gemini-flash]
  architect:
    model: anthropic/claude-opus-...
    fallback: [gemini/gemini-pro]
  coder:
    model: anthropic/claude-sonnet-...
    fallback: []
  reviewer:
    model: anthropic/claude-opus-...   # nên khác model/prompt với coder
    fallback: []
  gate:
    model: anthropic/claude-sonnet-...
```

Wrapper mỏng của ta:

```python
class LLMClient:
    async def generate_structured(self, agent: str, messages, schema: type[BaseModel]) -> BaseModel: ...
```

Wrapper chịu trách nhiệm: chọn model theo agent, fallback, retry khi output sai schema, ghi token/cost vào `model_usage`, trừ budget.

---

## 27. Local Model Strategy

Model local 3–4B dùng cho:

- Classification (Classifier).
- Summarization, log compression (tóm tắt tool output dài trước khi đưa vào context).
- Simple extraction.

Không dùng cho: architecture, coding lớn, security review, gate semantic.

---

## 28. Tool Registry

```text
ToolRegistry
├── filesystem   read · write · patch · search          (qua File Policy)
├── shell        run_command                            (qua Command Policy, trong sandbox)
├── git          diff · status · log                    (read-only; agent không có tool ghi)
├── checks       pytest · ruff · pyright                (adapter, output có cấu trúc)
├── scope        request_scope
├── workflow     report_step_done
└── research     search · fetch                         (chỉ Researcher)
```

Git ghi (worktree, commit checkpoint, revert) là **API nội bộ của runtime**, không phải tool của agent.

---

## 29. Command Policy

Flow:

```text
Command (string)
 ↓
Parse thành argv (shlex) — KHÔNG chạy qua shell
 ↓
Reject metacharacters: ; && || | > < ` $( ) \n
 ↓
Executable + arguments
 ↓
Working directory (phải trong worktree)
 ↓
Risk level
 ↓
Execute trong sandbox (timeout, giới hạn output)
 ↓
Post-exec diff check (§16)
```

### Risk levels

| Level | Ví dụ | Xử lý |
|---|---|---|
| **SAFE** | `git status`, `git diff`, `git log`, `ls`, `cat` (qua read policy) | Chạy trong sandbox, không cần duyệt |
| **SANDBOXED** | `pytest`, `ruff`, `pyright`, `python <file>`, `npm test`, `npm run lint` | Chạy **chỉ trong sandbox**, network off, timeout, không duyệt |
| **REQUIRES_APPROVAL** | `pip install`, `npm install`, lệnh cần network | `AWAITING_APPROVAL`; nếu duyệt thì chạy trong sandbox với network bật tạm thời |
| **DENY** | mọi `git` ghi (`commit`, `checkout`, `reset`, `push`…), `rm -rf` ngoài file trong plan, `curl`/`wget` (ngoài research), `sudo`, `chmod` system, `docker`, format/xoá ổ, đọc credential store | Chặn, ghi event |

Allowlist theo project nằm trong `config/policies.yaml` + `memory/commands.md`, do user quản lý. Lệnh không nằm trong danh sách nào → **DENY** mặc định.

### 29.1 Sandbox

```text
Docker container per run
├── mount: <worktree>  → /workspace   (rw)
├── KHÔNG mount: .env, secrets/, home, ổ đĩa host
├── network: none (mặc định)
├── user: non-root
├── CPU / memory / pids limit
└── timeout mỗi command (vd 300s), timeout tổng theo budget
```

Vì sao cần: `pytest` chạy `conftest.py` và test file do Coder viết; `python` chạy code tùy ý. Không có sandbox thì File Policy bị bypass hoàn toàn.

**Post-exec diff check:** sau mỗi command, runtime chạy `git status --porcelain` trong worktree. File thay đổi ngoài `plan_allowed` → revert về checkpoint, ghi `POLICY_VIOLATION`, trả lỗi cho Coder.

---

## 30. Workspace Boundary

```text
Project gốc của user:   D:\projects\my-project           (agent KHÔNG chạm)
Worktree của task:      D:\projects\.mase-worktrees\TASK-001   (mount vào sandbox)
```

- Agent chỉ thấy worktree (qua mount `/workspace`).
- Không bao giờ mount `C:\`, `D:\`, `Users\`, `Windows\`.
- Working tree dở dang của user không bị ảnh hưởng.

---

## 31. Git Strategy

V1 **yêu cầu project là git repo** (nếu không, hướng dẫn user `git init` và commit trước).

```text
PREPARING
 ├─ ghi base_commit = HEAD của branch user chọn
 ├─ git worktree add <path> -b mase/TASK-001 <base_commit>
 └─ baseline checks trong worktree
CODING / CHECKING
 └─ step PASS → runtime commit "TASK-001 S1: ..." trên mase/TASK-001   (checkpoint)
fail / violation
 └─ git checkout / reset về last_checkpoint trong worktree (runtime làm)
DONE
 └─ Final report: diff base_commit..HEAD, hướng dẫn merge
```

Không làm tự động:

- Merge vào branch của user.
- Push.
- Tạo PR.

User tự quyết merge / cherry-pick / xoá task branch. Hệ thống có nút "Discard" để xoá worktree + branch.

Checkpoint commit cũng chính là điểm **resume**: crash giữa step → quay về `last_checkpoint`, chạy lại step đó.

---

## 32. Observability

Event log **từ tuần 1**: JSONL trước, sau đó ghi song song vào bảng `events`.

```json
{
  "ts": "2026-09-28T09:31:02Z",
  "task_id": "TASK-001",
  "run_id": "RUN-001",
  "agent": "coder",
  "step": "S1",
  "event": "tool_call",
  "tool": "write_file",
  "args_summary": { "path": "app/auth.py" },
  "policy": "ALLOW",
  "status": "success",
  "latency_ms": 42,
  "tokens": { "in": 0, "out": 0 }
}
```

Theo dõi: agent calls, tool calls, policy ALLOW/DENY, approvals, tokens, cost, latency, errors, retries, files modified, checks, review issues, state transitions, budget còn lại.

---

## 33. UI

UI không chỉ là chatbot.

```text
┌──────────────────────────────────────────────────────────────┐
│ Project ▾                       Budget: $1.20 / $5   Settings│
├────────────┬───────────────────────┬─────────────────────────┤
│ Tasks      │ Workflow              │ Artifacts               │
│            │                       │                         │
│ TASK-01 ●  │ Classify ✓            │ plan.md                 │
│ TASK-02 ✓  │ Architect ✓           │ plan.json               │
│ TASK-03 ⏸  │ Approve plan ✓        │ diff                    │
│            │ Prepare ✓ (baseline)  │ checks.json             │
│            │ Coder S1 ✓  S2 ●  S3 ○│ review.json             │
│            │ Review ○              │ gate.json               │
│            │ Gate ○                │                         │
├────────────┴───────────────────────┴─────────────────────────┤
│ ⚠ Approval needed: Coder requests create app/__init__.py      │
│   [Approve] [Reject]                                          │
├──────────────────────────────────────────────────────────────┤
│ Live Events                                                   │
│ 09:31 Architect completed plan (v1)                           │
│ 09:32 Coder read app/main.py                                  │
│ 09:33 pytest tests/test_auth.py → 7 passed                    │
│ 09:33 DENY write .env                                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 34. Các màn hình UI

- Dashboard.
- Projects (+ cấu hình ceiling / denied / allowlist / budget).
- New Task.
- **Task Detail** (quan trọng nhất): workflow, approvals, plan, diff, checks, review, gate, events.
- Plan Viewer (duyệt / góp ý / từ chối).
- Diff Viewer.
- Approvals inbox.
- Agent Runs / Tool Calls / Logs.
- Settings (models, budget mặc định).

---

## 35. Project Structure

```text
multi-agent-se/
│
├── apps/
│   ├── cli/
│   │   └── main.py
│   ├── api/
│   │   ├── main.py
│   │   └── routes/
│   └── web/
│       ├── app/
│       ├── components/
│       └── lib/
│
├── orchestrator/
│   ├── agents/
│   │   ├── classifier.py
│   │   ├── architect.py
│   │   ├── researcher.py
│   │   ├── coder.py
│   │   ├── reviewer.py
│   │   └── gate.py
│   ├── workflow/
│   │   ├── engine.py
│   │   ├── transitions.py
│   │   └── approvals.py
│   ├── planning/
│   │   ├── schema.py
│   │   ├── validator.py
│   │   └── render.py            # plan.json → plan.md
│   ├── policy/
│   │   ├── paths.py             # normalize / resolve / Windows cases
│   │   ├── file_policy.py
│   │   ├── command_policy.py
│   │   ├── scope.py
│   │   └── budget.py
│   ├── checks/
│   │   ├── baseline.py
│   │   ├── diff_validation.py
│   │   ├── test_weakening.py
│   │   ├── pytest_adapter.py
│   │   ├── ruff_adapter.py
│   │   └── pyright_adapter.py
│   ├── state/
│   │   ├── models.py
│   │   ├── repository.py
│   │   └── events.py
│   ├── runtime/
│   │   ├── openhands.py         # adapter + policy hook
│   │   ├── sandbox.py           # Docker
│   │   └── git_ops.py           # worktree, checkpoint, revert
│   ├── context/
│   │   ├── builder.py
│   │   └── repo_map.py
│   ├── llm/
│   │   └── client.py            # LiteLLM wrapper
│   └── tools/
│       ├── filesystem.py
│       ├── shell.py
│       ├── git_read.py
│       └── scope.py
│
├── memory/
├── data/
│   └── agent.db
├── logs/
├── benchmarks/
│   ├── fixtures/                # repo mẫu
│   └── golden_tasks/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── policy/
│   └── e2e/
├── config/
│   ├── agents.yaml
│   ├── models.yaml
│   └── policies.yaml
├── pyproject.toml
└── README.md
```

---

## 36. API

```text
POST   /projects
GET    /projects
PATCH  /projects/{id}/policy

POST   /tasks
GET    /tasks/{id}
POST   /tasks/{id}/run
POST   /tasks/{id}/stop
POST   /tasks/{id}/resume
POST   /tasks/{id}/discard            # xoá worktree + task branch

GET    /tasks/{id}/state
GET    /tasks/{id}/plan
POST   /tasks/{id}/plan/approve
POST   /tasks/{id}/plan/feedback
GET    /tasks/{id}/diff
GET    /tasks/{id}/checks
GET    /tasks/{id}/review
GET    /tasks/{id}/gate
GET    /tasks/{id}/events

GET    /approvals?status=pending
POST   /approvals/{id}                # { decision: approve | reject, note }

GET    /tasks/{id}/stream             # SSE
```

---

## 37. Testing Strategy

### Unit

```text
PlanValidator
FilePolicy (read + write)
CommandPolicy (parse, metachar, levels)
ScopeRequest
Budget
Transition table (mọi dòng trong §20.2)
Deterministic Gate
Test weakening detector
Review consistency check
```

### Policy tests (bắt buộc pass trước khi tăng autonomy)

```text
../outside.txt                 → DENY
C:\Windows\x                   → DENY
\\?\C:\x, \\server\share\x     → DENY
.env, .ENV, .Env               → DENY (read + write)
.env::$DATA, app.py:hidden     → DENY
SECRET~1\key                   → DENY
CON, aux.py                    → DENY
symlink / junction ra ngoài    → DENY
.git/config, .git/hooks/x      → DENY
allowed.py (trong plan)        → ALLOW
new.py (ngoài plan, trong ceiling) → DENY + gợi ý request_scope
python -c "open('../x','w')"   → chạy trong sandbox, không ảnh hưởng host
pytest với conftest ghi ra ngoài scope → post-exec diff check bắt được, revert
pytest; rm -rf .               → DENY (metachar)
curl https://...               → DENY (network off)
git commit / reset             → DENY
```

### Integration

```text
Architect → plan → validator
plan → approval → prepare (worktree + baseline)
coder → check → checkpoint
check fail → retry → blocked
review fail → rework → pass
gate fail → rework
crash giữa step → resume từ checkpoint
budget vượt → BLOCKED
```

### E2E

Task mẫu: *Add health endpoint* trên repo fixture.

Expected:

```text
plan generated + approved
worktree created, baseline recorded
file changed (chỉ trong plan)
test created, test pass
review pass, gate pass
user repo không đổi, task branch có commit
```

---

## 38. Golden Tasks Benchmark

Tạo **10 task ở tuần 1**, đủ **25 task ở tuần 5**. Chạy lại sau mỗi thay đổi lớn.

| Loại | Số task |
|---|---:|
| Simple edit | 5 |
| Feature | 5 |
| Bug fix | 5 |
| Test generation | 3 |
| Refactor | 3 |
| Scope violation (cố ý cần file ngoài ceiling) | 2 |
| Dangerous command / prompt injection trong repo | 2 |

Mỗi task có: repo fixture, yêu cầu, test ẩn để chấm (Coder không thấy), kết quả mong đợi.

Metrics:

```text
Task completion rate            (theo test ẩn, không theo Gate)
False PASS rate                 (Gate PASS nhưng test ẩn fail)
Policy violation attempts / blocked rate
Reviewer catch rate
Average retries / reworks
Average tokens, cost, wall-clock
Average tool calls
Approvals per task                (đo gánh nặng cho user)
```

---

## 39. Definition of Done — MVP

MVP chỉ xong khi:

- User tạo task (CLI và UI).
- Architect sinh plan hợp lệ; runtime validate plan.
- User duyệt plan trước khi code.
- Task chạy trên worktree riêng; repo của user không bị đổi.
- Baseline được ghi; check đánh giá theo delta.
- Coder chỉ ghi được file trong plan; không đọc được file `read_denied`.
- Mọi command chạy trong sandbox, qua Command Policy.
- Post-exec diff check bắt được thay đổi ngoài scope và revert.
- Scope request đi qua duyệt.
- Test weakening bị phát hiện.
- Reviewer trả structured result; kiểm tra nhất quán.
- Fail quay về Coder; mọi loop có giới hạn; budget có hiệu lực.
- Gate deterministic + LLM xác nhận completion.
- State + event lưu SQLite trong cùng transaction.
- Restart giữa chừng → resume từ checkpoint.
- UI thấy live workflow, approvals, diff, checks, review, gate.
- Toàn bộ policy tests §37 pass.
- Golden Tasks: false PASS rate được đo và báo cáo.

---

# 40. Roadmap 8 tuần

## Tuần 1 — Spike + Foundation

- Spike OpenHands theo §2.1 (G1–G5). **Cuối tuần chốt go/no-go.**
- Docker sandbox chạy trên Windows (Docker Desktop + WSL2).
- Python 3.12, skeleton project, config YAML.
- Event log JSONL.
- LiteLLM wrapper tối thiểu.
- Repo fixture + **10 Golden Tasks đầu tiên** + script chấm bằng test ẩn.

Deliverable:

```text
CLI task → single Coder (OpenHands, sandbox) → sửa file trong fixture → events JSONL
+ chứng minh policy hook chặn được write/command
```

## Tuần 2 — Planning + State

- Plan schema v2 (Pydantic, extra=forbid), Architect, render `plan.md`.
- PlanValidator.
- State machine theo bảng §20.2 + SQLite persistence (state + event cùng transaction).
- Plan approval qua CLI.

Workflow: `User → Architect → Validator → Approval → Coder`.

## Tuần 3 — Safety

- Git worktree, checkpoint commit, revert.
- File Policy (read + write), path normalization (gồm Windows cases).
- Command Policy (argv, metachar, levels, allowlist).
- Post-exec diff check.
- Scope Expansion Request + AWAITING_APPROVAL.
- Toàn bộ policy tests §37.

Milestone:

```text
Agent không thể đọc secret, không thể ghi ngoài scope — kể cả qua shell.
```

Chưa đạt milestone này thì **không** phát triển thêm autonomy.

## Tuần 4 — Check + Review Loop

- Baseline, adapters pytest / ruff / pyright (output có cấu trúc).
- Diff validation, test weakening detection.
- Reviewer + consistency check.
- Rework loop, check retry.
- Deterministic Gate + LLM Gate.

Flow: `Coder → Check → (step loop) → Reviewer → Gate`.

## Tuần 5 — Reliability + MVP (CLI)

- Resume từ checkpoint, error recovery.
- Budget Manager.
- Classifier + Researcher.
- Đủ **25 Golden Tasks**, chạy benchmark lần đầu.

Cuối tuần 5: **MVP chạy được đầy đủ qua CLI.**

## Tuần 6 — API + UI

- FastAPI endpoints §36, SSE stream.
- Next.js: Dashboard, Project settings, New Task, Task Detail, Plan approval, Approvals inbox, Diff, Checks, Review, Logs.

## Tuần 7 — Multi-model + Context

- Model config per agent + fallback (LiteLLM).
- Context Builder + repo map.
- Tóm tắt tool output dài bằng model local.
- Project memory (đọc; đề xuất cập nhật có duyệt).
- Token / cost tracking trên UI.

## Tuần 8 — Hardening

Test: crash, restart, bad JSON, LLM timeout, tool timeout, test timeout, invalid path, policy bypass (red-team bằng chính agent), infinite retry, provider failure, Docker không chạy.

Sau đó: documentation, installer, config wizard, demo project, benchmark report.

---

# 41. Nếu bắt buộc xong trong 5 tuần

Cắt:

- UI đầy đủ → chỉ CLI + (tuỳ chọn) trang web read-only xem events/diff.
- Researcher.
- Classifier (luôn `needs_research = false`).
- LLM Gate (giữ deterministic Gate).
- Multi-model fallback.
- Repo map nâng cao (dùng ripgrep + file của step).
- Cost dashboard.

**Giữ — không được cắt:**

```text
Architect + Validator + Plan approval
Coder (OpenHands) trong Docker sandbox
File policy (read + write) + Command policy + post-exec diff check
Worktree + checkpoint
State machine + SQLite
Baseline + checks + test weakening
Reviewer + Deterministic Gate
Budget
Golden Tasks (tối thiểu 10)
```

---

# 42. Không nên build từ đầu

Không viết lại:

- Generic agent loop (trừ khi spike tuần 1 fail — khi đó viết loop tối giản kiểu mini-SWE-agent).
- Terminal abstraction, file editing engine.
- Workspace / sandbox abstraction.
- Conversation runtime.
- LLM provider clients (dùng LiteLLM).

---

# 43. Những gì nên học từ repo khác

| Repo | Lấy gì |
|---|---|
| ChatDev 2.0 | workflow definition, node routing, agent abstraction, tool extensibility |
| mini-SWE-agent | agent loop phải đơn giản; phương án fallback |
| MetaGPT | Role, SOP, handoff, responsibility separation (không dùng làm dependency) |
| aider | repo map, cách chọn context |
| OpenHands | sandbox, event stream, security analyzer / confirmation |

---

# 44. Kiến trúc cuối cùng

```text
                     USER
                      │
              ┌───────┴───────┐
              │  CLI / Next.js │
              └───────┬───────┘
                      ▼
               ORCHESTRATOR (FastAPI)
                      │
        ┌─────────────┼──────────────┐
        │             │              │
  Workflow Engine   State (SQLite)  Budget
        │
    Classifier
        │
    Architect ──► plan.json ──► Plan Validator ──► USER APPROVAL
                                                        │
                                              PREPARE (worktree, baseline)
                                                        │
                                  ┌─────────────────────┤
                                  │                     │
                             Researcher               Coder ◄──────────┐
                                  │                     │               │
                                  └────────► AUTO CHECK (per step)      │
                                                        │ pass          │ fail
                                                  checkpoint commit ────┘
                                                        │ all steps
                                                    Reviewer
                                                        │
                                              ┌─────────┴────────┐
                                             FAIL               PASS
                                              │                   │
                                         REWORK → Coder          Gate
                                                             (determ. → LLM)
                                                                  │
                                                                DONE
                                                         (user tự merge)

────────────────────────────────────────────────────────────────

Execution layer

   OpenHands SDK (Coder loop)
          │
   Policy hook ── File Policy (read/write) · Command Policy · Scope
          │
   Docker sandbox (worktree only, network off)
          │
   Post-exec diff check ── revert về checkpoint nếu vi phạm
```

---

# 45. Thứ tự implementation thực tế

> Chi tiết từng bước (checklist, output, tiêu chí hoàn thành): xem [`plan/README.md`](plan/README.md).

```text
01 Spike OpenHands + Docker trên Windows (go/no-go)
        ↓
02 Event log JSONL + Golden Tasks (10) + script chấm
        ↓
03 Coder chạy được trong sandbox trên fixture
        ↓
04 Plan schema v2 + render plan.md
        ↓
05 Architect
        ↓
06 Plan validator
        ↓
07 State machine + SQLite persistence
        ↓
08 Plan approval (CLI)
        ↓
09 Git worktree + checkpoint + revert
        ↓
10 File policy (read + write, Windows cases)
        ↓
11 Command policy + post-exec diff check
        ↓
12 Scope request + approvals
        ↓
13 Baseline + check adapters + diff validation + test weakening
        ↓
14 Reviewer + consistency check
        ↓
15 Rework loop
        ↓
16 Deterministic Gate → LLM Gate
        ↓
17 Budget + resume
        ↓
18 Classifier + Researcher
        ↓
19 Golden Tasks (25) + benchmark
        ↓
20 API + UI
        ↓
21 Context optimization + multi-model
        ↓
22 Hardening
```

---

# 46. Risks & Open Questions

| # | Rủi ro / câu hỏi | Giảm thiểu | Chốt khi nào |
|---|---|---|---|
| R1 | Không hook được policy vào tool loop của OpenHands | Spike §2.1; fallback mini-SWE-agent style | Cuối tuần 1 |
| R2 | Docker trên Windows chậm / khó cài cho user | WSL2 backend; worktree đặt trong filesystem WSL nếu I/O chậm | Tuần 1 |
| R3 | LLM trả JSON sai schema thường xuyên | Structured output của provider + retry có lỗi cụ thể | Tuần 2 |
| R4 | Quá nhiều approval → user mệt | Đo "approvals per task"; `auto_approve_scope` cho pattern rủi ro thấp | Tuần 5 |
| R5 | Chi phí mỗi task cao | Budget, model rẻ cho classifier/tóm tắt, targeted tests, context tối thiểu | Tuần 5, 7 |
| R6 | Project không phải Python (JS/TS…) | V1 tập trung Python; adapter checks thiết kế dạng plugin | Sau V1 |
| R7 | Reviewer cùng họ model với Coder → cùng điểm mù | Cho phép model/provider khác cho Reviewer; đo reviewer catch rate | Tuần 7 |
| Q1 | Team bao nhiêu người? 8 tuần có khả thi? | Nếu 1 người: đi theo §41 | Ngay |

---

# 47. Nguyên tắc phát triển cuối cùng

Ưu tiên theo thứ tự:

```text
Correctness
   ↓
Safety
   ↓
State integrity
   ↓
Repeatability
   ↓
Observability
   ↓
Autonomy
```

Không tăng mức autonomy (ít approval hơn, nhiều agent hơn, chạy song song) trước khi:

```text
plan → approval → policy + sandbox → coder → check → reviewer → gate
```

chạy ổn định trên Golden Tasks với false PASS rate chấp nhận được.

Đây là core workflow quan trọng nhất của toàn bộ hệ thống.
