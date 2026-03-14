# s02: Tool Use

`s01 > [ s02 ] s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Thêm một tool có nghĩa là thêm một handler"* -- vòng lặp vẫn giữ nguyên; các tool mới đăng ký vào dispatch map.

## Vấn đề

Với chỉ `bash`, agent phải dùng shell cho mọi thứ. `cat` cắt bớt một cách khó lường, `sed` thất bại với các ký tự đặc biệt, và mỗi lệnh gọi bash là một bề mặt bảo mật không bị ràng buộc (unconstrained security surface). Các tool chuyên dụng như `read_file` và `write_file` cho phép bạn thực thi path sandboxing ở cấp độ tool.

Sự thấu hiểu then chốt (The key insight): việc thêm các tool không yêu cầu thay đổi vòng lặp.

## Giải pháp

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+

Dispatch map là một dict: {tool_name: handler_function}.
Một lần tra cứu (lookup) thay thế cho bất kỳ chuỗi if/elif nào.
```

## Cách thức hoạt động

1. Mỗi tool nhận được một hàm handler. Path sandboxing ngăn chặn việc thoát khỏi workspace (workspace escape).

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text()
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]
```

2. Dispatch map liên kết tên tool với các handler.

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                        kw["new_text"]),
}
```

3. Trong vòng lặp, tra cứu (look up) handler bằng tên. Bản thân phần thân của vòng lặp (loop body) không thay đổi so với s01.

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

Thêm một tool = thêm một handler + thêm một mục nhập schema (schema entry). Vòng lặp không bao giờ thay đổi.

## Những gì đã thay đổi từ s01

| Thành phần     | Trước đây (s01)    | Sau này (s02)              |
|----------------|--------------------|----------------------------|
| Các tool       | 1 (chỉ bash)       | 4 (bash, read, write, edit)|
| Dispatch       | Lệnh gọi bash hardcode | `TOOL_HANDLERS` dict       |
| Path safety    | Không có           | Sandbox `safe_path()`      |
| Agent loop     | Không thay đổi     | Không thay đổi             |

## Dùng thử

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

1. `Read the file requirements.txt`
2. `Create a file called greet.py with a greet(name) function`
3. `Edit greet.py to add a docstring to the function`
4. `Read greet.py to verify the edit worked`
