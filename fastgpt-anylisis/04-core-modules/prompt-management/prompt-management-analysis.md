# FastGPT Prompt管理功能分析

## 一、概述

FastGPT **没有统一的Prompt管理功能**（如专门的Prompt库、Prompt CRUD API等），而是采用了**代码化的、分类组织的Prompt管理方式**。Prompt以TypeScript代码的形式定义，通过版本控制进行管理，并在应用运行时动态组合使用。

## 二、Prompt组织架构

### 2.1 文件结构

```
packages/global/core/ai/prompt/
├── agent.ts          # Agent相关prompt（分类、参数提取、问题引导等）
├── AIChat.ts         # 对话相关prompt（引用模板、系统/用户角色prompt等）
├── dataset.ts        # 数据集相关prompt（数据集搜索工具响应）
└── utils.ts          # Prompt工具函数（版本选择）
```

### 2.2 Prompt类型定义

```typescript
// packages/global/core/ai/type.d.ts
export type PromptTemplateItem = {
  title: string;        // 模板标题
  desc: string;         // 模板描述
  value: Record<string, string>;  // 版本号映射到具体prompt内容
}
```

## 三、各类Prompt详解

### 3.1 Agent Prompt (agent.ts)

#### 3.1.1 参数提取Prompt

**函数**: `getExtractJsonPrompt`

**用途**: 从用户输入中提取结构化JSON参数

**特性**:
- 支持系统背景知识(`systemPrompt`)
- 支持历史提取结果(`memory`)
- 基于JSON Schema生成参数
- 严格输出JSON格式

**示例结构**:
```typescript
getExtractJsonPrompt({
  systemPrompt: "特定领域知识",
  memory: "历史提取内容",
  schema: JSONSchema对象
})
```

#### 3.1.2 工具调用Prompt

**函数**: `getExtractJsonToolPrompt`

**用途**: 为function calling生成参数

**特性**:
- 结合历史记录、用户输入、背景知识
- 参数可选性处理
- 函数无法调用时仍返回JSON

#### 3.1.3 分类Prompt

**函数**: `getCQSystemPrompt`

**用途**: 对用户问题进行分类判断

**特性**:
- 支持背景知识
- 支持上一轮分类结果
- 输出分类标记

#### 3.1.4 问题引导Prompt

**常量**: `QuestionGuidePrompt`

**用途**: 生成引导性问题

**使用场景**: 在 `createQuestionGuide.ts` 中使用，支持自定义prompt覆盖

### 3.2 AI Chat Prompt (AIChat.ts)

#### 3.2.1 引用相关Prompt模板

**1. 用户角色引用Prompt列表**

```typescript
export const Prompt_userQuotePromptList: PromptTemplateItem[] = [
  {
    title: 'app:template.standard_template',  // 标准模板
    value: {
      '4.9.7': `...prompt内容...`
    }
  },
  {
    title: 'app:template.qa_template',  // 问答模板
    value: {
      '4.9.7': `...prompt内容...`
    }
  },
  {
    title: 'app:template.standard_strict',  // 严格标准模板
    value: {
      '4.9.7': `...prompt内容...`
    }
  }
]
```

**特点**:
- **追溯机制**: 使用 `[id](CITE)` 格式标记引用来源
- **每段话必须包含引用**: 确保可追溯性
- **Markdown格式**: 支持表格、图片等富文本
- **多语言**: 使用与问题相同的语言回答

**2. 系统角色引用Prompt列表**

```typescript
export const Prompt_systemQuotePromptList: PromptTemplateItem[]
```

**特点**:
- 与用户角色类似，但作为系统消息注入
- 在4.9.7版本中结构相同

**3. 引用内容模板列表**

```typescript
export const Prompt_QuoteTemplateList: PromptTemplateItem[]
```

**用途**: 定义如何格式化引用的数据集内容

#### 3.2.2 Prompt获取函数

**1. getQuotePrompt**

```typescript
getQuotePrompt(version?: string, role: 'user' | 'system')
```

- 根据版本和角色获取引用prompt
- 默认使用最新版本

**2. getQuoteTemplate**

```typescript
getQuoteTemplate(version?: string)
```

