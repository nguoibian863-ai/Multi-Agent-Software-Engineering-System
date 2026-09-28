# 19 — Golden Tasks (25) + benchmark → MVP CLI (Milestone M3)

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 5 — Reliability (tuần 5) |
| Phụ thuộc | 16, 17 (18 nếu có) |
| Mở khoá | 20, 21 |
| Master plan | §38, §39 |
| Ước lượng | 1.5–2 ngày |

## Mục tiêu

Đủ 25 Golden Tasks, chạy benchmark đầu tiên trên hệ thống thật, xác nhận **MVP CLI** đạt Definition of Done (trừ phần UI).

## Việc cần làm

### Bổ sung task
- [ ] Thêm 15 task để đủ phân bố §38: 5 simple edit, 5 feature, 5 bug fix, 3 test generation, 3 refactor, 2 scope violation, 2 dangerous command / prompt injection.
- [ ] Có thể thêm fixture thứ hai (vd `py-cli-tool`) để tránh overfit một repo.
- [ ] Mỗi task có hidden tests + `expect`.

### Runner
- [ ] `benchmarks/run.py`:
  - [ ] Mỗi task dùng một bản fixture riêng (một project = một run active).
  - [ ] Chế độ auto-approve plan / scope **chỉ dùng trong benchmark** (ghi rõ trong report), có tuỳ chọn dừng chờ duyệt.
  - [ ] Chạy song song N task (N cấu hình, mặc định 2).
  - [ ] Sau run: chạy hidden tests trên task branch; kiểm `must_not_touch`.
- [ ] Metrics §38: completion rate (theo hidden tests), false PASS rate, policy violation attempts / blocked, reviewer catch rate, retries / reworks, tokens / cost / wall-clock, tool calls, approvals per task.
- [ ] Report `benchmarks/reports/<date>.md` + JSON thô.

### Kiểm DoD MVP (§39, trừ UI)
- [ ] Đi qua từng dòng §39, ghi bằng chứng vào `docs/milestones/M3-mvp.md`.

## Output

```text
benchmarks/golden_tasks/GT-011..GT-025.yaml (+ hidden/)
benchmarks/run.py (hoàn thiện)
benchmarks/reports/<date>.md
docs/milestones/M3-mvp.md
```

## Tiêu chí hoàn thành

- [ ] 25 task chạy hết không crash runner.
- [ ] 4 task vi phạm: 100% bị chặn / BLOCKED, không để lại thay đổi cấm.
- [ ] False PASS rate được đo và ghi; đặt ngưỡng mục tiêu cho các lần sau (vd ≤ 5%).
- [ ] **M3:** mọi mục §39 (trừ UI) có bằng chứng.

## Rủi ro / Ghi chú

- Chạy benchmark sau mỗi thay đổi prompt / model để phát hiện regression.
- Auto-approve trong benchmark **không** được bật mặc định ở chế độ thường.
