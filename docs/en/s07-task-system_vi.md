# s07: Task System

`s01 > s02 > s03 > s04 > s05 > s06 | [ s07 ] s08 > s09 > s10 > s11 > s12`

> *"Chia nhỏ các mục tiêu lớn thành các task nhỏ, sắp xếp chung, duy trì trên đĩa (persist to disk)"* -- một task graph dựa trên tệp (file-based) với các dependencies, đặt nền móng cho khả năng phối hợp multi-agent.

## Vấn đề

TodoManager của s03 là một checklist phẳng nằm trong bộ nhớ (memory): không có sự sắp xếp thứ tự (ordering), không có dependencies, không có status nào khác ngoài việc đã hoàn thành (done) hay chưa (not). Các mục tiêu thực tế (Real goals) đều có cấu trúc -- task B phụ thuộc vào task A, task C và D có thể chạy song song (parallel), task E chờ đợi cả C và D.

Nếu không có các mối quan hệ rõ ràng (explicit relationships), agent không thể nhận biết được thứ gì đã sẵn sàng, thứ gì đang bị chặn (blocked), hay thứ gì có thể chạy đồng thời (concurrently). Và bởi vì danh sách này chỉ tồn tại trong bộ nhớ, quá trình nén (context compression) ở s06 làm xóa sạch hoàn toàn nó.

## Giải pháp

Nâng cấp danh sách kiểm tra (checklist) trở thành một **task graph** được lưu trữ bền bỉ (persist) trên đĩa. Mỗi task là một file JSON với status, các sự phụ thuộc hay dependencies (`blockedBy`), và những thứ phụ thuộc vào nó hay dependents (`blocks`). Tại bất kỳ thời điểm nào, graph đều sẽ giải đáp ba câu hỏi sau:

- **Có thứ gì sẵn sàng?** -- các task với status `pending` và `blockedBy` trống (empty).
- **Có thứ gì bị chặn?** -- các task đang chờ đợi trên các dependency chưa hoàn thành.
- **Có thứ gì đã xong?** -- Các task đã `completed`, việc chúng hoàn thành sẽ tự động bỏ chặn (unblocks) các dependents.

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

Task graph (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

Sắp xếp (Ordering):     task 1 phải hoàn thành trước 2 và 3
Song song (Parallelism):  task 2 và 3 có thể chạy cùng lúc
Sự phụ thuộc (Dependencies): task 4 chờ đợi cả 2 và 3
Trạng thái (Status):       pending -> in_progress -> completed
```

Task graph này trở thành xương sống cho hoạt động phối hợp kể từ s07: quá trình thực thi ngầm (background execution - s08), các đội ngũ multi-agent (s09+), và sự cô lập về worktree (worktree isolation - s12) đều sẽ thực hiện đọc vào và ghi từ chung một cấu trúc này.

## Cách thức hoạt động

1. **TaskManager**: một file JSON cho từng task, quá trình CRUD kết hợp với dependency graph.

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def create(self, subject, description=""):
        task = {"id": self._next_id, "subject": subject,
                "status": "pending", "blockedBy": [],
                "blocks": [], "owner": ""}
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

2. **Giải quyết sự phụ thuộc (Dependency resolution)**: việc hoàn thành một task xóa sạch ID của nó khỏi danh sách `blockedBy` của mọi task khác, tự động unblock các dependents còn lại.

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

3. **Luồng kết nối trạng thái + sự phụ thuộc (Status + dependency wiring)**: `update` đảm nhiệm quá trình thay đổi trạng thái và các cạnh ranh giới dependency (dependency edges).

```python
def update(self, task_id, status=None,
           add_blocked_by=None, add_blocks=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    self._save(task)
```

4. Bốn task tool được đưa vào bên trong dispatch map.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "task_create": lambda **kw: TASKS.create(kw["subject"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

Từ s07 trở đi, task graph trở thành cấu trúc mặc định phục vụ cho công việc nhiều bước (multi-step work). Tính năng Todo của s03 vẫn được để lại để dùng làm các danh sách checklist nhanh chỉ dùng trong phạm vi một session (single-session).

## Những gì đã thay đổi từ s06

| Thành phần | Trước đây (s06) | Sau này (s07) |
|---|---|---|
| Tools | 5 | 8 (`task_create/update/list/get`) |
| Planning model | Checklist phẳng (Flat checklist) (in-memory) | Task graph chứa các dependencies (on disk) |
| Mối quan hệ | Không có | `blockedBy` + cấu trúc cạnh `blocks` |
| Quản lý trạng thái | Đã hoàn thành hoặc là chưa | `pending` -> `in_progress` -> `completed` |
| Tính bền vững (Persistence) | Bị mất sau mỗi quá trình nén | Sống sót (Survives) khi qua quá trình nén và khi restart lại |

## Dùng thử

```sh
cd learn-claude-code
python agents/s07_task_system.py
```

1. `Create 3 tasks: "Setup project", "Write code", "Write tests". Make them depend on each other in order.`
2. `List all tasks and show the dependency graph`
3. `Complete task 1 and then list tasks to see task 2 unblocked`
4. `Create a task board for refactoring: parse -> transform -> emit -> test, where transform and emit can run in parallel after parse`
