# 📘 Dify 文档开源同步操作手册

> **方案**：Git Subtree 双向同步  
> **私有仓库**：`awesome-ai-projects-analysis/dify-analysis`  
> **开源子目录**：`dify-analysis/opensource`  
> **开源仓库**：`https://github.com/evilstar2016/awesome-dify-analysis`  
> **创建时间**：2025-12-30

---

## 📐 目录映射关系

```
awesome-ai-projects-analysis/          ← 私有仓库根目录
├─ dify-analysis/
│   ├─ opensource/                     ← 要开源的子目录
│   │   ├─ README.md
│   │   ├─ Dify-learning-map.md
│   │   ├─ 01-overview/
│   │   ├─ 02-architecture/
│   │   ├─ 03-layers/
│   │   ├─ 04-core-modules/
│   │   ├─ 05-data-architecture/
│   │   ├─ 06-third-party/
│   │   └─ 07-others/
│   └─ ... (其他私有内容)
└─ ...
```

```
awesome-dify-analysis/                 ← 开源仓库根目录
├─ README.md
├─ Dify-learning-map.md
├─ 01-overview/
├─ 02-architecture/
├─ 03-layers/
├─ 04-core-modules/
├─ 05-data-architecture/
├─ 06-third-party/
└─ 07-others/
```

> **关键点**：`opensource/` 的内容 = 开源仓库的根目录

---

## 🚀 一、首次初始化（只执行一次）

### 步骤 1：确保开源仓库已创建且为空

在 GitHub 上确认 `https://github.com/evilstar2016/awesome-dify-analysis` 已创建。

**如果是新仓库，建议不要勾选 "Add README" 等选项，保持完全空白。**

---

### 步骤 2：从私有仓库分离文档历史

```powershell
# 进入私有仓库根目录
cd E:\Github\awesome-ai-projects-analysis

# 从 opensource 子目录分离出独立的 Git 历史
git subtree split -P dify-analysis/opensource -b opensource-split
```

> ⚠️ 这一步会创建一个名为 `opensource-split` 的本地分支，包含 opensource 目录的完整历史。

---

### 步骤 3：推送到开源仓库

```powershell
# 推送分离的历史到开源仓库的 main 分支
git push https://github.com/evilstar2016/awesome-dify-analysis.git opensource-split:main
```

✅ 现在开源仓库已经拥有完整的文档内容和历史。

---

### 步骤 4：清理临时分支（可选）

```powershell
# 删除本地临时分支
git branch -D opensource-split
```

---

### 步骤 5：建立 Subtree 关联

```powershell
# 先删除现有的 opensource 目录（因为要用 subtree 重新关联）
Remove-Item -Recurse -Force "dify-analysis/opensource"
git add -A
git commit -m "chore: prepare for subtree setup"

# 用 subtree add 把开源仓库挂载回来
git subtree add --prefix=dify-analysis/opensource https://github.com/evilstar2016/awesome-dify-analysis.git main --squash
```

✅ 从这一刻起，`dify-analysis/opensource` = 一个可发布的文档产品。

---

## 📅 二、日常工作流

### 场景 A：在私有仓库中修改文档（最常见）

```powershell
# 1. 正常修改 dify-analysis/opensource 目录下的文件
# 2. 提交到私有仓库
git add -A
git commit -m "docs: 更新 xxx 文档"

# 3. 同步到开源仓库
git subtree push --prefix=dify-analysis/opensource https://github.com/evilstar2016/awesome-dify-analysis.git main
```

---

### 场景 B：直接在 GitHub 开源仓库修改（如接受 PR）

```powershell
# 拉取开源仓库的变更到私有仓库
git subtree pull --prefix=dify-analysis/opensource https://github.com/evilstar2016/awesome-dify-analysis.git main --squash
```

---

## ⚡ 三、配置快捷命令（强烈推荐）

### 设置 Git 别名

```powershell
# 在私有仓库根目录执行
cd E:\Github\awesome-ai-projects-analysis

# 设置推送别名
git config alias.docs-push "subtree push --prefix=dify-analysis/opensource https://github.com/evilstar2016/awesome-dify-analysis.git main"

# 设置拉取别名
git config alias.docs-pull "subtree pull --prefix=dify-analysis/opensource https://github.com/evilstar2016/awesome-dify-analysis.git main --squash"
```

### 使用方式

```powershell
# 推送文档到开源仓库
git docs-push

# 从开源仓库拉取更新
git docs-pull
```

---

## 🛡️ 四、安全加固

### 1. 开源仓库的 `.gitignore`

在 `dify-analysis/opensource/` 目录下创建 `.gitignore`：

```gitignore
# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db

# 临时文件
*.tmp
*.bak

# 私有内容（双重保险）
_SYNC_GUIDE.md
_OPENSOURCE_CHECKLIST.md
operator_strategy/
```

---

### 2. 开源仓库 README 声明

在开源仓库的 `README.md` 顶部添加：

```markdown
> 📌 **声明**：本仓库是从私有项目中提取的公开文档镜像。
> 代码示例仅用于说明目的，不保证生产可用性。
> 欢迎提交 Issue 和 PR 改进文档！
```

---

## 📋 五、完整命令速查表

| 操作 | 命令 |
|------|------|
| **首次分离历史** | `git subtree split -P dify-analysis/opensource -b opensource-split` |
| **首次推送** | `git push <开源仓库URL> opensource-split:main` |
| **建立关联** | `git subtree add --prefix=dify-analysis/opensource <开源仓库URL> main --squash` |
| **日常推送** | `git docs-push` 或完整命令 |
| **日常拉取** | `git docs-pull` 或完整命令 |

---

## ⚠️ 六、注意事项

### ❌ 不要做的事

1. **不要**在开源仓库中添加与私有仓库无关的文件
2. **不要**在私有仓库中直接 `git rm -rf opensource` 然后重建
3. **不要**在两端同时修改同一文件后忘记同步

### ✅ 最佳实践

1. **先 pull 再 push**：每次推送前，先拉取开源仓库的最新内容
2. **小步提交**：频繁提交和同步，避免大量冲突
3. **明确边界**：`opensource/` 目录只放公开内容

---

## 🔄 七、故障排除

### 问题 1：push 时提示历史不相关

```powershell
# 使用 --force 强制推送（仅首次或确认安全时使用）
git push https://github.com/evilstar2016/awesome-dify-analysis.git opensource-split:main --force
```

### 问题 2：subtree pull 冲突

```powershell
# 解决冲突后
git add -A
git commit -m "chore: resolve subtree merge conflict"
```

### 问题 3：忘记用 subtree 命令，直接修改了文件

没关系！正常 commit 后使用 `git docs-push` 即可，subtree 会处理好。

---

## 📊 八、当前开源内容统计

| 类型 | 数量 |
|------|------|
| PlantUML 架构图 (.puml) | 33 个 |
| Markdown 文档 (.md) | 17 个 |
| PNG/JPG 图片 | 9 个 |
| **总计** | **59 个文件** |

---

## 🎯 九、后续扩展

当文档需要扩展时：

1. **新增文档**：直接在 `opensource/` 对应目录添加
2. **新增模块**：在 `opensource/` 下创建新目录
3. **删除内容**：正常删除后 commit 并 push

---

**📅 最后更新**：2025-12-30  
**👤 维护者**：AI 架构解析
