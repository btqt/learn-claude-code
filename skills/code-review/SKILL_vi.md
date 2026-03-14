---
name: code-review
description: Thực hiện các đánh giá mã nguồn (code reviews) toàn diện với phân tích bảo mật, hiệu suất và khả năng bảo trì. Sử dụng khi người dùng yêu cầu đánh giá mã nguồn, kiểm tra lỗi (bugs), hoặc kiểm tra toàn bộ (audit) cơ sở mã nguồn (codebase).
---

# Kỹ Năng Đánh Giá Mã (Code Review Skill)

Bạn hiện có chuyên môn trong việc thực hiện các code reviews toàn diện. Hãy tuân theo cách tiếp cận có cấu trúc sau:

## Danh Sách Kiểm Tra Đánh Giá (Review Checklist)

### 1. Bảo mật (Nghiêm trọng)

Kiểm tra:
- [ ] **Lỗ hổng tấn công tiêm nhiễm (Injection vulnerabilities)**: SQL, command, XSS, template injection
- [ ] **Các vấn đề xác thực (Authentication issues)**: Hardcoded credentials (thông tin đăng nhập cố định), xác thực yếu
- [ ] **Lỗ hổng phân quyền (Authorization flaws)**: Thiếu các điều khiển truy cập, IDOR
- [ ] **Lộ lọt dữ liệu (Data exposure)**: Dữ liệu nhạy cảm xuất hiện trong logs, các thông báo lỗi
- [ ] **Mật mã (Cryptography)**: Các thuật toán yếu, quản lý khóa mã không đúng chuẩn
- [ ] **Các thành phần phụ thuộc (Dependencies)**: Các lỗ hổng bảo mật đã biết (kiểm tra bằng lệnh `npm audit`, `pip-audit`)

```bash
# Quét tìm lỗi bảo mật nhanh
npm audit                    # Node.js
pip-audit                    # Python
cargo audit                  # Rust
grep -r "password\|secret\|api_key" --include="*.py" --include="*.js"
```

### 2. Sự Chuẩn Xác (Correctness)

Kiểm tra:
- [ ] **Lỗi logic (Logic errors)**: Lỗi trượt một nhịp (Off-by-one), lỗi quản trị giá trị rỗng (null handling), các trường hợp bên rìa ngoại lệ (edge cases)
- [ ] **Sự cạnh tranh lỗi xung tín (Race conditions)**: Xảy ra màng kẹp gọi truy xuất chung đồng hồi mà vắng tính điều hòa khóa đồng bộ
- [ ] **Rò Rỉ Mất Mát Hệ Tài Nguyên (Resource leaks)**: Những thư mục files chưa dập đóng hoàn toàn, luồng ống kết nối chập chờn rò, vùng giữ không gian cấp bộ nhớ chập lãng
- [ ] **Xử Lí Giải Trình Gỡ Vướng Trở Ngại (Error handling)**: Đống lầm nhiễu lấp ngấm hỏng mất những báo vướng exceptions, dập đi bót mất nấc dấu nhánh báo tín lỗi sai hỏng đường paths
- [ ] **Tuyệt an cho nền gốc dạng dữ (Type safety)**: Khuất lộ ngấm ngầm tàng che mặt định chuyển tự Implicit conversions, mọi khung lỏng chắp any types

### 3. Hiệu Suất (Performance)

Kiểm tra:
- [ ] **Lệnh Gọi Truy Vấn Lặp Bóng Phá Dư N+1 (N+1 queries)**: Hàng chùm ngàn lệnh điều truy tìm móc nối gọi vô mảng nền Cơ Sỡ Dữ Liệu (Database calls) ngốn kẹp vòng vo loops
- [ ] **Bài Toán Sự Cố Mảng Bộ Nhớ (Memory issues)**: Khoét phễu to cho mảng lớn allocations, khựng đóng chốt chằng bị buộc tham chiếu đính mốc References bị cùm
- [ ] **Trì Trệ Vấp Dịch Chặn Đứng Khóa (Blocking operations)**: Gài cắm dòng thao tắc nghẽn tắc trệ Sync tính dứt điểm ở chóp đuôi I/O vô ngay lòng bụng Async mảng code
- [ ] **Thâm Thụt Dấu Vết Trơn Tru Của Thuật Phân Giải (Inefficient algorithms)**: Áp ngay phương diện kiểu vận thuật toán O(n^2) trong bối cảnh lề nhượng cho O(n) còn rất nhẽ sống có thể khả dụng possible
- [ ] **Che Khuyết Thiếu Bộ Tản Dựng Cache (Missing caching)**: Cứ lặp rình rập những bài đánh bắt tính rà lại nặng đè cực hao tốn tài lực Repeated expensive computations

