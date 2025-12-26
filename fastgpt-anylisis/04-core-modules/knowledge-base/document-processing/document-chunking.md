# 文档分块（Chunking）逻辑深度分析

## 1. 概述

文档分块是知识库管理中的核心功能，它将长文本分割成适合向量化和检索的小块。FastGPT 实现了一套智能的**多层级分块策略**，能够根据文档结构和内容特征进行自适应分割。

**核心文件路径**：
- `packages/global/common/string/textSplitter.ts` - 分块算法核心实现
- `packages/service/core/dataset/read.ts` - 分块触发逻辑
- `packages/service/core/dataset/collection/controller.ts` - 集合创建和分块调用

## 2. 分块模式概览

FastGPT 支持 **5 种分块模式**，适用于不同的文档类型和应用场景：

| 分块模式 | 英文标识 | 适用场景 | 分块特点 |
|---------|---------|---------|---------|
| **智能分块** | `chunk` | 通用文档、技术文档、文章 | 按语义边界分块，保留结构 |
| **问答对模式** | `qa` | FAQ、客服对话、问答数据集 | 问题-答案配对 |
| **图片索引** | `imageParse` | 图文混排文档 | 图片OCR识别 + 文本 |
| **备份导入** | `backup` | 数据迁移、批量导入 | CSV格式直接解析 |
| **模板导入** | `template` | 结构化数据导入 | 预定义格式解析 |

## 3. 智能分块（Chunk）详解

### 3.1 分块触发条件

智能分块有 **3 种触发策略**，通过 `chunkTriggerType` 参数控制：

#### 策略 1：按最大值触发 (maxSize)

```typescript
chunkTriggerType: ChunkTriggerConfigTypeEnum.maxSize
```

**触发逻辑**：
- 只有当文本长度超过 **模型最大上下文 × 0.7** 时才分块
- 默认最大值：16000 字符
- 适用场景：大模型处理场景，尽可能保持文档完整性

**实际例子**：
```
文档长度：8000 字符
maxSize = 16000 × 0.7 = 11200
结果：8000 < 11200，不分块，保持完整

文档长度：15000 字符  
结果：15000 > 11200，触发分块
```

**为什么是 0.7？**
- 预留 30% 空间给系统提示词、对话历史等
- 避免超出模型上下文限制导致截断

#### 策略 2：按最小值触发 (minSize)

```typescript
chunkTriggerType: ChunkTriggerConfigTypeEnum.minSize
chunkTriggerMinSize: 1000  // 手动设置最小值
```

**触发逻辑**：
- 文本长度 > `chunkTriggerMinSize` 时触发分块
- 文本长度 ≤ `chunkTriggerMinSize` 时保持完整
- 适用场景：希望控制分块粒度的场景

**实际例子**：
```
设置：chunkTriggerMinSize = 500

文档长度：300 字符 → 不分块（太短）
文档长度：800 字符 → 触发分块（超过阈值）
```

**最佳实践建议**：
```
短文本（客服回复、产品描述）：minSize = 200-300
中等文本（技术文档、知识条目）：minSize = 500-1000
长文本（论文、报告）：minSize = 1000-2000
```

#### 策略 3：强制分块 (forceChunk)

```typescript
chunkTriggerType: ChunkTriggerConfigTypeEnum.forceChunk
```

**触发逻辑**：
- 无条件分块，不管文本长度
- 适用场景：需要统一分块粒度的场景（如训练数据准备）

### 3.2 分块核心参数

| 参数 | 类型 | 默认值 | 说明 | 影响 |
|-----|------|--------|------|------|
| `chunkSize` | number | 512 | 单个分块的目标大小（tokens） | 分块粒度 |
| `overlapRatio` | number | 0.2 | 重叠比例（20%） | 上下文连贯性 |
| `paragraphChunkDeep` | number | 5 | Markdown 标题深度 | 保留文档结构 |
| `paragraphChunkMinSize` | number | 100 | 段落最小大小 | 避免碎片化 |
| `customReg` | string[] | [] | 自定义分隔符 | 特殊分割需求 |

### 3.3 多层级分块策略

FastGPT 的分块算法采用 **递归多层级分割**，按优先级尝试不同的分隔符：

