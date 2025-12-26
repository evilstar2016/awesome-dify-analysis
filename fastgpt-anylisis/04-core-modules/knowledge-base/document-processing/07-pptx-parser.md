# PowerPoint 解析器 (PPTX)

## 1. 概述

PPTX 解析器通过解压 PowerPoint 文件提取幻灯片文本内容，基于 XML 解析。

**文件路径**: 
- `packages/service/worker/readFile/extension/pptx.ts`
- `packages/service/worker/readFile/parseOffice.ts`

## 2. 核心依赖

```json
{
  "decompress": "^4.2.1",     // ZIP 解压
  "@xmldom/xmldom": "^0.8.10" // XML 解析
}
```

## 3. 解析流程

```
PPTX 文件 (ZIP 格式)
  ↓
1. 解压到临时目录
  ↓
2. 查找幻灯片 XML 文件
   - ppt/slides/slide*.xml
   - ppt/notesSlides/notesSlide*.xml
  ↓
3. 按幻灯片编号排序
  ↓
4. 解析 XML 提取文本节点
   - 查找 <a:p> 段落标签
   - 提取 <a:t> 文本标签
  ↓
5. 拼接所有幻灯片文本
  ↓
6. 清理临时文件
  ↓
7. 返回纯文本
```

## 4. PPTX 文件结构

```
presentation.pptx (ZIP)
├── ppt/
│   ├── slides/
│   │   ├── slide1.xml         # 幻灯片 1
│   │   ├── slide2.xml         # 幻灯片 2
│   │   └── slide3.xml         # 幻灯片 3
│   ├── notesSlides/
│   │   ├── notesSlide1.xml    # 演讲者备注 1
│   │   └── notesSlide2.xml    # 演讲者备注 2
│   └── slideLayouts/
│       └── ...
└── [Content_Types].xml
```

## 5. XML 解析

### 5.1 幻灯片 XML 结构

```xml
<p:sld>
  <p:cSld>
    <p:spTree>
      <p:sp>  <!-- 文本框 -->
        <p:txBody>
          <a:p>  <!-- 段落 -->
            <a:r>  <!-- 文本运行 -->
              <a:t>Hello World</a:t>  <!-- 文本内容 -->
            </a:r>
          </a:p>
        </p:txBody>
      </p:sp>
    </p:spTree>
  </p:cSld>
</p:sld>
```

### 5.2 文本提取逻辑

```typescript
// 查找所有段落节点
const paragraphs = xmlDoc.getElementsByTagName('a:p');

// 遍历段落
for (const paragraph of paragraphs) {
  // 查找文本节点
  const textNodes = paragraph.getElementsByTagName('a:t');
  
  // 提取文本
  const texts = Array.from(textNodes)
    .filter(node => node.childNodes[0]?.nodeValue)
    .map(node => node.childNodes[0].nodeValue)
    .join('');
  
  // 添加换行符
  result.push(texts);
}

return result.join('\n');
```

## 6. 幻灯片排序

```typescript
// 按幻灯片编号排序
const sortedFiles = files.sort((a, b) => {
  const getSlideNumber = (path: string) => {
    const match = path.match(/\d+/);
    return match ? parseInt(match[0]) : 0;
  };
  return getSlideNumber(a.path) - getSlideNumber(b.path);
});
```

**确保输出顺序正确**：
- slide1.xml → 第 1 张幻灯片
- slide2.xml → 第 2 张幻灯片
- ...

## 7. 临时文件管理

```typescript
// 临时目录
const tmpDir = '/tmp';
const filepath = `${tmpDir}/${nanoid()}.pptx`;
const decompressPath = `${tmpDir}/${nanoid()}`;

// 写入临时文件
fs.writeFileSync(filepath, buffer);

// 解压
await decompress(filepath, decompressPath);

// 处理完成后清理
fs.unlinkSync(filepath);
clearDirFiles(decompressPath);
```

## 8. 输出格式

```typescript
interface ReadFileResponse {
  rawText: string;  // 纯文本（按幻灯片顺序）
}
```

### 8.1 示例

**输入 PPTX**:
```
Slide 1:
- Title: Introduction
- Content: Welcome to the presentation

Slide 2:
- Title: Main Topic
- Content: This is the main content
```

**输出 rawText**:
```
Introduction
Welcome to the presentation

Main Topic
This is the main content
```

## 9. 支持的 PPTX 特性

| 特性 | 支持 | 说明 |
|------|------|------|
| 幻灯片文本 | ✅ | 提取所有文本框 |
| 演讲者备注 | ✅ | 提取 notesSlide |
| 表格 | ⚠️ | 提取文本，不保留结构 |
| 图片 | ❌ | 不提取 |
| 图表 | ❌ | 不提取 |
| 形状 | ❌ | 仅提取文本 |
| 动画 | ❌ | 不提取 |
| 超链接 | ❌ | 仅提取文本 |

## 10. 性能特点

- 🗜️ 需要解压（稍慢）
- 💾 临时文件占用磁盘
- 📄 文本提取准确

## 11. 使用示例

```typescript
const result = await readPptxRawText({
  buffer: pptxBuffer,
  encoding: 'utf-8'
});

console.log('文本:', result.rawText);

// 按幻灯片分割（假设每张幻灯片用双换行分隔）
const slides = result.rawText.split('\n\n');
console.log('幻灯片数量:', slides.length);
```

## 12. 错误处理

### 12.1 常见问题

**问题 1**: 解压失败
- **原因**: 文件损坏或格式错误
- **解决**: 检查文件完整性

**问题 2**: 文本乱码
- **原因**: 编码问题
- **解决**: 指定正确编码

**问题 3**: 临时目录权限
- **原因**: `/tmp` 目录无写权限
- **解决**: 修改临时目录路径

## 13. 最佳实践

1. **清理临时文件**: 确保及时删除临时文件
2. **错误处理**: 捕获解压和解析异常
3. **内存管理**: 大文件分批处理

## 14. 未来优化

- 🖼️ 图片提取
- 📊 表格结构保留
- 🔗 超链接提取
- 🎨 样式信息保留（标题、列表）