- 获取引用内容格式化模板

**3. getDocumentQuotePrompt**

```typescript
getDocumentQuotePrompt(version?: string)
```

- 获取文档引用prompt（工具调用场景）

### 3.3 Dataset Prompt (dataset.ts)

**函数**: `getDatasetSearchToolResponsePrompt`

**用途**: 数据集搜索工具的响应prompt

**特性**:
- 知识库回答助手角色
- 引用标记追溯规则
- Markdown格式优化

### 3.4 Prompt工具函数 (utils.ts)

**函数**: `getPromptByVersion`

```typescript
getPromptByVersion(
  promptMap: Record<string, string>, 
  version?: string
): string
```

**功能**:
- 版本号语义化排序（major.minor.patch）
- 未指定版本时返回最新版本
- 指定版本不存在时回退到最新版本

**排序逻辑**:
```typescript
versions.sort((a, b) => {
  const [majorA, minorA, patchA] = a.split('.').map(Number);
  const [majorB, minorB, patchB] = b.split('.').map(Number);
  
  if (majorA !== majorB) return majorB - majorA;
  if (minorA !== minorB) return minorB - minorA;
  return patchB - patchA;
})
```

## 四、Prompt在工作流中的应用

### 4.1 AI Chat节点中的Prompt配置

在工作流的AI Chat节点中，Prompt通过以下参数进行配置:

```typescript
// packages/global/core/workflow/template/system/aiChat/index.ts

export const AiChatQuoteRole = {
  key: NodeInputKeyEnum.aiChatQuoteRole,
  renderTypeList: [FlowNodeInputTypeEnum.hidden],
  valueType: WorkflowIOValueTypeEnum.string,
  value: 'system' // 'user' or 'system'
}

export const AiChatQuoteTemplate = {
  key: NodeInputKeyEnum.aiChatQuoteTemplate,
  renderTypeList: [FlowNodeInputTypeEnum.hidden],
  valueType: WorkflowIOValueTypeEnum.string
  // 用户自定义的引用内容模板
}

export const AiChatQuotePrompt = {
  key: NodeInputKeyEnum.aiChatQuotePrompt,
  renderTypeList: [FlowNodeInputTypeEnum.hidden],
  valueType: WorkflowIOValueTypeEnum.string
  // 用户自定义的引用prompt
}
```

**配置说明**:
- `AiChatQuoteRole`: 决定引用内容注入到系统消息还是用户消息
- `AiChatQuoteTemplate`: 格式化数据集引用内容的模板（支持变量：id, q, a, source, sourceId, updateTime, index）
- `AiChatQuotePrompt`: 包含引用内容的完整prompt（支持变量：quote, question）

### 4.2 工作流UI中的Prompt编辑

**文件**: `projects/app/src/pageComponents/app/detail/WorkflowComponents/Flow/nodes/render/RenderInput/templates/SettingQuotePrompt.tsx`

**功能**:
- **角色选择**: System / User
- **引用内容模板编辑**: PromptEditor组件，支持变量插入
- **引用Prompt编辑**: PromptEditor组件，支持变量插入
- **模板选择**: 通过 `PromptTemplate` 组件从预定义列表中选择

**变量支持**:

**引用模板变量**:
- `{{id}}`: 引用ID
- `{{source}}`: 数据源名称
- `{{sourceId}}`: 数据源ID
- `{{q}}`: 问题
- `{{a}}`: 答案
- `{{updateTime}}`: 更新时间
- `{{index}}`: 引用索引

**引用Prompt变量**:
- `{{quote}}`: 格式化后的引用内容
- `{{question}}`: 用户问题（仅在user角色时可用）

### 4.3 Prompt运行时处理

**文件**: `packages/service/core/workflow/dispatch/ai/chat.ts`

#### 4.3.1 数据集引用处理

```typescript
async function filterDatasetQuote({
  quoteQA,
  model,
  quoteTemplate
}) {
  function getValue({ item, index }) {
    return replaceVariable(quoteTemplate, {
      id: item.id,
      q: item.q,
      a: item.a || '',
      updateTime: formatTime2YMDHM(item.updateTime),
      source: item.sourceName,
      sourceId: String(item.sourceId || ''),
      index: index + 1
    });
  }
  
  const filterQuoteQA = await filterSearchResultsByMaxChars(
    quoteQA, 
    model.quoteMaxToken
  );
  
  const datasetQuoteText = filterQuoteQA
    .map((item, index) => getValue({ item, index }).trim())
    .join('\n------\n');
  
  return { datasetQuoteText };
}
```

