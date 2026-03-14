# Triết lý của các Agent

> **Mô hình đã biết cách trở thành một agent. Công việc của bạn là tránh ra khỏi đường đi của nó.**

## Sự Thật Cốt Lõi

Loại bỏ mọi Framework, mọi thư viện, mọi mẫu kiến trúc. Những gì còn lại là gì?

Một vòng lặp. Một mô hình. Một lời mời hành động.

Agent không phải là mã nguồn. Agent chính là bản thân mô hình - một mạng nơ-ron rộng lớn được đào tạo dựa trên khả năng giải quyết vấn đề, suy luận và sử dụng công cụ tập thể của nhân loại. Mã nguồn chỉ cung cấp cơ hội để mô hình thể hiện agent của nó.

## Tại Sao Điều Này Quan Trọng

Hầu hết các triển khai agent thất bại không phải do quá ít kỹ thuật, mà là do quá nhiều. Chúng ràng buộc. Chúng quy định. Chúng nghi ngờ chính trí thông minh mà chúng đang cố gắng tận dụng.

Hãy xem xét: Mô hình đã được đào tạo trên hàng triệu ví dụ về cách giải quyết vấn đề. Nó đã thấy cách các chuyên gia tiếp cận các nhiệm vụ phức tạp, cách sử dụng các công cụ, cách hình thành và sửa đổi các kế hoạch. Kiến thức này đã có sẵn ở đó, được mã hóa trong hàng tỷ tham số.

Công việc của bạn không phải là dạy nó cách suy nghĩ. Công việc của bạn là cung cấp cho nó phương tiện để hành động.

## Ba Yếu Tố

### 1. Capabilities (Tools)

Capabilities trả lời câu hỏi: **Agent có thể LÀM gì?**

Chúng là đôi bàn tay của mô hình - khả năng tác động đến thế giới của nó. Không có Capabilities, mô hình chỉ có thể giao tiếp. Với chúng, nó có thể hành động.

**Nguyên lý thiết kế**: Mỗi capability nên mang tính nguyên tử, rõ ràng và được mô tả tốt. Mô hình cần hiểu mỗi capability làm gì, nhưng không cần biết cách sử dụng chúng theo trình tự - nó sẽ tự tìm ra điều đó.

**Sai lầm phổ biến**: Quá nhiều Capabilities. Mô hình trở nên bối rối, bắt đầu sử dụng sai, hoặc tê liệt vì quá nhiều sự lựa chọn. Hãy bắt đầu với 3-5 capabilities. Chỉ thêm nhiều hơn khi mô hình liên tục thất bại trong việc hoàn thành các nhiệm vụ vì thiếu một capability.

### 2. Kiến Thức (Skills)

Kiến thức trả lời: **Agent BIẾT gì?**

Đây là chuyên môn lĩnh vực - sự hiểu biết chuyên sâu biến một trợ lý chung thành một chuyên gia lĩnh vực. Một agent dịch vụ khách hàng cần biết các chính sách của công ty. Một agent nghiên cứu cần biết phương pháp luận. Một agent sáng tạo cần biết các nguyên tắc về phong cách.

**Nguyên lý thiết kế**: Bơm kiến thức theo yêu cầu, không phải tất cả vào lúc đầu. Mô hình không cần biết mọi thứ cùng một lúc - chỉ những gì liên quan đến nhiệm vụ hiện tại. Sự tiết lộ lũy tiến bảo tồn ngữ cảnh cho những gì quan trọng.

**Sai lầm phổ biến**: Tải trước tất cả kiến thức có thể có vào lời nhắc hệ thống (system prompt). Điều này lãng phí ngữ cảnh, làm bối rối mô hình và làm cho mọi tương tác trở nên đắt đỏ. Thay vào đó, hãy làm cho kiến thức có sẵn nhưng không bắt buộc.

### 3. Ngữ Cảnh (The Conversation)

Ngữ cảnh là ký ức của cuộc tương tác - những gì đã được nói, những gì đã được thử, những gì đã được học. Nó là sợi dây kết nối các hành động riêng lẻ thành một hành vi gắn kết.

**Nguyên lý thiết kế**: Ngữ cảnh rất quý giá. Hãy bảo vệ nó. Cô lập các nhiệm vụ phụ tạo ra nhiễu. Cắt bớt các kết quả vượt quá tính hữu dụng. Tóm tắt khi lịch sử kéo dài.

**Sai lầm phổ biến**: Để ngữ cảnh phát triển không giới hạn, lấp đầy nó bằng các chi tiết khám phá, các nỗ lực thất bại và các kết quả công cụ dài dòng. Cuối cùng, mô hình không thể tìm thấy tín hiệu trong đống nhiễu.

## Mẫu Phổ Quát

Mọi agent hiệu quả - bất kể lĩnh vực, Framework, hay cách triển khai - đều tuân theo cùng một mẫu:

```
LOOP:
  Model sees: conversation history + available capabilities
  Model decides: act or respond
  If act: capability executed, result added to context, loop continues
  If respond: answer returned, loop ends
```

Đây không phải là một sự đơn giản hóa. Đây là kiến trúc thực tế. Mọi thứ khác đều là sự tối ưu hóa.

## Thiết Kế cho Agency

