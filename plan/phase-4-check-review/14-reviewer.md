# 14 — Reviewer

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 4 — Check + Review (tuần 4) |
| Phụ thuộc | 13 |
| Mở khoá | 15 |
| Master plan | §17, §25 |
| Ước lượng | 1 ngày |

## Mục tiêu

Reviewer độc lập, read-only, đánh giá toàn bộ thay đổi so với yêu cầu + plan, trả kết quả có cấu trúc; runtime kiểm tra nhất quán verdict.

## Việc cần làm

- [ ] Schema `ReviewResult` (`orchestrator/agents/reviewer.py` hoặc `planning/schema.py`):
  - `verdict: PASS | FAIL`
  - `acceptance: list[{id, satisfied, evidence}]` — phải có đủ mọi AC trong plan
  - `issues: list[{id, severity, file, line, category, description, required_fix}]`
- [ ] Context Builder cho Reviewer: requirement gốc, plan đã duyệt, **full diff `base_commit..HEAD`**, check results + baseline delta, weakening flags, policy events (DENY, scope requests), nội dung đầy đủ của file thay đổi.
- [ ] Reviewer **không** nhận hội thoại / reasoning của Coder.
- [ ] Model + prompt riêng (`config/models.yaml: reviewer`); khuyến nghị khác model với Coder.
- [ ] `effective_verdict(review)`:
  - PASS nhưng có issue `high` → FAIL
  - PASS nhưng có AC `satisfied=false` → FAIL
  - Thiếu AC trong `acceptance` → coi là output sai schema → retry
- [ ] Lưu `reviews` table + `runs/<task>/review.<n>.json`.

## Output

```text
orchestrator/agents/reviewer.py
prompts/reviewer.md
tests/unit/test_review_consistency.py
tests/integration/test_reviewer_fake_llm.py
```

## Tiêu chí hoàn thành

- [ ] Test consistency: 3 case trên → đúng effective verdict.
- [ ] Với model thật: GT có bug cố ý (vd JWT không check `exp`) → Reviewer FAIL với issue đúng category.
- [ ] Weakening flag từ bước 13 xuất hiện trong input và Reviewer phải phản hồi (issue hoặc giải thích trong evidence).

## Rủi ro / Ghi chú

- Diff lớn → cắt theo file, ưu tiên file trong plan; ghi rõ phần nào bị cắt để Reviewer biết.