```
【优先级 1】自定义分隔符（customReg）
  ↓ 如果块仍过大
【优先级 2】Markdown 标题（# ## ### ...）
  深度 1: # 一级标题
  深度 2: ## 二级标题
  深度 3: ### 三级标题
  ...最多到 8 级
  ↓ 如果块仍过大
【优先级 3】特殊语法块
  - 代码块：```code```
  - Markdown 表格：| col1 | col2 |
  ↓ 如果块仍过大
【优先级 4】段落边界
  - 双换行：\n\n
  - 单换行：\n
  ↓ 如果块仍过大
【优先级 5】句子边界（带重叠）
  - 中文句号：。
  - 英文句号：. 
  - 感叹号：！!
  - 问号：？?
  - 分号：；;
  - 逗号：，,
  ↓ 如果块仍过大
【优先级 6】字符切片
  - 按 chunkSize 强制切割（带重叠）
```

### 3.4 智能特性详解

#### 特性 1：Markdown 标题继承

**问题场景**：
```markdown
# 产品介绍

## 功能特性

### 核心功能
支持多种文档格式解析...

### 高级功能  
支持自定义分词...
```

如果简单按标题分块，会丢失上下文：
```
块1: "支持多种文档格式解析..."  ← 不知道这是什么产品的什么功能
块2: "支持自定义分词..."        ← 同样缺少上下文
```

**FastGPT 的解决方案**：
```
块1: "# 产品介绍\n## 功能特性\n### 核心功能\n支持多种文档格式解析..."
块2: "# 产品介绍\n## 功能特性\n### 高级功能\n支持自定义分词..."
```

**实现原理**：
- 在最深标题层级（`paragraphChunkDeep`）分块时
- 自动为每个块补充所有父级标题
- 保证每个块都有完整的上下文路径

**配置建议**：
```typescript
// 文档结构较浅（1-2级标题）
paragraphChunkDeep: 2

// 文档结构中等（3-5级标题）
paragraphChunkDeep: 5  // 推荐

// 文档结构很深（6+级标题）
paragraphChunkDeep: 8  // 最大支持
```

#### 特性 2：重叠（Overlap）机制

**为什么需要重叠？**

问题：简单切割会破坏语义连贯性
```
原文："...机器学习是人工智能的一个分支。它使用统计技术..."

简单切割：
  块1: "...机器学习是人工智能的一个分支。"
  块2: "它使用统计技术..."  ← "它"指代不明

带重叠切割：
  块1: "...机器学习是人工智能的一个分支。"
  块2: "机器学习是人工智能的一个分支。它使用统计技术..."
       ↑ 保留了前文，"它"的指代清晰
```

**重叠计算公式**：
```
overlapLen = chunkSize × overlapRatio

示例：
  chunkSize = 512
  overlapRatio = 0.2
  overlapLen = 512 × 0.2 = 102 tokens

结果：
  块1: tokens[0:512]
  块2: tokens[410:922]  ← 与块1重叠102个tokens
  块3: tokens[820:1332]
```

**重叠限制**：
- 只在句子边界（优先级5+）才启用重叠
- 自定义分隔符、Markdown标题、段落边界不重叠
- 最大重叠不超过 `chunkSize × 0.4`（40%）

**实际效果对比**：

| 场景 | 无重叠 | 20%重叠 | 效果 |
|------|--------|---------|------|
| 代词指代 | ❌ 指代丢失 | ✅ 指代保留 | 提升语义理解 |
| 跨块问答 | ❌ 检索困难 | ✅ 检索成功 | 提高召回率 |
| 碎片化内容 | ❌ 信息不完整 | ✅ 信息完整 | 提高准确率 |

#### 特性 3：段落最小值合并

**问题**：避免产生过小的块
```
原文（Markdown文档）：
  ## 第一节
  这是第一节的内容。
  
  ## 第二节
  这是第二节的内容。
  
  ## 第三节
  这是第三节很长很长的内容...（500字）
```

**简单分割的问题**：
```
块1: "## 第一节\n这是第一节的内容。"  ← 只有20字，太小！
块2: "## 第二节\n这是第二节的内容。"  ← 只有20字，太小！
块3: "## 第三节\n这是第三节很长..."    ← 500字，正常
```

**FastGPT 的智能合并**：
```typescript
paragraphChunkMinSize: 100  // 最小段落大小

结果：
块1: "## 第一节\n这是第一节的内容。\n## 第二节\n这是第二节的内容。"
     ↑ 合并了两个小段落，达到 100 字以上
块2: "## 第三节\n这是第三节很长..."
```