### 4. Năng Lực Bảo Dưỡng Duy Trì (Maintainability)

Kiểm tra:
- [ ] **Trạch Tên Gọi (Naming)**: Trong ngần trong suốt rõ (Clear), trọn tính bất tận xuyên thông suốt (consistent), lột tả bóc mảng trình thấu (descriptive)
- [ ] **Tiêu Thức Độ Quy Mảng Rối Loạn (Complexity)**: Các hệ khối Functions quá 50 lines, sự chồng xếp ngầm gài khoanh khoang lớp độ cuộn lún ngập trũng deep nesting > 3 tầng levels
- [ ] **Sự Tái Khớp Lại Sự Sao Dựng Lại Vết Mòn (Duplication)**: Hiện hữu mảng các trích tệp bị khuân nguyên xi tạc khắc rập sao in copy lại dán Copy-pasted code blocks
- [ ] **Mảng Mã Tử Chết (Dead code)**: Tồn tại vạ vất vật cản chướng vô nghĩa không lôi ra làm xài nhập Imports, hoặc là các phân luồng ngách chạy hoài vấp cụt ngõ tới unreachable branches
- [ ] **Vùng Định Trích Yếu Nhắc Diễn Lời Bình (Comments)**: Hoặc thiu rũ rêu cổ hủ (Outdated), dư sinh tạp nham tràng giang (redundant), ngõ đứt thiếu bóng hẳn khi vả chăng cần xướng chép có ích lợi hiện vóc

### 5. Kiểm Nghiệm (Testing)

Kiểm tra:
- [ ] **Biên Trùm Phủ Kín (Coverage)**: Tất cả những trục chốt lõi ngõ sống chết xương sống tủy cần bợ phải được lôi ra thẩm tra Tested đầy đủ
- [ ] **Nhánh Mép Ngụy Thác Trắc Ẩn Tình Cầu Khó Kiểm (Edge cases)**: Sự thả rơi giá trị vắng bặt vô hiệu Null, khoang rỗng, các tham biên báo độ mớm dốc đứng lằn mức chóp đỉnh boundary values
- [ ] **Các Ngõ Sự Tạo Điểm Đóng Giả Lập Cô Mạch Nhúng Thử (Mocking)**: Áp giải khống tỏa cách biệt tách chia External dependencies ra làm điểm nhốt trọn isolated
- [ ] **Kiểm Điểm Minh Định Xác Năng Khám Thác (Assertions)**: Toát hiển lộ chất hữu ý sâu nặng meaningfull, phác trình quy cụ rõ chi li xác mảng nhắm điểm specific checks

## Khung Mẫu Định Dạng Phơi Trình Xuất Tác Đánh Thẩm Cứu Định Code (Review Output Format)

```markdown
## Đánh giá Code (Code Review): [tên tệp/tên component]

### Tóm lược (Summary)
[1-2 dòng văn đoạn khái lược tổng thuật tổng quan]

### Điểm Vướng Gắng Có Tính Chốt Báo Chết Lỗi Nặng Cốt Tử (Critical Issues)
1. **[Tình vướng Lỗi ngặt nghẽo kịch độc (Issue)]** (tại mốc dòng line X): [Sự bóc diễn Mô Tả]
   - Đường Báo Nguy Gây Đâm Dựng Nào (Impact): [Nguy hại chi điều rủi sinh sẽ gây đâm xộc sai hỏng hóc lớn (What could go wrong)]
   - Lối Phất Cứu Gỡ Tình (Fix): [Thang chỉ chốt lại một chước lối cứu vãn (Suggested solution)]

### Những Quả Quả Trái Đề Định Được Nhắc Khuyên Tư Cải Thiện Thăng Phát (Improvements)
1. **[Lời Đề Xin Khuyên Cáo (Suggestion)]** (mốc ngõ dòng line X): [Khung Mô diễn Mô tả]

### Gạch Nét Tích Cực Ghi Nhận (Positive Notes)
- [Điểm mốc đánh được tán tán làm tuyệt tốt hay xuất sắc thành công]

### Quyết Phán Nơi Cửa Lệnh Đi Vào Merge (Verdict)
[ ] Định dứt điểm ngon nghẻ sẵn tay mở (Ready to merge)
[ ] Tính trớ sửa tút lại tí mọn chỉnh nhẹ xíu dặm biên (Needs minor changes)
[ ] Móc móc cần bóc đại phẫu xét thấu rò đập phang tháo độ lớn đổi sửa (Needs major revision)
```

## Các Mảng Hiện Điển Hình Quy Chiếu Hay Cần Lột Rẽ Áp Vạch Sự Gọi Phải Cảnh Giác Đánh Động Chỉ Tỏ Rõ Nhãn Dấu Flag (Common Patterns to Flag)

