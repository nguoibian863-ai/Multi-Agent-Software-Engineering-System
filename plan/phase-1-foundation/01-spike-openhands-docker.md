# 01 — Spike OpenHands + Docker trên Windows

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 1 — Foundation (tuần 1) |
| Phụ thuộc | — |
| Mở khoá | 03 |
| Master plan | §2, §2.1, §29.1, §42, §46 (R1, R2) |
| Ước lượng | 2–3 ngày |

## Mục tiêu

Trả lời câu hỏi rủi ro nhất của dự án: **có chèn được Policy Engine của ta vào giữa Coder (OpenHands) và việc thực thi tool không, và sandbox có chạy ổn trên Windows không.** Kết quả là quyết định go/no-go, không phải code production.

## Việc cần làm

### Môi trường
- [ ] Cài Docker Desktop với WSL2 backend; xác nhận `docker run --rm hello-world`.
- [ ] Python 3.12 + `uv`; tạo venv riêng cho spike trong `spikes/openhands/`.
- [ ] Cài OpenHands Software Agent SDK (+ Agent Server nếu cần); ghi lại version chính xác.
- [ ] Một repo fixture nhỏ (FastAPI 1 file + 1 test) để agent thao tác.

### Chạy agent cơ bản
- [ ] Chạy một Coder agent của OpenHands với Docker workspace, yêu cầu "thêm endpoint /health".
- [ ] Xác nhận agent đọc/sửa được file trong workspace và chạy được `pytest`.

### Thử điểm hook policy (theo thứ tự ưu tiên)
- [ ] **H1 – Custom tools:** thay file editor / terminal mặc định bằng tool của ta, bên trong gọi `policy.check(...)` trước khi thực thi.
- [ ] **H2 – Security analyzer / confirmation policy** của SDK: có chặn được *trước khi* tool chạy không, có nhận đủ thông tin (path, command) không.
- [ ] **H3 – Wrapper ở Agent Server:** chặn ở tầng request/event.
- [ ] Với mỗi hook: viết một policy giả — deny mọi path chứa `..`, deny command bắt đầu bằng `curl`.

### Kiểm tiêu chí G1–G5
- [ ] **G1** Agent được yêu cầu ghi `../outside.txt` → bị chặn trước khi ghi, có event DENY.
- [ ] **G2** Agent chạy `curl https://example.com` → bị chặn bởi policy.
- [ ] **G3** Trong container: `ls /` không thấy ổ host; đọc file ngoài mount thất bại; `python -c "import urllib.request; urllib.request.urlopen('https://example.com')"` thất bại (network none).
- [ ] **G4** Chạy end-to-end trên máy Windows của team; đo thời gian một lệnh `pytest` trong container với bind mount từ NTFS và từ filesystem WSL.
- [ ] **G5** Lấy được event stream: tool call, arguments, kết quả, token usage.

### Quyết định
- [ ] Viết `docs/decisions/ADR-001-agent-runtime.md`: hook được chọn, lý do, giới hạn, version SDK, đoạn code mẫu.
- [ ] Nếu G1 hoặc G2 fail với cả H1–H3 → **No-go**: ADR ghi quyết định dùng agent loop tự viết (kiểu mini-SWE-agent) + Docker, và cập nhật bước 03.
- [ ] Ghi quyết định vị trí đặt worktree (NTFS hay WSL fs) dựa trên số đo G4.

## Output

```text
spikes/openhands/            # code thử, không dùng lại trực tiếp
docs/decisions/ADR-001-agent-runtime.md
docs/decisions/ADR-002-worktree-location.md
```

## Tiêu chí hoàn thành

- [ ] G1–G5 mỗi tiêu chí có bằng chứng (log / event / output lệnh) đính kèm trong ADR-001.
- [ ] ADR-001 có kết luận rõ **Go (hook Hx)** hoặc **No-go (fallback)**.
- [ ] Team khác đọc ADR và tái hiện được spike trong < 30 phút.

## Rủi ro / Ghi chú

- Không tối ưu, không viết test cho code spike — mục tiêu là học nhanh.
- Nếu H2 chỉ cho phép "hỏi xác nhận" mà không cho phép "từ chối có lý do trả về agent" → vẫn dùng được nhưng ghi rõ trong ADR.
- Bind mount từ NTFS vào container Linux thường chậm; nếu `pytest` chậm > 3× thì đặt worktree trong WSL filesystem.