#### 4.3.2 消息组装

```typescript
async function getChatMessages({
  aiChatQuoteRole,  // 'user' or 'system'
  datasetQuotePrompt,
  datasetQuoteText,
  version,
  userChatInput,
  systemPrompt,
  // ...
}) {
  // 1. 确定prompt注入位置
  const quoteRole = 
    aiChatQuoteRole === 'user' || 
    datasetQuotePrompt.includes('{{question}}') 
      ? 'user' 
      : 'system';
  
  // 2. 获取默认prompt
  const defaultQuotePrompt = getQuotePrompt(version, quoteRole);
  const datasetQuotePromptTemplate = 
    datasetQuotePrompt || defaultQuotePrompt;
  
  // 3. 替换变量
  if (quoteRole === 'user') {
    replaceInputValue = replaceVariable(datasetQuotePromptTemplate, {
      quote: datasetQuoteText,
      question: userChatInput
    });
  }
  
  // 4. 组装消息
  // ...
}
```

## 五、UI组件

### 5.1 PromptTemplate组件

**文件**: `projects/app/src/components/PromptTemplate/index.tsx`

**功能**: 展示预定义prompt模板供用户选择

**使用场景**: 
- AI Chat节点的引用prompt设置
- 数据集训练的自定义prompt选择

### 5.2 PromptEditor组件

**位置**: `@fastgpt/web/components/common/Textarea/PromptEditor`

**功能**:
- 多行文本编辑
- 变量插入提示
- 语法高亮（变量用 `{{}}` 包裹）

## 六、自定义Prompt支持

### 6.1 应用层自定义

**位置**: `packages/global/core/app/type.d.ts`

```typescript
export type AppDetailType = {
  // ...
  chatConfig: {
    questionGuide: {
      customPrompt?: string;  // 自定义问题引导prompt
    }
  }
}
```

**使用**: 在 `createQuestionGuide.ts` 中:

```typescript
const messages = [
  {
    role: ChatCompletionRequestMessageRoleEnum.System,
    content: `${customPrompt || QuestionGuidePrompt}\n${QuestionGuideFooterPrompt}`
  }
]
```

### 6.2 数据集训练自定义

**位置**: `projects/app/src/pageComponents/dataset/detail/Form/CollectionChunkForm.tsx`

**组件**: `PromptTextarea`

**功能**: 
- 编辑数据集训练时的自定义prompt
- 默认使用 `Prompt_AgentQA`
- 支持固定文本和可编辑部分

**代码示例**:

```typescript
const PromptTextarea = ({ defaultValue, onChange, onClose }) => {
  return (
    <MyModal title="自定义prompt">
      <Textarea defaultValue={defaultValue} />
      <Box>{Prompt_AgentQA.fixedText}</Box>
      <Button onClick={() => {
        const val = ref.current?.value || Prompt_AgentQA.description;
        onChange(val);
      }}>确认</Button>
    </MyModal>
  );
}
```

## 七、版本管理机制

### 7.1 版本号结构

FastGPT使用语义化版本号（Semantic Versioning）: `major.minor.patch`

当前最新版本: **4.9.7**

### 7.2 版本兼容策略

```typescript
// 1. Prompt定义时包含多个版本
const Prompt_userQuotePromptList: PromptTemplateItem[] = [
  {
    title: 'standard_template',
    value: {
      '4.9.7': '...新版本prompt...',
      '4.8.0': '...旧版本prompt...',
      // 可能的未来版本
      '5.0.0': '...未来版本prompt...'
    }
  }
]

// 2. 运行时根据工作流节点版本选择对应prompt
const prompt = getPromptByVersion(templateItem.value, nodeVersion);
```

### 7.3 版本回退机制

如果请求的版本不存在，自动使用最新版本:

```typescript
if (version in promptMap) {
  return promptMap[version];
}
return promptMap[versions[0]];  // 返回最新版本
```

## 八、Prompt变量替换机制

### 8.1 replaceVariable函数

**功能**: 将prompt中的 `{{variableName}}` 替换为实际值

**使用示例**:

```typescript
const prompt = "用户问题: {{question}}\n引用内容: {{quote}}";
const result = replaceVariable(prompt, {
  question: "什么是FastGPT?",
  quote: "FastGPT是一个开源的LLM应用平台..."
});
// 结果: "用户问题: 什么是FastGPT?\n引用内容: FastGPT是一个开源的LLM应用平台..."
```

### 8.2 支持的变量类型

#### 8.2.1 全局变量

- 系统配置节点的输出
- 应用的聊天配置参数

#### 8.2.2 数据集引用变量

- `{{id}}`: 引用数据块ID
- `{{q}}`: 问题
- `{{a}}`: 答案
- `{{source}}`: 数据源名称
- `{{sourceId}}`: 数据源ID
- `{{updateTime}}`: 更新时间
- `{{index}}`: 引用序号

#### 8.2.3 对话相关变量

- `{{quote}}`: 格式化后的引用内容
- `{{question}}`: 用户问题

## 九、工具调用中的Prompt

### 9.1 工具Prompt模板

**文件**: `packages/service/core/ai/llm/promptCall/prompt.ts`

**特性**:
- 使用JSON Schema声明工具参数
- 结构化的工具调用指令
- 0/1前缀标识是否调用工具

**格式示例**:

```
<ToolSkill>
工具使用了 JSON Schema 的格式声明，格式为：
{name: 工具名; description: 工具描述; parameters: 工具参数}

请你根据工具描述，决定回答问题或是使用工具。你的每次输出都必须以0,1开头：
0: 不使用工具，直接回答内容。
1: 使用工具，返回工具调用的参数。

## 回答示例
- 0: 你好，有什么可以帮助你的么？
- 1: {"name": "searchToolId1"}
- 1: {"name": "searchToolId2", "arguments": {"city": "杭州"}}

## 可用工具列表
"""
{{toolSchema}}
"""
</ToolSkill>
```

### 9.2 文档引用Prompt

**函数**: `getDocumentQuotePrompt(version)`

**用途**: 在工具调用场景中注入文档引用内容

**使用位置**: `packages/service/core/workflow/dispatch/ai/tool/index.ts`

```typescript
const concatenateSystemPrompt = [
  toolModel.defaultSystemChatPrompt,
  systemPrompt,
  documentQuoteText 
    ? replaceVariable(getDocumentQuotePrompt(version), {
        quote: documentQuoteText
      })
    : ''
].join('\n\n===---===---===\n\n');
```

## 十、Prompt管理特点总结

### 10.1 优点

#### 1. **代码化管理**
- ✅ 版本控制友好（Git跟踪变更）
- ✅ 类型安全（TypeScript类型检查）
- ✅ 易于代码审查

#### 2. **版本化设计**
- ✅ 支持多版本并存
- ✅ 向后兼容
- ✅ 自动回退到最新版本

#### 3. **模块化组织**
- ✅ 按功能分类（agent、chat、dataset）
- ✅ 职责清晰
- ✅ 易于维护

#### 4. **灵活的自定义**
- ✅ 用户可在UI中自定义prompt
- ✅ 支持变量替换
- ✅ 提供预定义模板选择

### 10.2 局限性

#### 1. **非数据库管理**
- ❌ 无CRUD API
- ❌ 无运行时动态添加/修改
- ❌ 需要代码部署才能更新系统级prompt

#### 2. **无中心化管理界面**
- ❌ 没有专门的Prompt库管理界面
- ❌ 无法可视化查看所有prompt
- ❌ 无法统计prompt使用情况

#### 3. **用户自定义prompt的存储**
- ⚠️ 用户自定义的prompt存储在工作流节点配置中
- ⚠️ 无法跨应用共享自定义prompt
- ⚠️ 无prompt模板市场

### 10.3 与其他LLM平台对比

