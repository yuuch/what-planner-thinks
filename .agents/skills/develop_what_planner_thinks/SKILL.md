---
name: Develop What Planner Thinks
description: Guide for modifying and developing the "what-planner-thinks" PostgreSQL query optimizer source code analysis mdBook project.
---

# Develop "What Planner Thinks"

这个 Skill 用于指导如何修改和开发本仓库的 PostgreSQL 查询优化器（Planner）源码深度解析书籍 ——《那个叫 Planner 的家伙在想什么？》。

## 📌 项目概览

本仓库使用 [mdBook](https://rust-lang.github.io/mdBook/) 作为静态站点生成器，专注于深度解析 PostgreSQL (默认版本基于 postgresql_18_1) 的 Planner 流程。
- **构建工具**: mdBook。
- **源码对照**: 包含对应版本的 PostgreSQL 源码，尤其是 `postgresql_18_1/src/backend/optimizer` 路径下的核心逻辑。
- **目录结构**: 
  - `src/` 包含所有的 markdown 源文件。
  - `src/SUMMARY.md` 为全局目录，所有的章节新增都必须在这里注册。
  - `src/phase1` 到 `src/phase3` 以及 `src/appendix` 对应查询处理和计划生成的不同阶段。

## 🛠️ 环境配置及预览

在修改内容前或修改完成后，如果需要在本地预览文档页面，应当执行以下命令（前提已安装 Rust 以及 cargo）：

```bash
# 1. 安装必要的工具（仅首次需要）
cargo install mdbook

# 2. 本地启动服务并实时预览
mdbook serve --open
```

## ✍️ 写作及修改规范

开发本电子书时，请严格遵守以下规范：

### 1. 结构与目录
- **文件位置**: 所有书籍章节必须放在 `src/` 目录或其对应逻辑阶段的对应子目录（如 `phase1`, `phase2` 等）中。
- **更新大纲**: 添加新文章后，**绝对不要忘记** 更新 `src/SUMMARY.md` 文件。如果不更新，mdBook 不会将其编译进书中，且左侧导航栏也不会展示。

### 2. 语言与行文
- **目标语言**: 全文统一使用中文（**zh-CN**）表述。
- **语言风格**: 保持通俗易懂的“口语化、笔记化”风格（正如书名一样），但对于关键的技术细节（如数据结构、算法和指针）必须严谨、准确和客观。
- **排版要求**: 中英文之间建议留出一个半角空格（例如：`在处理 TargetEntry 时`），以提供更好的阅读体验。

### 3. 代码与图表
- **代码块**: 引用 C 语言源码时，标注语言以获得高亮支持 ` ```c `。对于大型函数分析，请通过适当裁剪并使用注释说明这段代码的意图，避免直接粘贴整屏无用代码。
- **Excalidraw 绘制图表**: 在解释复杂流程、AST/Plan Tree 修改或者结构体关联时，务必使用 Excalidraw 制图。
  - **绘制与导出**：使用 Excalidraw 进行绘制。完成后，使用导出功能将其导出为 **SVG 或 PNG 图片格式**。
  - **嵌入场景（Embed Scene）**：在导出弹窗中，**务必勾选 "Embed scene" 选项**！这会将画板的 JSON 数据内嵌到图片中，使图片可以直接通过 mdBook 静态渲染，同时之后您可以随时将图片拖拽回 Excalidraw 中继续无损编辑。
  - **保存**：将导出的图片保存在本仓库合适的相对路径下（如 `src/images/` 目录下新建该目录保存）。
  - **嵌入 Markdown**：在书中通过标准 Markdown 图片语法引用导出的图片：`![图表描述](./images/your_diagram.svg)`。

### 4. 引证与版本兼容
- 明确标注 PostgreSQL 的版本，当描述的具体行为或者结构体是特定于某个版本时，请告知读者（默认语境为本库中包含的 18.1 代码）。
- 在文档中如果引用了 `postgresql_18_1` 下的相关注释（如 `TODO`），应该将其背景及当前解决状态进行梳理分析。

## 📝 常用操作示例

**新增一节关于连接顺序（Join Order）优化**：
1. 创建文件：`src/phase2/join_order.md`。
2. 写入章节标题及内容：
   ```markdown
   # 连接顺序优化
   
   在生成所有的 Join 路径之前，...
   ```
3. 修改大纲：
   打开 `src/SUMMARY.md`，在 `phase2` 部分下方添加：
   `- [连接顺序优化](./phase2/join_order.md)`
4. 启动预览检查：`mdbook serve`

遵循上述指南开展开发任务，将确保电子书生成规范统一且高质量。
