# Dify 知识问答场景 - 会话级上下文工程应用详解

## ⚠️ 重要说明

**本文档讨论的是"会话级上下文工程"（Session-Level Context Engineering）**，即在**单次对话会话内**管理上下文信息。

如果您在寻找"用户级上下文工程"（跨会话的用户画像、个性化记忆等），请参阅 [上下文工程概念澄清.md](./上下文工程概念澄清.md)。

---

## 📋 概述

本文档详细解析 Dify 在知识问答场景中如何应用**会话级上下文工程（Session-Level Context Engineering）**来优化用户体验。通过深入代码分析，揭示 Dify 如何通过多层次的上下文管理技术，实现准确、连贯、可追溯的智能问答。

## 🎯 什么是会话级上下文工程？

会话级上下文工程是指在 LLM 应用的**单次对话会话内**，系统化地**收集、组织、注入和管理上下文信息**的技术实践。

**范围：** 仅在当前对话会话中有效，会话结束后上下文不保留

**核心内容：**
- 📚 知识库检索结果 (`{{#context#}}`)
- 💬 对话历史 (`{{#histories#}}`) - 当前会话的历史消息
- ❓ 用户查询 (`{{#query#}}`) - 当前问题

**目的：**

1. **提高回答准确性** - 基于相关知识而非模型幻觉
2. **保持对话连贯性** - 记忆和利用历史对话
3. **增强可控性** - 通过上下文引导模型行为
4. **优化 Token 使用** - 动态裁剪，避免超限

---

## 🏗️ Dify 的上下文工程架构

### 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                    知识问答场景架构                           │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  [用户输入] → [知识检索] → [上下文构建] → [Prompt组装]        │
│                    ↓            ↓            ↓               │
│              向量检索     文档过滤     模板变量注入            │
│              元数据过滤    重排序      历史对话管理            │
│              混合检索     Top-K       Token管理               │
│                    ↓            ↓            ↓               │
│              [LLM生成] ← [上下文注入] ← [质量追踪]            │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔍 分阶段详解

### 阶段一：用户输入与上下文变量准备

#### 1.1 前端 - PromptEditor 组件

**文件位置：** `web/app/components/base/prompt-editor/index.tsx`

```typescript
// 用户可以在可视化编辑器中插入上下文块
<PromptEditor
  contextBlock={{
    show: true,                    // 显示上下文块
    selectable: true,              // 可选择知识库
    datasets: [{id, name}, ...]    // 关联的数据集
  }}
  historyBlock={{
    show: true,                    // 显示历史对话块
    history: {user: 'Human', assistant: 'Assistant'}
  }}
  queryBlock={{
    show: true,                    // 显示查询块
  }}
/>
```

**支持的特殊变量：**
- `{{#context#}}` - 知识库检索结果
- `{{#histories#}}` - 对话历史
- `{{#query#}}` - 用户当前问题
- `{{自定义变量}}` - 用户自定义输入

#### 1.2 后端 - 初始化运行时状态

**文件位置：** `api/core/app/apps/advanced_chat/app_generator.py`

```python
class AdvancedChatAppGenerator:
    def generate(self, ...):
        # 创建应用生成实体
        application_generate_entity = AdvancedChatAppGenerateEntity(
            task_id=str(uuid.uuid4()),
            query=query,                    # 用户问题
            inputs=inputs,                  # 输入变量
            conversation_id=conversation.id, # 对话ID
            files=files,                    # 上传的文件
            ...
        )
        
        # 初始化变量池 - 存储所有上下文变量
        # 后续节点可以从变量池获取/设置变量
```

**关键数据结构：**
```python
SystemVariable(
    query="用户的问题",
    conversation_id="会话ID",
    user_id="用户ID",
    dialogue_count=1,  # 对话轮次
    workflow_id="工作流ID",
    ...
)
```

---

### 阶段二：知识库检索 - 构建上下文

这是上下文工程的**核心阶段**，通过多层次优化确保检索质量。

#### 2.1 知识检索节点

**文件位置：** `api/core/workflow/nodes/knowledge_retrieval/knowledge_retrieval_node.py`

