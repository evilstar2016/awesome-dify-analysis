# 07-others - Supplementary Analysis Documents

This directory contains supplementary analysis documents that do not belong to the main seven categories (overview, architecture, layers, core-modules, data-architecture, third-party).

## 📚 Document List

### 1. [Context Engineering Concept Clarification.md](./上下文工程概念澄清.md)

**Type:** Concept clarification and comparative analysis

**Content:**
- 📊 **Session-Level Context Engineering**
  - Definition, time scope, core content, application scenarios
  - Dify's complete support status

- 📊 **User-Level Context Engineering**
  - Definition, time scope, core content, application scenarios
  - Cross-session user profiles and personalized customization
  - Current implementation status (basic support)

- 🔄 **Comparative Analysis**: Differences and applicable scenarios between two types of context engineering

**Target Readers:** Developers and product designers who need to understand Dify's context management mechanisms

---

### 2. [Dify Knowledge Q&A Scenario - Context Engineering Application Details.md](./Dify知识问答场景-上下文工程应用详解.md)

**Type:** Technical deep analysis

**Content:**
- 🏗️ **Architecture Design**: Knowledge retrieval → Context construction → Prompt assembly → LLM generation
- 🔄 **Complete Processing Flow** (6 stages):
  1. User input and context variable preparation
  2. Knowledge base retrieval and context acquisition
  3. Context construction and filtering
  4. Prompt assembly and variable injection
  5. Token management and conversation history trimming
  6. LLM calling and quality tracking

- 💻 **Code Implementation Analysis**:
  - Frontend: PromptEditor component
  - Backend: Context processing, vector retrieval, re-ranking, Token management

- 📊 **Best Practices**

**Core Variables:**
```
{{#context#}}   - Knowledge base retrieval results
{{#histories#}} - Conversation history (current session)
{{#query#}}     - User's current question
{{自定义}}      - User-defined variables
```

**Target Readers:** Developers who need to deeply understand Dify's knowledge Q&A internal implementation

---

### 3. [context_engineering_qa_flow.puml](./context_engineering_qa_flow.puml)

**Type:** PlantUML flow diagram

**Visualization Content:**
- Complete data flow and component interaction for knowledge Q&A scenarios
- Context engineering processing steps
- End-to-end system flow

**Viewing Methods:**
- Online: https://www.plantuml.com/plantuml/uml/ (copy content and paste)
- VS Code: Install PlantUML plugin, press Alt + D to preview
- Command line: `plantuml context_engineering_qa_flow.puml`

**Suitable Readers:** All developers who need to quickly understand the knowledge Q&A system flow

---

## 🔗 Related Main Analysis

- [04-core-modules/knowledgeBase](../04-core-modules/knowledgeBase/) - Knowledge base module details
- [04-core-modules/prompt](../04-core-modules/prompt/) - Prompt management and templates
- [03-layers/api](../03-layers/api/) - API layer design and controllers

---

📱 **Follow WeChat Official Account 「柒叔代码阁」**

Regularly publish Dify in-depth content and production practices~

![Official Account QR Code](../qrcode.png)

💡 **Why follow?**

Not just tell you "what" and "how", but more importantly **"why"** — understanding the thinking behind the design allows you to apply it flexibly.

💬 **Discussion and Exchange**
- GitHub Issue/Discussion: Technical problem discussions
- Official Account Messages

---