**合并逻辑**：
1. 如果当前块 < `paragraphChunkMinSize`
2. 且下一个块也 < `paragraphChunkMinSize`
3. 则合并到一起
4. 直到总大小 ≥ `paragraphChunkMinSize`

**配置建议**：
```typescript
// 严格场景（精确匹配）
paragraphChunkMinSize: 50-100

// 通用场景（平衡质量）
paragraphChunkMinSize: 100-200  // 推荐

// 宽松场景（长上下文）
paragraphChunkMinSize: 200-300
```

#### 特性 4：Markdown 表格处理

**特殊处理原因**：
- 表格是结构化数据，不能随意切割
- 需要保持表头和数据行的完整性

**识别逻辑**：
```typescript
// 必须满足所有条件：
1. 包含 | 字符
2. 至少 2 行
3. 第一行是表头（以 | 开始和结束）
4. 第二行是分隔符（| :---: | 格式）
5. 后续行都是数据行（以 | 开始和结束）
```

**切割策略**：
```markdown
原始表格（100行）：
| 列1 | 列2 | 列3 |
|-----|-----|-----|
| 数据1 | 数据2 | 数据3 |
| 数据4 | 数据5 | 数据6 |
... 98行数据

切割结果：
块1:
| 列1 | 列2 | 列3 |
|-----|-----|-----|
| 数据1 | 数据2 | 数据3 |
| 数据4 | 数据5 | 数据6 |
... 直到达到 chunkSize

块2:
| 列1 | 列2 | 列3 |  ← 重复表头
|-----|-----|-----|  ← 重复分隔符
| 数据N | 数据N+1 | 数据N+2 |
...
```

**关键点**：
- 每个分块都包含完整的表头和分隔符
- 按行分割，不会切断数据行
- 自动调整 `chunkSize × 1.2` 以容纳更多行

#### 特性 5：代码块保护

**问题**：代码块中的换行符有语义
```python
def hello():
    print("Hello")  ← 这个换行符不能作为分块边界
    return True
```

**FastGPT 的保护机制**：
```typescript
// 1. 临时替换代码块中的换行符
const codeBlockMarker = 'CODE_BLOCK_LINE_MARKER';
text = text.replace(/(```[\s\S]*?```|~~~[\s\S]*?~~~)/g, (match) => {
  return match.replace(/\n/g, codeBlockMarker);
});

// 2. 进行分块处理

// 3. 恢复换行符
chunks = chunks.map(chunk => chunk.replaceAll(codeBlockMarker, '\n'));
```

**效果**：
```
代码块会作为一个整体被保留在同一个分块中，不会被中间的换行符拆散
```

## 4. 其他分块模式详解

### 4.1 问答对模式（QA）

**适用场景**：
- FAQ 文档
- 客服对话记录
- 问答数据集

**输入格式**：
```
Q: 如何重置密码？
A: 点击"忘记密码"链接，输入注册邮箱...

Q: 支持哪些支付方式？
A: 支持支付宝、微信支付、银行卡...
```

**处理特点**：
- 不进行文本分块
- 直接提取 Q&A 对
- 每个 Q&A 对作为一个独立的知识条目
- `chunkIndex` 按 Q&A 对顺序编号

**参数特点**：
```typescript
// QA 模式下，这些参数被忽略：
chunkTriggerType    // 无需触发条件
chunkTriggerMinSize // 无需最小值
chunkSize           // 无需分块大小
chunkSplitter       // 无需分隔符
overlapRatio: 0     // 无重叠
```

**最佳实践**：
1. 问题尽量简洁明确
2. 答案完整包含必要信息
3. 避免答案之间相互引用

### 4.2 图片索引模式（ImageParse）

**适用场景**：
- 图文混排文档
- 产品目录（图片+说明）
- 技术手册（配图说明）

**处理流程**：
```
1. 提取图片 → OCR识别 → 生成图片描述
2. 提取文本 → 正常分块
3. 将图片ID关联到对应的文本块
```

**数据结构**：
```typescript
{
  q: "文本内容",
  a: "",
  imageIdList: ["img_001", "img_002"],  // 关联的图片
  chunkIndex: 0
}
```

**注意事项**：
- 图片需要先上传并获得 `imageId`
- OCR 识别需要配置视觉模型（`vlmModel`）
- 图片内容会被向量化用于检索

