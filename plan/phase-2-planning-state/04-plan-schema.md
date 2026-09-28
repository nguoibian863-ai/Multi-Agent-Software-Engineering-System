# 04 — Plan schema v2 + render plan.md

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 2 — Planning + State (tuần 2) |
| Phụ thuộc | 02 |
| Mở khoá | 05, 06 |
| Master plan | §9, §10 |
| Ước lượng | 1 ngày |

## Mục tiêu

Định nghĩa contract giữa Architect và runtime bằng Pydantic, tách rõ **cái LLM được đề xuất** (`Plan`) và **giới hạn do runtime đặt** (`PolicySnapshot`).

## Việc cần làm

- [ ] `orchestrator/planning/schema.py`
  - [ ] `PlanFile(path: str, op: Literal["create","modify","delete"])`
  - [ ] `Verify(type: Literal["test","review","test+review"], ref: str | None)`
  - [ ] `AcceptanceCriterion(id, text, verify)`
  - [ ] `Step(id, title, depends_on: list[str], files: list[str], acceptance: list[str])`
  - [ ] `Plan(schema_version: Literal["2.0"], task_id, goal, files, steps, acceptance_criteria, proposed_commands)`
  - [ ] Tất cả model: `model_config = ConfigDict(extra="forbid", frozen=True)`.
- [ ] `orchestrator/policy/snapshot.py`
  - [ ] `Limits(max_replan, max_check_retry_per_step, max_rework, max_scope_requests)`
  - [ ] `Budget(max_tokens, max_cost_usd, max_wall_clock_min, max_tool_calls)`
  - [ ] `PolicySnapshot(policy_version, user_ceiling, write_denied, read_denied, limits, budget)`
  - [ ] Hàm `build_snapshot(project_config) -> PolicySnapshot` (đọc `config/policies.yaml` + override của project).
- [ ] Xuất JSON Schema: `schemas/plan.v2.json` (script `scripts/export_schemas.py`), dùng để đưa vào prompt / structured output.
- [ ] `orchestrator/planning/render.py`: `render_plan_md(plan) -> str` bằng Jinja2 template `templates/plan.md.j2` (bố cục như §9).
- [ ] Ví dụ chuẩn `tests/data/plan_jwt.json` (lấy từ §10).

## Output

```text
orchestrator/planning/schema.py
orchestrator/policy/snapshot.py
orchestrator/planning/render.py
templates/plan.md.j2
schemas/plan.v2.json
config/policies.yaml           (default ceiling / denied / limits / budget)
tests/unit/test_plan_schema.py
tests/unit/test_render.py
```

## Tiêu chí hoàn thành

- [ ] `plan_jwt.json` parse thành công.
- [ ] Plan có field `user_ceiling` / `denied` / `max_rework` / `budget` → `ValidationError`.
- [ ] Snapshot test cho `render_plan_md` (so với file mong đợi).
- [ ] `schemas/plan.v2.json` được sinh tự động và khớp với model (test so sánh).

## Rủi ro / Ghi chú

- Giữ schema phẳng, ít optional — structured output của LLM ổn định hơn.
- Tăng `schema_version` khi đổi schema; validator từ chối version lạ.
