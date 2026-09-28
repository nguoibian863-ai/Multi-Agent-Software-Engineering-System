# 07 — State machine + SQLite persistence

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 2 — Planning + State (tuần 2) |
| Phụ thuộc | 02 |
| Mở khoá | 08, 09, 12, 15, 17, 18 |
| Master plan | §7, §20, §21, §22, §23 |
| Ước lượng | 2 ngày |

## Mục tiêu

Workflow Engine deterministic điều khiển toàn bộ vòng đời task bằng **bảng transition dạng dữ liệu**, state và event được lưu SQLite **trong cùng transaction**.

## Việc cần làm

### Database
- [ ] `orchestrator/state/models.py` (SQLModel): các bảng §23 — `projects`, `tasks`, `task_runs`, `plans`, `policy_snapshots`, `steps`, `agent_runs`, `tool_calls`, `approvals`, `file_changes`, `check_runs`, `reviews`, `gate_results`, `events`, `model_usage`.
- [ ] SQLite WAL mode, foreign keys bật; `data/agent.db`.
- [ ] Migration đơn giản (Alembic hoặc `create_all` + version table).

### Repository
- [ ] `orchestrator/state/repository.py`
  - [ ] `transition(run_id, from_state, to_state, event_payload)` — cập nhật `task_runs.status` + insert `events` trong **một transaction**, kiểm tra `from_state` khớp (optimistic lock).
  - [ ] Counters: `replan_count`, `check_retry_count[step]`, `rework_count`, `scope_request_count`.
  - [ ] Lock một run active mỗi project (unique partial index hoặc bảng lock).
- [ ] `EventLogger` ghi song song JSONL + bảng `events`.

### Workflow
- [ ] `orchestrator/workflow/transitions.py`: bảng §20.2 dưới dạng dữ liệu — `(state, trigger) -> (guard, next_state)`.
- [ ] `orchestrator/workflow/engine.py`
  - [ ] Vòng lặp: load state → gọi handler của state → nhận `trigger` → tra bảng → `transition`.
  - [ ] Handler mỗi state là hàm riêng, **idempotent** (chạy lại an toàn — cần cho resume ở bước 17).
  - [ ] Transition không có trong bảng → `IllegalTransition` → `FAILED`.
  - [ ] Handler cho state chưa làm (REVIEWING, GATING…) tạm là stub trả trigger mặc định.
- [ ] `TaskState` (§21) là view đọc từ DB, không phải nguồn sự thật riêng.

## Output

```text
orchestrator/state/models.py
orchestrator/state/repository.py
orchestrator/workflow/transitions.py
orchestrator/workflow/engine.py
tests/unit/test_transitions.py
tests/integration/test_engine_persistence.py
```

## Tiêu chí hoàn thành

- [ ] Test cho **mọi dòng** trong bảng §20.2 (tham số hoá).
- [ ] Transition trái phép → exception, state không đổi.
- [ ] Test atomicity: lỗi giữa chừng → không có event mồ côi, status không đổi.
- [ ] Hai run cùng project → run thứ hai bị từ chối.
- [ ] Engine chạy hết luồng với toàn bộ agent là fake → tới `DONE`.

## Rủi ro / Ghi chú

- LLM không bao giờ gọi `repository` trực tiếp — chỉ engine gọi.
- Giữ `transitions.py` là dữ liệu thuần để UI (bước 20) render được sơ đồ.
