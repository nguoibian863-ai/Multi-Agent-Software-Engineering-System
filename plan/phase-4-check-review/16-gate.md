# 16 — Gate + Final report

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 4 — Check + Review (tuần 4) |
| Phụ thuộc | 15 |
| Mở khoá | 19 |
| Master plan | §19, §31 (final report) |
| Ước lượng | 1 ngày |

## Mục tiêu

Xác nhận task **thực sự hoàn thành**: deterministic gate trước, LLM gate sau (chỉ có thể chặn thêm). Sinh final report cho user.

## Việc cần làm

- [ ] `orchestrator/agents/gate.py`
  - [ ] `deterministic_gate(run) -> GateResult` đúng như §19.1: policy violations, out-of-scope, incomplete steps, check delta, weakening chưa xử lý, AC có `verify=test` phải có test pass theo `ref`, effective review verdict, high severity.
  - [ ] `llm_gate(run) -> {complete, missing[]}` — chỉ chạy khi deterministic PASS; context: requirement, plan, diff summary, review.
  - [ ] Kết hợp: deterministic FAIL → FAIL; LLM `complete=false` → FAIL với `missing` làm feedback.
- [ ] Handler `GATING`: PASS → `DONE`; FAIL + còn `rework` → `REWORK`; hết → `BLOCKED`.
- [ ] Lưu `gate_results`.
- [ ] Final report `runs/<task>/report.md` (template `templates/report.md.j2`):
  - tóm tắt yêu cầu, plan version cuối
  - danh sách AC + bằng chứng (test nodeid / review evidence)
  - diff stat, check delta so với baseline
  - review cuối, số rework, token / cost / thời gian
  - lệnh gợi ý: `git diff <base>..mase/<task>`, `git merge mase/<task>`, `mase discard <task>`
- [ ] CLI: `mase report TASK`.

## Output

```text
orchestrator/agents/gate.py
prompts/gate.md
templates/report.md.j2
tests/unit/test_deterministic_gate.py
tests/integration/test_gate_flow.py
```

## Tiêu chí hoàn thành

- [ ] Mỗi nhánh FAIL của deterministic gate có unit test.
- [ ] LLM gate trả `complete=true` nhưng deterministic FAIL → vẫn FAIL (test).
- [ ] E2E GT-001 đến `DONE` với report đầy đủ.
- [ ] Chạy 10 Golden Tasks: đo **false PASS rate** (Gate PASS nhưng hidden test fail) lần đầu.

## Rủi ro / Ghi chú

- Phương án 5 tuần: bỏ `llm_gate`, giữ deterministic.