```python
class KnowledgeRetrievalNode(Node):
    def _run(self) -> NodeRunResult:
        # 1. 从变量池提取用户查询
        variable = self.graph_runtime_state.variable_pool.get(
            self._node_data.query_variable_selector
        )
        query = variable.value
        
        # 2. 执行知识检索
        results, usage = self._fetch_dataset_retriever(
            node_data=self._node_data, 
            query=query
        )
        
        # 3. 输出检索结果到变量池
        return NodeRunResult(
            outputs={"result": ArrayObjectSegment(value=results)}
        )
```

#### 2.2 多层次检索优化

##### 🎯 优化 1：元数据过滤（Metadata Filtering）

**作用：** 在向量检索前先过滤文档，缩小检索范围

**三种模式：**

1. **Disabled（禁用）** - 不进行过滤
2. **Automatic（自动）** - LLM 自动从查询中提取过滤条件
3. **Manual（手动）** - 用户预设过滤规则

**自动过滤示例：**

```python
# 用户查询："2023年的产品文档中关于API的内容"
# 
# 系统自动调用 LLM 提取元数据：
def _automatic_metadata_filter_func(self, dataset_ids, query, node_data):
    # 使用专门的 Prompt 模板
    prompt_messages = self._get_prompt_template(
        metadata_fields=["year", "type", "category"],
        query=query
    )
    
    # LLM 返回过滤条件
    result = model_instance.invoke_llm(prompt_messages)
    
    # 解析 JSON：
    # {
    #   "metadata_map": [
    #     {"metadata_field_name": "year", 
    #      "metadata_field_value": "2023",
    #      "comparison_operator": "="},
    #     {"metadata_field_name": "type", 
    #      "metadata_field_value": "product_doc",
    #      "comparison_operator": "="}
    #   ]
    # }
```

**文件位置：** `api/core/workflow/nodes/knowledge_retrieval/knowledge_retrieval_node.py:L523`

##### 🎯 优化 2：检索策略选择

**文件位置：** `api/core/rag/retrieval/dataset_retrieval.py:L88`

**两种模式：**

```python
# 模式一：Single（单数据集路由）
# 适用场景：多个专业领域数据集，需要智能选择
if retrieve_config.retrieve_strategy == "SINGLE":
    # 使用 LLM 进行路由决策
    if model_supports_function_calling:
        # Function Calling 路由
        planning_strategy = PlanningStrategy.ROUTER
    else:
        # ReAct 路由
        planning_strategy = PlanningStrategy.REACT_ROUTER
    
    all_documents = self.single_retrieve(
        available_datasets=datasets,
        query=query,
        planning_strategy=planning_strategy,
        ...
    )

# 模式二：Multiple（多数据集并行检索）
# 适用场景：需要综合多个数据源的信息
elif retrieve_config.retrieve_strategy == "MULTIPLE":
    all_documents = self.multiple_retrieve(
        available_datasets=datasets,
        query=query,
        top_k=4,
        score_threshold=0.7,
        rerank_mode="reranking_model",  # 或 "weighted_score"
        weights={"vector_weight": 0.7, "keyword_weight": 0.3},
        ...
    )
```

**混合检索（Hybrid Search）：**

```python
# 在 Multiple 模式下，可以启用混合检索
# 同时使用向量相似度和关键词匹配

# 向量检索得分
vector_scores = self.calculate_vector_score(query, documents)

# 关键词检索得分
keyword_scores = self.calculate_keyword_score(query, documents)

# 加权合并
final_scores = (
    vector_scores * vector_weight + 
    keyword_scores * keyword_weight
)
```

##### 🎯 优化 3：重排序（Reranking）

**作用：** 使用专门的重排序模型提升相关性排序质量

```python
# 支持的重排序模型：
# - Cohere Rerank
# - Jina Rerank
# - BGE Reranker（本地）

if reranking_enabled:
    # 将 Top-N 候选文档发送给重排序模型
    reranked_documents = rerank_model.rerank(
        query=query,
        documents=candidate_documents,
        top_n=top_k
    )
```