### 4.3 备份导入模式（Backup）

**适用场景**：
- 从其他系统迁移数据
- 批量导入历史数据
- 数据恢复

**输入格式**（CSV）：
```csv
问题,答案,索引1,索引2,索引3
如何登录系统？,使用用户名和密码登录,登录,用户,系统
如何重置密码？,点击忘记密码链接...,密码,重置,账号
```

**解析逻辑**：
```typescript
// 使用 Papa.parse 解析 CSV
const csvArr = Papa.parse(rawText).data;

// 转换为分块格式
const chunks = csvArr.slice(1).map(item => ({
  q: item[0],           // 第1列：问题
  a: item[1],           // 第2列：答案
  indexes: item.slice(2) // 第3列开始：自定义索引
}));
```

**关键特点**：
- 跳过 CSV 第一行（表头）
- 自动过滤空行
- 自定义索引可以提高检索准确率

### 4.4 模板导入模式（Template）

**适用场景**：
- 结构化数据批量导入
- 预定义格式的数据集
- API 集成导入

**与 Backup 的区别**：
- Backup：CSV 格式，通用
- Template：自定义格式，灵活

**处理特点**：
- 跳过所有分块参数
- 直接使用模板定义的结构
- 适合程序化批量导入

## 5. 分块配置最佳实践

### 5.1 场景化配置推荐

#### 场景 1：技术文档（有明确结构）

**特点**：
- 有清晰的章节标题
- 内容逻辑性强
- 可能包含代码块

**推荐配置**：
```typescript
{
  chunkTriggerType: ChunkTriggerConfigTypeEnum.minSize,
  chunkTriggerMinSize: 500,
  chunkSize: 800,
  overlapRatio: 0.2,
  paragraphChunkDeep: 5,      // 保留5级标题
  paragraphChunkMinSize: 150
}
```

**预期效果**：
- ✅ 保留完整的章节结构
- ✅ 代码块完整性
- ✅ 合理的上下文重叠

#### 场景 2：长篇文章（连续叙述）

**特点**：
- 少量或无标题
- 段落紧密相连
- 叙事连贯

**推荐配置**：
```typescript
{
  chunkTriggerType: ChunkTriggerConfigTypeEnum.minSize,
  chunkTriggerMinSize: 300,
  chunkSize: 512,
  overlapRatio: 0.25,          // 更大的重叠
  paragraphChunkDeep: 2,       // 较浅的标题
  paragraphChunkMinSize: 100
}
```

**预期效果**：
- ✅ 保持叙述连贯性
- ✅ 段落语义完整
- ✅ 较大重叠避免上下文丢失

#### 场景 3：API 文档（结构化）

**特点**：
- 标准化结构（端点、参数、返回值）
- 包含代码示例
- 信息密度高

**推荐配置**：
```typescript
{
  chunkTriggerType: ChunkTriggerConfigTypeEnum.forceChunk,
  chunkSize: 1024,             // 更大的块
  overlapRatio: 0.15,          // 较小的重叠
  paragraphChunkDeep: 6,
  paragraphChunkMinSize: 200,
  customReg: [
    '---\n',                   // API 分隔符
    '## Endpoint'              // 特定标题
  ]
}
```

**预期效果**：
- ✅ 每个 API 端点独立成块
- ✅ 完整的代码示例
- ✅ 参数说明完整

#### 场景 4：FAQ（问答对）

**特点**：
- 简短的问答
- 问题和答案紧密关联
- 独立性强

**推荐配置**：
```typescript
{
  trainingType: DatasetCollectionDataProcessModeEnum.qa,
  // QA 模式下，分块参数自动忽略
}
```

**预期效果**：
- ✅ 问答对完整性
- ✅ 独立检索
- ✅ 精准匹配

#### 场景 5：产品手册（图文混排）

**特点**：
- 图片和文字说明
- 步骤化内容
- 表格数据

**推荐配置**：
```typescript
{
  chunkTriggerType: ChunkTriggerConfigTypeEnum.minSize,
  chunkTriggerMinSize: 400,
  chunkSize: 600,
  overlapRatio: 0.2,
  paragraphChunkDeep: 4,
  paragraphChunkMinSize: 120,
  imageIndex: true              // 启用图片索引
}
```

**预期效果**：
- ✅ 图文对应清晰
- ✅ 表格完整性
- ✅ 步骤连贯性

### 5.2 参数调优指南

