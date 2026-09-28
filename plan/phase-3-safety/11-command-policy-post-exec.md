# 11 — Command Policy + post-exec diff check

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 3 — Safety (tuần 3) |
| Phụ thuộc | 09, 10 |
| Mở khoá | 12, 13 |
| Master plan | §29, §29.1, §16 (diff validation), §37 |
| Ước lượng | 1.5–2 ngày |

## Mục tiêu

Mọi command của Coder (và `proposed_commands` của Architect) được parse, phân loại rủi ro, chạy **trong sandbox**; sau mỗi command runtime kiểm tra worktree để bắt thay đổi ngoài scope.

## Việc cần làm

### Command Policy
- [ ] `orchestrator/policy/command_policy.py`
  - [ ] `parse(command: str) -> argv` bằng `shlex.split` (posix — lệnh chạy trong container Linux).
  - [ ] Reject metacharacters: `; && || | > < \` $( ) \n` và biến môi trường mở rộng.
  - [ ] `classify(argv, cwd) -> Level` với `Level = SAFE | SANDBOXED | REQUIRES_APPROVAL | DENY`.
  - [ ] Rule từ `config/policies.yaml`:

    ```yaml
    commands:
      safe:        [["ls"], ["cat"]]
      sandboxed:   [["pytest"], ["ruff", "check"], ["pyright"], ["python", "<file>"], ["npm", "test"], ["npm", "run", "lint"]]
      approval:    [["pip", "install"], ["npm", "install"], ["uv", "add"]]
      deny:        [["git"], ["curl"], ["wget"], ["sudo"], ["docker"], ["rm", "-rf"], ["chmod"]]
    default: deny
    ```

  - [ ] `cwd` phải nằm trong worktree; `python -c` → DENY (chỉ cho chạy file).
  - [ ] Lệnh không khớp rule nào → DENY.
- [ ] Plan Validator dùng Command Policy thật cho `proposed_commands` (≤ SANDBOXED).

### Thực thi
- [ ] `orchestrator/tools/shell.py`: `run_command` → Command Policy → `Sandbox.exec` (timeout, cắt output) → post-exec check → trả kết quả cho Coder.
- [ ] `REQUIRES_APPROVAL` → tạm trả lỗi "cần duyệt" (luồng duyệt làm ở bước 12).

### Post-exec diff check
- [ ] Sau **mỗi** command và mỗi lần Coder báo xong step: `git_ops.changed_files(worktree, last_checkpoint)`.
- [ ] File thay đổi không thuộc `plan_allowed` hoặc match `write_denied` → `restore_paths` về checkpoint, event `POLICY_VIOLATION`, trả lỗi rõ ràng cho Coder.
- [ ] File trong plan thay đổi hợp lệ → giữ nguyên.

## Output

```text
orchestrator/policy/command_policy.py
orchestrator/tools/shell.py
orchestrator/checks/diff_validation.py   (phần scope; phần test weakening ở bước 13)
config/policies.yaml (mục commands)
tests/policy/test_command_policy.py
tests/integration/test_post_exec_check.py
```

## Tiêu chí hoàn thành

- [ ] Case command trong §37 pass: `pytest; rm -rf .` → DENY (metachar), `curl …` → DENY, `git commit` / `git reset` → DENY, `python -c "open('../x','w')"` → DENY; `python script.py` ghi `../x` → không ảnh hưởng host (mount).
- [ ] Integration: Coder viết `conftest.py` ghi `app/other.py` (ngoài plan) rồi chạy `pytest` → post-exec check bắt được, `app/other.py` bị khôi phục, event `POLICY_VIOLATION`.
- [ ] Output lệnh dài bị cắt đúng giới hạn; lệnh treo bị kill theo timeout.

## Rủi ro / Ghi chú

- Allowlist là của **project**, do user sửa — không bao giờ lấy từ output của LLM.
- `conftest.py` / `pytest.ini` / `pyproject.toml` là cấu hình có thể chạy code → nếu nằm trong plan với `op=modify`, đánh dấu cho Reviewer.