**文件位置：** `api/core/rag/rerank/rerank_model.py`

##### 🎯 优化 4：Top-K 与阈值过滤

```python
# 在 DatasetRetrieval.retrieve() 方法中：

# 1. 按相关性得分排序
document_context_list = sorted(
    document_context_list, 
    key=lambda x: x.score or 0.0, 
    reverse=True
)

# 2. 应用 Top-K 限制
top_k_documents = document_context_list[:top_k]

# 3. 应用分数阈值过滤
filtered_documents = [
    doc for doc in top_k_documents 
    if doc.score >= score_threshold
]

# 4. 构建上下文字符串
context_string = "\n".join([
    doc.content for doc in filtered_documents
])
```

#### 2.3 上下文格式化

**文件位置：** `api/core/rag/retrieval/dataset_retrieval.py:L252`

```python
# 格式化检索结果为上下文字符串

for record in records:
    segment = record.segment
    
    # 如果是 Q&A 格式
    if segment.answer:
        context += f"question:{segment.get_sign_content()}\n"
        context += f"answer:{segment.answer}\n\n"
    else:
        # 普通文档片段
        context += f"{segment.get_sign_content()}\n\n"

# 最终上下文示例：
"""
question: 什么是RAG?
answer: RAG（检索增强生成）是一种结合检索系统和生成模型的技术...

question: Dify支持哪些向量数据库?
answer: Dify支持多种向量数据库，包括Weaviate、Qdrant、Milvus...

Dify的RAG系统具有以下特点：
1. 多种检索策略支持
2. 智能重排序
3. 元数据过滤
...
"""
```

---

### 阶段三：Prompt 组装 - 上下文注入

#### 3.1 Prompt 模板解析

**文件位置：** `api/core/prompt/utils/prompt_template_parser.py`

```python
class PromptTemplateParser:
    """
    识别并提取模板中的变量
    
    支持的变量格式：
    - {{variable_name}}  # 普通变量
    - {{#context#}}      # 特殊：知识库上下文
    - {{#histories#}}    # 特殊：对话历史
    - {{#query#}}        # 特殊：用户查询
    """
    
    # 正则表达式
    REGEX = re.compile(
        r'\{\{([a-zA-Z_][a-zA-Z0-9_]{0,29}|'
        r'#histories#|#query#|#context#)\}\}'
    )
    
    def extract(self):
        """提取模板中所有变量"""
        return re.findall(self.REGEX, self.template)
    
    def format(self, inputs: dict) -> str:
        """将变量值替换到模板中"""
        return re.sub(
            self.REGEX, 
            lambda m: inputs.get(m.group(1), m.group(0)), 
            self.template
        )
```

#### 3.2 高级 Prompt 转换器

**文件位置：** `api/core/prompt/advanced_prompt_transform.py`

这是上下文工程的**关键类**，负责将所有上下文变量注入到 Prompt 中。

```python
class AdvancedPromptTransform(PromptTransform):
    def get_prompt(
        self,
        prompt_template: list[ChatModelMessage],
        inputs: dict,
        query: str,
        context: str | None,         # 知识库上下文
        memory: TokenBufferMemory,   # 对话历史
        model_config: ModelConfig,
        ...
    ) -> list[PromptMessage]:
        """
        组装最终的 Prompt Messages
        """
        prompt_messages = []
        
        for prompt_item in prompt_template:
            raw_prompt = prompt_item.text
            
            # 1. 解析模板
            parser = PromptTemplateParser(
                template=raw_prompt, 
                with_variable_tmpl=True
            )
            
            # 2. 注入上下文
            prompt_inputs = self._set_context_variable(
                context, parser, inputs
            )
            
            # 3. 注入对话历史
            if memory:
                prompt_inputs = self._set_histories_variable(
                    memory, parser, prompt_inputs, model_config
                )
            
            # 4. 注入用户查询
            prompt_inputs = self._set_query_variable(
                query, parser, prompt_inputs
            )
            
            # 5. 格式化最终 Prompt
            final_prompt = parser.format(prompt_inputs)
            
            prompt_messages.append(
                UserPromptMessage(content=final_prompt)
            )
        
        return prompt_messages
```

