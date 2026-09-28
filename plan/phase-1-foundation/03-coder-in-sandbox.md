# 03 — Coder chạy trong sandbox

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 1 — Foundation (tuần 1) |
| Phụ thuộc | 01 (ADR-001), 02 |
| Mở khoá | 09, 10 |
| Master plan | §2, §14, §29.1, §30 |
| Ước lượng | 1–2 ngày |

## Mục tiêu

Biến kết quả spike thành code thật: một **Coder duy nhất** nhận yêu cầu thô, chạy trong Docker sandbox, sửa được fixture, mọi tool call đi qua một **policy hook** (tạm thời là stub).

## Việc cần làm

- [ ] `orchestrator/runtime/sandbox.py`
  - [ ] `Sandbox.start(mount_path) -> container`: network `none`, user non-root, giới hạn CPU / memory / pids.
  - [ ] `Sandbox.exec(argv, cwd, timeout) -> ExecResult` (stdout/stderr bị cắt theo giới hạn, exit code, duration).
  - [ ] `Sandbox.stop()`; luôn dọn container kể cả khi lỗi.
  - [ ] Image base: `python:3.12-slim` + deps của fixture; build script `docker/sandbox.Dockerfile`.
- [ ] `orchestrator/runtime/openhands.py` — adapter theo hook đã chọn trong ADR-001
  - [ ] Đăng ký tool của ta (`read_file`, `write_file`, `run_command`, `report_step_done`).
  - [ ] Mỗi tool gọi `policy_hook.check(...)` trước khi thực thi.
  - [ ] Chuyển event của OpenHands → `EventLogger`.
- [ ] `orchestrator/policy/hook.py` — **stub**: allow mọi path trong workspace, deny path có `..`, deny command ngoài danh sách tạm `[pytest, ruff, python]`. Sẽ bị thay ở bước 10–11.
- [ ] `orchestrator/agents/coder.py`: system prompt tối thiểu, input = yêu cầu thô.
- [ ] `apps/cli/main.py`: `mase run --project <path> --task "<text>"`.
  - [ ] Tạm thời copy project sang thư mục tạm và mount bản copy (worktree thật ở bước 09).

## Output

```text
docker/sandbox.Dockerfile
orchestrator/runtime/sandbox.py
orchestrator/runtime/openhands.py
orchestrator/policy/hook.py        (stub)
orchestrator/agents/coder.py
apps/cli/main.py
tests/integration/test_sandbox.py
```

## Tiêu chí hoàn thành

- [ ] `mase run` giải được GT-001 (health endpoint) trên bản copy fixture; hidden test pass.
- [ ] Project gốc không bị thay đổi (so hash thư mục trước/sau).
- [ ] Integration test: trong sandbox không có network, không thấy file ngoài mount.
- [ ] Mỗi tool call có event với `policy: ALLOW|DENY`.
- [ ] Container được dọn sau khi run kết thúc hoặc crash (test bằng exception giả).

## Rủi ro / Ghi chú

- Nếu ADR-001 là No-go: file `openhands.py` được thay bằng `orchestrator/runtime/agent_loop.py` (loop tối giản: LLM → parse tool call → policy → exec → observation), phần còn lại giữ nguyên.
- Chưa có plan, chưa có scope — chỉ chứng minh đường ống end-to-end.
