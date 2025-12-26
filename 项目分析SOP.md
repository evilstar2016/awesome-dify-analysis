1. 使用 serena 工具分析项目，生成初始信息
**输出文档**：
serena memory信息。

2. 使用serena工具深度分析项目代码，编写md格式的项目overview文档，输出到anylisis目录下。中文，代码结构改成树形展示
**输出文档**：
anylisis\01-overview\project-overview.md

3. 使用serena工具深度分析项目代码，编写md格式的系统架构图描述，使用puml语法绘制系统的组件架构图，风格参考 .github\copilot-instructions.md
**输出文档**：
anylisis\02-architecture\system-architecture.md
anylisis\02-architecture\system-componet-architecture.puml

4. 使用serena工具深度分析代码，参照组件架构图，依次从编写前端-API层-... 各组件的说明文档（md格式），每个组件1个文件（类似于索引文件，下一步会详细编写组件内部细节）

**输出文档（示例）**：
anylisis\03-layers\01-frontend-layer.md     ---- 前端展示层
anylisis\03-layers\02-api-routes-layer.md   ---- API 路由层
………………


5. 使用serena阅读代码， 深度分析 {{模块名}}， 编写md格式说明文档，识别核心流程并绘制puml格式的时序图。
该步骤需要依据步骤3生成的 system-componet-architecture.puml 架构中，针对每个模块进行分析。

**输入示例**：
接下来 使用serena阅读代码， 深度分析 知识库管理功能， 编写md格式说明文档，识别核心流程并绘制puml格式的时序图

**输出文档示例**：
anylisis\04-core-modules\knowledge-base\README.md
anylisis\04-core-modules\knowledge-base\knowledge-base-management.md
anylisis\04-core-modules\knowledge-base\01-dataset-create-sequence.puml
anylisis\04-core-modules\knowledge-base\02-xxxx.puml
…………


6. 步骤5中的所有模块内容都生成后，针对核心模块中的核心功能或特色功能，再进行深度分析，并生成相关文档。

**输入示例**：
使用serena阅读代码， 深度分析 知识库管理 中的文档处理功能：
都支持处理哪些文档？
每类文档是如何被解析、分块的？
针对每类文档 分别编写md格式说明文档 以及puml格式的文档导时序图

**输出示例**：
anylisis\04-core-modules\knowledge-base\document-processing\00-overview.md
anylisis\04-core-modules\knowledge-base\document-processing\01-txt-md-parser.md
anylisis\04-core-modules\knowledge-base\document-processing\01-txt-md-sequence.puml
………………

7. 分析第三方集成方案

**输入示例**：
使用serena阅读代码，该项目中 都可以集成哪些第三方？编写md文档， 整理到 anylisis\05-ThirdParty 目录下


最终的目录结构参考：
E:\GITHUB\FASTGPT\ANYLISIS
├─01-overview
├─03-layers
├─04-core-modules
│  ├─agent
│  ├─knowledge-base
│  │  └─document-processing
│  ├─mcp-tools-plugins
│  ├─prompt-management
│  └─workflow
└─05-ThirdParty

