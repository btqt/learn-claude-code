---
name: agent-builder
description: |
  Thiết kế và xây dựng AI agents cho mọi lĩnh vực. Sử dụng khi người dùng:
  (1) yêu cầu "tạo một agent", "xây dựng một trợ lý", hoặc "thiết kế một hệ thống AI"
  (2) muốn hiểu về kiến trúc agent, các mẫu agentic, hoặc AI tự chủ
  (3) cần sự trợ giúp với capabilities, subagents, planning, hoặc các cơ chế kỹ năng
  (4) hỏi về Claude Code, Cursor, hoặc phần internals của các agent tương tự
  (5) muốn xây dựng agents cho kinh doanh, nghiên cứu, công việc sáng tạo, hoặc các nhiệm vụ vận hành
  Keywords: agent, assistant, autonomous, workflow, tool use, multi-step, orchestration
---

# Agent Builder

Xây dựng AI agents cho bất kỳ lĩnh vực nào - dịch vụ khách hàng, nghiên cứu, vận hành, công việc sáng tạo hoặc các quy trình kinh doanh chuyên biệt.

## Triết Lý Cốt Lõi

> **Mô hình đã biết cách trở thành một agent. Công việc của bạn là tránh ra khỏi đường đi của nó.**

Một agent không phải là một kỹ thuật phức tạp. Nó là một vòng lặp đơn giản mời gọi mô hình hành động:

```
LOOP:
  Model sees: context + available capabilities
  Model decides: act or respond
  If act: execute capability, add result, continue
  If respond: return to user
```

**Đó là tất cả.** Phép thuật không nằm trong hệ thống mã - nó ở bên trong mô hình. Code của bạn chỉ việc cung cấp cơ hội cho nó thực hiện.

## Ba Yếu Tố

### 1. Capabilities (Nó có thể LÀM gì?)

Các hành động mang tính nguyên tử mà agent có thể thi hành: tìm kiếm, đọc, tạo lập, gửi đi, truy vấn, nội dung chỉnh sửa.

**Nguyên lý thiết kế**: Bắt đầu luôn từ 3-5 capabilities. Chỉ đưa thêm nhiều hơn cho tới khi agent kiên quyết tạo ra thất bại chỉ bởi vì thiểu sót 1 capability đó.

### 2. Kiến Thức (Nó BIẾT những gì?)

Sự chuyên môn ở trong loại lĩnh vực được nạp vào mỗi khi cần phát sinh (on-demand): các thể chế chính sách, workflows, cách thức chuẩn best practices, schemas.

**Nguyên lý thiết kế**: Làm cho mọi kiến thức sẵn có để tham chiếu, chớ không ép buộc. Mời gọi tải nó đi vào trong lúc hữu quan, không mang từ ban đầu.

### 3. Ngữ Cảnh (Tất cả điều gì đã từng diễn ra?)

Lịch sử của cuộc nội hội thoại – chiếc sợi dẫn chỉ xuyên kết dính các hoạt động hành vi lại với nhau trở thành duy nhất.

**Nguyên lý thiết kế**: Ngữ cảnh thật sự cực kỳ trân quý. Cần tập ly cô mảng các loại subtasks tạo mức độ t nhiễu. Phải rút kết hoặc ngắt đi các báo kết dài dòng không cần thiết. Đảm bảo triệt để cho tính dễ hiểu minh bạch.

## Tư Duy Thiết Kế Agent

Trước khi xúc tiến xây dựng, cần ngộ thấu:

- **Mục đích**: Chức năng thực hiện agent hướng về điều gì?
- **Lĩnh vực**: Phân vùng môi trường thế giới agent đang thao tác ở đâu? (Dịch vụ hỗ trợ chăm sóc khách hàng, phân môn nghiên cứu học thuật, hệ thống tác nghiệp vận hành, công tác sáng tạo đổi mới...)
- **Capabilities**: Điều bắt buộc cho yếu cốt 3-5 loại hành vi hành động là gì?
- **Kiến thức**: Những phân nguồn kiến thức tri thức nào chúng bắt buộc xin quyền truy nhập tìm hiểu?
- **Sự tin tưởng**: Có thể tự ủy phân uỷ quyền những cách ra yêu cầu để quyết định nào gửi cho model kia?

