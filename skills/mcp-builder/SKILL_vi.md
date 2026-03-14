---
name: mcp-builder
description: Xây dựng các máy chủ MCP (Model Context Protocol) cho phép thêm thắt bổ cấp năng lực capabilities cho Claude. Sử dụng áp xuất vào hễ lúc người chơi dùng User ham thiết muốn mặn nồng tạc tạo 1 cỗ MCP Server, hoặc dỡ mớm đưa cung tool cho vô Claude xài, hay rả ráp gom cài cắm ngoại diên quy tập external services bên ngoài vào chụm vô khối chung.
---

# Kỹ năng Xây dựng Máy chủ MCP (MCP Server Building Skill)

Giờ đây bạn nắm chóp tay bọc chuyên sâu kinh lược đắc thủ tuyệt tay rành rộc chuyên môn (expertise) ở mảng thiết tạo lên Các máy chủ MCP. Dàn thể MCP bung ban cung phụng chắp đường mở nẻo cho lịnh kích Claude đi thông móc kết giăng dây qua các tuyến dịch vụ phụ trợ ngoại lai chằng chéo ở mảng khối thông đi qua ngõ vạch sẵn quy chuẩn trói nghi thức bài bản protocol.

## MCP nó là Cái thể Cớ Chi Thế?

Những cục MCP servers này chuyên bóc phốt mớm ban cung cấp mấy thứ:
- **Các Ngón Bộ Lệnh Tools**: Bày ra mấy chuỗi mảng thể Hàm Function cho bãi sân Claude đánh hiệu kích còi call sử dụng (mường tượng y xì mấy ngõ API endpoints trỏ đầu)
- **Chuỗi Mảng Món Resources Dữ Liệu**: Đây là Mảng cục xôi dữ data thứ bóc Claude khượi ban xới trọc cho xem đọc lấy xơi read thủ được (ví trọn tựa phác tập File cục bộ hoặc mảng dòng trệ quy vạch vọc trong database records cõi gốc)
- **Tập Thể Prompts Phác Mẫu Thơ Nền**: Chủng dạng mâm cơm lót ổ sẳn rải bày nền đệp sẵn trọn lốc phím lời Prompts Templates Pre-built lên mâm lót khung

## Lao Nhào Vào Thử Nhanh Nào (Quick Start): Ốp Nhanh Với Bé MCP Server Cho Form Phía Python Nào

### 1. Phác Cụm Nền Cơ Đặt Dự Án Setup Đầu Mối Dựng Vỏ (Project Setup)

```bash
# Tao Khởi Nắn Vỏ Kiến Tạo Project
mkdir my-mcp-server && cd my-mcp-server
python3 -m venv venv && source venv/bin/activate

# Dần Lắp Lên Cổ Bộc Tẩy SDK Của Phía Dòng MCP
pip install mcp
```

### 2. Mâm Rập Khung Template Bộ Máy Server Cơ Hồ Định Khối Cực Chuẩn (Basic Server Template)

```python
#!/usr/bin/env python3
"""my_server.py - Một dạng thức cấu MCP server nằm mức tối đoan cực đơn giãn mộc"""

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# Nhào Bột Ráp Thể Hiện Thể Hiện Thân Định Vị Nắn instance Bản Sao server Cục Máy Chạy
server = Server("my-server")

# Áp Phạt Dựng Khắc Tạc Lên Mã 1 Cây Công Cụ Khía Đẽo
@server.tool()
async def hello(name: str) -> str:
    """Nở lời thân kiệu ái chào chào khẩy hé lô đến ai đó (Say hello to someone).

    Args:
        name: Thả tên búa vô cho tao rót lời điệt cớ tên mầy để sái chào mừng chớ bộ
    """
    return f"Hello, {name}!"

@server.tool()
async def add_numbers(a: int, b: int) -> str:
    """Xáp nạp tốn trói lôi chặp đôi hai hòn số nọ quấn lụn nháo nối gợp lại vào chung dội với nhau.

    Args:
        a: Tén con đệ báo Số thứ tự đầu 1
        b: Gả con nhỏ bám đu Số rớt cắp cái hạng hở thứ ngỏ phỉnh 2
    """
    return str(a + b)

# Dận Bàn Điện Cho Cỗ Trâu Server Đội Máy Đi Càn Quét Hành Động Kích Đề
async def main():
    async with stdio_server() as (read, write):
        await server.run(read, write)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 3. Ấn Cọ Ghi Phác Bút Làm Sổ Nhật Nhập Biên Tố Sổ Kể Định Vị Register With Đi Cho Nấm Cục Claude Thấy Biến Mình Nữa Kia Ạ

Gia bồi bổ móc đẩy nhồi vô mốc góc ổ `~/.claude/mcp.json`:
```json
{
  "mcpServers": {
    "my-server": {
      "command": "python3",
      "args": ["/path/to/my_server.py"]
    }
  }
}
```

## Bé Máy Chủ MCP Cho Form Bên Phía TypeScript

### 1. Ngự Tính Set Setup Lên Mối Đặt Chỗ

```bash
mkdir my-mcp-server && cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk
```

### 2. Vẩy Sơn Nhấp Mực Bản Template

```typescript
// src/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "my-server",
  version: "1.0.0",
});

