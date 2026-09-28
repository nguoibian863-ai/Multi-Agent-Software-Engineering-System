# 17 — Budget + Resume

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 5 — Reliability (tuần 5) |
| Phụ thuộc | 07, 09 (có thể làm song song với 13–16) |
| Mở khoá | 19 |
| Master plan | §10 (budget), §20.2, §21, §31 (checkpoint = resume point) |
| Ước lượng | 1.5–2 ngày |

## Mục tiêu

Mỗi task có trần chi phí cứng, và hệ thống chịu được crash / restart: resume từ checkpoint gần nhất mà không mất hay nhân đôi công việc.

## Việc cần làm

### Budget
- [ ] `orchestrator/policy/budget.py`: `BudgetTracker(run_id, snapshot.budget)`
  - [ ] `charge_llm(tokens_in, tokens_out, cost)` — gọi từ `LLMClient`.
  - [ ] `charge_tool_call()` — gọi từ policy hook.
  - [ ] `wall_clock_used()` — trừ thời gian ở `AWAITING_*`.
  - [ ] `check()` trước **mỗi** LLM call và tool call → vượt → exception `BudgetExceeded`.
- [ ] Engine bắt `BudgetExceeded` → `BLOCKED` (reason=BUDGET), report ghi rõ mục nào vượt.
- [ ] Cảnh báo event khi dùng 80% budget.
- [ ] CLI: `mase resume TASK --add-budget cost=2 tokens=500000`.

### Resume
- [ ] Khi khởi động: tìm run ở state active → đánh dấu `interrupted` (event).
- [ ] `mase resume TASK`:
  - [ ] `git_ops.revert_to(last_checkpoint)` (bỏ thay đổi dở của step đang chạy).
  - [ ] Dựng lại sandbox, vào lại state tương ứng: `CODING` (step hiện tại), `CHECKING` (chạy lại check), `REVIEWING` / `GATING` (gọi lại agent), `AWAITING_*` (giữ nguyên chờ user).
- [ ] Rà lại tất cả handler: **idempotent** (chạy lại không tạo commit / approval / event trùng).
- [ ] Error recovery: LLM timeout / provider error → retry có backoff (tối đa 3) → sau đó `BLOCKED` (không phải `FAILED`).

## Output

```text
orchestrator/policy/budget.py
orchestrator/workflow/resume.py
tests/integration/test_budget.py
tests/integration/test_crash_resume.py
```

## Tiêu chí hoàn thành

- [ ] Budget token nhỏ → run dừng đúng lúc vượt, `BLOCKED`, không có LLM call nào sau thời điểm vượt.
- [ ] Crash injection (exception giả) tại: giữa `CODING`, sau write trước checkpoint, giữa `CHECKING`, giữa `REVIEWING` → `resume` đều đi tới `DONE` với fake LLM; không có commit trùng.
- [ ] Kill process thật (`taskkill`) giữa run GT-002 → resume → `DONE`.
- [ ] Container mồ côi sau crash được dọn khi khởi động lại.

## Rủi ro / Ghi chú

- Cost ước tính từ bảng giá LiteLLM có thể lệch — hiển thị là "ước tính".