**ĐIỀU CỰC KỲ QUAN TRỌNG**: Đặt Niềm tin vào cho model. Nghiêm cấm tạo dạng cấu kiện xây dựng quá thặng thừa (over-engineer). Tuyệt đối tránh việc quy đặt chuẩn workflows ở trước mức. Tự động cấp cấp phát quyền năng capabilities cùng theo cho phép nhận quyền suy luận phân xử.

## Độ Phức Tạp Lũy Tiến

Thuở khởi đi thật tự nhiên giản dị. Góp đẩy theo tiến độ chỉ trừ ở hiện tượng bắt sử dụng vào ứng dụng môi sinh hiện thực hiển tạo yêu cầu cấp bách:

| Cấp độ | Cần nạp điều gì | Áp dụng ở ngưỡng thời điểm nhịp diễn này |
|-------|-------------|----------------|
| Cơ bản | 3-5 capabilities | Mọi lúc bắt đầu từ nấc này đây |
| Lập Kế hoạch | Tracking lưu giữ lại các quy trình quá trình tiến độ | Dòng tác vụ chắp nhiều chuỗi quy đoạn bị loãng rớt mất mạch tính dính dãn gắn kết |
| Subagents | Vùng môi sinh tách biệt cho mấy agents phần hệ chi nhánh dạng con | Yếu tố thu thập khám phá tìm tòi vượt tạo dư âm mức ô nhiễm cho không gian Context |
| Các Đơn vị Kỹ Năng Skills | Thông tin trí năng tại định tuyến đáp ứng cấp on-demand theo gọi mời yêu cầu | Tính chuyên tu thạo thuần được nhắc nhớ đến |

**Đại bộ phận mô thức các agents chẳng cần vươn tớii vượt hẳn cấp ngưỡng cao Level 2.**

## Ví Dụ Về Các Lĩnh Vực

**Kinh Doanh Doanh nghiệp**: Các phân mảng về CRM queries, dạng truyền tải email, quản lý biểu thời gian calendar, sự xét duyệt approvals
**Phân Mảng Nghiên Cứu**: Nghiền tìm lọc trong Database, quy trình đánh mổ xẻ rã dữ liệu phân tích document, hệ chỉ báo nhắm đích citations
**Công tác Khối Vận Hành Hoạt Động**: Điều khiển quan trắc monitoring, sự quản vận về vé thẻ tickets, mạng báo đẩy hiện notifications, tăng thang thang biểu nâng độ escalation
**Mảng Công Tác Giải Quyết Tính Sáng Tạo Tạo Mới Thẩm Mỹ**: Mảng sự sáng xuất sáng tạo nguyên liệu đầu vào tài sản Asset generation, chế tác biên chỉnh edit, tham gia hợp khối đồng thao tác collaboration, đánh định bình xét duyệt lại review

Giao thức mang dấp tính phổ biên chung quy phổ quát. Dấu chỉ duy chỉ khác biệt ở khả lực capabilities xoay chuyển mà thôi.

## Những Nguyên Tắc Cốt Yếu

1. **Mô hình LÀ agent** - Yếu tố mã hóa Code rốt quy tụ thuần tạo sự thực thi xoáy lặp the loop
2. **Khả năng Capabilities tiếp dẫn năng lực** - Nẵng sức những thứ nó CÓ THỂ hiện thân nhúng tay can thiệp tạo làm
3. **Tri Năng Kiến Thức mang chỉ báo mách gọi** - Việc chi nó phải TƯỜNG nắm định hình phương quy thực hiện thế nào
4. **Hệ Ràng Buộc sinh hội tụ tập trung** - Tính giới nghiêm quy định hạn chế kích đẻ thêm nét minh rành hiển lộ sự rõ ràng
5. **Đức Tin kích thả sự Tự do Tự chủ** - Uỷ thác quyền chủ quản năng động cho quy cách phân tư của chính chiếc model
6. **Phiên bản Tiến Hóa tạo trình chiếu bộc bạch** - Chạy mầm đầu thô sơ cốt giản, bước tiếp lột gọt tiến dần tịnh biến chiểu nương nhờ hiệu sinh từ Usage (Dữ kiện đo dùng số thật tế)

## Những Lối Rẽ Mẫu Sử Dụng Sai Hại (Anti-Patterns)

