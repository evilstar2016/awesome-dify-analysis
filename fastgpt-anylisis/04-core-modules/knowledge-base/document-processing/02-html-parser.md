# HTML 文档解析器

## 1. 概述

HTML 解析器负责将 HTML 文档转换为 Markdown 格式，并提取其中的图片资源。

**文件路径**: `packages/service/worker/readFile/extension/html.ts`

## 2. 支持的文件类型

- `.html` - HTML 文档
- `.htm` - HTML 文档（旧扩展名）

## 3. 核心依赖

### 3.1 第三方库

```json
{
  "turndown": "^7.1.2",              // HTML 转 Markdown
  "joplin-turndown-plugin-gfm": "*"  // GitHub Flavored Markdown 插件
}
```

### 3.2 内部依赖

- `worker/htmlStr2Md/utils` - HTML 转 Markdown 工具
- `readFile/extension/rawText` - 纯文本读取器

## 4. 解析流程

```
HTML 文件
  ↓
1. 读取 HTML 文本（使用 rawText 解析器）
  ↓
2. 提取 Base64 图片并替换为 UUID
  ↓
3. 检查 HTML 大小（限制 100KB）
  ↓
4. 移除无用标签 (script, style, iframe)
  ↓
5. 使用 Turndown 转换为 Markdown
  ↓
6. 返回 Markdown 文本和图片列表
```

## 5. HTML 转 Markdown

### 5.1 Turndown 配置

```typescript
const turndownService = new TurndownService({
  headingStyle: 'atx',           // 标题样式: # Heading
  bulletListMarker: '-',         // 无序列表标记
  codeBlockStyle: 'fenced',      // 代码块样式: ```
  fence: '```',                  // 代码块围栏
  emDelimiter: '_',              // 斜体: _italic_
  strongDelimiter: '**',         // 粗体: **bold**
  linkStyle: 'inlined',          // 链接样式: [text](url)
  linkReferenceStyle: 'full'     // 链接引用样式
});
```

### 5.2 移除的标签

```typescript
turndownService.remove([
  'i',        // 图标标签
  'script',   // 脚本
  'iframe',   // 内嵌框架
  'style'     // 样式
]);
```

### 5.3 GitHub Flavored Markdown (GFM)

支持的 GFM 特性：
- ✅ 表格 (Tables)
- ✅ 删除线 (Strikethrough)
- ✅ 任务列表 (Task Lists)

### 5.4 自定义媒体标签处理

```typescript
turndownService.addRule('media', {
  filter: ['video', 'source', 'audio'],
  replacement: function (content, node) {
    const src = node.getAttribute('src');
    if (src) {
      return `[${src}](${src}) `;
    }
    return content;
  }
});
```

**处理效果**:
```html
<!-- 输入 -->
<video src="demo.mp4"></video>
<audio src="music.mp3"></audio>

<!-- 输出 -->
[demo.mp4](demo.mp4)
[music.mp3](music.mp3)
```

## 6. 图片处理

### 6.1 Base64 图片提取

```typescript
const base64Regex = /src="data:([^;]+);base64,([A-Za-z0-9+/=]+)"/g;

// 1. 提取 Base64 图片
// 2. 生成唯一 UUID
// 3. 替换为 UUID 占位符
// 4. 保存图片信息到 imageList
```

**示例**:
```html
<!-- 原始 HTML -->
<img src="data:image/png;base64,iVBORw0KGgoAAAANS..." />

<!-- 替换后 -->
<img src="IMAGE_abc123xyz_IMAGE" />
```

### 6.2 图片数据结构

```typescript
interface ImageType {
  uuid: string;    // 唯一标识符 (如 IMAGE_abc123_IMAGE)
  base64: string;  // Base64 编码的图片数据
  mime: string;    // MIME 类型 (如 image/png, image/jpeg)
}
```

## 7. 性能优化

### 7.1 HTML 大小限制

```typescript
const MAX_HTML_SIZE = 100 * 1000; // 100KB

if (processedHtml.length > MAX_HTML_SIZE) {
  return { rawText: processedHtml, imageList: [] };
}
```

**原因**:
- HTML 转 Markdown 的正则匹配成本高
- 超大 HTML 可能导致内存溢出
- Base64 图片会显著增加 HTML 大小

**降级策略**:
- 超过 100KB 时，直接返回原始 HTML
- 不提取图片列表
- 跳过 Markdown 转换

### 7.2 Base64 图片优化

**问题**: 内联 Base64 图片占用大量内存

**解决方案**:
1. 提取 Base64 图片到独立数组
2. 用 UUID 替换原始 Base64 数据
3. 降低 HTML 字符串大小

**效果**:
```
原始 HTML: 500KB (含大量 Base64 图片)
  ↓