##### 🔧 上下文变量注入

```python
def _set_context_variable(
    self, 
    context: str | None, 
    parser: PromptTemplateParser, 
    prompt_inputs: dict
) -> dict:
    """注入知识库上下文"""
    prompt_inputs = dict(prompt_inputs)
    
    if '#context#' in parser.variable_keys:
        if context:
            # 注入检索到的知识库内容
            prompt_inputs['#context#'] = context
        else:
            # 如果没有检索结果，使用空字符串
            prompt_inputs['#context#'] = ""
    
    return prompt_inputs
```

##### 🔧 对话历史注入（Token 管理）

**文件位置：** `api/core/prompt/advanced_prompt_transform.py:L270`

```python
def _set_histories_variable(
    self,
    memory: TokenBufferMemory,
    memory_config: MemoryConfig,
    parser: PromptTemplateParser,
    prompt_inputs: dict,
    model_config: ModelConfig,
) -> dict:
    """
    注入对话历史 - 包含智能 Token 管理
    """
    prompt_inputs = dict(prompt_inputs)
    
    if '#histories#' in parser.variable_keys:
        if memory:
            # **关键：计算剩余 Token 容量**
            # 先构建一个临时消息，计算已使用的 Token
            tmp_prompt = parser.format({'#histories#': "", **prompt_inputs})
            tmp_message = UserPromptMessage(content=tmp_prompt)
            
            # 计算剩余 Token
            rest_tokens = self._calculate_rest_token(
                [tmp_message], 
                model_config
            )
            
            # 从历史记录中提取适量的对话
            histories = memory.get_history_prompt_text(
                max_token_limit=rest_tokens,
                message_limit=memory_config.window.size
            )
            
            # 格式化历史对话
            # Human: 问题1
            # Assistant: 回答1
            # Human: 问题2
            # Assistant: 回答2
            
            prompt_inputs['#histories#'] = histories
        else:
            prompt_inputs['#histories#'] = ""
    
    return prompt_inputs
```

**TokenBufferMemory 工作原理：**

```python
# 文件位置：api/core/memory/token_buffer_memory.py

class TokenBufferMemory:
    def get_history_prompt_text(
        self, 
        max_token_limit: int,
        message_limit: int
    ) -> str:
        """
        获取对话历史，受两个限制：
        1. Token 数量限制
        2. 消息数量限制
        """
        messages = self.get_messages(message_limit)
        
        history_text = ""
        current_tokens = 0
        
        # 从最近的消息开始，逐条添加
        for message in reversed(messages):
            message_text = self._format_message(message)
            message_tokens = self._count_tokens(message_text)
            
            if current_tokens + message_tokens > max_token_limit:
                break  # 超出 Token 限制，停止添加
            
            history_text = message_text + history_text
            current_tokens += message_tokens
        
        return history_text
```

#### 3.3 最终 Prompt 示例

**假设用户模板：**
```
你是一个专业的AI助手。

背景知识：
{{#context#}}

对话历史：
{{#histories#}}

用户问题：
{{#query#}}

请基于背景知识准确回答用户问题，如果背景知识中没有相关信息，请明确说明。
```

**经过转换后的最终 Prompt：**
```
你是一个专业的AI助手。

背景知识：
question: 什么是RAG?
answer: RAG（检索增强生成）是一种结合检索系统和生成模型的技术，通过检索相关文档来增强生成质量。

question: Dify支持哪些向量数据库?
answer: Dify支持多种向量数据库，包括Weaviate、Qdrant、Milvus、PGVector等。

Dify的RAG系统具有以下特点：
1. 多种检索策略支持（单数据集路由、多数据集并行）
2. 智能重排序功能
3. 元数据过滤能力
4. 混合检索（向量+关键词）
5. Token智能管理

对话历史：
Human: Dify支持哪些LLM?
Assistant: Dify支持OpenAI、Claude、Azure OpenAI、Anthropic等多种LLM提供商。

用户问题：
RAG系统中的重排序是如何工作的？

请基于背景知识准确回答用户问题，如果背景知识中没有相关信息，请明确说明。
```

