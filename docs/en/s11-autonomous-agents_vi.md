# s11: Autonomous Agents

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > [ s11 ] s12`

> *"Các đồng đội (Teammates) quét bảng công việc (board) và tự mình nhận task"* -- lead không cần phải phân công từng task.

## Vấn đề

Trong s09-s10, các teammate chỉ làm việc khi được bảo rõ ràng. Lead phải tạo (spawn) từng người với một prompt cụ thể. Có 10 task không ai nhận (unclaimed) trên bảng? Lead phân công từng cái một một cách thủ công. Không thể mở rộng (Doesn't scale).

Sự tự chủ thực sự (True autonomy): các teammate tự mình quét bảng task, nhận (claim) các task unclaimed, làm việc trên chúng, sau đó tìm kiếm thêm.

Một sự tinh tế nhỏ (One subtlety): sau quá trình nén (context compression - s06), agent có thể quên nó là ai. Việc tiêm lại danh tính (Identity re-injection) sẽ khắc phục điều này.

## Giải pháp

```
Vòng đời đồng đội (Teammate lifecycle) với chu kỳ rảnh rỗi (idle cycle):

+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (hoặc tool idle được gọi)
    v
+--------+
|  IDLE  |  kiểm tra (poll) mỗi 5s cho đến tối đa 60s
+---+----+
    |
    +---> check inbox --> message? ----------> WORK
    |
    +---> scan .tasks/ --> unclaimed? -------> claim -> WORK
    |
    +---> 60s timeout ----------------------> SHUTDOWN

Tiêm lại danh tính (Identity re-injection) sau quá trình nén:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

## Cách thức hoạt động

1. Vòng lặp của teammate có hai giai đoạn (phases): WORK và IDLE. Khi LLM ngừng gọi các tool (hoặc gọi tool `idle`), teammate sẽ tiến vào trạng thái IDLE.

```python
def _loop(self, name, role, prompt):
    while True:
        # -- GIAI ĐOẠN LÀM VIỆC (WORK PHASE) --
        messages = [{"role": "user", "content": prompt}]
        for _ in range(50):
            response = client.messages.create(...)
            if response.stop_reason != "tool_use":
                break
            # execute tools...
            if idle_requested:
                break

        # -- GIAI ĐOẠN RẢNH RỖI (IDLE PHASE) --
        self._set_status(name, "idle")
        resume = self._idle_poll(name, messages)
        if not resume:
            self._set_status(name, "shutdown")
            return
        self._set_status(name, "working")
```

2. Giai đoạn rảnh rỗi sẽ kiên tục quét (polls) hộp thư (inbox) và bảng công việc bằng một vòng lặp.

```python
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12
        time.sleep(POLL_INTERVAL)
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            return True
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                           f"{unclaimed[0]['subject']}</auto-claimed>"})
            return True
    return False  # timeout -> shutdown
```

3. Quét bảng task: tìm các task mang trạng thái pending (đang chờ), unowned (chưa ai nhận), và unblocked (không bị chặn).

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

4. Việc tiêm lại danh tính: khi context quá ngắn (quá trình nén đã xảy ra), nó sẽ tự động chèn một identity block.

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

## Những gì đã thay đổi từ s10

| Thành phần      | Trước đây (s10)  | Sau này (s11)              |
|----------------|------------------|----------------------------|
| Tools          | 12               | 14 (+idle, +claim_task)    |
| Sự tự chủ (Autonomy)       | Tùy thuộc vào Lead (Lead-directed)    | Tự tổ chức (Self-organizing)            |
| Giai đoạn rảnh rỗi (Idle phase)     | Không có             | Quét (Poll) inbox + bảng board task    |
| Việc nhận task (Task claiming)  | Chỉ làm thủ công (Manual only)      | Tự động nhận các task unclaimed |
| Danh tính (Identity)       | System prompt    | + tiêm lại (re-injection) sau quá trình compress |
| Thời gian chờ (Timeout)        | Không có             | 60s idle -> tự động shutdown  |

## Dùng thử

```sh
cd learn-claude-code
python agents/s11_autonomous_agents.py
```

1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim.`
2. `Spawn a coder teammate and let it find work from the task board itself`
3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
4. Type `/tasks` to see the task board với các owners của từng task
5. Type `/team` to monitor who is working vs idle