| 特性 | FastGPT | LangSmith | Dify |
|------|---------|-----------|------|
| Prompt版本管理 | ✅ 代码版本 | ✅ 数据库版本 | ✅ 数据库版本 |
| Prompt库 | ❌ | ✅ | ✅ |
| 动态更新 | ❌ | ✅ | ✅ |
| 类型安全 | ✅ | ❌ | ❌ |
| A/B测试 | ❌ | ✅ | ✅ |
| Prompt市场 | ❌ | ✅ | ✅ |

## 十一、Prompt最佳实践建议

### 11.1 系统开发者

#### 1. **添加新Prompt时**

```typescript
// 1. 在对应分类文件中定义prompt函数
export const getMyNewPrompt = ({ param1, param2 }: MyPromptParams) => {
  return `
## 角色
你是一个${param1}助手

## 任务
${param2}

## 要求
...
  `.trim();
};

// 2. 如果是模板列表，使用PromptTemplateItem[]格式
export const Prompt_MyTemplateList: PromptTemplateItem[] = [
  {
    title: 'my_template_name',
    desc: 'Template description',
    value: {
      '4.9.7': `...prompt content...`,
      // 未来版本
      '5.0.0': `...updated content...`
    }
  }
];

// 3. 导出供其他模块使用
```

#### 2. **Prompt版本更新**

```typescript
// 不要直接修改现有版本的prompt！
// ❌ 错误做法
value: {
  '4.9.7': '修改后的内容'  // 破坏了版本兼容性
}

// ✅ 正确做法
value: {
  '4.9.7': '原有内容',
  '4.10.0': '新版本内容'  // 添加新版本
}
```

### 11.2 应用开发者

#### 1. **使用预定义模板**

在工作流AI Chat节点配置中:
1. 优先选择预定义模板（标准模板、问答模板、严格标准模板）
2. 根据实际需求选择角色（System/User）
3. 必要时基于模板进行微调

#### 2. **自定义Prompt变量使用**

```
## 推荐的变量命名规范
{{userInput}}     ✅ 驼峰命名
{{dataset_quote}} ✅ 下划线分隔
{{DatasetQuote}}  ❌ 避免首字母大写

## 变量使用示例
用户问题: {{question}}
相关知识: {{quote}}
历史对话: {{history}}
```

#### 3. **引用内容模板设计**

```
## 简洁模板（适合短文本）
{{index}}. {{q}}
{{a}}

## 详细模板（适合需要溯源）
### 引用 {{index}}
**问题**: {{q}}
**答案**: {{a}}
**来源**: {{source}}
**更新时间**: {{updateTime}}
```

## 十二、未来可能的改进方向

### 12.1 短期改进

1. **Prompt统计分析**
   - 记录各类prompt的使用频率
   - 监控prompt效果（用户反馈、重试率等）

2. **Prompt测试工具**
   - 单元测试框架集成
   - Prompt回归测试

3. **文档增强**
   - 为每个prompt添加详细注释
   - 生成Prompt使用文档

### 12.2 长期改进

1. **混合管理模式**
   - 系统级prompt保持代码化
   - 用户自定义prompt存储到数据库
   - 提供Prompt库管理界面

2. **Prompt市场**
   - 用户可分享自定义prompt
   - 社区投票最佳prompt
   - 一键导入优质prompt

3. **智能优化**
   - 基于使用数据自动优化prompt
   - A/B测试不同prompt版本
   - LLM辅助生成/优化prompt

## 十三、结论

FastGPT采用的是**代码化的、版本化的、分类组织的Prompt管理方式**，而非传统意义上的"统一Prompt管理功能"（如专门的数据库表、CRUD API、管理界面等）。

**这种方式的核心价值在于**:
- ✅ 开发者友好（代码化、类型安全）
- ✅ 版本控制（Git管理）
- ✅ 模块化设计（职责清晰）
- ✅ 灵活扩展（用户可自定义）

**但也存在明显的局限**:
- ❌ 运营人员无法直接管理prompt
- ❌ 无法动态更新系统级prompt
- ❌ 缺乏prompt效果分析工具

**总体评价**: FastGPT的prompt管理方式**适合技术驱动的团队**，对于需要频繁调整prompt或由非技术人员管理prompt的场景，可能需要进一步增强prompt管理功能。