处理后 HTML: 50KB (UUID 占位符)
图片数组: 450KB (分离存储)
```

### 7.3 正则表达式优化

```typescript
// ❌ 低效: 回溯过多
const badRegex = /src="data:.*base64,.*"/g;

// ✅ 高效: 精确字符集
const goodRegex = /src="data:([^;]+);base64,([A-Za-z0-9+/=]+)"/g;
```

**优化点**:
- 使用精确字符集 `[A-Za-z0-9+/=]+` 而非 `.*`
- 明确捕获 MIME 类型和 Base64 数据
- 减少不必要的捕获组

## 8. 输出格式

```typescript
interface ReadFileResponse {
  rawText: string;        // Markdown 格式的文本
  imageList: ImageType[]; // 提取的 Base64 图片列表
}
```

### 8.1 示例输出

**输入 HTML**:
```html
<!DOCTYPE html>
<html>
<head><title>示例</title></head>
<body>
  <h1>标题</h1>
  <p>这是一段 <strong>重要</strong> 的文字。</p>
  <img src="data:image/png;base64,iVBORw0KGgo..." />
  <ul>
    <li>项目 1</li>
    <li>项目 2</li>
  </ul>
</body>
</html>
```

**输出 Markdown**:
```markdown
# 标题

这是一段 **重要** 的文字。

![](IMAGE_abc123_IMAGE)

- 项目 1
- 项目 2
```

**输出图片列表**:
```json
[
  {
    "uuid": "IMAGE_abc123_IMAGE",
    "base64": "iVBORw0KGgo...",
    "mime": "image/png"
  }
]
```

## 9. 错误处理

### 9.1 降级策略

```typescript
try {
  // 尝试转换为 Markdown
  const md = turndownService.turndown(processedHtml);
  return { rawText: md, imageList: images };
} catch (error) {
  // 失败时返回原始 HTML
  return { rawText: html, imageList: [] };
}
```

### 9.2 常见问题

**问题 1**: 表格转换失败
- **原因**: 表格格式不规范（缺少 `<thead>`, `<tbody>`）
- **解决**: GFM 插件自动修复大部分问题

**问题 2**: 图片提取不完整
- **原因**: Base64 格式错误或截断
- **解决**: 正则表达式只匹配完整的 Base64 字符串

**问题 3**: HTML 过大导致超时
- **原因**: 超过 100KB 限制
- **解决**: 自动降级为返回原始 HTML

## 10. 使用场景

### 10.1 适用场景

- 🌐 网页内容抓取
- 📧 富文本邮件转换
- 📄 HTML 格式文档导入
- 🖼️ 图文混排内容

### 10.2 不适用场景

- ❌ 复杂网页（大量 JavaScript）
- ❌ 超大 HTML（> 100KB）
- ❌ 需要保留样式的场景

## 11. 代码示例

### 11.1 基本用法

```typescript
import { readHtmlRawText } from './html';

const result = await readHtmlRawText({
  buffer: htmlBuffer,
  encoding: 'utf-8'
});

console.log('Markdown:', result.rawText);
console.log('图片数量:', result.imageList.length);
```

### 11.2 处理图片

```typescript
const { rawText, imageList } = await readHtmlRawText({
  buffer: htmlBuffer,
  encoding: 'utf-8'
});

// 保存图片到存储服务
for (const image of imageList) {
  const imageBuffer = Buffer.from(image.base64, 'base64');
  await saveImage(image.uuid, imageBuffer, image.mime);
}

// 替换 UUID 为实际图片 URL
let finalMarkdown = rawText;
for (const image of imageList) {
  const imageUrl = getImageUrl(image.uuid);
  finalMarkdown = finalMarkdown.replace(
    image.uuid,
    imageUrl
  );
}
```

## 12. 最佳实践

### 12.1 HTML 预处理

1. **清理无用标签**: 移除 `<script>`, `<style>` 等
2. **统一编码**: 转换为 UTF-8
3. **修复格式**: 确保 HTML 结构完整

### 12.2 图片处理

1. **限制图片大小**: 避免超大 Base64 图片
2. **异步上传**: 图片上传与文本处理分离
3. **CDN 加速**: 图片存储到 CDN

### 12.3 性能优化

1. **分批处理**: 大量 HTML 文件分批解析
2. **缓存结果**: 相同 HTML 的解析结果可缓存
3. **限制并发**: 避免同时解析过多文件

## 13. 相关配置

### 13.1 环境变量

- `MAX_HTML_SIZE`: HTML 大小限制（默认 100KB）

### 13.2 可调参数

- Turndown 配置选项
- Base64 正则表达式
- 图片 UUID 生成规则

## 14. 未来优化方向

- 🔄 支持更大的 HTML 文件（流式处理）
- 🖼️ 图片压缩和优化
- 📊 保留部分样式信息（如颜色、对齐）
- 🌐 支持更多 HTML5 标签（如 `<details>`, `<summary>`）