---

### 阶段四：LLM 生成

#### 4.1 上下文质量追踪

**文件位置：** `api/core/app/apps/advanced_chat/app_generator.py`

```python
# 在生成过程中，系统会追踪上下文质量指标

trace_manager.add_trace(
    trace_type="knowledge_retrieval",
    metadata={
        "query": query,
        "dataset_ids": dataset_ids,
        "retrieval_count": len(documents),
        "top_k": top_k,
        "score_threshold": score_threshold,
        "reranking_enabled": True,
        "retrieval_time_ms": elapsed_time,
        "documents": [
            {
                "content": doc.content[:100],  # 前100字符
                "score": doc.score,
                "dataset_id": doc.dataset_id,
                "document_id": doc.document_id,
                "segment_id": doc.segment_id,
            }
            for doc in documents
        ]
    }
)
```

#### 4.2 流式生成

```python
# LLM 生成支持流式输出
for chunk in model_instance.invoke(
    prompt_messages=final_prompt_messages,
    stream=True
):
    yield chunk
```

---

### 阶段五：结果存储与反馈

#### 5.1 消息存储

```python
# 将用户问题和AI回答存入数据库
message = Message(
    conversation_id=conversation.id,
    query=query,
    answer=answer,
    message_metadata={
        "retriever_resources": [
            {
                "dataset_id": doc.dataset_id,
                "document_id": doc.document_id,
                "segment_id": doc.segment_id,
                "score": doc.score,
                "content": doc.content,
            }
            for doc in retrieved_documents
        ]
    }
)

db.session.add(message)
db.session.commit()
```

#### 5.2 反馈机制

用户可以对回答进行评分，系统会记录：

```python
message_feedback = MessageFeedback(
    message_id=message.id,
    rating="like",  # or "dislike"
    content="回答很准确"
)

# 这些反馈可用于：
# 1. 调整检索策略
# 2. 优化重排序模型
# 3. 改进 Prompt 模板
```

---

## 📊 完整流程时序图

已在 `analysis/context_engineering_qa_flow.puml` 文件中创建，展示了完整的 5 阶段流程。

---

## 🎯 6 大上下文工程优化技术总结

### 1. 元数据过滤（Metadata Filtering）
- **作用位置：** 知识检索前
- **优化目标：** 缩小检索范围，提高精准度
- **实现方式：** 自动/手动设置过滤条件
- **核心代码：** `KnowledgeRetrievalNode._get_metadata_filter_condition()`

### 2. 检索策略（Retrieval Strategy）
- **Single路由：** LLM智能选择最合适的单个数据集
- **Multiple并行：** 从多个数据集并行检索，合并结果
- **混合检索：** 向量相似度 + 关键词匹配
- **核心代码：** `DatasetRetrieval.retrieve()`

### 3. 重排序（Reranking）
- **作用位置：** 初步检索后
- **优化目标：** 使用专门模型重新排序，提升相关性
- **支持模型：** Cohere, Jina, BGE
- **核心代码：** `api/core/rag/rerank/rerank_model.py`

### 4. Token 管理
- **动态计算：** 根据模型上下文窗口和已用 Token 动态调整
- **优先级策略：** 保留最近的对话历史
- **防止超限：** 自动裁剪，避免超出 Token 限制
- **核心代码：** `AdvancedPromptTransform._set_histories_variable()`

### 5. Prompt 模板组装
- **变量注入：** `{{#context#}}`, `{{#histories#}}`, `{{#query#}}`
- **格式灵活：** 用户可自定义模板结构
- **类型安全：** 正则表达式验证变量格式
- **核心代码：** `PromptTemplateParser.format()`

### 6. 质量追踪
- **检索追踪：** 记录检索的数据集、文档、分数
- **性能监控：** 记录检索耗时、Token使用
- **用户反馈：** 收集点赞/点踩，用于优化
- **核心代码：** `trace_manager.add_trace()`

