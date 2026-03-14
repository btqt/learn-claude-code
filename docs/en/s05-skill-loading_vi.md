# s05: Skills

`s01 > s02 > s03 > s04 > [ s05 ] s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Tải kiến thức khi bạn cần, không phải tải trước toàn bộ (upfront)"* -- chèn (inject) thông qua tool_result, không phải qua system prompt.

## Vấn đề

Bạn muốn agent tuân theo các workflow mang tính đặc thù miền (domain-specific): các quy ước git (git conventions), các mẫu kiểm thử (testing patterns), danh sách kiểm tra xem xét mã nguồn (code review checklists). Việc đặt mọi thứ vào system prompt làm lãng phí các token cho những skill không được sử dụng. 10 skill với 2000 token mỗi cái = 20,000 token, hầu hết trong số đó không liên quan đến bất kỳ task cụ thể nào.

## Giải pháp

```
System prompt (Layer 1 -- always present):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
+--------------------------------------+

Khi model gọi load_skill("git"):
+--------------------------------------+
| tool_result (Layer 2 -- on demand):  |
| <skill name="git">                   |
|   Các hướng dẫn git workflow đầy đủ...  |  ~2000 tokens
|   Bước 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

Layer 1: các *tên* skill trong system prompt (giá rẻ - cheap). Layer 2: toàn bộ *nội dung* thông qua tool_result (khi có nhu cầu - on demand).

## Cách thức hoạt động

1. Mỗi skill là một thư mục chứa một file `SKILL.md` với YAML frontmatter.

```
skills/
  pdf/
    SKILL.md       # ---\n name: pdf\n description: Process PDF files\n ---\n ...
  code-review/
    SKILL.md       # ---\n name: code-review\n description: Review code\n ---\n ...
```

2. SkillLoader quét tìm các file `SKILL.md`, sử dụng tên thư mục làm định danh cho skill.

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

3. Layer 1 đi vào system prompt. Layer 2 chỉ là một tool handler khác.

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

TOOL_HANDLERS = {
    # ...base tools...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

Model học được những skill nào tồn tại (giá rẻ - cheap) và tải chúng lên khi thực sự liên quan (đắt - expensive).

## Những gì đã thay đổi từ s04

| Thành phần     | Trước đây (s04)  | Sau này (s05)              |
|----------------|------------------|----------------------------|
| Tools          | 5 (base + task)  | 5 (base + load_skill)      |
| System prompt  | Chuỗi tĩnh (Static string) | + skill descriptions       |
| Knowledge      | Không có             | skills/\*/SKILL.md files   |
| Injection      | Không có             | Hai lớp (system + result)  |

## Dùng thử

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

1. `What skills are available?`
2. `Load the agent-builder skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`
4. `Build an MCP server using the mcp-builder skill`
