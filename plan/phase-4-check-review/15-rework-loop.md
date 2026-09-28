# 15 — Rework loop

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 4 — Check + Review (tuần 4) |
| Phụ thuộc | 07, 14 |
| Mở khoá | 16 |
| Master plan | §18, §20.2 |
| Ước lượng | 1 ngày |

## Mục tiêu

Review FAIL → Coder sửa đúng những gì được yêu cầu → check lại → review lại, có giới hạn rõ ràng.

## Việc cần làm

- [ ] Handler `REVIEWING`:
  - effective PASS → `GATING`
  - FAIL và `rework_count < max_rework` → `REWORK`
  - FAIL và hết lượt → `BLOCKED`
- [ ] Handler `REWORK`: tăng `rework_count`, dựng `review_feedback`:
  - chỉ issue có `required_fix` + AC `satisfied=false`
  - gắn với step / file liên quan
  - → `CODING` (chế độ rework: Coder làm việc trên các file bị nêu, vẫn trong `plan_allowed`)
- [ ] Sau rework: `CHECKING` (targeted + full suite) → `REVIEWING`.
- [ ] Check fail trong rework tính vào `check_retry_count`, không tính `rework_count`.
- [ ] `BLOCKED` report (`runs/<task>/blocked.md`): lý do, review cuối, diff stat, check results, gợi ý cho user (thêm budget, sửa plan, sửa tay).
- [ ] CLI: `mase resume TASK [--note "<hướng dẫn>"]` từ `BLOCKED` → `CODING` (reset counters theo cấu hình) hoặc `--replan` → `PLANNING`.

## Output

```text
orchestrator/workflow/engine.py (handlers REVIEWING, REWORK)
templates/blocked.md.j2
tests/integration/test_rework_loop.py
```

## Tiêu chí hoàn thành

- [ ] Fake LLM: review FAIL lần 1 → rework → PASS lần 2 → `GATING`.
- [ ] Fake LLM: FAIL liên tục → dừng đúng sau `max_rework` → `BLOCKED` + `blocked.md`.
- [ ] Rework không làm Coder ghi được file ngoài plan (policy vẫn áp dụng).
- [ ] Resume từ `BLOCKED` với note → Coder nhận được note trong context.

## Rủi ro / Ghi chú

- Rework dễ "dao động" (sửa A làm hỏng B). Theo dõi trong benchmark: số rework trung bình và tỷ lệ BLOCKED.
