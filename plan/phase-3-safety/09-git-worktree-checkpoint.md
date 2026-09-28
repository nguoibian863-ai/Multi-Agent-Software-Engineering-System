# 09 — Git worktree + checkpoint + revert

Trạng thái: ☐ Chưa làm

| | |
|---|---|
| Phase | 3 — Safety (tuần 3) |
| Phụ thuộc | 03, 07 |
| Mở khoá | 11, 13, 17 |
| Master plan | §30, §31 |
| Ước lượng | 1.5 ngày |

## Mục tiêu

Mỗi task chạy trên một **git worktree + branch riêng**. Runtime (không phải agent) sở hữu mọi thao tác git ghi: tạo worktree, checkpoint commit, revert, discard.

## Việc cần làm

- [ ] `orchestrator/runtime/git_ops.py` (chạy trên host, gọi Git CLI)
  - [ ] `ensure_repo(project)`: không phải git repo → lỗi hướng dẫn `git init` + commit đầu.
  - [ ] Cảnh báo nếu working tree của user có thay đổi chưa commit (chúng **không** có trong worktree).
  - [ ] `create_worktree(project, task_id, base_ref) -> Worktree(path, branch, base_commit)`
    - branch `mase/<task_id>`, vị trí theo ADR-002.
    - bật `core.longpaths=true` trên Windows.
  - [ ] `checkpoint(worktree, step_id, message) -> sha` — author `mase-bot <mase@local>`, không chạy hook của user (`-c core.hooksPath=/dev/null` hoặc tương đương).
  - [ ] `changed_files(worktree, since_sha) -> list[Change]` (tracked + untracked).
  - [ ] `restore_paths(worktree, sha, paths)` — khôi phục **chỉ** các path chỉ định (tracked: `git checkout <sha> -- p`; untracked: xoá).
  - [ ] `revert_to(worktree, sha)` — `reset --hard` + `clean -fd` **chỉ trong worktree**; assert cwd là worktree trước khi chạy.
  - [ ] `diff(worktree, base, head) -> str`, `diff_stat(...)`.
  - [ ] `discard(project, task_id)` — `worktree remove` + `branch -D`; chỉ gọi từ lệnh user.
- [ ] Handler `PREPARING`: tạo worktree, lưu `base_commit`, `task_branch`, `worktree_path` vào state.
- [ ] Sandbox mount worktree thay vì bản copy (xoá cơ chế copy ở bước 03).
- [ ] Git **read** tools cho Coder (`git_diff`, `git_status`, `git_log`) chạy **host-side** bởi runtime — trong container file `.git` của worktree trỏ tới đường dẫn host không được mount, nên git không chạy được trong sandbox (đây là điều mong muốn).
- [ ] CLI: `mase discard TASK`.

## Output

```text
orchestrator/runtime/git_ops.py
orchestrator/tools/git_read.py
tests/integration/test_git_ops.py
```

## Tiêu chí hoàn thành

- [ ] Test trên repo tạm: create → sửa file → checkpoint → sửa tiếp → `restore_paths` / `revert_to` đúng.
- [ ] Sau cả run (kể cả FAILED): branch và working tree hiện tại của user **không đổi** (so `git status` + HEAD trước/sau).
- [ ] `revert_to` từ chối chạy nếu path không phải worktree của task.
- [ ] `discard` dọn sạch worktree + branch.
- [ ] GT-001 chạy lại qua worktree, hidden test pass trên task branch.

## Rủi ro / Ghi chú

- Git hook của user (pre-commit…) có thể chạy code tùy ý trên host khi runtime commit → phải tắt hook cho commit của runtime.
- Không bao giờ `push`, không `merge` vào branch của user.
