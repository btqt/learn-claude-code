# s06: Context Compact

`s01 > s02 > s03 > s04 > s05 > [ s06 ] | s07 > s08 > s09 > s10 > s11 > s12`

> *"Context sẽ bị đầy; bạn cần một cách để tạo thêm không gian"* -- chiến lược nén (compression) ba lớp (three-layer) cho các phiên (sessions) vô hạn.

## Vấn đề

Context window là có giới hạn. Một lệnh gọi `read_file` duy nhất trên một file có độ dài 1000 dòng sẽ tiêu tốn ~4000 token. Sau khi đọc 30 file và chạy 20 bash command, bạn sẽ đạt tới ngưỡng 100,000+ token. Agent không thể làm việc trên các codebase lớn mà không có tính năng nén (compression).

## Giải pháp

Ba lớp, mức độ quyết liệt tăng dần:

```
Mỗi lượt (turn):
+------------------+
| Tool call result |
+------------------+
        |
        v
[Layer 1: micro_compact]        (âm thầm, mỗi lượt)
  Thay thế tool_result cũ > 3 lượt (turns)
  bằng "[Previous: used {tool_name}]"
        |
        v
[Check: tokens > 50000?]
   |               |
   no              yes
   |               |
   v               v
continue    [Layer 2: auto_compact]
              Lưu bản ghi chép (transcript) vào .transcripts/
              LLM tóm tắt đoạn hội thoại.
              Thay thế toàn bộ messages bằng [summary].
                    |
                    v
            [Layer 3: compact tool]
              Model gọi compact một cách rõ ràng (explicitly).
              Tóm tắt tương tự như auto_compact.
```

## Cách thức hoạt động

1. **Layer 1 -- micro_compact**: Trước mỗi lần gọi LLM, thay thế các tool result cũ bằng các placeholder.

```python
def micro_compact(messages: list) -> list:
    tool_results = []
    for i, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for j, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((i, j, part))
    if len(tool_results) <= KEEP_RECENT:
        return messages
    for _, _, part in tool_results[:-KEEP_RECENT]:
        if len(part.get("content", "")) > 100:
            part["content"] = f"[Previous: used {tool_name}]"
    return messages
```

2. **Layer 2 -- auto_compact**: Khi các token vượt quá ngưỡng, lưu lại toàn bộ bản ghi (transcript) vào ổ đĩa, sau đó yêu cầu LLM thực hiện tóm tắt.

```python
def auto_compact(messages: list) -> list:
    # Save transcript for recovery
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    # LLM summarizes
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity..."
            + json.dumps(messages, default=str)[:80000]}],
        max_tokens=2000,
    )
    return [
        {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
        {"role": "assistant", "content": "Understood. Continuing."},
    ]
```

3. **Layer 3 -- manual compact**: Tool `compact` kích hoạt quá trình tóm tắt tương tự dựa trên nhu cầu theo yêu cầu.

4. Vòng lặp tích hợp cả ba:

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # Layer 1
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)       # Layer 2
        response = client.messages.create(...)
        # ... tool execution ...
        if manual_compact:
            messages[:] = auto_compact(messages)       # Layer 3
```

Các bản ghi chép (Transcripts) bảo toàn toàn bộ lịch sử lịch sử trên đĩa. Không có gì thực sự bị mất đi -- chỉ là được di chuyển ra khỏi active context.

## Những gì đã thay đổi từ s05

| Thành phần     | Trước đây (s05)  | Sau này (s06)              |
|----------------|------------------|----------------------------|
| Tools          | 5                | 5 (base + compact)         |
| Context mgmt   | Không có             | Three-layer compression    |
| Micro-compact  | Không có             | Các kết quả (results) cũ -> placeholders|
| Auto-compact   | Không có             | Token threshold trigger    |
| Transcripts    | Không có             | Được lưu vào .transcripts/ |

## Dùng thử

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

1. `Read every Python file in the agents/ directory one by one` (quan sát micro-compact thay thế các kết quả cũ)
2. `Keep reading files until compression triggers automatically`
3. `Use the compact tool to manually compress the conversation`