---

## 💡 实战建议

### 配置最佳实践

1. **Top-K 设置**
   - 小型知识库（< 1000文档）：Top-K = 3-5
   - 中型知识库（1000-10000）：Top-K = 5-8
   - 大型知识库（> 10000）：Top-K = 8-12

2. **Score Threshold**
   - 精确匹配场景：threshold = 0.8
   - 模糊匹配场景：threshold = 0.6
   - 探索性场景：threshold = 0.4

3. **重排序启用条件**
   - Top-K > 5 时建议启用
   - 多数据集并行检索时强烈推荐
   - 对响应速度要求高时可禁用

4. **对话历史窗口**
   - 闲聊场景：保留 10-20 轮
   - 专业问答：保留 5-10 轮
   - 紧密上下文依赖：保留 15-30 轮

### 性能优化技巧

1. **缓存检索结果**
   ```python
   # 对相同查询的检索结果缓存 5 分钟
   cache_key = f"retrieval:{hash(query)}:{dataset_ids}"
   cached_result = redis.get(cache_key)
   if cached_result:
       return json.loads(cached_result)
   ```

2. **异步并行检索**
   ```python
   # 多数据集并行检索时使用异步
   async def parallel_retrieve():
       tasks = [
           retrieve_from_dataset(dataset_id, query)
           for dataset_id in dataset_ids
       ]
       results = await asyncio.gather(*tasks)
       return merge_results(results)
   ```

3. **智能分块**
   ```python
   # 将大文档分块时，考虑重叠区域
   chunk_size = 500  # Token
   chunk_overlap = 50  # 重叠 Token，保持上下文连续性
   ```

---

## 🔗 相关文件索引

### 核心文件清单

| 文件路径 | 核心功能 | 关键类/方法 |
|---------|---------|------------|
| `api/core/prompt/advanced_prompt_transform.py` | Prompt组装与上下文注入 | `AdvancedPromptTransform.get_prompt()` |
| `api/core/rag/retrieval/dataset_retrieval.py` | 知识检索编排 | `DatasetRetrieval.retrieve()` |
| `api/core/workflow/nodes/knowledge_retrieval/knowledge_retrieval_node.py` | 知识检索节点 | `KnowledgeRetrievalNode._run()` |
| `api/core/prompt/utils/prompt_template_parser.py` | 模板变量解析 | `PromptTemplateParser.format()` |
| `api/core/memory/token_buffer_memory.py` | Token管理内存 | `TokenBufferMemory.get_history_prompt_text()` |
| `web/app/components/base/prompt-editor/` | 前端Prompt编辑器 | `PromptEditor` 组件 |
| `api/core/app/apps/advanced_chat/app_generator.py` | 对话应用生成器 | `AdvancedChatAppGenerator.generate()` |

### 测试文件

| 文件路径 | 测试内容 |
|---------|---------|
| `api/tests/unit_tests/core/prompt/test_advanced_prompt_transform.py` | Prompt转换测试 |
| `api/tests/unit_tests/core/rag/test_dataset_retrieval.py` | 检索逻辑测试 |
| `api/tests/unit_tests/core/memory/test_token_buffer_memory.py` | Token管理测试 |

---

## 🎓 总结

Dify 通过**多层次、系统化的上下文工程技术**，在知识问答场景中实现了：

✅ **高准确性** - 通过元数据过滤和重排序确保检索质量  
✅ **强连贯性** - 通过对话历史管理保持上下文连续  
✅ **高可控性** - 通过灵活的Prompt模板引导LLM行为  
✅ **高效率** - 通过智能Token管理避免浪费和超限  
✅ **可追溯性** - 通过质量追踪实现问题诊断和持续优化

这些技术的有机结合，使得 Dify 能够在企业级应用中提供**准确、可靠、高效**的知识问答服务。

---

## 📚 扩展阅读

- 参见：`analysis/context_engineering_qa_flow.puml` - 完整流程时序图
- 参见：`docs/zh-CN/` - Dify 官方中文文档
- 参见：`api/core/rag/README.md` - RAG 系统架构说明