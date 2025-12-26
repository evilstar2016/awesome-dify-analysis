# 文档处理功能概览

## 1. 支持的文档类型

FastGPT 的知识库管理系统支持 **8 种文档类型**的处理：

| 文档类型 | 文件扩展名 | 解析库 | 主要功能 |
|---------|-----------|--------|---------|
| 纯文本 | `.txt`, `.md` | iconv-lite | 编码检测和转换 |
| HTML | `.html`, `.htm` | cheerio, turndown | HTML转Markdown |
| PDF | `.pdf` | pdfjs-dist | 文本提取，支持自定义解析服务 |
| Word文档 | `.docx` | mammoth | DOCX转Markdown，图片提取 |
| Excel表格 | `.xlsx`, `.xls` | node-xlsx | 表格转CSV/Markdown |
| CSV文件 | `.csv` | papaparse | CSV转Markdown表格 |
| PowerPoint | `.pptx` | xmldom, decompress | 幻灯片文本提取 |

## 2. 架构设计

### 2.1 Worker 线程架构

文档解析采用 **Worker 线程** 模式，实现主线程和解析线程的分离：

```
主线程 (Main Thread)
  ↓ SharedArrayBuffer (零拷贝)
Worker 线程 (Parse Thread)
  ↓ 路由到对应解析器
解析器 (Extension Parser)
  ↓ 调用解析库
文本提取 (Text Extraction)
```

**优势：**
- 🚀 **零拷贝传输**: 使用 SharedArrayBuffer 在线程间共享内存
- 🔄 **异步非阻塞**: 主线程不会被文件解析任务阻塞
- 💪 **高性能**: CPU 密集型任务在独立线程执行
- 🛡️ **稳定性**: Worker 崩溃不会影响主进程

### 2.2 核心代码路径

```
packages/service/
├── worker/
│   ├── readFile/
│   │   ├── index.ts              # Worker线程入口
│   │   ├── type.ts                # 类型定义
│   │   ├── parseOffice.ts         # Office文件解压解析
│   │   └── extension/             # 各类文档解析器
│   │       ├── rawText.ts         # 纯文本 (txt, md)
│   │       ├── html.ts            # HTML文档
│   │       ├── pdf.ts             # PDF文档
│   │       ├── docx.ts            # Word文档
│   │       ├── xlsx.ts            # Excel表格
│   │       ├── csv.ts             # CSV文件
│   │       └── pptx.ts            # PowerPoint
│   └── htmlStr2Md/
│       └── utils.ts               # HTML转Markdown工具
└── common/
    └── file/
        └── read/
            └── utils.ts           # 高级文件读取(自定义PDF解析)
```

## 3. 文档解析流程

### 3.1 通用流程

```
1. 文件上传
   ↓
2. 确定文件类型（扩展名）
   ↓
3. 创建Worker线程
   ↓
4. 通过SharedArrayBuffer传输文件Buffer
   ↓
5. Worker线程路由到对应解析器
   ↓
6. 解析器调用库提取文本
   ↓
7. 返回解析结果
   {
     rawText: string,      // 原始文本
     formatText?: string,  // 格式化文本（如Markdown表格）
     imageList?: Array     // 提取的图片列表
   }
   ↓
8. 文本分块（Chunking）
   ↓
9. 向量化存储
```

### 3.2 特殊处理

#### PDF 三级解析策略

1. **系统解析** (默认)：使用 pdfjs-dist 库
2. **自定义解析服务**：通过 `customPdfParse` 外部 API
3. **Doc2X 服务**：高级 PDF 解析（表格、公式识别）

#### 图片处理

文档类型 | 图片提取 | 存储方式
---------|---------|----------
DOCX | ✅ | Base64 + UUID
HTML | ✅ | Base64 + UUID
TXT/MD | ✅ (Markdown图片) | URL + UUID
PDF | ❌ (可选自定义服务) | -
XLSX | ❌ | -
CSV | ❌ | -
PPTX | ❌ | -

## 4. 解析结果格式

### 4.1 标准输出

```typescript
interface ReadFileResponse {
  rawText: string;          // 原始提取的文本
  formatText?: string;      // 格式化后的文本（如Markdown表格）
  imageList?: ImageType[];  // 图片列表
}

interface ImageType {
  uuid: string;     // 唯一标识符
  base64: string;   // Base64编码的图片数据
  mime: string;     // MIME类型 (如 image/png)
}
```

