# 06 — Plan Validator

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 2 — Planning + State (tuần 2) |
| Phụ thuộc | 04 |
| Mở khoá | 08, 10 |
| Master plan | §11, §12 |
| Ước lượng | 1 ngày |

## Mục tiêu

Kiểm tra plan theo mọi rule §11 trước khi user thấy nó. Trả lỗi **có cấu trúc** để Architect sửa.

## Việc cần làm

- [ ] `orchestrator/policy/paths.py` — bản đầu
  - [ ] `normalize(path, workspace) -> RelPath | PathError`: thống nhất `/`, bỏ `.`/`..`, reject absolute, reject ra ngoài workspace.
  - [ ] `match_any(path, globs) -> bool` (case-insensitive trên Windows).
  - [ ] Bước 10 sẽ bổ sung symlink/junction/ADS/device name.
- [ ] `orchestrator/planning/validator.py`
  - [ ] `validate(plan, snapshot, workspace) -> ValidationResult(ok, errors: list[PlanError])`
  - [ ] `PlanError(code, location, message)`; codes: `PLAN_SCHEMA_INVALID`, `PLAN_INCOMPLETE`, `PLAN_SCOPE_CONFLICT`, `PLAN_COMMAND_REJECTED`.
  - [ ] Rules:
    - [ ] `steps` và `files` không rỗng
    - [ ] step id duy nhất; `depends_on` tồn tại và không có vòng (topo sort)
    - [ ] `step.files ⊆ plan.files`
    - [ ] mỗi path trong `plan.files`: không glob, normalize được, trong workspace
    - [ ] mỗi path match `user_ceiling`, không match `write_denied`
    - [ ] `create` → file chưa tồn tại; `modify`/`delete` → file đã tồn tại
    - [ ] mọi AC id được tham chiếu đều tồn tại; mỗi AC được ≥ 1 step đảm nhận
    - [ ] `proposed_commands`: tạm dùng allowlist trong `config/policies.yaml`; bước 11 thay bằng Command Policy
  - [ ] Thu thập **tất cả** lỗi (không dừng ở lỗi đầu) để Architect sửa một lần.
- [ ] `format_errors_for_llm(errors) -> str`.

## Output

```text
orchestrator/policy/paths.py
orchestrator/planning/validator.py
tests/unit/test_plan_validator.py     (table-driven, mỗi rule ≥ 1 case pass + 1 case fail)
```

## Tiêu chí hoàn thành

- [ ] Mỗi rule có test pass/fail.
- [ ] Ví dụ §12 cũ (plan rộng hơn ceiling) → `PLAN_SCOPE_CONFLICT`.
- [ ] Plan có `../x.py`, `C:\x.py`, `app/*.py` → bị từ chối với code đúng.
- [ ] Coverage `validator.py` ≥ 95%.

## Rủi ro / Ghi chú

- Validator là hàm thuần (không gọi LLM, không I/O ngoài `exists()`) → dễ test, chạy nhanh.
