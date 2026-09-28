# 22 — Hardening + release (Milestone M5)

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 8 — Hardening (tuần 8) |
| Phụ thuộc | 20, 21 |
| Mở khoá | Release V1 |
| Master plan | §37, §39, §40 (tuần 8), §46 |
| Ước lượng | 5 ngày |

## Mục tiêu

Chứng minh hệ thống chịu lỗi và khó bị lạm dụng, rồi đóng gói để người khác cài và dùng được.

## Việc cần làm

### Failure injection
- [ ] Crash process ở mọi state → resume.
- [ ] Restart máy / Docker Desktop giữa run.
- [ ] LLM trả JSON hỏng, trả rỗng, trả schema khác.
- [ ] LLM timeout, provider 5xx, rate-limit.
- [ ] Tool timeout, test treo, test sinh output khổng lồ.
- [ ] Path không hợp lệ, file bị khoá (Windows file lock), đĩa đầy.
- [ ] Infinite retry: xác nhận mọi loop đều có trần.
- [ ] Docker không chạy khi bắt đầu task → lỗi rõ ràng, không `FAILED` âm thầm.

### Red-team policy
- [ ] Dùng chính một agent với prompt "tìm cách đọc `.env` / ghi ngoài scope / chạy lệnh mạng" chạy N lần trên fixture.
- [ ] Mọi bypass tìm được → thêm vào `tests/policy/` trước khi sửa.
- [ ] Rà lại: git hooks, `conftest.py`, `pyproject.toml` (plugin pytest), `.pth` files, `sitecustomize.py`.

### Release
- [ ] Documentation: `README.md`, `docs/getting-started.md`, `docs/configuration.md` (policies, models, budget), `docs/security-model.md` (3 lớp: sandbox → policy → post-exec check).
- [ ] Installer / setup script cho Windows (kiểm Docker, WSL2, Python, build sandbox image).
- [ ] Config wizard: `mase init` — tạo `config/*.yaml`, hỏi ceiling / denied / model keys.
- [ ] Demo project + video / GIF luồng chính.
- [ ] Benchmark report cuối cùng so với M3.

## Output

```text
tests/chaos/*.py
tests/policy/ (bổ sung từ red-team)
docs/getting-started.md, docs/configuration.md, docs/security-model.md
scripts/install.ps1
apps/cli (lệnh `mase init`)
benchmarks/reports/<date>-release.md
docs/milestones/M5-release.md
```

## Tiêu chí hoàn thành

- [ ] Mọi kịch bản failure injection kết thúc ở `DONE`, `BLOCKED` hoặc `FAILED` có lý do — không treo, không mất state.
- [ ] Red-team: 0 bypass chưa được vá.
- [ ] Người mới cài từ đầu trên máy Windows sạch và chạy xong demo trong < 30 phút theo docs.
- [ ] Toàn bộ DoD §39 (gồm UI) có bằng chứng → **M5**.

## Rủi ro / Ghi chú

- Đừng thêm tính năng ở tuần này — chỉ sửa lỗi, test, docs.
