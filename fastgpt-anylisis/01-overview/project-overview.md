# FastGPT 项目 Overview

## 项目定位
- AI Agent 构建平台：提供开箱即用的数据处理、模型调用与可视化 Flow 编排，支持复杂对话与知识库检索场景。
- 单仓多包（pnpm workspaces）架构，主应用基于 Next.js 14 + TypeScript + Chakra UI。

## 技术栈要点
- 前端/UI：Next.js 14、React 18、Chakra UI、Framer Motion、React Query、React Flow 等。
- 服务端/Agent 能力：Next 14（API 路由）、BullMQ 任务队列、Mongoose（MongoDB）、pg/PGVector、Milvus SDK、Redis (ioredis)、MinIO、OpenTelemetry（pino/winston 集成）、MCP SDK。
- 向量/数据：MongoDB、PostgreSQL+PGVector 或 Milvus；MinIO 对象存储。
- 包管理/工具：pnpm (Node >=20, 推荐 9.15.9)，Vitest 单测，ESLint + Prettier，Husky + lint-staged。

## 代码结构（核心）
```
repo
├─ packages
│  ├─ global          # 跨端共用：common/core/openapi/sdk/support
│  ├─ service         # 服务侧：common/core/support/thirdProvider/type/worker
│  └─ web             # Web UI：common/components/context/core/hooks/i18n/store/styles/support
├─ projects
│  ├─ app             # 主应用 (Next.js)，含 build:workers 脚本
│  ├─ marketplace     # 市场相关子项目
│  ├─ mcp_server      # MCP 相关子项目
│  └─ sandbox         # 工作流代码执行（Python 3.11，需更新 requirements.txt/SYSTEM_CALLS）
├─ document           # 官方文档站 (fumadoc/Next)，需 .env.local 与 npm install
├─ deploy             # Helm / Docker / Sealos 模板
├─ scripts            # icon/i18n 等脚本；Makefile dev/build 包装
├─ test               # Vitest 配置与用例
└─ 其他根目录文件     # dev.md、README*、配置/锁文件等
```

## 环境与依赖
- Node >= 20（若使用 20+，安装时需 `NODE_OPTIONS=--no-node-snapshot`）。
- pnpm 工作区；根 `package.json` 定义 lint/format/test 等脚本，postinstall 会生成 Chakra 主题类型。
- ESLint 规则：继承 `next/core-web-vitals`，偏好 type-only imports；Prettier：100 列、单引号、无尾逗号、LF。
- i18n：`t(namespace:key)`，key 使用小写+下划线。

## 开发与运行
- 依赖安装（PowerShell，Node>=20）：`$env:NODE_OPTIONS="--no-node-snapshot"; pnpm i`
- 启动主应用：`pnpm --prefix ./projects/app dev` 或进入 `projects/app` 执行 `pnpm dev`；若有 make：`make dev name=app`。
- 文档站：在 `document/` 运行 `npm install` 与 `npm run dev`（默认 3000）。

## 质量与测试
- 代码格式：`pnpm format-code`（Prettier 作用于 `**/src/**/*.{ts,tsx,scss}`）。
- 文档格式：`pnpm format-doc` + `pnpm initDocTime` + `pnpm initDocToc`（lint-staged 自动处理 MDX）。
- Lint：`pnpm lint`（ESLint，带 `--fix`）。
- 测试：`pnpm test`（Vitest 全量）、`pnpm test:workflow`（工作流相关）。

## 构建与发布
- 主应用构建：`pnpm --prefix ./projects/app build`（包含 `build:workers`）。
- Docker/镜像：
  - 直接：`docker build -f ./projects/app/Dockerfile -t <image> . --build-arg name=app [--build-arg proxy=taobao]`
  - Make：`make build name=app image=<image> [proxy=taobao|clash]`

## 其他提示
- 工作流/Sandbox 若新增 Python 包，请更新 `requirements.txt` 并根据需要调整 `SYSTEM_CALLS` 白名单。
- 部署可参考 `deploy/` 下的 Helm、Docker Compose 与 Sealos 模板。
- 共享逻辑优先放入 `packages/global`，前后端复用通过 workspace `@fastgpt/*` 引用。
