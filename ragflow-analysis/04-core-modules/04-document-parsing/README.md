# 文档解析模块 (Document Parsing Module)

## 模块概述

文档解析模块是 RAGFlow 的内容提取核心，负责将各种格式的文档（PDF、Word、Excel、PPT、图片等）解析为结构化文本。支持 OCR 识别、表格提取、版面分析、多语言支持等高级功能。

### 核心价值
- **多格式支持**：支持 10+ 种文档格式解析
- **深度理解**：OCR、表格提取、版面分析一体化
- **高精度**：基于 PaddleOCR、Tesseract 等先进引擎
- **可配置**：灵活的解析策略和参数配置

---

## 功能清单

### 1. 文档解析器 (Parsers)
- **PDF Parser** - PDF 文档解析（pdfplumber + PyMuPDF）
- **Word Parser** - Word 文档解析（python-docx）
- **Excel Parser** - Excel 表格解析（openpyxl + pandas）
- **PPT Parser** - PowerPoint 解析（python-pptx）
- **Markdown Parser** - Markdown 解析（mistune）
- **HTML Parser** - HTML 网页解析（BeautifulSoup）
- **Email Parser** - 邮件解析（email.parser）
- **JSON/XML Parser** - 结构化数据解析

### 2. OCR 识别引擎
- **PaddleOCR** - 中英文 OCR（默认）
- **Tesseract** - 多语言 OCR
- **RapidOCR** - 轻量级 OCR
- **EasyOCR** - 深度学习 OCR
- **版面分析** - 检测文本、图片、表格区域

### 3. 表格提取
- **TableStructureRecognizer** - 基于深度学习的表格结构识别
- **Table Transformer** - 表格组件检测 (Header/Row/Column)
- **结构化输出** - Markdown、HTML 格式

### 4. 图像处理
- **图像提取** - 从文档提取图片
- **图像增强** - 去噪、二值化、倾斜校正
- **图像 OCR** - 识别图片中的文字
- **图表识别** - 识别图表并提取数据

---

## 目录结构

```
deepdoc/
  ├── parser/
  │   ├── pdf_parser.py                    # PDF 解析 (RAGFlowPdfParser/PlainParser/VisionParser)
  │   ├── docx_parser.py                   # Word 解析
  │   ├── excel_parser.py                  # Excel 解析
  │   ├── ppt_parser.py                    # PPT 解析
  │   ├── markdown_parser.py               # Markdown 解析
  │   ├── html_parser.py                   # HTML 解析
  │   ├── json_parser.py                   # JSON 解析
  │   ├── txt_parser.py                    # 纯文本解析
  │   ├── figure_parser.py                 # 图片解析
  │   ├── mineru_parser.py                 # MinerU 解析器
  │   ├── docling_parser.py                # Docling 解析器
  │   └── tcadp_parser.py                  # 腾讯云 ADP 解析器
  ├── vision/
  │   ├── layout_recognizer.py             # 版面识别 (LayoutRecognizer/YOLOv10/Ascend)
  │   ├── ocr.py                           # OCR 引擎 (TextDetector/TextRecognizer/OCR)
  │   ├── table_structure_recognizer.py    # 表格结构识别
  │   └── recognizer.py                    # 通用识别器基类
  └── README.md
```

---

## 技术栈

### PDF 处理
- **pdfplumber** - 文本提取和页面渲染
- **PyMuPDF (fitz)** - 高性能 PDF 处理、图像提取、大纲解析
- **MinerU** - 高精度 PDF 解析引擎 (可选)
- **Docling** - 高级 PDF 解析引擎 (可选)
- **TCADP** - 腾讯云文档解析服务 (可选)

### OCR 引擎
- **PaddleOCR** - 主力 OCR 引擎
- **Tesseract** - 备选引擎
- **OpenCV** - 图像预处理

### 文档处理
- **python-docx** - Word 处理
- **openpyxl** - Excel 处理
- **python-pptx** - PPT 处理
- **BeautifulSoup4** - HTML 解析
- **mistune** - Markdown 解析

---

## 核心流程

### PDF 解析流程
1. **文件加载** - 打开 PDF 文件
2. **页面遍历** - 逐页处理
3. **文本提取** - 提取文本内容
4. **表格提取** - 识别并提取表格
5. **图像提取** - 提取图片并 OCR
6. **版面分析** - 分析文档结构
7. **内容合并** - 组装完整内容

### OCR 识别流程
1. **图像预处理** - 去噪、增强、二值化
2. **版面分析** - 检测文本区域
3. **文字识别** - OCR 识别文字
4. **后处理** - 文本校正和格式化
5. **结构化输出** - 输出 JSON/Markdown

---

## 配置参数

### Parser 配置
```json
{
  "chunk_token_num": 128,
  "layout_recognize": true,
  "raptor": {
    "use_raptor": false
  },
  "filename_embd_weight": 0.1,
  "task_page_size": 12
}
```

### OCR 配置
```python
{
  "lang": ["ch", "en"],
  "use_angle_cls": true,
  "use_gpu": true,
  "det_model": "ch_PP-OCRv3_det",
  "rec_model": "ch_PP-OCRv3_rec"
}
```

---

## 性能优化

### 并行处理
- 多进程页面解析
- 批量 OCR 识别
- 异步文件读取

### 缓存策略
- 解析结果缓存
- OCR 模型缓存
- 图像预处理缓存

---

## 相关文档

- [PDF 解析时序图](./01-pdf-parsing-sequence.puml)
- [OCR 识别时序图](./02-ocr-recognition-sequence.puml)
- [表格提取时序图](./03-table-extraction-sequence.puml)