### Python
```python
# Đại kỵ Bậy (Bad): Trúng trỏ SQL injection
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
# Trúng điệu Tốt (Good):
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))

# Bậy bạ (Bad): Sự gài mớm lệnh nhập hại command injection
os.system(f"ls {user_input}")
# Đúng bóc (Good):
subprocess.run(["ls", user_input], check=True)

# Hihi hãm lồng quá (Bad): Các mớ biến Đối số gán gởi quy hàm mặc định hở bung phập phồng (Mutable default argument)
def append(item, lst=[]):  # Cấy Rệp Bug: Bức cấu tạo chung đùn xài phập phồng mặc định chia đổi san nhau (shared mutable default)
# Đúng đắn Tốt (Good):
def append(item, lst=None):
    lst = lst or []
```

### JavaScript/TypeScript
```javascript
// Dở tệ gài thối (Bad): Bơm ngụy nhiễm khuếch độc khuôn mẫu nguyên thủy (Prototype pollution)
Object.assign(target, userInput)
// Tuyệt chuẩn (Good):
Object.assign(target, sanitize(userInput))

// Bậy độc dâm (Bad): Kẹp xài lấy eval vô tội vạ
eval(userCode)
// Chóp Tối Thượng Tốt đỉnh (Good): Chẳng đời nào, Không muôn thuở được khơi xài rước mặt thằng eval dính bọc đệm tới vùng mảng của nợ nhập tiếp lấy vào bởi người dùng User input

// Siêu Dở Nhão đứt chán (Bad): Sự lọt bám đu dây móc gọi văng Callback hell
getData(x => process(x, y => save(y, z => done(z))))
// Êm Chắc (Good):
const data = await getData();
const processed = await process(data);
await save(processed);
```

## Lệnh Chạy Cấu Thao Gọi Giúp Hỗ Trợ Đánh Giá Kiểm Duyệt (Review Commands)

```bash
# Phơi vóc diễn nét dáng những màn xáo nhào thay thế gần nhất hiện thời (Show recent changes)
git diff HEAD~5 --stat
git log --oneline -10

# Bới tung đào sới ra những góc kẹt mầm ấp bạo loạn (Find potential issues)
grep -rn "TODO\|FIXME\|HACK\|XXX" .
grep -rn "password\|secret\|token" . --include="*.py"

# Test thử xới coi xem độ cuộn nhồi mức rối đan Complexity là tới đâu (áp trù Python)
pip install radon && radon cc . -a

# Test coi xem mấy cái thứ phụ nhằng phụ thuộc vạ vật ở kề lề ngoài (Check dependencies)
npm outdated  # Node
pip list --outdated  # Python
```

## Guồng Lưu Vận Luân Chuyển Chuỗi (Review Workflow)

1. **Am Thấu bọc gộp Context (Understand context)**: Ngấu nghiến cho dứt trọn nấc nội mô PR description, cả đám móc nối issues bị ghim Linked
2. **Ra mạn nện vận chạy xách the code ra trui rèn (Run the code)**: Build ra dựng đứng lên, test coi trúng trật xách, bật tại chỗ vọc chơi lủi cục bộ run locally in máy test hễ có cách if possible
3. **Mòi xơi rạch ròi dứt dạt chóp đến tận rễ ngọn (Read top-down)**: Đi bộ từ đỉnh đầu ngách main mạch ngõ nhập cảnh chính xộc dô lề entry points
4. **Áp trù mớ Tests kiêm duyệt**: Bộ các pha quậy changes có kinh qua bẫy nẹt hầm test kiểm thử khâu chưa rứa? Các khâu kiêm tests có đậu ngã xuôi lôi trót vát thoát rành mạch Pass nổi không chứ?
5. **Rà lưới trùm tóm bủa diệt mấu mảng nọc nanh (Security scan)**: Ấn nện cắm nút khởi động phóng hàng mớ cỗ máy soi mói bọc dò móng bảo mật tự động automated tools
6. **Xách sức trâu phán thẩm ròi xoi lấy tay quất gõ đánh soát tự bằng mắt (Manual review)**: Xách theo mớ cái bản tóm yết checklist giơ bảng gõ đồn trạm ở phia trên ra mà điểm tích vô danh mục chỉ soát (Use checklist above)
7. **Đúc thảo viết diễn mảng miếng bồi nạp rày yết góp trỏ mưu cáo feedback (Write feedback)**: Bắt rõ đích đến bắt chính nhắm ngõ chốt mấu (Be specific), nhả hột ban trao ra ý cớ dứt đề cứu vãn khuyên chữa (suggest fixes), làm người ôn hòa mở mở nhã thanh hòa bớt độc mồm chua lè ráng đàng hoàng thiện lương lịch lãm thân dâng cống tận (be kind)
