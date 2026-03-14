[English](./README.md) | [中文](./README-zh.md) | [日本語](./README-ja.md)  
# Học về Claude Code -- Một agent dạng nano giống Claude Code, được xây dựng từ 0 đến 1

```
                    THE AGENT PATTERN
                    =================

    User --> messages[] --> LLM --> response
                                      |
                            stop_reason == "tool_use"?
                           /                          \
                         yes                           no
                          |                             |
                    execute tools                    return text
                    append results
                    loop back -----------------> messages[]


    Đó là loop tối giản. Mọi AI coding agent đều cần loop này.
    Các agent thực tế (production) sẽ thêm các lớp policy, permissions, và lifecycle.
```

**12 session lũy tiến, từ một loop đơn giản đến việc thực thi tự trị (autonomous execution) biệt lập.**
**Mỗi session thêm vào một cơ chế. Mỗi cơ chế có một phương châm (motto).**

> **s01** &nbsp; *"Chỉ cần một loop & Bash là đủ"* &mdash; một tool + một loop = một agent
>
> **s02** &nbsp; *"Thêm một tool nghĩa là thêm một handler"* &mdash; loop giữ nguyên; các tool mới được đăng ký vào dispatch map
>
> **s03** &nbsp; *"Một agent không có kế hoạch sẽ bị trôi dạt"* &mdash; liệt kê các bước trước, sau đó thực thi; tỷ lệ hoàn thành tăng gấp đôi
>
> **s04** &nbsp; *"Chia nhỏ các nhiệm vụ lớn; mỗi subtask nhận một context sạch"* &mdash; các subagents sử dụng messages[] độc lập, giữ cho cuộc hội thoại chính luôn sạch sẽ
>
> **s05** &nbsp; *"Tải kiến thức khi cần, không tải trước"* &mdash; chèn qua tool_result, không phải qua system prompt
>
> **s06** &nbsp; *"Context sẽ đầy; bạn cần một cách để tạo chỗ trống"* &mdash; chiến lược nén ba lớp cho các session vô hạn
>
> **s07** &nbsp; *"Chia các mục tiêu lớn thành các task nhỏ, sắp xếp chúng, lưu vào đĩa (persist to disk)"* &mdash; một task graph dựa trên file với các phụ thuộc, đặt nền móng cho sự hợp tác multi-agent
>
> **s08** &nbsp; *"Chạy các hoạt động chậm trong background; agent tiếp tục suy nghĩ"* &mdash; các daemon threads chạy lệnh, chèn notifications khi hoàn tất
>
> **s09** &nbsp; *"Khi nhiệm vụ quá lớn cho một người, hãy ủy thác (delegate) cho đồng đội"* &mdash; đồng đội bền vững + các JSONL mailboxes bất đồng bộ
>
> **s10** &nbsp; *"Đồng đội cần các quy tắc giao tiếp (communication rules) chung"* &mdash; một mô hình request-response thúc đẩy mọi cuộc đàm phán
>
> **s11** &nbsp; *"Đồng đội tự quét bảng công việc và tự nhận (auto-claim) task"* &mdash; không cần người dẫn đầu phải chỉ định từng việc
>
> **s12** &nbsp; *"Mỗi người làm việc trong thư mục riêng của mình, không can thiệp lẫn nhau"* &mdash; task quản lý mục tiêu, worktree quản lý thư mục, liên kết bằng ID

---