### Tin Tưởng Mô Hình

Nguyên lý quan trọng nhất: **tin tưởng mô hình**.

Đừng cố gắng lường trước mọi trường hợp biên. Đừng xây dựng các cây quyết định phức tạp. Đừng xác định trước quy trình làm việc (workflow).

Mô hình có khả năng suy luận tốt hơn bất kỳ hệ thống quy tắc nào bạn có thể viết. Logic có điều kiện của bạn sẽ thất bại với các trường hợp biên. Mô hình sẽ suy luận qua chúng.

**Hãy cung cấp cho mô hình capabilities và kiến thức. Hãy để nó tự tìm ra cách sử dụng chúng.**

### Ràng Buộc Kích Hoạt

Điều này có vẻ nghịch lý, nhưng các ràng buộc không giới hạn các agents - chúng tập trung chúng.

Một danh sách cần làm (todo list) với "chỉ một nhiệm vụ đang tiến hành" buộc sự tập trung tuần tự. Một subagent với "quyền truy cập chỉ xem" ngăn chặn các sửa đổi ngẫu nhiên. Một phản hồi với "dưới 100 từ" đòi hỏi sự rõ ràng.

Những ràng buộc tốt nhất là những ràng buộc ngăn mô hình bị lạc, chứ không phải những ràng buộc quản lý vi mô (micromanage) cách tiếp cận của nó.

### Độ Phức Tạp Lũy Tiến

Đừng bao giờ xây dựng mọi thứ ngay từ đầu.

```
Level 0: Model + one capability
Level 1: Model + 3-5 capabilities
Level 2: Model + capabilities + planning
Level 3: Model + capabilities + planning + subagents
Level 4: Model + capabilities + planning + subagents + skills
```

Bắt đầu ở cấp độ thấp nhất có thể hoạt động. Chỉ thăng cấp khi việc sử dụng thực tế cho thấy sự cần thiết. Hầu hết các agents không bao giờ cần vượt qua Level 2.

## Tư Duy Agent

Việc xây dựng agents đòi hỏi sự thay đổi trong suy nghĩ:

**Từ**: "Làm thế nào để tôi làm cho hệ thống thực hiện X?"
**Sang**: "Làm thế nào để tôi cho phép mô hình thực hiện X?"

**Từ**: "Điều gì nên xảy ra khi người dùng nói Y?"
**Sang**: "Capabilities nào sẽ giúp giải quyết Y?"

**Từ**: "Workflow cho nhiệm vụ này là gì?"
**Sang**: "Mô hình cần gì để tự tìm ra workflow?"

Mã agent tốt nhất gần như nhàm chán. Các vòng lặp đơn giản. Các định nghĩa capability rõ ràng. Quản lý ngữ cảnh sạch sẽ. Phép thuật không nằm trong mã nguồn - nó nằm trong mô hình.

## Nền Tảng Triết Học

### Mô Hình Như Agent Nổi Lên (Emergent Agent)

Các mô hình ngôn ngữ được đào tạo dựa trên văn bản của con người không chỉ học ngôn ngữ, mà còn các mô hình tư duy. Chúng đã tiếp thu cách con người tiếp cận vấn đề, sử dụng công cụ và hoàn thành mục tiêu. Đây là agency nổi lên - không được lập trình, mà được học.

Khi bạn cung cấp cho mô hình capabilities, bạn không dạy nó trở thành một agent. Bạn đang cho nó quyền thể hiện agency mà nó vốn đã có.

### Vòng Lặp Là Sự Giải Phóng

Vòng lặp agent đơn giản một cách lừa dối: nhận phản hồi, kiểm tra việc sử dụng công cụ, thực thi, lặp lại. Nhưng chính sự đơn giản này là sức mạnh của nó.

Vòng lặp không giới hạn mô hình vào các chuỗi cụ thể. Nó không ép buộc các workflows cụ thể. Nó chỉ đơn giản nói: "Bạn có capabilities. Sử dụng chúng khi bạn thấy phù hợp. Tôi sẽ thực thi những gì bạn yêu cầu và cho bạn xem kết quả."

Đây là sự giải phóng, không phải sự hạn chế.

### Capabilities Như Biểu Hiện

Mỗi capability mà bạn cung cấp là một hình thức biểu hiện của mô hình. "Đọc tệp" cho phép nó nhìn thấy. "Ghi tệp" cho phép nó tạo ra. "Tìm kiếm" cho phép nó khám phá. "Gửi tin nhắn" cho phép nó giao tiếp.

Nghệ thuật trong việc thiết kế agent là việc chọn các hình thức biểu hiện nào được kích hoạt. Quá ít, và mô hình câm lặng. Quá nhiều, và nó nói bằng những thứ ngôn ngữ khó hiểu.

## Kết Luận

Agent là mô hình. Mã nguồn chỉ là vòng lặp. Công việc của bạn là tránh sang một bên.

Cung cấp cho mô hình capabilities rõ ràng. Làm cho kiến thức có sẵn khi cần. Bảo vệ ngữ cảnh khỏi nhiễu loạn. Tin tưởng vào mô hình để nó tự làm phần còn lại.

Đó là tất cả. Đó chính là triết lý.

Mọi thứ còn lại chỉ là sự tinh chỉnh.
