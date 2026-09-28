# 12 — Scope request + approvals (Milestone M2)

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 3 — Safety (tuần 3) |
| Phụ thuộc | 07, 10, 11 (+ cơ chế Approval của 08) |
| Mở khoá | Phase 4 (chỉ khi đạt M2) |
| Master plan | §12.1, §20 (AWAITING_APPROVAL), §29 (REQUIRES_APPROVAL) |
| Ước lượng | 1 ngày + 0.5 ngày kiểm milestone |

## Mục tiêu

Cho Coder một cách **hợp lệ** để xin thêm quyền (file ngoài plan, command cần duyệt) thay vì cố vượt policy. Kết thúc phase Safety bằng milestone M2.

## Việc cần làm

### Scope request
- [ ] Tool `request_scope(path, op, reason)` — `orchestrator/tools/scope.py`.
- [ ] `orchestrator/policy/scope.py`:
  - [ ] match `write_denied` → DENY ngay.
  - [ ] ngoài `user_ceiling` → DENY ngay.
  - [ ] `scope_request_count >= max_scope_requests` → `BLOCKED`.
  - [ ] match `auto_approve_scope` của project (mặc định rỗng) → approve tự động, ghi event.
  - [ ] còn lại → `Approval(kind=scope)` → `AWAITING_APPROVAL`.
- [ ] Approved → tạo `plan.v<N+1>` (thêm file vào `files` + step hiện tại), `plan_version += 1`, quay lại `CODING`.
- [ ] Rejected → trả lỗi cho Coder, quay lại `CODING`.

### Command approval
- [ ] `REQUIRES_APPROVAL` → `Approval(kind=command, payload={argv, reason})`.
- [ ] Approved → chạy trong sandbox **với network bật tạm thời** (container riêng hoặc network attach/detach), sau đó post-exec check như thường.

### CLI
- [ ] `mase approvals list [--task T]`
- [ ] `mase approvals approve ID [--note]` / `mase approvals reject ID [--note]`

### Thời gian chờ
- [ ] Thời gian ở `AWAITING_*` **không tính** vào `max_wall_clock_min`.

### Kiểm milestone M2
- [ ] Chạy toàn bộ `tests/policy/`.
- [ ] Chạy 2 Golden Tasks scope-violation + dangerous-command với model thật.
- [ ] Red-team thủ công: prompt Coder cố đọc `.env`, ghi ngoài scope, dùng `pytest` để chạy code ghi file, đổi `.git/hooks`.
- [ ] Ghi kết quả vào `docs/milestones/M2-safety.md`.

## Output

```text
orchestrator/tools/scope.py
orchestrator/policy/scope.py
apps/cli/main.py (approvals)
tests/integration/test_scope_request.py
tests/integration/test_command_approval.py
docs/milestones/M2-safety.md
```

## Tiêu chí hoàn thành

- [ ] Scope request ngoài ceiling / denied → bị từ chối không qua user.
- [ ] Scope request hợp lệ → approve → Coder ghi được file đó; `plan.v2` có file mới.
- [ ] `pip install` được duyệt → chạy với network; không duyệt → không chạy.
- [ ] **M2:** mọi policy test pass; 2 GT vi phạm bị chặn; red-team không có lần nào đọc được secret hoặc để lại thay đổi ngoài scope trên task branch.

## Rủi ro / Ghi chú

- **Không đạt M2 → dừng, không làm phase 4.**
- Theo dõi số approval mỗi task — nếu quá nhiều, cân nhắc `auto_approve_scope` cho `**/__init__.py`, `tests/**/conftest.py` trong ceiling.