### 4.2 不同文档类型的输出特点

| 文档类型 | rawText | formatText | imageList |
|---------|---------|-----------|-----------|
| TXT/MD | 文本内容 | - | Markdown图片 |
| HTML | Markdown文本 | - | Base64图片 |
| PDF | 纯文本 | - | - |
| DOCX | Markdown文本 | - | Base64图片 |
| XLSX | CSV格式 | Markdown表格 | - |
| CSV | CSV格式 | Markdown表格 | - |
| PPTX | 纯文本 | - | - |

## 5. 编码处理

系统支持多种文件编码：

**原生编码**（Node.js Buffer 支持）：
- UTF-8, UTF-16LE, ASCII, Latin1, Base64, Hex

**扩展编码**（iconv-lite 库）：
- GBK, GB2312, Big5, Shift-JIS, ISO-8859-* 等

**自动降级**：
- 当指定编码解析失败时，自动回退到 UTF-8

## 6. 性能优化

### 6.1 PDF 优化

- **页面级处理**: 逐页加载，避免内存占用
- **内存清理**: 及时释放 PDF 页面对象
- **头尾过滤**: 过滤页眉页脚（5% 阈值）
- **EOL 检测**: 智能识别段落边界

### 6.2 HTML 优化

- **大小限制**: HTML 超过 100KB 时直接返回原始内容
- **Base64 替换**: 将内联图片替换为 UUID，避免内存膨胀
- **标签移除**: 移除 `<i>`, `<script>`, `<iframe>`, `<style>` 等无用标签

### 6.3 Office 文件优化

- **流式解析**: PPTX 使用 XML 流式解析
- **临时文件清理**: 解压后立即删除临时文件
- **排序处理**: PPTX 幻灯片按序号排序

## 7. 错误处理

```typescript
// 编码降级
try {
  return iconv.decode(buffer, encoding);
} catch (error) {
  return buffer.toString('utf-8');
}

// 解析失败降级
const text = await (async () => {
  try {
    return await parseDocument();
  } catch (error) {
    addLog.error('Parse error', { error });
    return '';
  }
})();
```

## 8. 分块策略

解析后的文本进入分块环节：

### 8.1 分块模式

- **chunk**: 固定大小分块（如 512 tokens）
- **qa**: 问答对分块
- **imageParse**: 图片识别分块
- **backup**: 备份导入
- **template**: 模板导入

### 8.2 分块参数

- `chunkSize`: 分块大小（默认 512 tokens）
- `chunkOverlap`: 重叠大小（默认 50 tokens）
- `separator`: 自定义分隔符

## 9. 相关文档

- [01-txt-md-parser.md](./01-txt-md-parser.md) - 纯文本解析详解
- [02-html-parser.md](./02-html-parser.md) - HTML文档解析详解
- [03-pdf-parser.md](./03-pdf-parser.md) - PDF文档解析详解
- [04-docx-parser.md](./04-docx-parser.md) - Word文档解析详解
- [05-xlsx-parser.md](./05-xlsx-parser.md) - Excel表格解析详解
- [06-csv-parser.md](./06-csv-parser.md) - CSV文件解析详解
- [07-pptx-parser.md](./07-pptx-parser.md) - PowerPoint解析详解

## 10. 时序图索引

- [01-txt-md-sequence.puml](./01-txt-md-sequence.puml) - 纯文本解析时序图
- [02-html-sequence.puml](./02-html-sequence.puml) - HTML解析时序图
- [03-pdf-sequence.puml](./03-pdf-sequence.puml) - PDF解析时序图
- [04-docx-sequence.puml](./04-docx-sequence.puml) - DOCX解析时序图
- [05-xlsx-sequence.puml](./05-xlsx-sequence.puml) - XLSX解析时序图
- [06-csv-sequence.puml](./06-csv-sequence.puml) - CSV解析时序图
- [07-pptx-sequence.puml](./07-pptx-sequence.puml) - PPTX解析时序图
- [08-worker-thread-sequence.puml](./08-worker-thread-sequence.puml) - Worker线程架构时序图
