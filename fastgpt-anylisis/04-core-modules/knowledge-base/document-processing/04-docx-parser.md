# Word 文档解析器 (DOCX)

## 1. 概述

DOCX 解析器使用 Mammoth 库将 Word 文档转换为 Markdown 格式，并提取内嵌图片。

**文件路径**: `packages/service/worker/readFile/extension/docx.ts`

## 2. 核心依赖

```json
{
  "mammoth": "^1.6.0"  // DOCX 转 HTML
}
```

## 3. 解析流程

```
DOCX 文件
  ↓
1. Mammoth 转换 DOCX → HTML
  ↓
2. 提取内嵌图片（Base64）
  ↓
3. 为图片生成 UUID
  ↓
4. HTML 转 Markdown (html2md)
  ↓
5. 返回 Markdown 和图片列表
```

## 4. 关键代码

```typescript
// Mammoth 配置
const result = await mammoth.convertToHtml(
  { buffer },
  {
    convertImage: mammoth.images.imgElement((image) => {
      // 提取图片并返回 UUID
      return image.read('base64').then((imageBuffer) => {
        const uuid = `IMAGE_${getNanoid()}_IMAGE`;
        const base64 = imageBuffer.toString();
        
        images.push({
          uuid,
          base64,
          mime: image.contentType
        });
        
        return { src: uuid };
      });
    })
  }
);

// 转换 HTML 为 Markdown
const { rawText, imageList: htmlImageList } = html2md(result.value);
```

## 5. 图片处理

### 5.1 支持的图片格式

- PNG
- JPEG
- GIF
- BMP
- TIFF

### 5.2 图片提取流程

```
内嵌图片 (Binary)
  ↓
读取为 Base64
  ↓
生成唯一 UUID
  ↓
保存到 imageList
  ↓
HTML 中替换为 UUID
```

## 6. 输出格式

```typescript
interface ReadFileResponse {
  rawText: string;        // Markdown 文本
  imageList: ImageType[]; // 图片列表
}

interface ImageType {
  uuid: string;    // IMAGE_abc123_IMAGE
  base64: string;  // Base64 图片数据
  mime: string;    // image/png, image/jpeg
}
```

## 7. 支持的 DOCX 特性

| 特性 | 支持 | 说明 |
|------|------|------|
| 标题 | ✅ | 转为 Markdown 标题 |
| 粗体/斜体 | ✅ | `**bold**`, `_italic_` |
| 列表 | ✅ | 有序/无序列表 |
| 表格 | ✅ | 转为 Markdown 表格 |
| 图片 | ✅ | Base64 提取 |
| 超链接 | ✅ | `[text](url)` |
| 页眉页脚 | ❌ | 不提取 |
| 批注 | ❌ | 不提取 |
| 修订 | ❌ | 不提取 |

## 8. 性能特点

- ⚡ 速度快（基于流式解析）
- 💾 内存友好（图片分离存储）
- 📄 格式保留度高

## 9. 使用示例

```typescript
const result = await readDocxRawText({
  buffer: docxBuffer,
  encoding: 'utf-8'
});

console.log('Markdown:', result.rawText);
console.log('图片数量:', result.imageList.length);

// 保存图片
for (const img of result.imageList) {
  await saveImage(img.uuid, img.base64, img.mime);
}
```

## 10. 最佳实践

1. **文档预处理**: 移除页眉页脚，简化格式
2. **图片优化**: 压缩大图片，避免 Base64 过大
3. **分批处理**: 大文档分段解析

## 11. 未来优化

- 🔄 支持页眉页脚提取
- 📝 保留批注信息
- 🎨 保留部分样式（字体颜色）