#### chunkSize 调优

**影响因素**：
1. **检索精度**：小块更精确，但可能信息不完整
2. **上下文完整性**：大块信息更完整，但可能引入噪音
3. **向量化成本**：块越多，成本越高

**调优建议**：
```
小文档（< 5000字）：chunkSize = 256-512
中文档（5000-20000字）：chunkSize = 512-800  ← 推荐
大文档（> 20000字）：chunkSize = 800-1024

注意：chunkSize 是 token 数，不是字符数
中文：约 1.5 字符 = 1 token
英文：约 4 字符 = 1 token
```

**实际测试**：
```
文档：技术文档，12000字
chunkSize = 256 → 60个块 → 检索准确，但缺少上下文
chunkSize = 512 → 30个块 → 平衡 ✅
chunkSize = 1024 → 15个块 → 信息完整，但检索不够精确
```

#### overlapRatio 调优

**核心权衡**：
- 重叠越大 → 上下文越完整 → 存储成本越高
- 重叠越小 → 存储成本越低 → 语义连贯性越差

**调优建议**：
```
无重叠需求（结构化文档）：overlapRatio = 0-0.1
轻度重叠（技术文档）：overlapRatio = 0.15-0.2  ← 推荐
重度重叠（叙事文本）：overlapRatio = 0.25-0.3
```

**成本计算**：
```
文档：10000 tokens
chunkSize = 500
overlapRatio = 0.2 (overlap = 100 tokens)

无重叠：10000 / 500 = 20 块
有重叠：10000 / (500-100) = 25 块 → 成本增加 25%
```

#### paragraphChunkDeep 调优

**文档结构分析**：
```
扁平结构（1-2级标题）：
  # 主标题
  ## 章节
  内容...
  
  → paragraphChunkDeep = 2

标准结构（3-5级标题）：
  # 文档标题
  ## 第一章
  ### 第一节
  #### 小节
  ##### 细节
  内容...
  
  → paragraphChunkDeep = 5 ← 推荐

深层结构（6+级标题）：
  # 到 ######## (8级)
  
  → paragraphChunkDeep = 8
```

**注意事项**：
- 深度越大，标题继承的开销越大
- 如果文档实际只有3级标题，设置deep=5也不会有问题
- 建议略大于文档实际深度（+1或+2）

### 5.3 常见问题和解决方案

#### 问题 1：分块过多，向量化成本高

**症状**：
```
文档：5000字
配置：chunkSize = 256, overlapRatio = 0.3
结果：产生 40+ 个块，成本高
```

**解决方案**：
```typescript
// 方案 A：增大 chunkSize
{
  chunkSize: 512,  // 256 → 512
  overlapRatio: 0.2
}
// 效果：块数减少约 50%

// 方案 B：减小重叠
{
  chunkSize: 256,
  overlapRatio: 0.15  // 0.3 → 0.15
}
// 效果：块数减少约 20%

// 方案 C：提高触发阈值
{
  chunkTriggerType: ChunkTriggerConfigTypeEnum.minSize,
  chunkTriggerMinSize: 1000  // 只有较大文档才分块
}
```

#### 问题 2：检索结果不完整

**症状**：
```
用户查询："如何配置 HTTPS？"
返回："在服务器上生成证书..."
问题：缺少前文"HTTPS 配置步骤"
```

**原因**：分块切断了关键上下文

**解决方案**：
```typescript
// 方案 A：增加重叠
{
  overlapRatio: 0.25  // 0.2 → 0.25
}

// 方案 B：降低标题深度（保留更多父级标题）
{
  paragraphChunkDeep: 5  // 保留到5级标题
}

// 方案 C：增大块大小
{
  chunkSize: 800  // 512 → 800
}
```

#### 问题 3：Markdown 表格被切断

**症状**：
```
表格有 100 行，但被切成多个不完整的块
```

**原因**：单个表格超过了 `chunkSize`

**解决方案**：
```typescript
// 表格自动处理已内置，但可以优化：
{
  chunkSize: 1024,  // 增大块大小以容纳更多表格行
  paragraphChunkMinSize: 200  // 避免产生小表格碎片
}

// 或者使用自定义分隔符
{
  customReg: ['\n\n---\n\n']  // 在表格前后添加分隔符
}
```

#### 问题 4：代码块被破坏

