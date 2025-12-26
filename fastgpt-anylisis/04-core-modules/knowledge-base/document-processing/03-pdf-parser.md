# PDF 文档解析器

## 1. 概述

PDF 解析器提供 **三级解析策略**，从基础的系统解析到高级的 AI 识别服务，满足不同场景的 PDF 文本提取需求。

**文件路径**: 
- `packages/service/worker/readFile/extension/pdf.ts` (系统解析)
- `packages/service/common/file/read/utils.ts` (自定义解析服务)

## 2. 支持的文件类型

- `.pdf` - PDF 文档

## 3. 三级解析策略

```
┌─────────────────────────────────────┐
│ 1. 系统解析 (pdfjs-dist)            │
│    - 免费                           │
│    - 速度快                         │
│    - 适合普通文本 PDF               │
└─────────────────────────────────────┘
           ↓ (可选切换)
┌─────────────────────────────────────┐
│ 2. 自定义解析服务 (customPdfParse)  │
│    - 外部 API                       │
│    - 支持表格、公式识别              │
│    - 需要配置 API 地址和密钥         │
└─────────────────────────────────────┘
           ↓ (可选切换)
┌─────────────────────────────────────┐
│ 3. Doc2X 服务                        │
│    - AI 驱动的高级识别               │
│    - 支持复杂布局、手写内容          │
│    - 收费服务，需要 API Key          │
└─────────────────────────────────────┘
```

## 4. 系统解析 (pdfjs-dist)

### 4.1 核心依赖

```json
{
  "pdfjs-dist": "^3.11.174"
}
```

### 4.2 解析流程

```typescript
1. 初始化 PDF 文档
   ↓
2. 逐页加载页面对象
   ↓
3. 提取文本内容（getTextContent）
   ↓
4. 过滤页眉页脚（5% 阈值）
   ↓
5. EOL (End of Line) 检测
   ↓
6. 拼接文本并释放内存
   ↓
7. 返回纯文本
```

### 4.3 页眉页脚过滤

```typescript
// 计算页面高度阈值
const minY = viewport.height * 0.05;  // 下 5%
const maxY = viewport.height * 0.95;  // 上 5%

// 过滤页眉页脚
items = items.filter(item => {
  const y = item.transform[5];
  return y >= minY && y <= maxY;
});
```

**目的**: 
- 移除重复的页眉（如文档标题、页码）
- 移除重复的页脚（如版权信息）
- 提高文本质量

### 4.4 EOL (End of Line) 检测

```typescript
// 检测是否为行尾
const hasEOL = item.hasEOL;

// 添加换行符或空格
if (hasEOL) {
  text += '\n';
} else {
  text += ' ';
}
```

**作用**:
- 保留段落结构
- 避免单词粘连
- 识别列表和表格

### 4.5 内存管理

```typescript
// 处理完每一页后立即清理
page.cleanup();
textContent = null;

// 文档关闭后销毁
pdf.destroy();
```

**优势**:
- 避免内存泄漏
- 支持大文件处理
- 提高并发能力

## 5. 自定义解析服务 (customPdfParse)

### 5.1 配置方式

**环境变量**:
```bash
# 自定义 PDF 解析 API
CUSTOM_PDF_PARSE_URL=https://your-service.com/parse
CUSTOM_PDF_PARSE_KEY=your_api_key
```

### 5.2 API 调用

```typescript
const response = await fetch(CUSTOM_PDF_PARSE_URL, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${CUSTOM_PDF_PARSE_KEY}`,
    'Content-Type': 'application/pdf'
  },
  body: pdfBuffer
});