| Hình Thái Mẫu Lỗi | Vấn Đề Rắc Rối | Lời Quyết Giải |
|---------|---------|----------|
| Kiến trúc độ nặng kỹ nghệ quá dư (Over-engineering) | Biến rườm rà phức loạn mọi việc lên trước ở cả ngưỡng cấp độ nhu cầu cần khởi đáp sinh | Hãy Khơi thủy đầu một cách gọn lẹ nhẹ bình dị (Start simple) |
| Đưa bồi dắp nhét vô kể capabilities | Model sẽ rớt kẹt đứng sinh lú hỗn loạn tẩu hỏa (Model confusion) | Chọn dắt tay cho từ 3-5 phần món khả dụng khi bước bắt đầu xuất phát |
| Bộ Workflows máy móc bưng bít cứng khừ | Yếu rớt không đủ xoay vần dẻo dai thích biên | Vứt bỏ, giao việc phán xét quy trình lại cho đích tay con hệ the model được quyết chỉ định |
| Đưa nhét dội tải trước kiến thức quá thể (Front-loaded) | Làm ngập cồng nghẹt nghẽn bloat béo căng Context bộ đệm | Được rót nạp duy trì thả xuống lúc yêu cầu truy hoán on-demand (Tải on-demand) |
| Rình rập siết bóp nhúng tay tiểu tiết (Micromanagement) | Tiệt tiêu gọt bào mài vứt năng sinh bộ não intelligence thông minh nhận định nhạy bén | Tôn vinh nhượng thác toàn bộ trọng tín trao the model xử sự |

## Các Nguồn Hệ Lực Tham Khảo

**Phần Triết Thuyết & Mảng Lý Luận**:
- `references/agent-philosophy.md` - Đâm vục khơi lặn sâu khám xét vì phương sự đâu khiến sức mạnh thực của các loại agents thành tác diệu năng hiệu quả

**Sự Thực Chế Quy Trình Mảng Tích Hợp Đưa Chạy (Implementation)**:
- `references/minimal-agent.py` - Hệ agent thi hành đầy trọn quy trình tóm gói toàn chỉnh (~80 dòng code lines)
- `references/tool-templates.py` - Biểu cấu các Capability definitions
- `references/subagent-pattern.py` - Tuyến tách khoang sự ngăn lập phân tỏa Context rạch mảng dứt điểm cô lập

**Phần Bộ Hệ Tạo Biểu Cấu Dựng Dàn Khung Scaffolding**:
- `scripts/init_agent.py` - Sự động khởi kiến tạo nẩy dự án mới dựng hình agent

## Nếp Suy Tư Nhập Tâm Nơi Hệ Của Thể Hệ Phôi Agent (The Agent Mindset)

**Từ Chỗ**: "Tôi cần xoay xở đánh vật kiểu gì đẩy cái con framework đó cày đập làm việc ra thành chi cái việc cấu hệ ra X?"
**Đến Lúc**: "Mình phải cởi mở tung đường cho chiếc the model mở khả năng giải nén để vờn sinh ra cho được chi thức năng của bản thể X?"

**Từ Chỗ**: "Quy trình thiết trình của riêng cho luồng mảng task này thì là gì đây?"
**Đến Lúc**: "Tự nó, thể mảng cấu capabilities khả dụng thứ chi cần thiết cung trao để tự tóm trọn gói thầu chặng đường đi tới cái đích này?"

Câu mã lập code xây mầm tốt nhất thường được đắp tạo ra trong tình cảnh chán ngắt cực đơn thuần. Chỉ là mấy Vòng loops sạch trần đơn giản. Giới mảng khả sinh Capabilities thật tinh bạch rõ sáng. Gói bọc Context thông lưu cực sạch không đóng màng. Thứ phép ma phù phép thì vốn nó đã chả bao giờ khuất nấp ở dạng đống giàn code đó.

**Nhanh tay cấp phát capabilities cùng gói nạp knowledge quy thuộc về nó nhường dọn lại cho bản mẫu model. Uỷ thác lòng tin tưởng toàn tâm ủy nhượng gán mọi thao lược đằng sau lại về cho nó tự gieo tự lo định định phần còn thấu còn gỡ tiếp diễn xoay.**