**症状**：
```python
# 块1（不完整）
def calculate():
    x = 10

# 块2（缺少上文）
    y = 20
    return x + y
```

**原因**：FastGPT 应该已经处理了这个问题，如果出现说明代码块标记有问题

**排查步骤**：
```markdown
1. 确认代码块使用了正确的标记：
   ✅ ```python
   ✅ ~~~python
   ❌ `single backtick`（单反引号不会被保护）

2. 检查代码块是否闭合：
   ✅ ```python ... ```
   ❌ ```python ... （缺少结束标记）

3. 如果仍有问题，增大 chunkSize：
   chunkSize: 1024  // 确保整个代码块在一个分块中
```

#### 问题 5：小块太多，合并不够

**症状**：
```
生成了大量 50-80 字的小块
```

**解决方案**：
```typescript
// 增大最小段落大小
{
  paragraphChunkMinSize: 200  // 100 → 200
}

// 或者使用更大的 chunkTriggerMinSize
{
  chunkTriggerMinSize: 500
}
```

## 6. 性能优化建议

### 6.1 大文档处理

**问题**：单个文档 > 100MB
```
普通处理：一次性加载到内存 → OOM
```

**优化方案**：
```typescript
// 1. 分段读取和处理
const chunkSize = 10 * 1024 * 1024; // 10MB per chunk
for (let offset = 0; offset < fileSize; offset += chunkSize) {
  const chunk = await readFileChunk(file, offset, chunkSize);
  const chunks = await rawText2Chunks({ rawText: chunk });
  await insertChunks(chunks);
}

// 2. 使用流式处理
import { pipeline } from 'stream/promises';
await pipeline(
  fs.createReadStream(file),
  splitStream(chunkSize),
  processChunks(),
  insertToDatabase()
);
```

### 6.2 批量文档处理

**问题**：1000+ 个文档需要处理

**优化方案**：
```typescript
// 1. 并发控制（避免过载）
import pLimit from 'p-limit';
const limit = pLimit(5); // 最多5个并发

await Promise.all(
  documents.map(doc => limit(() => processDocument(doc)))
);

// 2. 批量插入向量（减少数据库操作）
const batchSize = 100;
for (let i = 0; i < chunks.length; i += batchSize) {
  const batch = chunks.slice(i, i + batchSize);
  await insertVectorsBatch(batch);
}

// 3. 分块进度跟踪
let processed = 0;
for (const doc of documents) {
  await processDocument(doc);
  processed++;
  console.log(`Progress: ${processed}/${documents.length}`);
}
```

### 6.3 内存优化

**问题**：分块过程占用大量内存

**优化方案**：
```typescript
// 1. 及时清理临时数据
let chunks = await rawText2Chunks({ rawText });
await processChunks(chunks);
chunks = null; // 显式释放

// 2. 避免深度递归（使用迭代）
// FastGPT 已经优化，但处理超大文档时注意栈溢出

// 3. 使用流式处理大文本
const { Readable } = require('stream');
const textStream = Readable.from(largeText);
// 分块读取和处理
```

## 7. 监控和调试

### 7.1 分块质量检查

```typescript
// 检查点 1：分块数量
const chunks = await rawText2Chunks({ rawText });
console.log(`Total chunks: ${chunks.length}`);
// 预期：根据文档长度和 chunkSize 估算
// 异常：过多（> 100）或过少（< 5）需要调整参数

// 检查点 2：分块大小分布
const sizes = chunks.map(c => c.q.length);
console.log(`Min: ${Math.min(...sizes)}`);
console.log(`Max: ${Math.max(...sizes)}`);
console.log(`Avg: ${sizes.reduce((a, b) => a + b) / sizes.length}`);
// 预期：大小分布相对均匀，最小值 > paragraphChunkMinSize

// 检查点 3：重叠检查
for (let i = 1; i < chunks.length; i++) {
  const prev = chunks[i-1].q;
  const curr = chunks[i].q;
  const overlap = findOverlap(prev, curr);
  console.log(`Overlap ${i}: ${overlap.length} chars`);
}
// 预期：重叠长度 ≈ chunkSize × overlapRatio
```

### 7.2 常见错误排查

