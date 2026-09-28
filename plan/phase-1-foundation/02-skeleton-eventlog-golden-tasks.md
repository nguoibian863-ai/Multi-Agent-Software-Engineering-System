# 02 — Skeleton, event log, Golden Tasks (10)

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 1 — Foundation (tuần 1) |
| Phụ thuộc | — (song song với 01) |
| Mở khoá | 03, 04, 07 |
| Master plan | §4, §26, §32, §35, §38 |
| Ước lượng | 2–3 ngày |

## Mục tiêu

Dựng khung dự án, cơ chế ghi event và **bộ đo** (Golden Tasks + grader) ngay từ đầu, để mọi bước sau đều đo được.

## Việc cần làm

### Skeleton
- [ ] Tạo cấu trúc thư mục theo §35 (thư mục rỗng + `__init__.py`).
- [ ] `pyproject.toml` (uv): python 3.12, ruff, pyright (strict cho `orchestrator/policy/**`), pytest, pydantic, pydantic-settings, sqlmodel, litellm, typer.
- [ ] `config/` loader: YAML + `.env` qua `pydantic-settings` → `orchestrator/config.py`.
- [ ] CI local: `uv run ruff check . && uv run pyright && uv run pytest`.

### Event log
- [ ] `orchestrator/state/events.py`: model `Event` (các field §32) + `EventLogger` ghi JSONL vào `logs/<run_id>.jsonl`.
- [ ] Không ghi nội dung file hay secret vào event — chỉ `args_summary`.
- [ ] Unit test: ghi/đọc lại event, validate schema.

### LLM client tối thiểu
- [ ] `orchestrator/llm/client.py`: `LLMClient.generate_structured(agent, messages, schema)` qua LiteLLM.
- [ ] Đọc model theo agent từ `config/models.yaml`.
- [ ] Ghi token in/out + cost vào event.
- [ ] `tests/fakes/fake_llm.py`: LLM giả trả response theo kịch bản — dùng cho mọi test sau này.

### Golden Tasks + grader
- [ ] Fixture `benchmarks/fixtures/py-fastapi-basic/`: FastAPI app nhỏ, vài test, **cố ý có 2–3 lỗi lint sẵn** (để test baseline ở bước 13). Là git repo riêng (script tạo lại từ template).
- [ ] Định dạng task:

  ```yaml
  # benchmarks/golden_tasks/GT-001.yaml
  id: GT-001
  type: simple_edit
  fixture: py-fastapi-basic
  request: "Add GET /health returning {\"status\": \"ok\"}"
  hidden_tests: hidden/test_gt001.py      # Coder không bao giờ thấy
  expect:
    outcome: DONE                          # DONE | BLOCKED (cho task vi phạm)
    must_not_touch: [".env", "secrets/**"]
  ```

- [ ] 10 task đầu: 3 simple edit, 3 feature, 2 bug fix, 1 scope violation, 1 dangerous command / prompt injection trong repo.
- [ ] `benchmarks/run.py`: copy fixture → gọi hệ thống (hiện là stub) → chạy hidden tests → ghi `benchmarks/results/<timestamp>.json`.

## Output

```text
pyproject.toml
orchestrator/config.py
orchestrator/state/events.py
orchestrator/llm/client.py
tests/fakes/fake_llm.py
benchmarks/fixtures/py-fastapi-basic/
benchmarks/golden_tasks/GT-001..GT-010.yaml (+ hidden/)
benchmarks/run.py
```

## Tiêu chí hoàn thành

- [ ] `ruff`, `pyright`, `pytest` xanh trên skeleton.
- [ ] `EventLogger` ghi được event hợp lệ; test pass.
- [ ] Gọi thật được 1 model qua `LLMClient` với structured output; token được ghi.
- [ ] `benchmarks/run.py` chạy với stub "không làm gì" → báo đúng 0/8 task thường pass, 2 task vi phạm được đánh dấu chưa kiểm.

## Rủi ro / Ghi chú

- Hidden tests phải nằm ngoài fixture được mount vào sandbox.
- Task scope-violation nên đòi sửa file ngoài `user_ceiling` mặc định (vd `infra/deploy.yaml`) → kỳ vọng `BLOCKED` hoặc bị từ chối ở validator.
