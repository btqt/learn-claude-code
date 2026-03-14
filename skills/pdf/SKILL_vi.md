---
name: pdf
description: Xử lý các tệp PDF - trích xuất văn bản, tạo PDF, hợp nhất các tài liệu. Sử dụng khi người dùng yêu cầu đọc PDF, tạo PDF hoặc làm việc với các tệp PDF.
---

# Kỹ Năng Xử Lý PDF (PDF Processing Skill)

Bạn hiện có chuyên môn trong việc làm việc với tệp PDF. Hãy tuân theo các quy trình làm việc sau:

## Đọc PDF (Reading PDFs)

**Lựa chọn 1: Trích xuất văn bản nhanh (được ưu tiên)**
```bash
# Sử dụng pdftotext (poppler-utils)
pdftotext input.pdf -  # Xuất ra stdout
pdftotext input.pdf output.txt  # Xuất ra tệp

# Nếu không có sẵn pdftotext, hãy thử:
python3 -c "
import fitz  # PyMuPDF
doc = fitz.open('input.pdf')
for page in doc:
    print(page.get_text())
"
```

**Lựa chọn 2: Từng trang một cùng với siêu dữ liệu (metadata)**
```python
import fitz  # pip install pymupdf

doc = fitz.open("input.pdf")
print(f"Số trang (Pages): {len(doc)}")
print(f"Siêu dữ liệu (Metadata): {doc.metadata}")

for i, page in enumerate(doc):
    text = page.get_text()
    print(f"--- Trang {i+1} ---")
    print(text)
```

## Tạo PDF (Creating PDFs)

**Lựa chọn 1: Từ Markdown (khuyên dùng)**
```bash
# Sử dụng pandoc
pandoc input.md -o output.pdf

# Với việc định dạng (styling) tùy chỉnh
pandoc input.md -o output.pdf --pdf-engine=xelatex -V geometry:margin=1in
```

**Lựa chọn 2: Bằng lập trình (Programmatically)**
```python
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

c = canvas.Canvas("output.pdf", pagesize=letter)
c.drawString(100, 750, "Hello, PDF!")
c.save()
```

**Lựa chọn 3: Từ HTML**
```bash
# Sử dụng wkhtmltopdf
wkhtmltopdf input.html output.pdf

# Hoặc bằng Python
python3 -c "
import pdfkit
pdfkit.from_file('input.html', 'output.pdf')
"
```

## Hợp nhất PDF (Merging PDFs)

```python
import fitz

result = fitz.open()
for pdf_path in ["file1.pdf", "file2.pdf", "file3.pdf"]:
    doc = fitz.open(pdf_path)
    result.insert_pdf(doc)
result.save("merged.pdf")
```

## Chia tách PDF (Splitting PDFs)

```python
import fitz

doc = fitz.open("input.pdf")
for i in range(len(doc)):
    single = fitz.open()
    single.insert_pdf(doc, from_page=i, to_page=i)
    single.save(f"page_{i+1}.pdf")
```

## Thư viện chính (Key Libraries)

| Tác vụ (Task) | Thư viện (Library) | Cài đặt (Install) |
|------|---------|---------|
| Đọc/Ghi/Hợp nhất (Read/Write/Merge) | PyMuPDF | `pip install pymupdf` |
| Tạo mới từ đầu (Create from scratch) | ReportLab | `pip install reportlab` |
| HTML sang PDF | pdfkit | `pip install pdfkit` + wkhtmltopdf |
| Trích xuất văn bản | pdftotext | `brew install poppler` / `apt install poppler-utils` |

## Thực tiễn Tốt nhất (Best Practices)

1. **Luôn kiểm tra xem công cụ đã được cài đặt chưa** trước khi sử dụng chúng
2. **Xử lý các vấn đề về mã hóa (encoding)** - PDF có thể chứa nhiều loại mã hóa ký tự (character encodings) khác nhau
3. **PDF dung lượng lớn**: Xử lý từng trang để tránh lỗi bộ nhớ
4. **OCR cho PDF dạng hình ảnh scan**: Sử dụng `pytesseract` nếu quá trình trích xuất văn bản trả về kết quả trống