## The Core Pattern

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS,
        )
        messages.append({"role": "assistant",
                         "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

Mọi session đều xếp chồng một cơ chế lên trên loop này -- mà không làm thay đổi chính loop đó.

## Phạm vi (Quan trọng)

Repository này là một dự án học tập từ 0->1 để xây dựng một agent dạng nano giống Claude Code.
Nó cố ý đơn giản hóa hoặc bỏ qua một số cơ chế production:

- Các bus event/hook đầy đủ (ví dụ: PreToolUse, SessionStart/End, ConfigChange).  
  s12 chỉ bao gồm một dòng sự kiện lifecycle tối giản (append-only) phục vụ cho việc giảng dạy.
- Quản trị quyền dựa trên quy tắc (Rule-based permission governance) và quy trình tin cậy (trust workflows)
- Kiểm soát lifecycle của session (resume/fork) và kiểm soát lifecycle của worktree nâng cao
- Các chi tiết runtime MCP đầy đủ (transport/OAuth/resource subscribe/polling)

Hãy xem giao thức team JSONL mailbox trong repo này như một triển khai phục vụ giảng dạy, không phải là một khẳng định về bất kỳ cơ chế nội bộ production cụ thể nào.

## Bắt đầu nhanh

```sh
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env   # Chỉnh sửa .env với ANTHROPIC_API_KEY của bạn
102: 
python agents/s01_agent_loop.py       # Bắt đầu tại đây
python agents/s12_worktree_task_isolation.py  # Điểm kết thúc lộ trình đầy đủ
python agents/s_full.py               # Capstone: tất cả các cơ chế kết hợp lại
```

### Web Platform

Các hình ảnh trực quan tương tác, sơ đồ từng bước, trình xem mã nguồn và tài liệu.

```sh
cd web && npm install && npm run dev   # http://localhost:3000
```

## Lộ trình học tập

```
Phase 1: THE LOOP                    Phase 2: PLANNING & KNOWLEDGE
==================                   ==============================
s01  The Agent Loop          [1]     s03  TodoWrite               [5]
     while + stop_reason                  TodoManager + nag reminder
     |                                    |
     +-> s02  Tool Use            [4]     s04  Subagents            [5]
              dispatch map: name->handler     fresh messages[] per child
                                              |
                                         s05  Skills               [5]
                                              SKILL.md via tool_result
                                              |
                                         s06  Context Compact      [5]
                                              3-layer compression

Phase 3: PERSISTENCE                 Phase 4: TEAMS
==================                   =====================
s07  Tasks                   [8]     s09  Agent Teams             [9]
     file-based CRUD + deps graph         teammates + JSONL mailboxes
     |                                    |
s08  Background Tasks        [6]     s10  Team Protocols          [12]
     daemon threads + notify queue        shutdown + plan approval FSM
                                          |
                                     s11  Autonomous Agents       [14]
                                          idle cycle + auto-claim
                                     |
                                     s12  Worktree Isolation      [16]
                                          task coordination + optional isolated execution lanes

                                     [N] = số lượng tools
```

## Kiến trúc

```
learn-claude-code/
|
|-- agents/                        # Các triển khai tham chiếu Python (s01-s12 + s_full capstone)
|-- docs/{en,zh,ja}/               # Tài liệu ưu tiên mô hình tư duy (3 ngôn ngữ)
|-- web/                           # Nền tảng học tập tương tác (Next.js)
|-- skills/                        # Các file Skill cho s05
+-- .github/workflows/ci.yml      # CI: typecheck + build
```

## Tài liệu

Ưu tiên mô hình tư duy (mental-model-first): vấn đề, giải pháp, sơ đồ ASCII, mã nguồn tối giản.
Có sẵn bằng [English](./docs/en/) | [中文](./docs/zh/) | [日本語](./docs/ja/).

| Session | Chủ đề | Phương châm |
|---------|-------|-------|
| [s01](./docs/en/s01-the-agent-loop.md) | The Agent Loop | *Chỉ cần một loop & Bash là đủ* |
| [s02](./docs/en/s02-tool-use.md) | Tool Use | *Thêm một tool nghĩa là thêm một handler* |
| [s03](./docs/en/s03-todo-write.md) | TodoWrite | *Một agent không có kế hoạch sẽ bị trôi dạt* |
| [s04](./docs/en/s04-subagent.md) | Subagents | *Chia nhỏ các nhiệm vụ lớn; mỗi subtask nhận một context sạch* |
| [s05](./docs/en/s05-skill-loading.md) | Skills | *Tải kiến thức khi cần, không tải trước* |
| [s06](./docs/en/s06-context-compact.md) | Context Compact | *Context sẽ đầy; bạn cần một cách để tạo chỗ trống* |
| [s07](./docs/en/s07-task-system.md) | Tasks | *Chia các mục tiêu lớn thành các task nhỏ, sắp xếp chúng, lưu vào đĩa (persist to disk)* |
| [s08](./docs/en/s08-background-tasks.md) | Background Tasks | *Chạy các hoạt động chậm trong background; agent tiếp tục suy nghĩ* |
| [s09](./docs/en/s09-agent-teams.md) | Agent Teams | *Khi nhiệm vụ quá lớn cho một người, hãy ủy thác (delegate) cho đồng đội* |
| [s10](./docs/en/s10-team-protocols.md) | Team Protocols | *Đồng đội cần các quy tắc giao tiếp (communication rules) chung* |
| [s11](./docs/en/s11-autonomous-agents.md) | Autonomous Agents | *Đồng đội tự quét bảng công việc và tự nhận (auto-claim) task* |
| [s12](./docs/en/s12-worktree-task-isolation.md) | Worktree + Task Isolation | *Mỗi người làm việc trong thư mục riêng của mình, không can thiệp lẫn nhau* |

## Tiếp theo là gì -- từ hiểu biết đến phát hành

Sau 12 session, bạn sẽ hiểu cách một agent hoạt động từ trong ra ngoài. Có hai cách để áp dụng kiến thức đó:

### Kode Agent CLI -- Open-Source Coding Agent CLI (Mã nguồn mở)

> `npm i -g @shareai-lab/kode`

Hỗ trợ Skill & LSP, Windows-ready, có thể cắm (pluggable) với GLM / MiniMax / DeepSeek và các mô hình mở khác. Cài đặt và sử dụng ngay.

GitHub: **[shareAI-lab/Kode-cli](https://github.com/shareAI-lab/Kode-cli)**

### Kode Agent SDK -- Nhúng khả năng Agent vào ứng dụng của bạn

Claude Code Agent SDK chính thức giao tiếp với một quy trình CLI đầy đủ bên dưới -- mỗi người dùng đồng thời nghĩa là một quy trình terminal riêng biệt. Kode SDK là một thư viện độc lập không có chi phí quy trình trên mỗi người dùng, có thể nhúng vào backends, browser extensions, thiết bị nhúng hoặc bất kỳ runtime nào.

GitHub: **[shareAI-lab/Kode-agent-sdk](https://github.com/shareAI-lab/Kode-agent-sdk)**

---

## Repo chị em: từ *các session theo yêu cầu* đến *trợ lý luôn bật*

Agent mà repo này hướng dẫn là loại **dùng-xong-bỏ (use-and-discard)** -- mở terminal, giao nhiệm vụ, đóng khi xong, session tiếp theo bắt đầu từ con số không. Đó là mô hình của Claude Code.

[OpenClaw](https://github.com/openclaw/openclaw) đã chứng minh một khả năng khác: trên cùng một lõi agent, hai cơ chế biến agent từ "chạm vào để nó di chuyển" thành "nó thức dậy sau mỗi 30 giây để tìm việc":

- **Heartbeat** -- sau mỗi 30 giây, hệ thống gửi một tin nhắn cho agent để kiểm tra xem có việc gì cần làm không. Không có gì? Tiếp tục ngủ. Có gì đó? Hành động ngay lập tức.
- **Cron** -- agent có thể tự lên lịch cho các tác vụ tương lai của chính mình, được thực thi tự động khi đến giờ.

Thêm định tuyến IM đa kênh (WhatsApp / Telegram / Slack / Discord, 13+ nền tảng), bộ nhớ context bền vững, và hệ thống Soul tính cách, agent sẽ chuyển từ một công cụ dùng một lần thành một trợ lý AI cá nhân luôn bật.

**[claw0](https://github.com/shareAI-lab/claw0)** là repo hướng dẫn đồng hành của chúng tôi, phân tích các cơ chế này từ đầu:

```
claw agent = agent core + heartbeat + cron + IM chat + memory + soul
```

```
learn-claude-code                   claw0
(agent runtime core:                (trợ lý chủ động luôn bật:
 loop, tools, planning,              heartbeat, cron, kênh IM,
 teams, worktree isolation)          memory, tính cách soul)
```

## Giới thiệu
<img width="260" src="https://github.com/user-attachments/assets/fe8b852b-97da-4061-a467-9694906b5edf" /><br>

Quét mã Wechat để theo dõi chúng tôi,  
hoặc theo dõi trên X: [shareAI-Lab](https://x.com/baicai003)  

## Giấy phép

MIT

---

**Mô hình chính là agent. Việc của chúng ta là cung cấp cho nó các tool và tránh đường.**