const result = await response.json();
```

### 5.3 返回格式

```typescript
interface CustomPdfParseResponse {
  text: string;           // 提取的文本
  tables?: Table[];       // 识别的表格
  formulas?: Formula[];   // 识别的公式
}
```

### 5.4 计费方式

```typescript
// 按 Token 计费
const tokens = await getMarkdownImageContentTokens(result.text);
await createTrainingUsage({
  billId: billId,
  appId: appId,
  teamId: teamId,
  tokens: tokens,
  type: UsageSourceEnum.customPdfParse
});
```

## 6. Doc2X 服务

### 6.1 配置方式

**环境变量**:
```bash
# Doc2X API 配置
DOC2X_API_URL=https://api.doc2x.com/v1/parse
DOC2X_API_KEY=your_doc2x_key
```

### 6.2 特色功能

- 🧠 **AI 识别**: 基于深度学习的文档理解
- 📊 **表格还原**: 保留表格结构和数据
- 🖼️ **图片描述**: 自动生成图片说明
- 🔢 **公式识别**: LaTeX 格式的数学公式
- ✍️ **手写识别**: OCR 手写内容
- 📐 **复杂布局**: 多栏、图文混排

### 6.3 API 调用

```typescript
const response = await fetch(`${DOC2X_API_URL}/parse`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${DOC2X_API_KEY}`,
    'Content-Type': 'multipart/form-data'
  },
  body: formData
});

const result = await response.json();
```

### 6.4 返回格式

```typescript
interface Doc2XResponse {
  markdown: string;       // Markdown 格式的文本
  images: Array<{         // 提取的图片
    id: string;
    url: string;
    description?: string;
  }>;
  tables: Array<{         // 识别的表格
    markdown: string;
  }>;
  formulas: Array<{       // 识别的公式
    latex: string;
  }>;
}
```

## 7. 性能对比

| 策略 | 速度 | 准确率 | 表格 | 公式 | 图片 | 成本 |
|------|------|--------|------|------|------|------|
| 系统解析 | ⚡⚡⚡ | 85% | ❌ | ❌ | ❌ | 免费 |
| 自定义服务 | ⚡⚡ | 90% | ✅ | ✅ | ⚠️ | 低 |
| Doc2X | ⚡ | 95% | ✅ | ✅ | ✅ | 中 |

## 8. 输出格式

### 8.1 系统解析输出

```typescript
interface ReadFileResponse {
  rawText: string;  // 纯文本
}
```

### 8.2 自定义服务输出

```typescript
interface ReadFileResponse {
  rawText: string;        // Markdown 格式文本
  formatText?: string;    // 格式化文本
  imageList?: ImageType[]; // 提取的图片
}
```

## 9. 使用场景选择

### 9.1 系统解析

**适用场景**:
- ✅ 普通文本 PDF（电子书、论文）
- ✅ 布局简单的文档
- ✅ 对速度要求高的场景
- ✅ 免费服务

**不适用场景**:
- ❌ 扫描版 PDF（需要 OCR）
- ❌ 表格密集的文档
- ❌ 包含大量公式
- ❌ 复杂的多栏布局

### 9.2 自定义解析服务

**适用场景**:
- ✅ 表格识别需求
- ✅ 公式识别需求
- ✅ 预算有限
- ✅ 需要自定义识别逻辑

### 9.3 Doc2X 服务

**适用场景**:
- ✅ 扫描版 PDF
- ✅ 手写内容
- ✅ 复杂布局
- ✅ 图文混排
- ✅ 对准确率要求极高

## 10. 错误处理

### 10.1 解析失败降级

```typescript
try {
  // 尝试自定义服务
  return await customPdfParse();
} catch (error) {
  // 降级到系统解析
  return await systemPdfParse();
}
```

### 10.2 常见问题

**问题 1**: PDF 加密
- **检测**: `pdf.isEncrypted()`
- **解决**: 提示用户移除密码保护

**问题 2**: 内存溢出
- **原因**: PDF 文件过大（> 100MB）
- **解决**: 分页处理，及时释放内存

**问题 3**: 文字乱码
- **原因**: 编码问题或字体嵌入问题
- **解决**: 尝试自定义解析服务

## 11. 代码示例

### 11.1 系统解析

```typescript
import { readPdfRawText } from './pdf';

const result = await readPdfRawText({
  buffer: pdfBuffer,
  encoding: 'utf-8'
});

console.log('文本:', result.rawText);
```

### 11.2 自定义服务

```typescript
import { readFileByCustomApi } from '../../common/file/read/utils';

const result = await readFileByCustomApi({
  teamId: 'team_123',
  buffer: pdfBuffer,
  extension: 'pdf',
  metadata: {
    apiServer: process.env.CUSTOM_PDF_PARSE_URL,
    apiKey: process.env.CUSTOM_PDF_PARSE_KEY
  }
});

console.log('Markdown:', result.rawText);
console.log('表格数量:', result.tables?.length);
```

## 12. 最佳实践

### 12.1 解析策略选择

```typescript
// 根据文件大小和内容选择策略
const strategy = selectStrategy(pdf);

if (pdf.pages > 100 || pdf.hasComplexLayout) {
  // 使用 Doc2X
  return await doc2xParse(pdf);
} else if (pdf.hasTables || pdf.hasFormulas) {
  // 使用自定义服务
  return await customParse(pdf);
} else {
  // 使用系统解析
  return await systemParse(pdf);
}
```

### 12.2 分页处理

```typescript
const PAGES_PER_BATCH = 10;

for (let i = 0; i < totalPages; i += PAGES_PER_BATCH) {
  const batch = await parsePages(i, i + PAGES_PER_BATCH);
  await saveBatch(batch);
}
```

### 12.3 缓存机制

```typescript
// 缓存解析结果
const cacheKey = `pdf:${fileHash}`;
let result = await cache.get(cacheKey);

if (!result) {
  result = await parsePdf(pdf);
  await cache.set(cacheKey, result, 3600); // 1 hour
}
```

## 13. 相关配置

### 13.1 环境变量

```bash
# 系统解析（无需配置）
# pdfjs-dist 内置

# 自定义解析服务
CUSTOM_PDF_PARSE_URL=https://api.example.com/parse
CUSTOM_PDF_PARSE_KEY=your_api_key

# Doc2X 服务
DOC2X_API_URL=https://api.doc2x.com/v1
DOC2X_API_KEY=your_doc2x_key
```

### 13.2 性能参数

```typescript
// pdfjs-dist 配置
const loadingTask = pdfjsLib.getDocument({
  data: buffer,
  maxImageSize: 2000,        // 最大图片尺寸
  disableFontFace: true,     // 禁用字体加载（提高速度）
  cMapPacked: true           // 使用压缩的 CMap
});
```

## 14. 未来优化方向

- 🔄 混合解析策略（系统解析 + AI 增强）
- 📊 表格结构保留（Markdown 表格格式）
- 🖼️ 图片文字识别（OCR）
- 🔗 超链接和书签提取
- 📝 PDF 批注和高亮提取
- 🗜️ 流式解析（支持超大 PDF）
