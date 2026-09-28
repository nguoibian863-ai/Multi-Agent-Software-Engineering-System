# 10 — File Policy (read + write)

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 3 — Safety (tuần 3) |
| Phụ thuộc | 03, 06 |
| Mở khoá | 11, 12 |
| Master plan | §12, §13, §37 (policy tests) |
| Ước lượng | 1.5–2 ngày |

## Mục tiêu

Thay policy stub bằng File Policy thật: mọi `read_file` / `write_file` / `create_file` / `apply_patch` / `delete` của Coder được kiểm theo `read_denied`, `write_denied`, `plan_allowed` (+ op) và `user_ceiling`.

## Việc cần làm

- [ ] `orchestrator/policy/paths.py` — hoàn thiện
  - [ ] Reject: NUL byte, ADS (`:` ngoài drive letter), device names (`CON`, `PRN`, `AUX`, `NUL`, `COM1-9`, `LPT1-9`, kể cả có đuôi `aux.py`), `\\?\`, `\\.\`, UNC.
  - [ ] Case-fold khi so sánh trên Windows.
  - [ ] Resolve symlink / junction / reparse point (`os.path.realpath` + kiểm `os.lstat` / `is_junction`) rồi kiểm lại workspace boundary **sau** resolve.
  - [ ] Phát hiện 8.3 short name (`~1`) → reject hoặc expand bằng `GetLongPathNameW`.
- [ ] `orchestrator/policy/file_policy.py`
  - [ ] `check_read(path) -> Decision`
  - [ ] `check_write(path, op, plan_allowed, snapshot) -> Decision`
  - [ ] `Decision(allow: bool, code: str, message: str, suggest_scope_request: bool)`
  - [ ] Codes: `OUTSIDE_WORKSPACE`, `READ_DENIED`, `WRITE_DENIED`, `NOT_IN_PLAN`, `OP_MISMATCH`, `OUTSIDE_CEILING`, `INVALID_PATH`.
  - [ ] `NOT_IN_PLAN` nhưng trong ceiling → `suggest_scope_request = true`.
- [ ] Thay `policy/hook.py` stub → gọi File Policy; mọi quyết định ghi event.
- [ ] Context Builder (bước 05) chuyển sang dùng `check_read`.
- [ ] Plan Validator (bước 06) dùng chung `paths.py` mới.

## Output

```text
orchestrator/policy/paths.py (hoàn thiện)
orchestrator/policy/file_policy.py
tests/policy/test_paths.py
tests/policy/test_file_policy.py
tests/policy/test_windows_paths.py      (skip nếu không phải Windows)
```

## Tiêu chí hoàn thành

- [ ] Toàn bộ case file trong §37 "Policy tests" pass:
  `../outside.txt`, `C:\Windows\x`, `\\?\C:\x`, `\\server\share\x`, `.env` / `.ENV` / `.Env` (read + write), `.env::$DATA`, `app.py:hidden`, `SECRET~1\key`, `CON`, `aux.py`, symlink ra ngoài, junction ra ngoài, `.git/config`, `.git/hooks/x`, file trong plan → ALLOW, file ngoài plan trong ceiling → DENY + gợi ý scope request.
- [ ] Property-based test (hypothesis): path sinh ngẫu nhiên có `..` / separator lẫn lộn không bao giờ được ALLOW nếu resolve ra ngoài workspace.
- [ ] Test Windows chạy thật trên máy Windows (junction, ADS, short name).
- [ ] Pyright strict pass cho `orchestrator/policy/**`.

## Rủi ro / Ghi chú

- Coder chạy trong container Linux nhưng tool file của runtime có thể chạy host-side (tuỳ ADR-001) — policy phải đúng cho **cả hai** phía.
- File Policy là lớp 2; lớp 1 là mount của sandbox, lớp 3 là post-exec diff check (bước 11).
