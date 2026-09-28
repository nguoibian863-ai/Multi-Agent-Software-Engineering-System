# 20 — API + UI (Milestone M4)

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 6 — API + UI (tuần 6) |
| Phụ thuộc | 19 |
| Mở khoá | 22 |
| Master plan | §33, §34, §36 |
| Ước lượng | 5 ngày (API 1.5 · UI 3.5) |
| Phương án 5 tuần | Chỉ trang web read-only: danh sách task, events, diff |

## Mục tiêu

Mở toàn bộ chức năng CLI qua HTTP và dựng UI tập trung vào **Task Detail** và **Approvals**.

## Phần A — API (FastAPI)

- [ ] `apps/api/main.py` + routers trong `apps/api/routes/`.
- [ ] Endpoints §36: projects (+ `PATCH /projects/{id}/policy`), tasks (`run`, `stop`, `resume`, `discard`), `state`, `plan` (`approve`, `feedback`), `diff`, `checks`, `review`, `gate`, `events`, `approvals`.
- [ ] `GET /tasks/{id}/stream` — SSE, đẩy event mới từ bảng `events` (poll DB hoặc pub/sub nội bộ).
- [ ] Engine chạy trong background worker (asyncio task) — API chỉ phát lệnh, không chạy agent trong request.
- [ ] Chỉ bind `127.0.0.1`; token local đơn giản trong `.env` để tránh trang web khác gọi vào (CSRF / DNS rebinding).
- [ ] Response model Pydantic dùng chung với orchestrator; xuất OpenAPI → sinh TS client.

## Phần B — UI (Next.js + shadcn/ui)

- [ ] Layout §33: Tasks | Workflow | Artifacts + Live Events + banner approval.
- [ ] Màn hình (§34):
  - [ ] Dashboard — task đang chạy, cần duyệt, gần đây.
  - [ ] Projects — thêm project, sửa ceiling / denied / command allowlist / budget.
  - [ ] New Task.
  - [ ] **Task Detail** — sơ đồ workflow render từ `transitions.py` + state hiện tại, step progress, budget bar.
  - [ ] Plan Viewer — render `plan.md`, nút Approve / Feedback / Reject.
  - [ ] Approvals inbox — scope / command, hiển thị lý do + rủi ro.
  - [ ] Diff Viewer — theo file, highlight file có weakening flag.
  - [ ] Checks — baseline vs hiện tại.
  - [ ] Review / Gate / Report.
  - [ ] Logs / Tool calls — lọc theo agent, DENY.
  - [ ] Settings — models, budget mặc định.
- [ ] Realtime qua SSE; reconnect + lấy lại event bị lỡ theo `last_event_id`.

## Output

```text
apps/api/main.py, apps/api/routes/*.py
apps/web/app/**, apps/web/components/**, apps/web/lib/api.ts (generated)
tests/integration/test_api.py
apps/web/e2e/*.spec.ts (Playwright, tối thiểu)
```

## Tiêu chí hoàn thành

- [ ] Toàn bộ luồng GT-001 làm được chỉ qua UI: tạo task → duyệt plan → theo dõi live → xem diff / checks / review → report.
- [ ] Scope request hiện banner trong < 2s; approve từ UI → Coder tiếp tục.
- [ ] Reload trang giữa run → state và events hiển thị đầy đủ.
- [ ] API từ chối request không có token local.
- [ ] **M4** đạt.

## Rủi ro / Ghi chú

- Nếu Next.js nặng cho tool local, Vite + React là lựa chọn thay thế — quyết định ghi ADR trước khi bắt đầu phần B.
