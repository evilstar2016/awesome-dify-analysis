# 内容审核子模块概览

## 功能定位

Dify 内容审核模块支持**三种审核策略**，满足不同场景的内容安全需求。用户可根据业务特点灵活选择单一策略或组合使用。

---

## 支持的审核策略

| 审核类型 | 文件路径 | 特点 | 适用场景 |
|---------|--------|------|---------|
| **关键词审核** | `keywords/keywords.py` | 快速、本地、低成本 | 简单的词汇过滤、敏感词库 |
| **OpenAI 审核** | `openai_moderation/openai_moderation.py` | 准确、智能、多语言 | 复杂内容理解、多语言支持 |
| **自定义 API 审核** | `api/api.py` | 灵活、可定制、可扩展 | 专有审核逻辑、第三方集成 |

---

## 技术架构

### 核心设计

```
┌─────────────────────────────────┐
│     审核工厂 (Factory)           │
│   ModerationFactory             │
└────────────┬────────────────────┘
             │
             ├─ 关键词审核
             │  KeywordsModeration
             │
             ├─ OpenAI 审核
             │  OpenAIModeration
             │
             └─ API 审核
                ApiModeration
```

### 核心接口

所有审核策略继承自 **`Moderation` 基类**，必须实现：

| 方法 | 签名 | 说明 |
|-----|------|------|
| `validate_config()` | `@classmethod` | 验证配置有效性 |
| `moderation_for_inputs()` | 实例方法 | 输入内容审核 |
| `moderation_for_outputs()` | 实例方法 | 输出内容审核 |

---

## 各策略对比

### 性能对比

| 指标 | 关键词 | OpenAI | 自定义 API |
|-----|-------|--------|-----------|
| 响应时间 | < 1ms | 500-2000ms | 取决于实现 |
| 计算消耗 | 极低 | 中 | 中等 |
| 网络依赖 | 否 | 是 | 是 |

### 准确性对比

| 指标 | 关键词 | OpenAI | 自定义 API |
|-----|-------|--------|-----------|
| 准确率 | ⭐⭐ 低 | ⭐⭐⭐⭐⭐ 高 | ⭐⭐⭐ 中高 |
| 误报率 | 高 | 低 | 取决于实现 |
| 漏报率 | 高 | 低 | 取决于实现 |

### 成本对比

| 指标 | 关键词 | OpenAI | 自定义 API |
|-----|-------|--------|-----------|
| 成本 | 💰 无 | 💰💰💰 高 | 💰💰 中 |
| 配置难度 | 低 | 低 | 高 |
| 运维成本 | 无 | 低 | 中 |

---

## 配置结构统一规范

### 基础配置字段

所有审核策略共享以下配置结构：

```json
{
  "enabled": true,
  "type": "{strategy_type}",
  "config": {
    "inputs_config": {
      "enabled": true,
      "preset_response": "输入包含不当内容"
    },
    "outputs_config": {
      "enabled": true,
      "preset_response": "输出包含不当内容"
    },
    // 策略特定字段...
  }
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `enabled` | boolean | ✅ | 是否启用审核 |
| `type` | string | ✅ | 审核策略类型 |
| `config.inputs_config.enabled` | boolean | ✅ | 是否启用输入审核 |
| `config.outputs_config.enabled` | boolean | ✅ | 是否启用输出审核 |

---

## 使用流程

### 1. 选择策略
根据业务需求选择合适的审核策略

### 2. 配置参数
填写策略特定的配置参数

### 3. 验证配置
系统自动验证配置有效性（`validate_config`）

### 4. 保存应用
将配置保存到 `App.sensitive_word_avoidance`

### 5. 执行审核
在输入/输出时自动执行审核（通过 Factory 自动选择）

---

## 审核流程

### 输入审核

```
用户输入
  ↓
InputModeration.check()
  ↓
ModerationFactory.moderation_for_inputs()
  ↓
具体策略实现（Keywords/OpenAI/API）
  ↓
审核结果
  └─→ flagged=True  → 返回预设响应
  └─→ flagged=False → 继续处理
```

### 输出审核

```
LLM 生成内容
  ↓
OutputModeration 缓冲累积
  ↓
达到缓冲区大小或生成完成
  ↓
ModerationFactory.moderation_for_outputs()
  ↓
具体策略实现（Keywords/OpenAI/API）
  ↓
审核结果
  └─→ flagged=True  → 返回预设响应
  └─→ flagged=False → 返回原始内容
```

---

## 子模块文档清单

### 1. 关键词审核 (Keywords Moderation)
- **文件**: [01-keywords-moderation.md](01-keywords-moderation.md)
- **核心类**: `KeywordsModeration`
- **特点**: 本地、快速、成本低
- **适用**: 敏感词过滤、简单关键词匹配

### 2. OpenAI 审核 (OpenAI Moderation)
- **文件**: [02-openai-moderation.md](02-openai-moderation.md)
- **核心类**: `OpenAIModeration`
- **特点**: 智能、准确、多语言
- **适用**: 复杂内容理解、多语言内容

### 3. 自定义 API 审核 (API Moderation)
- **文件**: [03-api-moderation.md](03-api-moderation.md)
- **核心类**: `ApiModeration`
- **特点**: 灵活、可定制、可扩展
- **适用**: 专有审核逻辑、第三方集成

---

## 扩展指南

### 开发新审核策略

创建新的审核策略需要：

1. **继承基类**
```python
from core.moderation.base import Moderation

class CustomModeration(Moderation):
    name: str = "custom"
```

2. **实现必要方法**
```python
@classmethod
def validate_config(cls, tenant_id: str, config: dict):
    pass

def moderation_for_inputs(self, inputs: dict, query: str = "") -> ModerationInputsResult:
    pass

def moderation_for_outputs(self, text: str) -> ModerationOutputsResult:
    pass
```

3. **注册到扩展系统**
通过 `ExtensionModule.MODERATION` 框架自动注册

4. **配置使用**
```python
app_config.sensitive_word_avoidance = {
    "type": "custom",
    "config": { ... }
}
```

---

## 常见问题

### Q: 如何同时使用多种审核策略？

**A**: 内置的三种策略只能同时使用一种（通过 `type` 字段指定）。如需多层审核，可以：
- 使用自定义 API 审核，在 API 端集合多种策略
- 开发新的策略类，在内部组合多种审核方法

### Q: 审核失败会影响用户体验吗？

**A**: 取决于配置的 `action`：
- `DIRECT_OUTPUT`: 直接返回预设响应，无法看到原始内容
- `OVERRIDDEN`: 返回修改后的内容，内容被过滤但不被完全拒绝

### Q: OpenAI 审核需要什么配置？

**A**: 需要配置 OpenAI 账户和 API Key。系统会自动使用 `omni-moderation-latest` 模型。

### Q: 自定义 API 审核如何集成？

**A**: 需要通过"API-based Extensions"功能创建一个自定义端点，暴露 `app.moderation.input` 和 `app.moderation.output` 接口点。

---

## 参考资源

- **基类定义**: [api/core/moderation/base.py](../../../../api/core/moderation/base.py)
- **工厂类**: [api/core/moderation/factory.py](../../../../api/core/moderation/factory.py)
- **输入审核**: [api/core/moderation/input_moderation.py](../../../../api/core/moderation/input_moderation.py)
- **输出审核**: [api/core/moderation/output_moderation.py](../../../../api/core/moderation/output_moderation.py)
- **测试用例**: [api/tests/unit_tests/core/moderation/](../../../../api/tests/unit_tests/core/moderation/)

---

**文档版本**: 1.0  
**最后更新**: 2026-01-21
