# 21 — Context optimization + multi-model

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 7 — Context + Models (tuần 7) |
| Phụ thuộc | 19 (song song với 20) |
| Mở khoá | 22 |
| Master plan | §24, §25, §26, §27 |
| Ước lượng | 4–5 ngày |
| Phương án 5 tuần | **Bỏ** |

## Mục tiêu

Giảm token / chi phí và tăng chất lượng bằng context chính xác hơn; cho phép cấu hình model theo agent kèm fallback. Mọi thay đổi phải được xác nhận bằng benchmark (không regression).

## Việc cần làm

### Repo map
- [ ] `orchestrator/context/repo_map.py`: tree-sitter (hoặc ctags) → symbols (class, function, signature) + quan hệ import.
- [ ] Cache theo commit sha.
- [ ] Architect nhận repo map thay vì nhiều nội dung file.
- [ ] Coder nhận: file của step + file import/được import bởi file của step (theo repo map), cắt theo token budget.

### Tóm tắt output dài
- [ ] Tool output (pytest log, lint) > N token → tóm tắt bằng model local / rẻ trước khi đưa vào context; giữ nguyên dòng lỗi quan trọng (traceback cuối, nodeid fail).

### Multi-model
- [ ] `config/models.yaml` per agent + `fallback` (§26).
- [ ] `LLMClient`: fallback khi provider lỗi / timeout / rate-limit; ghi event `MODEL_FALLBACK`.
- [ ] Cho phép Reviewer dùng provider khác Coder; đo reviewer catch rate cho từng cấu hình.

### Project memory
- [ ] Context Builder đọc `memory/*.md`.
- [ ] Sau `DONE`: agent đề xuất cập nhật `decisions.md` / `commands.md` → Approval(kind=memory) → user duyệt mới ghi.

### Token / cost
- [ ] Bảng `model_usage` tổng hợp theo task / agent / model; hiển thị trên UI (bước 20).

## Output

```text
orchestrator/context/repo_map.py
orchestrator/context/builder.py (nâng cấp)
orchestrator/context/summarize.py
orchestrator/llm/client.py (fallback)
benchmarks/reports/<date>-context.md
```

## Tiêu chí hoàn thành

- [ ] Benchmark 25 task: token trung bình giảm ≥ 30% so với report M3, completion rate **không giảm**, false PASS rate không tăng.
- [ ] Tắt provider chính (key sai) → fallback hoạt động, task vẫn hoàn thành.
- [ ] Memory chỉ bị ghi sau khi user duyệt.

## Rủi ro / Ghi chú

- Context quá ít → Coder đoán sai API nội bộ. Luôn giữ tool `search_code` / `read_file` để Coder tự lấy thêm (qua read policy).
