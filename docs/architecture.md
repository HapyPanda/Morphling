# Morphling 架构与开发指导

> 方案 v3（2026-11-05）。后续开发以本文为准；改动架构时先更新本文。

Morphling 保存你与 AI 聊天的原始记录，归纳成一棵本地 Markdown 记忆树，供 AI 在对话中读取，并通过 Dashboard 浏览。核心只写一次，通过轻量的壳运行在 Mac App、DSH 插件和网页中。

---

## 一、总体架构

### 架构图

```
   写入                                                   读取
   ┌─────────────────────────────┐          ┌─────────────────────────────────┐
   │ 归纳 agent                   │          │ AI 助手（Codex / Claude Code / DSH）│
   │ morph ingest 或 ingest skill │          │ 按 recall skill 直接读文件 + grep  │
   └──────────────┬──────────────┘          └────────────────┬────────────────┘
                  ▼                                          ▼
   ┌────────────────────────────────────────────────────────────────────────┐
   │ 用户数据目录 ~/.morphling/（本地，不在代码库中）                          │
   │   raw/ 原始聊天（不可变）   memory/ Markdown 记忆树   .cache/ 可重建缓存   │
   └───────────────────────────────────▲────────────────────────────────────┘
                                       │ FileAccess
   ┌───────────────────────────────────┴────────────────────────────────────┐
   │ 核心 packages/ + skills/：core · cli · dashboard · AI 说明文件            │
   └───────────┬───────────────────────┬───────────────────────┬────────────┘
       ┌───────▼───────┐       ┌───────▼────────┐      ┌───────▼───────┐
       │ apps/desktop  │       │ apps/dsh-plugin│      │ apps/web      │
       │ Electron      │       │ DSH 插件        │      │ 浏览器         │
       └───────────────┘       └────────────────┘      └───────────────┘
```

### 原则

1. **不做常驻 server**：写入由归纳 agent 完成，读取直接读文件。
2. **文件即真相**：主存储是 Markdown 目录树，不用数据库；`.cache/` 可删可重建。
3. **代码与数据分离**：数据只在用户本地目录，绝不进代码库；路径统一通过 `resolveHome()` 获取。
4. **核心只写一次**：核心不得 import Electron、DSH SDK 或浏览器专属 API，平台能力通过接口由壳提供。
5. **原始数据不可变、归纳可重跑**：每条记忆记录来源、模型和 prompt 版本。
6. **暂不处理多进程同时写**：需要时再加文件锁。

### 代码目录

```
Morphling/
├── docs/architecture.md   # 本文
├── packages/              # 核心（第二部分）
│   ├── core/
│   ├── cli/
│   └── dashboard/
├── skills/                # 给 AI 的说明文件（第二部分）
│   ├── memory-ingest/
│   └── memory-recall/
└── apps/                  # 壳（第三部分）
    ├── desktop/
    ├── dsh-plugin/
    └── web/
```

技术栈：TypeScript + Node 24 + pnpm workspaces；界面用 React 19 + Vite。

### 开发路线

| 阶段 | 内容 | 完成标志 |
|---|---|---|
| 1 | `core` + `morph ingest`：导入 DSH / Claude Code 会话，存档并生成会话摘要 | 一条命令后 `memory/` 出现摘要 |
| 2 | 主题 / 决定 / 待办归纳、`_index.md` 维护、recall skill | 在 AI 对话中能答出"我之前怎么决定的" |
| 3 | `dashboard` + `apps/desktop` | Mac 上打开 App 能浏览记忆 |
| 4 | `apps/dsh-plugin` | Dashboard 出现在 DSH 面板 |
| 5 | `apps/web`、`morph search`、MCP、更多来源、周期汇总 | 按需 |

### 待决定

| 问题 | 当前倾向 |
|---|---|
| 写摘要用的大模型 | DeepSeek API 或本地 Ollama |
| 最先接入的来源 | DSH + Claude Code，随后 Codex |
| 多设备同步 | 暂只本机；以后由用户自行同步数据目录 |

---

## 二、核心部分（与壳无关）

### 各包职责

| 包 | 职责 | 主要依赖 |
|---|---|---|
| `packages/core` | 数据结构、导入器、原始存档、记忆树读写、`_index.md` 维护、归纳流水线、检索 | `zod`、`gray-matter`、Vercel `ai` SDK |
| `packages/cli` | `morph ingest`、`morph search`；用 Node `fs` 实现 `FileAccess` | `commander`、Node `zlib`、ripgrep |
| `packages/dashboard` | 全部界面（组件库，导出 `<Dashboard files host />`） | React 19、TanStack Query/Virtual、Recharts、Radix UI；样式用 CSS Modules + CSS 变量 |
| `skills/memory-ingest` | 告诉 AI 如何归纳、存到哪里、用什么格式 | Markdown |
| `skills/memory-recall` | 告诉 AI 如何检索：读 `_index.md` → 进子目录 → 读文件 / `rg` | Markdown |

