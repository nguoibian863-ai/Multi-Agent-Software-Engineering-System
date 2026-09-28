# 05 — Architect

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 2 — Planning + State (tuần 2) |
| Phụ thuộc | 04 |
| Mở khoá | 08 |
| Master plan | §8, §25 |
| Ước lượng | 1–1.5 ngày |

## Mục tiêu

Agent Architect nhận yêu cầu + context do runtime chuẩn bị, trả về `Plan` hợp lệ về schema.

## Việc cần làm

- [ ] `orchestrator/context/builder.py` — bản tối thiểu cho Architect
  - [ ] `project_tree`: liệt kê file (tôn trọng `.gitignore`, loại `read_denied`), giới hạn độ sâu / số dòng.
  - [ ] `relevant_files`: ripgrep theo từ khoá trong yêu cầu, lấy top N file, cắt theo token budget.
  - [ ] `project_summary`: đọc `memory/architecture.md` nếu có.
  - [ ] `policy_summary`: `user_ceiling` + `write_denied` dạng text (chỉ để thông báo, không để Architect ghi lại).
  - [ ] Mọi file đọc đều qua `read_denied` (tạm dùng glob match đơn giản; bước 10 thay bằng File Policy thật).
- [ ] `orchestrator/agents/architect.py`
  - [ ] Prompt: vai trò, quy tắc (path cụ thể, op đúng, mỗi AC có verify, ưu tiên test cho verify).
  - [ ] Gọi `LLMClient.generate_structured(agent="architect", schema=Plan)`.
  - [ ] Output sai schema → retry tối đa 2 lần, kèm thông báo lỗi Pydantic.
  - [ ] Nhận thêm `validator_errors` và `user_feedback` (cho replan ở bước 06/08).
- [ ] Lưu artifact: `runs/<task_id>/plan.v<N>.json` + `plan.v<N>.md`.

## Output

```text
orchestrator/context/builder.py
orchestrator/agents/architect.py
prompts/architect.md
tests/integration/test_architect_fake_llm.py
```

## Tiêu chí hoàn thành

- [ ] Test với fake LLM: output hợp lệ → trả `Plan`; output sai 1 lần → retry thành công; sai 3 lần → lỗi có cấu trúc.
- [ ] Chạy với model thật trên 10 Golden Tasks: ≥ 9/10 plan hợp lệ về schema.
- [ ] Context gửi cho Architect không chứa nội dung `.env` (test với fixture có `.env` giả).

## Rủi ro / Ghi chú

- Architect không cần OpenHands — chỉ là một LLM call.
- Theo dõi token context; nếu > 30k token cho fixture nhỏ thì builder đang gửi thừa.
