# s04: Subagents

`s01 > s02 > s03 > [ s04 ] s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Chia nhỏ các task lớn; mỗi subtask nhận một context sạch"* -- các subagent sử dụng mảng messages[] độc lập, giữ cho cuộc trò chuyện chính (main conversation) sạch sẽ.

## Vấn đề

Khi agent làm việc, mảng messages của nó lớn dần. Mỗi file được đọc, mỗi bash output vẫn còn ở trong context vĩnh viễn. "Testing framework nào mà project này sử dụng?" có thể yêu cầu đọc 5 file, nhưng parent chỉ cần câu trả lời: "pytest."

## Giải pháp

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- fresh
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Context của parent giữ sự sạch sẽ. Context của subagent bị loại bỏ.
```

## Cách thức hoạt động

1. Parent nhận được một tool `task`. Child nhận tất cả base tools ngoại trừ `task` (không sinh ra đệ quy).

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

2. Subagent bắt đầu với `messages=[]` và chạy loop của riêng nó. Chỉ có text cuối cùng mới trả về (return) parent.

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

Toàn bộ lịch sử message của child (có thể 30+ tool calls) bị loại bỏ. Parent nhận được một đoạn summary như một `tool_result` bình thường.

## Những gì đã thay đổi từ s03

| Thành phần      | Trước đây (s03)  | Sau này (s04)             |
|----------------|------------------|---------------------------|
| Tools          | 5                | 5 (base) + task (parent)  |
| Context        | Đơn lẻ, chia sẻ chung | Cô lập (isolation) Parent + child |
| Subagent       | Không có             | Hàm `run_subagent()` |
| Return value   | N/A              | Chỉ tóm tắt bằng text         |

## Dùng thử

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`