`core` 内部模块：`importers/`（dsh、codex、claude-code…）、`archive/`、`store/`、`indexer/`、`summarize/`、`llm/`（大模型，负责写文字）、`decide/`（决策模型，负责做选择）、`search/`。

### 数据目录

位置优先级：`--home` 参数 > 环境变量 `MORPHLING_HOME` > 默认 `~/.morphling/`。用户可把它放进 Obsidian 库、iCloud 或私有 Git 仓库。

```
~/.morphling/
├── config.toml                    # 模型选择、API key 引用
├── raw/<source>/<hash>.*          # 原始聊天，按哈希去重，只增不改
├── memory/
│   ├── _index.md                  # 总目录摘要（AI 检索入口）
│   ├── sessions/2026/11/*.md      # 会话摘要
│   ├── topics/<topic>/            # 主题：_index.md、decisions.md、todos.md
│   ├── profile/preferences.md     # 用户长期事实与偏好
│   └── digests/2026-W45.md        # 周期汇总
└── .cache/manifest.json           # Dashboard 统计，可重建
```

每个目录都有 `_index.md`，供 AI 当路标。记忆文件格式：

```markdown
---
id: 2026-11-04-morphling-architecture
source: claude-code
raw: raw/claude-code/ab12cd.jsonl
date: 2026-11-04
topics: [morphling]
model: deepseek-chat
prompt_version: session-summary@v1
---
## 摘要
## 决定
## 待办
```

### 写入与读取

**写入**：发现新会话 → 复制到 `raw/` → 解析 → 归纳 → 写入 `memory/` → 更新 `_index.md` 和 manifest。运行方式：批量用 `morph ingest`，单次用对话中的 ingest skill。

**读取**：先让 AI 按 recall skill 直接读文件；以后按需增加 `morph search` 脚本或 MCP 工具 `get_memory()`。

**双模型分工**（参考 Jev-Mem，arXiv 2609.23986）：决策模型 Jev 只返回选项、分数、是/否，不写文字。

| 环节 | 决策模型（Jev） | 大模型 |
|---|---|---|
| 写入 | 值不值得记、属于哪个主题、重要程度、关联哪些记忆 | 写摘要、决定、待办和 `_index.md` |
| 读取 | 选择下一步进入的子目录、给搜索结果打分、判断是否找够 | 用找到的记忆组织回答 |

`core/decide/` 只定义 `choose / score / judge`；各家决策模型共用 Decisions API，换模型只改配置。

### 给壳的接口

```ts
// 文件读写：core 和 dashboard 只依赖它
interface FileAccess {
  readText(path: string): Promise<string>
  writeText(path: string, content: string): Promise<void>
  list(dir: string): Promise<{ name: string; isDir: boolean }[]>
  exists(path: string): Promise<boolean>
}

// 平台能力
interface HostBridge {
  pickFiles(): Promise<File[]>
  openExternal(url: string): void
  notify(text: string): void
  theme(): 'light' | 'dark'
  locale(): string
}
```

---

## 三、apps 壳部分

每个壳只做三件事：实现 `FileAccess`、实现 `HostBridge`、挂载 `<Dashboard />`。

| 壳 | `FileAccess` 实现 | 额外职责 | 技术 |
|---|---|---|---|
| `apps/desktop` | 主进程用 Node `fs`，界面经 preload 的 IPC 调用 | 窗口、托盘、原生选文件 | Electron（`electron-vite`、`electron-builder`） |
| `apps/dsh-plugin` | 插件 Host 端用 Node `fs`，Client 端经 DSH 通信调用 | 挂进 DSH 面板，跟随 DSH 主题与语言 | DSH 插件 SDK（Cordis） |
| `apps/web` | 浏览器 File System Access API，用户选择一次数据文件夹 | 无 | Vite；仅支持 Chrome / Edge |

**DSH 插件注意事项**：开发前先读 DSH 插件文档，确认 Host 端如何读文件、Client 端如何挂载面板、是否必须用原生 React（非 iframe）。Dashboard 样式不能写全局样式，颜色走 CSS 变量，以便跟随 DSH 主题。

**其他 AI 工具**（不需要壳代码）：

| 工具 | 写入来源 | 读取方式 |
|---|---|---|
| Codex | `~/.codex/sessions/` | `AGENTS.md` 引导读文件；可选 MCP |
| Claude Code | `~/.claude/projects/**/*.jsonl` | skill 引导读文件；可选 MCP |
| DSH | `~/.dsh/sessions/**/session.v4.jsonl.zstd` | 读文件；可选 MCP |
