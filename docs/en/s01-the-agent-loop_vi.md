# s01: Vòng lặp Agent

`[ s01 ] s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Một vòng lặp & Bash là tất cả những gì bạn cần"* -- một tool + một vòng lặp = một agent.

## Vấn đề

Một mô hình ngôn ngữ có thể suy luận về code, nhưng nó không thể *chạm* vào thế giới thực -- không thể đọc file, chạy test, hoặc kiểm tra lỗi. Nếu không có một vòng lặp, mỗi lệnh gọi tool yêu cầu bạn phải sao chép và dán kết quả trở lại một cách thủ công. Bạn trở thành vòng lặp đó.

## Giải pháp

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (lặp cho đến khi stop_reason != "tool_use")
```

Một điều kiện thoát sẽ điều khiển toàn bộ luồng luân chuyển. Vòng lặp chạy cho đến khi mô hình ngừng gọi các tool.

## Cách thức hoạt động

1. User prompt trở thành tin nhắn đầu tiên.

```python
messages.append({"role": "user", "content": query})
```

2. Gửi các tin nhắn + định nghĩa tool đến LLM.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

3. Nối thêm (append) phản hồi của assistant. Kiểm tra `stop_reason` -- nếu mô hình không gọi một tool, chúng ta đã hoàn thành.

```python
messages.append({"role": "assistant", "content": response.content})
if response.stop_reason != "tool_use":
    return
```

4. Thực thi từng lệnh gọi tool, thu thập kết quả, nối thêm dưới dạng tin nhắn của user. Vòng lặp quay lại bước 2.

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
messages.append({"role": "user", "content": results})
```

Được lắp ráp lại thành một hàm duy nhất:

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

Đó là toàn bộ agent gói gọn trong dưới 30 dòng mã. Mọi thứ khác trong khóa học này sẽ được xếp lớp lên trên -- mà không làm thay đổi vòng lặp.

## Những gì đã thay đổi

| Thành phần | Trước đây | Sau này |
|---------------|------------|--------------------------------|
| Vòng lặp Agent | (không có) | `while True` + stop_reason |
| Các tool | (không có) | `bash` (một tool) |
| Tin nhắn | (không có) | Danh sách tích lũy |
| Luồng điều khiển | (không có) | `stop_reason != "tool_use"` |

## Dùng thử

```sh
cd learn-claude-code
python agents/s01_agent_loop.py
```

1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`
4. `Create a directory called test_output and write 3 files in it`
