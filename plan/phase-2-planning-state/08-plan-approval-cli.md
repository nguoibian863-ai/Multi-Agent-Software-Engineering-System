# 08 — Plan approval (CLI)

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 2 — Planning + State (tuần 2) |
| Phụ thuộc | 05, 06, 07 |
| Mở khoá | 12 (dùng chung cơ chế approval) |
| Master plan | §1, §20 (AWAITING_PLAN_APPROVAL), §36 |
| Ước lượng | 1 ngày |

## Mục tiêu

Nối Architect → Validator → **User duyệt** → Coder qua Workflow Engine, điều khiển bằng CLI.

## Việc cần làm

- [ ] `orchestrator/workflow/approvals.py`
  - [ ] Model `Approval(id, run_id, kind: plan|command|scope, payload, status: pending|approved|rejected, note, decided_at)`.
  - [ ] `request(kind, payload)` → tạo approval + chuyển state sang `AWAITING_*`.
  - [ ] `decide(id, decision, note)` → ghi kết quả + phát trigger cho engine.
- [ ] Handler `PLANNING` → gọi Architect (kèm `validator_errors` / `user_feedback` nếu có).
- [ ] Handler `PLAN_VALIDATION` → gọi Validator; invalid → `PLANNING` (tăng `replan_count`) hoặc `FAILED`.
- [ ] Handler `AWAITING_PLAN_APPROVAL` → engine dừng, chờ quyết định.
- [ ] CLI (`apps/cli/main.py`, typer):
  - [ ] `mase task new --project P "<request>"`
  - [ ] `mase plan show TASK` (in `plan.md`)
  - [ ] `mase plan approve TASK`
  - [ ] `mase plan feedback TASK "<text>"` → `PLANNING`, **không** tăng `replan_count`
  - [ ] `mase plan reject TASK` → `CANCELLED`
  - [ ] `mase status TASK`
- [ ] Sau approve: `PREPARING` (tạm: copy project như bước 03) → `CODING`; Coder nhận plan + step hiện tại thay vì yêu cầu thô.

## Output

```text
orchestrator/workflow/approvals.py
apps/cli/main.py (mở rộng)
tests/integration/test_plan_approval_flow.py
```

## Tiêu chí hoàn thành

- [ ] E2E với model thật: `task new` → `plan show` → `plan approve` → Coder làm GT-001 → hidden test pass.
- [ ] `plan feedback` sinh `plan.v2` phản ánh feedback; `replan_count` không đổi.
- [ ] Plan invalid 3 lần liên tiếp (fake LLM) → `FAILED`, có thông báo lỗi.
- [ ] Tắt tiến trình khi đang `AWAITING_PLAN_APPROVAL`, mở lại → vẫn approve được.

## Rủi ro / Ghi chú

- Cơ chế `Approval` này dùng lại cho command và scope ở bước 12 — thiết kế tổng quát ngay từ đầu.
