# s12: Worktree + Task Isolation

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > [ s12 ]`

> *"Mỗi người làm việc trong thư mục riêng của mình, không có sự can thiệp (no interference)"* -- các task quản lý các mục tiêu (goals), các worktree quản lý các thư mục (directories), được liên kết (bound) bởi ID.

## Vấn đề

Cho đến s11, các agent có thể tự chủ (autonomously) claim và hoàn thành các task. Nhưng mỗi task chạy trong một thư mục dùng chung (shared directory). Hai agent cùng thực hiện quá trình refactoring cho các module khác nhau vào cùng một thời điểm sẽ dẫn đến sự va chạm (collide): agent A sửa `config.py`, agent B sửa `config.py`, các thay đổi unstaged bị trộn lẫn, và cả hai đều không thể quay lui (roll back) một cách gọn gàng được.

Bảng task theo dõi *cần làm gì* nhưng không có ý kiến gì về việc *làm điều đó ở đâu*. Cách khắc phục (The fix): cung cấp cho mỗi task một thư mục git worktree riêng biệt của nó. Các task quản lý các mục tiêu (goals), các worktree quản lý execution context. Liên kết chúng bằng task ID.

## Giải pháp

```
Control plane (.tasks/)             Execution plane (.worktrees/)
+------------------+                +------------------------+
| task_1.json      |                | auth-refactor/         |
|   status: in_progress  <------>   branch: wt/auth-refactor
|   worktree: "auth-refactor"   |   task_id: 1             |
+------------------+                +------------------------+
| task_2.json      |                | ui-login/              |
|   status: pending    <------>     branch: wt/ui-login
|   worktree: "ui-login"       |   task_id: 2             |
+------------------+                +------------------------+
                                    |
                          index.json (worktree registry)
                          events.jsonl (lifecycle log)

Các máy trạng thái (State machines):
  Task:     pending -> in_progress -> completed
  Worktree: absent  -> active      -> removed | kept
```

## Cách thức hoạt động

1. **Tạo ra một task.** Lưu giữ (Persist) trạng thái mục tiêu (goal) trước.

```python
TASKS.create("Implement auth refactor")
# -> .tasks/task_1.json  status=pending  worktree=""
```

2. **Tạo ra một worktree và kết nối (bind) vào task đó.** Việc truyền (Passing) `task_id` sẽ tự động chuyển đổi task đó qua thành `in_progress`.

```python
WORKTREES.create("auth-refactor", task_id=1)
# -> git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
# -> index.json nhận entry mới, task_1.json thay đổi thành worktree="auth-refactor"
```

Việc kết nối (binding) sẽ ghi đè (writes) các trạng thái cho cả hai phía:

```python
def bind_worktree(self, task_id, worktree):
    task = self._load(task_id)
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"
    self._save(task)
```

3. **Chạy các command trong worktree.** Thuộc tính `cwd` được trỏ trự tiếp tới một thư mục cô lập.

```python
subprocess.run(command, shell=True, cwd=worktree_path,
               capture_output=True, text=True, timeout=300)
```

4. **Đóng lại (Close out).** Hai lựa chọn:
   - `worktree_keep(name)` -- bảo toàn (preserve) thư mục phục vụ cho truy xuất về sau.
   - `worktree_remove(name, complete_task=True)` -- xóa bỏ thư mục, hoàn thành quá trình ràng buộc task, phát (emit) event. Một call duy nhất có thể xoay sở cả công cuộc tháo dỡ (teardown) + completion.

```python
def remove(self, name, force=False, complete_task=False):
    self._run_git(["worktree", "remove", wt["path"]])
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(wt["task_id"], status="completed")
        self.tasks.unbind_worktree(wt["task_id"])
        self.events.emit("task.completed", ...)
```

5. **Luồng sự kiện (Event stream).** Mỗi bước của lifecycle sẽ emit (phát ra) đến `.worktrees/events.jsonl`:

```json
{
  "event": "worktree.remove.after",
  "task": {"id": 1, "status": "completed"},
  "worktree": {"name": "auth-refactor", "status": "removed"},
  "ts": 1730000000
}
```

Các sự kiện được phát ra gồm: `worktree.create.before/after/failed`, `worktree.remove.before/after/failed`, `worktree.keep`, `task.completed`.

Sau sự cố (crash), trạng thái (state) sẽ tự cấu trúc lại từ `.tasks/` + `.worktrees/index.json` trên ổ đĩa. Bộ nhớ hội thoại (Conversation memory) có tính chất dễ bay hơi (volatile); trạng thái file có tính bền bỉ (durable).

## Những gì đã thay đổi từ s11

| Thành phần          | Trước đây (s11)               | Sau này (s12)                                  |
|--------------------|----------------------------|----------------------------------------------|
| Sự phối hợp (Coordination)       | Bảng task (owner/status)  | Bảng task + liên kết (binding) worktree rõ ràng       |
| Phạm vi thực thi (Execution scope)    | Thư mục chia sẻ chung (Shared directory)           | Thư mục bị cô lập ở phạm vi task (Task-scoped isolated directory)               |
| Khả năng phục hồi (Recoverability)     | Chỉ có status task           | Status task + worktree index                 |
| Gỡ bỏ (Teardown)           | Sự hoàn thành của task            | Task completion + giữ lại/xóa bỏ rõ ràng (explicit keep/remove)       |
| Tính hữu hình vòng đời (Lifecycle visibility) | Ngầm hiểu (Implicit) trong các bản log logs         | Các sự kiện rõ ràng (Explicit events) trong `.worktrees/events.jsonl` |

## Dùng thử

```sh
cd learn-claude-code
python agents/s12_worktree_task_isolation.py
```

1. `Create tasks for backend auth and frontend login page, then list tasks.`
2. `Create worktree "auth-refactor" for task 1, then bind task 2 to a new worktree "ui-login".`
3. `Run "git status --short" in worktree "auth-refactor".`
4. `Keep worktree "ui-login", then list worktrees and inspect events.`
5. `Remove worktree "auth-refactor" with complete_task=true, then list tasks/worktrees/events.`
