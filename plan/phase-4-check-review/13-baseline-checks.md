# 13 — Baseline + auto checks + test weakening

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 4 — Check + Review (tuần 4) |
| Phụ thuộc | 09, 11 |
| Mở khoá | 14 |
| Master plan | §16, §16.1, §16.2 |
| Ước lượng | 2 ngày |

## Mục tiêu

Sau mỗi step, runtime tự kiểm tra deterministic và so với **baseline**. Pass → checkpoint commit; fail → feedback có cấu trúc cho Coder.

## Việc cần làm

### Adapters (chạy trong sandbox, output có cấu trúc)
- [ ] `checks/pytest_adapter.py`: `--junitxml` → `TestResult(passed, failed[{nodeid, message}], errors, duration)`; hỗ trợ chạy theo nodeid / file.
- [ ] `checks/ruff_adapter.py`: `--output-format json` → list `LintIssue(file, code, message, line)`.
- [ ] `checks/pyright_adapter.py`: `--outputjson` → list `TypeIssue`.
- [ ] Syntax check: `python -m py_compile` trên file thay đổi.

### Baseline
- [ ] `checks/baseline.py`: chạy trong `PREPARING` sau khi tạo worktree; lưu vào `check_runs` (kind=baseline).
- [ ] So sánh delta: key lint/type theo `(file, code, message)` — **không** theo số dòng (dòng dịch chuyển khi sửa).
- [ ] Test: pass trước → phải pass sau; test fail sẵn trong baseline không chặn (nhưng ghi vào report).

### Diff validation (mở rộng từ bước 11)
- [ ] Scope (đã có), file binary, file > N KB bất thường.
- [ ] Security rules đơn giản: pattern secret (AWS key, private key, token dài), `eval(` / `exec(`, SQL ghép chuỗi → issue cho Reviewer.

### Test weakening
- [ ] `checks/test_weakening.py` — so sánh AST test file giữa `base_commit` và HEAD:
  - [ ] test function bị xoá / test file bị xoá
  - [ ] thêm `pytest.mark.skip` / `skipif` / `xfail` / `if False:`
  - [ ] số `assert` trong test có sẵn giảm
  - [ ] test có sẵn bị sửa mà không có `op=modify` trong plan
  - [ ] test mới không có assert hoặc chỉ `assert True`
  - Kết quả: list `WeakeningFlag(file, test, kind)` — severity high.

### Handler CHECKING
- [ ] Chọn test: test của step (file test trong `step.files`) + test import module bị sửa (grep) → targeted; step cuối → full suite.
- [ ] Pass → `git_ops.checkpoint(step)` → step kế hoặc `REVIEWING`.
- [ ] Fail → `CODING` với `check_feedback` (lỗi cụ thể, đã rút gọn), tăng `check_retry_count[step]`; hết lượt → `BLOCKED`.

## Output

```text
orchestrator/checks/{pytest,ruff,pyright}_adapter.py
orchestrator/checks/baseline.py
orchestrator/checks/diff_validation.py (mở rộng)
orchestrator/checks/test_weakening.py
tests/unit/test_baseline_delta.py
tests/unit/test_test_weakening.py
```

## Tiêu chí hoàn thành

- [ ] Fixture có sẵn 2–3 lỗi lint → task hợp lệ vẫn PASS check (delta = 0).
- [ ] Thêm 1 lỗi lint mới → FAIL với đúng lỗi đó.
- [ ] Mỗi loại test weakening có test phát hiện đúng; test không bị làm yếu → không flag.
- [ ] Checkpoint commit được tạo sau mỗi step pass.
- [ ] GT feature nhiều step chạy qua được `CODING → CHECKING → CODING(step kế) → … → REVIEWING`.

## Rủi ro / Ghi chú

- Full suite có thể chậm — cho phép cấu hình `full_suite_command` trong `memory/commands.md`.