// Chạm Mảng Vẽ Đeo Tools Xuống Khung
server.setRequestHandler("tools/list", async () => ({
  tools: [
    {
      name: "hello",
      description: "Say hello to someone",
      inputSchema: {
        type: "object",
        properties: {
          name: { type: "string", description: "Name to greet" },
        },
        required: ["name"],
      },
    },
  ],
}));

server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "hello") {
    const name = request.params.arguments.name;
    return { content: [{ type: "text", text: `Hello, ${name}!` }] };
  }
  throw new Error("Unknown tool");
});

// Châm Ngòi Cho Đạn Server Chạm Đề Xuất Bến Start
const transport = new StdioServerTransport();
server.connect(transport);
```

## Khung Tủ Đựng Phác Mẫu Điệu Pattern Rắc Rối Đâm Sâu Advanced Patterns Nâng Cao

### Áp Trù Thiết Đảm Móc Luồng Chồng Nắn Khúc Cho Chấu API Ngoại Tạp Integration Xáp Ngoại Cập Biến External

```python
import httpx
from mcp.server import Server

server = Server("weather-server")

@server.tool()
async def get_weather(city: str) -> str:
    """Nắm bắt lấy chộp rút thẩu gom cái giong khí hậu thời tiết của thực thời cho ngay cái cục chốn mảng cái xó thành phố ấy."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"https://api.weatherapi.com/v1/current.json",
            params={"key": "YOUR_API_KEY", "q": city}
        )
        data = resp.json()
        return f"{city}: {data['current']['temp_c']}C, {data['current']['condition']['text']}"
```

### Lách Cửa Ngoáy Nhảy Vào Can Thấy Hệ Access Cơ Nền Đáy Đệm Cơ Sở Gốc Rễ Database

```python
import sqlite3
from mcp.server import Server

server = Server("db-server")

@server.tool()
async def query_db(sql: str) -> str:
    """Hò réo khứ hành đẩy lịnh tống tráo vận chuyển 1 luồng truy vấn mác SQL dợm ở chóp read-only điệu dạng chỉ nhõn có xem đọc tuột luốt ngóng lén thôi nhen."""
    if not sql.strip().upper().startswith("SELECT"):
        return "Error: Only SELECT queries allowed"

    conn = sqlite3.connect("data.db")
    cursor = conn.execute(sql)
    rows = cursor.fetchall()
    conn.close()
    return str(rows)
```

### Xó Dữ Liệu Tịch Chỉ Lướt Liếc Data Chay (Resources - Read-only Data)

```python
@server.resource("config://settings")
async def get_settings() -> str:
    """Những Thiết đặt lọt chổ khung ngắm trích settings thiết định tại Application."""
    return open("settings.json").read()

@server.resource("file://{path}")
async def read_file(path: str) -> str:
    """Nhóp nhép nhá tệp chữ xơi read moi ra một con tập từ cõi môi trượng ổ bãi workspace."""
    return open(path).read()
```

## Khám Kiểm Ợ Khâu Dập Bệ Thử Test (Testing)

```bash
# Phang test gá chung vô kẹp gá Inspector tóm thóp thử với kẹp MCP
npx @anthropics/mcp-inspector python3 my_server.py

# Bằng không vút đạn phím cược gửi tông sấn rảo đè xáng chực tiệp dội chùm khối test bửa thẳng luồng mớ messages test vô họng sún (Or send test messages directly)
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | python3 my_server.py
```

## Những Đỉnh Chóp Hàng Chuẩn Dạng Mực Thước Mực Áp Best Practices Cho Chuyên Bang Nghiệp

1. **Kháng rõ rành rọt bập bóc phím nét trong việc phơi bầy Mô tả (tool descriptions) rạch ròi ra hồn**: Vì lão đần Claude sẽ ngợm chực chỉ chăm dòm măm bám vú vào chóp lọn tụt tọt những dòng giải ngữ trên đó ngõ đành làm cớ bắt đưa ra lệnh ngã ngũ định hướng trạc tòm coi chừng gõ tiếng gào bới bóc xó nào gọi là tóm xỏ được the tools call dăm công cục lề nứt ranh ấy mớ đời rỡ
2. **Quy Bắt Ấn Phạt Sự Dò Xác Thực Input Tiết Chuẩn (Input validation)**: Luật sắt thép buộc mãi là luôn bắt kẹt áp giải rành sòng soát định lượng làm độ chuẩn đôn dập mọi kẽ hở vệ sanh thông thụt làm cho rành lõi lọc sạch rác uế sanitize đám inputs cục đà đút mớm nạp khơi thục vô từ cái vệt đầu phểu dậu vô input
3. **Máng Xử Đấu Xước Trở Rắn Sượng Lỗi Mấu Cọc Lỗi Cứng Lòi (Error handling)**: Phải sất đưa văng tớ về thảy trả về ném trịch lại vô mâm những dòng thư chửi củi tin báo tát Error trả lại tạt đong mang xô trọng lượng thấm chĩu sớ đục xéo nội ngụ nặng đầm trĩu chất có dọng có sức vóc mang trọn bầu nghĩa ý hàm sinh meaningful báo sất lỗi
4. **Luôn mặc mặc Trọn Trịch Luồng Ngầm Trôi Ngót Lọt Chọn Tính Kẽ Cho Áp Theo Mệnh Async Không Ngầm Đi Bộ (Async by default)**: Sức sài chiêu găm miết rặc quăng dốc hết chụm bộ đôi quyền Async/Await múa quạt vặt cho cái đám sớ các cuộc vần thao xáp vận đánh dồn chùm đi ải xốc I/O operations vận xuất nhập nén luồn
5. **Độn Sắc Chắc Bức Tường Phòng Trị Tính Căn Bệnh Security Tội Táy Ngoáy Vệ Thể Trọn Tính Tàng Hình (Security)**: Nhất quỷ diệt ma, sống quật chết nạt muôn đời không cho nát cắn bở Không bao giờ được hớ hênh lòi ngực nẩy mầm thả tọt tung hô lột phơi bày ba rành mọi cái chi rà các thứ phép thao tác phỉnh sensitive vờn chọt sâu kín xó nhạy mà lại phang khong chẳng qua tay xác thực định định qua phổng qua kiểm auth chốt thẻ cấp căn cước
6. **Tá Tính Vừa Tẩm Thẩn Rũ Tính Nếp Tráo Nịch Thuần Idempotency Đóng Tính Bất Lợi Bất Biến Đồng Quất Chẳng Cụt (Idempotency)**: Bộ ngạch Tools nên đi được phác thảo ngự theo chiều thế bưng mâm ôm an ổn cẩn an trơn lu chít chốc không bứt quắc xốc tai nạn nếu trường tai biến hỏng phải ép ấn cho xài luân lại quất trần đẻ vòng xoay quay đánh đứt nhỡ cuốc cuộn gượng nhả sức quánh tọt đi khứ hồi chầu khói nẹt Retry mâm lại vòng hai vòng chục vệt bẩn xoáy vẫn thoi lội an nhiên lặn ngoi ngót bọt
