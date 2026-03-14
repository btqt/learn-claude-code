# s08: Background Tasks

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > [ s08 ] s09 > s10 > s11 > s12`

> *"Chạy các hoạt động chậm trong background; agent tiếp tục suy nghĩ"* -- các luồng daemon (daemon threads) chạy các command, chèn (inject) các thông báo (notifications) khi hoàn thành.

## Vấn đề

Một số command mất nhiều phút: `npm install`, `pytest`, `docker build`. Với một vòng lặp chặn (blocking loop), mô hình ngồi không chờ đợi (sits idle waiting). Nếu user yêu cầu "install dependencies and while that runs, create the config file," agent thực hiện chúng một cách tuần tự (sequentially), không phải song song (in parallel).

## Giải pháp

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue(result) |
|  ^drain queue   |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]      (parallel)
             |          |
             +-- các kết quả (results) được chèn vào trước LLM call tiếp theo --+
```

## Cách thức hoạt động

1. BackgroundManager theo dõi các task với một hàng đợi thông báo an toàn luồng (thread-safe notification queue).

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self._notification_queue = []
        self._lock = threading.Lock()
```

2. `run()` khởi chạy một daemon thread và trả về (returns) ngay lập tức.

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"
```

3. Khi subprocess hoàn thành, kết quả của nó đi vào notification queue.

```python
def _execute(self, task_id, command):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "result": output[:500]})
```

4. Agent loop vét trống (drains) các notifications trước mỗi lần gọi LLM.

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()
        if notifs:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n"
                           f"</background-results>"})
            messages.append({"role": "assistant",
                "content": "Noted background results."})
        response = client.messages.create(...)
```

Vòng lặp vẫn giữ cấu trúc đơn luồng (single-threaded). Chỉ có I/O của subprocess là được thực thi song song (parallelized).

## Những gì đã thay đổi từ s07

| Thành phần      | Trước đây (s07)  | Sau này (s08)              |
|----------------|------------------|----------------------------|
| Tools          | 8                | 6 (base + background_run + check)|
| Thực thi      | Chỉ Blocking (chặn)    | Blocking + background threads|
| Notification   | Không có             | Hàng đợi (Queue) được vét trống (drained) trên mỗi vòng lặp     |
| Đồng thời (Concurrency)    | Không có             | Các luồng Daemon (Daemon threads)             |

## Dùng thử

```sh
cd learn-claude-code
python agents/s08_background_tasks.py
```

1. `Run "sleep 5 && echo done" in the background, then create a file while it runs`
2. `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Check their status.`
3. `Run pytest in the background and keep working on other things`