| 错误现象 | 可能原因 | 排查方法 |
|---------|---------|---------|
| 分块数 = 0 | 文档为空或触发条件未满足 | 检查 `chunkTriggerMinSize` |
| 分块数过多 | `chunkSize` 太小 | 增大 `chunkSize` |
| 表格被切断 | 表格未被识别或太大 | 检查表格格式，增大 `chunkSize` |
| 代码块被破坏 | 代码块标记不完整 | 检查 ``` 是否闭合 |
| 内存溢出 | 文档太大，一次性处理 | 使用分段或流式处理 |
| 检索结果不完整 | 重叠太小或标题丢失 | 增大 `overlapRatio` 或 `paragraphChunkDeep` |

## 8. 完整配置示例

### 示例 1：技术文档（推荐配置）

```typescript
const config = {
  // 触发条件
  chunkTriggerType: ChunkTriggerConfigTypeEnum.minSize,
  chunkTriggerMinSize: 500,
  
  // 分块大小
  chunkSize: 800,
  overlapRatio: 0.2,
  
  // Markdown 处理
  paragraphChunkDeep: 5,
  paragraphChunkMinSize: 150,
  
  // 自定义分隔符（可选）
  chunkSplitter: '',
  
  // 数据增强（可选）
  autoIndexes: true,
  imageIndex: false
};
```

### 示例 2：FAQ（问答对）

```typescript
const config = {
  trainingType: DatasetCollectionDataProcessModeEnum.qa,
  // 其他参数自动忽略
};
```

### 示例 3：长篇小说（连续叙述）

```typescript
const config = {
  chunkTriggerType: ChunkTriggerConfigTypeEnum.minSize,
  chunkTriggerMinSize: 300,
  
  chunkSize: 512,
  overlapRatio: 0.3,  // 更大的重叠保持叙述连贯性
  
  paragraphChunkDeep: 2,  // 小说标题较少
  paragraphChunkMinSize: 100
};
```

### 示例 4：API 文档（高密度信息）

```typescript
const config = {
  chunkTriggerType: ChunkTriggerConfigTypeEnum.forceChunk,
  
  chunkSize: 1024,  // 更大的块容纳完整 API 说明
  overlapRatio: 0.15,
  
  paragraphChunkDeep: 6,
  paragraphChunkMinSize: 200,
  
  customReg: ['---\n', '## Endpoint']  // API 特定分隔符
};
```

## 9. 总结

### 9.1 核心要点

1. **选择合适的分块模式**
   - 通用文档 → `chunk` 模式
   - 问答数据 → `qa` 模式
   - 图文混排 → `imageParse` 模式

2. **根据文档特征调整参数**
   - 有明确结构 → 增大 `paragraphChunkDeep`
   - 连续叙述 → 增大 `overlapRatio`
   - 信息密度高 → 增大 `chunkSize`

3. **平衡质量和成本**
   - 块越小 → 检索越精确 → 成本越高
   - 块越大 → 上下文越完整 → 可能引入噪音
   - 推荐从 `chunkSize = 512-800` 开始测试

4. **利用智能特性**
   - Markdown 标题继承 → 保留文档结构
   - 重叠机制 → 保持语义连贯性
   - 段落合并 → 避免碎片化
   - 表格保护 → 保持数据完整性

### 9.2 快速决策表

| 您的文档类型 | 推荐配置 |
|-------------|---------|
| 📄 技术文档、用户手册 | `minSize`, `chunkSize=800`, `overlap=0.2`, `deep=5` |
| 📝 文章、博客 | `minSize`, `chunkSize=512`, `overlap=0.25`, `deep=3` |
| 💬 FAQ、对话 | `qa` 模式 |
| 🔧 API 文档 | `forceChunk`, `chunkSize=1024`, `overlap=0.15`, `deep=6` |
| 📖 小说、故事 | `minSize`, `chunkSize=512`, `overlap=0.3`, `deep=2` |
| 📊 产品手册（带图） | `minSize`, `chunkSize=600`, `imageIndex=true` |

### 9.3 性能优化清单

- [x] 使用合适的 `chunkTriggerMinSize` 避免不必要的分块
- [ ] 批量处理多文档时控制并发数（5-10个）
- [ ] 大文档使用分段处理避免内存溢出
- [ ] 定期检查分块质量（数量、大小分布、重叠）
- [ ] 根据实际检索效果调整参数
- [ ] 为特定领域文档配置自定义分隔符

---

**相关文档**：
- [文档处理概览](./document-processing/00-overview.md)
- [文本嵌入与检索](./embedding-and-retrieval.md)
- [知识库管理时序图](./knowledge-base-management.md)
