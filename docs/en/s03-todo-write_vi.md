# s03: TodoWrite

`s01 > s02 > [ s03 ] s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Một agent không có một kế hoạch là một agent trôi dạt"* -- liệt kê các bước trước, sau đó thực thi.

## Vấn đề

Trong các tác vụ có nhiều bước, model sẽ bị mất dấu (loses track). Nó lặp lại công việc, bỏ qua các bước, hoặc đi chệch hướng. Các cuộc trò chuyện dài làm điều này trở nên tồi tệ hơn -- system prompt bị nhạt đi khi các tool result lấp đầy context. Một quá trình refactoring gồm 10 bước có thể hoàn thành các bước 1-3, sau đó model bắt đầu ứng biến vì nó đã quên các bước 4-10.

## Giải pháp

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | Trạng thái TodoManager     |
              | [ ] task A            |
              | [>] task B  <- doing  |
              | [x] task C            |
              +-----------------------+
                          |
              if rounds_since_todo >= 3:
                inject <reminder> vào tool_result
```

## Cách thức hoạt động

1. TodoManager lưu trữ các item với trạng thái (statuses). Mỗi lần, chỉ một item có thể là `in_progress`.

```python
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()
```

2. Tool `todo` đi vào dispatch map giống như bất kỳ tool nào khác.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

3. Một nag reminder chèn (injects) một nhắc nhở nếu model trải qua 3+ round mà không gọi `todo`.

```python
if rounds_since_todo >= 3 and messages:
    last = messages[-1]
    if last["role"] == "user" and isinstance(last.get("content"), list):
        last["content"].insert(0, {
            "type": "text",
            "text": "<reminder>Update your todos.</reminder>",
        })
```

Ràng buộc "một in_progress tại một thời điểm" bắt buộc tập trung theo tuần tự. Nag reminder tạo ra tính tự chịu trách nhiệm (accountability).

## Những gì đã thay đổi từ s02

| Thành phần      | Trước đây (s02)  | Sau này (s03)              |
|----------------|------------------|----------------------------|
| Tools          | 4                | 5 (+todo)                  |
| Lên kế hoạch       | Không có             | TodoManager với statuses  |
| Nag injection  | Không có             | `<reminder>` sau 3 vòng (rounds)|
| Agent loop     | Simple dispatch  | + bộ đếm rounds_since_todo |

## Dùng thử

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

1. `Refactor the file hello.py: add type hints, docstrings, and a main guard`
2. `Create a Python package with __init__.py, utils.py, and tests/test_utils.py`
3. `Review all Python files and fix any style issues`
