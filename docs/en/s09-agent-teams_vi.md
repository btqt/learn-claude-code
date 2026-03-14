# s09: Agent Teams

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > [ s09 ] s10 > s11 > s12`

> *"Khi task là quá lớn đối với một người, hãy ủy quyền cho các đồng đội (teammates)"* -- các đồng đội liên tục (persistent teammates) + hộp thư bất đồng bộ (async mailboxes).

## Vấn đề

Các subagent (s04) mang tính chất dùng một lần (disposable): sinh ra (spawn), làm việc, trả về bản tóm tắt (summary), sau đó chết đi. Không có danh tính (identity), không có bộ nhớ (memory) giữa các lần gọi (invocations). Các background task (s08) chạy các shell command nhưng không thể đưa ra các quyết định dựa trên sự điều hướng của LLM.

Làm việc nhóm thực sự cần: (1) các agent liên tục (persistent agents) sống lâu hơn một prompt đơn lẻ, (2) việc quản lý vòng đời (lifecycle) và danh tính, (3) một kênh giao tiếp (communication channel) giữa các agent.

## Giải pháp

```
Vòng đời đồng đội (Teammate lifecycle):
  spawn -> WORKING -> IDLE -> WORKING -> ... -> SHUTDOWN

Giao tiếp (Communication):
  .team/
    config.json           <- danh sách đội nhóm (team roster) + trạng thái (statuses)
    inbox/
      alice.jsonl         <- chỉ nối thêm (append-only), làm trống khi đọc (drain-on-read)
      bob.jsonl
      lead.jsonl

              +--------+    send("alice","bob","...")    +--------+
              | alice  | -----------------------------> |  bob   |
              | loop   |    bob.jsonl << {json_line}    |  loop  |
              +--------+                                +--------+
                   ^                                         |
                   |        BUS.read_inbox("alice")          |
                   +---- alice.jsonl -> read + drain ---------+
```

## Cách thức hoạt động

1. TeammateManager duy trì file config.json cùng với danh sách của đội (team roster).

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()
        self.threads = {}
```

2. `spawn()` tạo ra một teammate và bắt đầu vòng lặp agent của nó trong một thread.

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)
    self._save_config()
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()
    return f"Spawned teammate '{name}' (role: {role})"
```

3. MessageBus: Các inbox dùng dữ liệu dạng JSONL mang bản chất append-only (chỉ thêm vào). `send()` sẽ gắn thêm một dòng JSON (JSON line); `read_inbox()` sẽ đọc tất cả và làm trống (drains).

```python
class MessageBus:
    def send(self, sender, to, content, msg_type="message", extra=None):
        msg = {"type": msg_type, "from": sender,
               "content": content, "timestamp": time.time()}
        if extra:
            msg.update(extra)
        with open(self.dir / f"{to}.jsonl", "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, name):
        path = self.dir / f"{name}.jsonl"
        if not path.exists(): return "[]"
        msgs = [json.loads(l) for l in path.read_text().strip().splitlines() if l]
        path.write_text("")  # drain
        return json.dumps(msgs, indent=2)
```

4. Mỗi teammate kiểm tra inbox của nó trước mỗi LLM call, chèn các message nhận được vào trong context.

```python
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            messages.append({"role": "assistant",
                "content": "Noted inbox messages."})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
        # execute tools, append results...
    self._find_member(name)["status"] = "idle"
```

## Những gì đã thay đổi từ s08

| Thành phần      | Trước đây (s08)  | Sau này (s09)              |
|----------------|------------------|----------------------------|
| Tools          | 6                | 9 (+spawn/send/read_inbox) |
| Các Agent         | Đơn lẻ (Single)           | Lead + N đồng đội (teammates)         |
| Persistence    | Không có             | config.json + JSONL inboxes|
| Threads        | Background cmds  | Toàn bộ agent loops mỗi thread|
| Vòng đời (Lifecycle)      | Bắn-và-quên (Fire-and-forget)  | idle -> working -> idle    |
| Giao tiếp  | Không có             | message + broadcast        |

## Dùng thử

```sh
cd learn-claude-code
python agents/s09_agent_teams.py
```

1. `Spawn alice (coder) and bob (tester). Have alice send bob a message.`
2. `Broadcast "status update: phase 1 complete" to all teammates`
3. `Check the lead inbox for any messages`
4. Type `/team` to see the team roster với các trạng thái tương ứng
5. Type `/inbox` to manually check the lead's inbox
