<p align="center">
  <img src="assets/Morphling.png" alt="Morphling" width="280">
</p>

<h1 align="center">Morphling</h1>

<p align="center">
  <b>把你与 AI 的每一次对话，变成会生长的本地记忆。</b><br>
  一个核心，多种形态：Mac App · DeepSeek Harness 插件 · 网页
</p>

<p align="center">
  <img alt="status" src="https://img.shields.io/badge/status-early%20design-blue">
  <img alt="storage" src="https://img.shields.io/badge/storage-local%20markdown-38bdf8">
</p>

---

## Morphling 是什么

你和 Codex、Claude Code、DeepSeek Harness 等 AI 工具的聊天里，藏着大量决定、想法和待办，但它们散落在各处，聊完就被遗忘。

Morphling 会：

- **保存**：把各个 AI 工具的原始聊天记录原样归档到本地。
- **归纳**：用大模型把对话抽象成一棵 Markdown 记忆树，包括会话摘要、主题、决定、待办、长期偏好和周期汇总。
- **回忆**：让 AI 在新对话中读取这些记忆，例如"我之前对这件事是怎么决定的？"
- **浏览**：通过 Dashboard 查看、搜索和管理你的记忆。

## 特点

- **本地优先，数据归你**：记忆只存放在你电脑上的 `~/.morphling/`，不在代码库里，也不上传。
- **纯文件，无数据库**：记忆就是 Markdown 文件，可以直接用 Obsidian 打开，也可以用 Git 或 iCloud 备份。
- **无需常驻服务**：写入由归纳 agent 完成，读取直接读文件。
- **双模型分工**：决策模型（如 Jev）负责分类、打分、选目录，大模型只负责写文字，更快也更省。
- **一次开发，多处运行**：核心与平台无关，每个平台只需一层薄壳。

## 工作方式

```
AI 聊天记录 ──► 归纳 agent ──► ~/.morphling/ ◄── AI 助手读取记忆
 (DSH / Codex /   (morph ingest)    raw/    原始记录
  Claude Code)                      memory/ Markdown 记忆树
                                         ▲
                                         │
                       Dashboard：Mac App / DSH 插件 / 网页
```

## 项目结构

```
Morphling/
├── packages/          # 核心：与平台无关
│   ├── core/          #   记忆引擎：导入、归纳、存储、检索
│   ├── cli/           #   命令行工具 morph
│   └── dashboard/     #   Dashboard 界面组件库
├── skills/            # 给 AI 的说明：如何归纳、如何回忆
├── apps/              # 壳：每个平台一层薄适配
│   ├── desktop/       #   macOS App（Electron）
│   ├── dsh-plugin/    #   DeepSeek Harness 插件
│   └── web/           #   浏览器版
└── docs/              # 设计文档
```

## 开发路线

- [ ] **阶段 1**：`morph ingest` 导入 DSH / Claude Code 会话，生成会话摘要
- [ ] **阶段 2**：主题、决定、待办归纳，AI 可通过 skill 回忆记忆
- [ ] **阶段 3**：Dashboard + macOS App
- [ ] **阶段 4**：DeepSeek Harness 插件
- [ ] **阶段 5**：网页版、`morph search`、MCP、更多数据来源

## 当前状态

项目处于早期设计阶段，尚无可运行代码。完整设计见 [docs/architecture.md](docs/architecture.md)。

技术栈：TypeScript · Node 24 · pnpm workspaces · React 19 · Vite · Electron
