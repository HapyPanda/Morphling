# Morphling 架构与开发指导

> 状态：方案 v2（2026-11-05）：取消常驻 server，简化流程。后续开发以本文为准；改动架构时先更新本文。

## 1. 目标

- 保存与 AI 聊天的**原始文件**（DSH、Codex、Claude Code、ChatGPT 等），原样不改。
- 由一个**归纳 agent** 把聊天抽象成一棵本地 **Markdown 记忆树**（会话摘要 → 主题 → 长期事实 → 周期汇总）。
- AI 在对话中能**读取记忆**（先靠说明文件直接读文件，以后再按需加脚本或 MCP）。
- 提供一个 **Dashboard**，可以用 **macOS App**、**DSH 插件**、**网页**三种方式打开。

## 2. 核心原则

1. **不做常驻 server**：没有后台服务。写入由归纳 agent 完成，读取直接读文件。
2. **文件即真相**：主存储是本地 Markdown 目录树，不使用数据库；`.cache/` 里的内容都可删除、可重建。
3. **主体只写一次**：核心代码在 `packages/`，与任何壳无关；壳（`apps/`）只做薄适配。
4. **核心层禁止平台专属代码**：不得 import Electron、DSH SDK 或浏览器专属 API；需要平台能力时，通过 `FileAccess`、`HostBridge` 接口由壳提供。
5. **原始数据不可变、归纳可重跑**：每条记忆记录来源文件、模型和 prompt 版本。
6. **暂不处理多进程同时写**：当前假设同一时间只有一个写入者，以后需要时再在 `core/store` 加文件锁。

## 3. 架构图

```
                 写入                                              读取
   ┌──────────────────────────────┐              ┌──────────────────────────────────┐
   │ 归纳 agent                    │              │ AI 助手（Codex / Claude Code / DSH）│
   │  · CLI：memory ingest         │              │  A. 按 skills/ 说明直接读文件+grep │
   │  · 或 skill：由当前对话 AI 执行 │              │  B. memory search 脚本（可选）     │
   │ 读原始聊天 → 大模型归纳        │              │  C. MCP 工具 get_memory（可选）    │
   │ → 写 Markdown → 更新 _index   │              └────────────────┬─────────────────┘
   └──────────────┬───────────────┘                               │
                  ▼                                               ▼
   ┌───────────────────────────────────────────────────────────────────────────────┐
   │ 数据目录（不进 Git）~/.memory-system/                                            │
   │   raw/  原始聊天文件（不可变）   memory/  Markdown 记忆树   .cache/  可重建缓存   │
   └───────────────────────────────────────▲───────────────────────────────────────┘
                                           │ 通过 FileAccess 读写
   ┌───────────────────────────────────────┴───────────────────────────────────────┐
   │ Dashboard（packages/dashboard，React 组件库，只写一次）                          │
   └──────────┬──────────────────────────────┬──────────────────────────────┬──────┘
   ┌──────────▼─────────┐        ┌───────────▼──────────┐       ┌───────────▼────────┐
   │ apps/desktop        │        │ apps/dsh-plugin       │       │ apps/web            │
   │ Electron：主进程用   │        │ 插件 Host 端（Node）   │       │ 浏览器 File System   │
   │ Node 读文件，IPC 传给 │        │ 读文件，Client 端挂载  │       │ Access API：用户选择  │
   │ 界面                 │        │ 面板                  │       │ 记忆文件夹后直接读写  │
   └─────────────────────┘        └──────────────────────┘       └────────────────────┘
                            （三个壳都只是实现 FileAccess + HostBridge）
```

## 4. 分层总表

| 类别 | 层 / 包 | 职责 | 技术栈 | 推荐库 |
|---|---|---|---|---|
| 数据 | `~/.memory-system/` | 原始文件 + Markdown 记忆树 + 可重建缓存 | 文件系统 | — |
| **核心** | `packages/core` | 数据结构、导入器、原始存档、记忆树读写、`_index.md` 维护、归纳流水线、关键词检索；不直接调用 `fs`，通过 `FileAccess` 接口读写 | TypeScript | `zod`、`gray-matter`（frontmatter）、Vercel `ai` SDK（大模型统一接口） |
| **核心** | `packages/cli` | `memory ingest`（导入+归纳）、`memory search`（检索）；用 Node `fs` 实现 `FileAccess` | Node 24 | `commander`、Node `zlib`（zstd 解压）、ripgrep |
| **核心** | `packages/dashboard` | 全部界面页面与组件（组件库形式，可被挂载） | React 19 + Vite 库模式 | `@tanstack/react-query`、`@tanstack/react-virtual`、`recharts`、Radix UI；样式用 CSS Modules + CSS 变量 |
| **核心** | `skills/` | 给 AI 的说明文件：归纳规则、存储格式、检索方法 | Markdown | — |
| 壳 | `apps/desktop` | Mac App：主进程用 Node 读写文件，通过 IPC 提供给界面；窗口、托盘 | Electron | `electron-vite`、`electron-builder` |
| 壳 | `apps/dsh-plugin` | Host 端读写文件，Client 端把 Dashboard 挂进 DSH 面板，适配主题与语言 | DSH 插件 SDK（Cordis） | 以 DSH 插件文档为准 |
| 壳 | `apps/web` | 浏览器版：用 File System Access API 让用户选择记忆文件夹（Chrome / Edge 支持，Safari 不支持） | Vite | — |
| 壳 | `apps/codex`（可选） | 给 Codex 的 `AGENTS.md` 片段 / MCP 配置示例，几乎无代码 | — | — |
| 工程 | 根目录 | monorepo、构建、测试 | pnpm workspaces | `tsup`、`vitest` |

## 5. 两个关键接口

```ts
// 文件读写：core 和 dashboard 只依赖它，每个壳实现一次
interface FileAccess {
  readText(path: string): Promise<string>
  writeText(path: string, content: string): Promise<void>
  list(dir: string): Promise<{ name: string; isDir: boolean }[]>
  exists(path: string): Promise<boolean>
}

// 平台能力：每个壳实现一次
interface HostBridge {
  pickFiles(): Promise<File[]>
  openExternal(url: string): void
  notify(text: string): void
  theme(): 'light' | 'dark'
  locale(): string
}
```

| 实现者 | `FileAccess` 的实现方式 |
|---|---|
| `packages/cli` | Node `fs` |
| `apps/desktop` | 主进程用 Node `fs`，界面通过 preload 暴露的 IPC 调用 |
| `apps/dsh-plugin` | 插件 Host 端用 Node `fs`，Client 端通过 DSH 提供的通信方式调用 |
| `apps/web` | 浏览器 File System Access API（`showDirectoryPicker`） |

## 6. 数据目录与记忆格式

```
~/.memory-system/
├── raw/                               # 原始聊天文件，按内容哈希命名，只增不改
│   ├── dsh/2026/11/<hash>.jsonl.zstd
│   ├── codex/<hash>.jsonl
│   └── claude-code/<hash>.jsonl
├── memory/                            # Markdown 记忆树（主存储，可用 Obsidian 打开）
│   ├── _index.md                      # 总目录摘要（AI 读取记忆的入口）
│   ├── sessions/2026/11/
│   │   ├── _index.md
│   │   └── 2026-11-04-memory-system-architecture.md
│   ├── topics/<topic>/
│   │   ├── _index.md
│   │   ├── decisions.md
│   │   └── todos.md
│   ├── profile/preferences.md         # 关于用户本人的长期事实与偏好
│   └── digests/2026-W45.md            # 周报 / 月报
└── .cache/
    └── manifest.json                  # Dashboard 统计用，可重建
```

每个目录都有 `_index.md`，由归纳 agent 在写入后更新，作用是给 AI 当路标。记忆文件格式如下：

```markdown
---
id: 2026-11-04-memory-system-architecture
source: claude-code
raw: raw/claude-code/ab12cd.jsonl
date: 2026-11-04
topics: [memory-system]
model: deepseek-chat
prompt_version: session-summary@v1
---
## 摘要
## 决定
## 待办
```

## 7. 写入与读取流程

**写入（归纳 agent）**

1. 扫描来源目录，发现新的会话文件。
2. 原样复制到 `raw/`（按哈希去重）。
3. 解析成统一结构，调用大模型生成会话摘要、主题、决定、待办。
4. 写入 `memory/` 下对应的 Markdown 文件。
5. 更新相关目录的 `_index.md` 和 `.cache/manifest.json`。

运行方式：`memory ingest` 命令（可手动执行，也可定时执行），或在 AI 对话中按 `skills/memory-ingest/SKILL.md` 执行。

**读取（get_memory）**

| 方式 | 实现 | 阶段 |
|---|---|---|
| A. 直接读文件 | AI 按 `skills/memory-recall/SKILL.md`：读 `memory/_index.md` → 进子目录 → 读文件；需要时用 `rg` 搜关键词 | 先做 |
| B. 脚本 | `memory search "<关键词>"`，返回匹配文件与片段 | 按需 |
| C. MCP | `get_memory()` 工具，由 AI 工具按需拉起、用完退出 | 按需 |

### 双系统分工：决策模型 + 大模型

参考 Jev-Mem（arXiv 2609.23986）的思路：快速、便宜的**决策模型**（Jev）负责"做选择"，**大模型**只负责"写文字"。Jev 只返回结构化结果（选项、分数、是/否），不生成文字，所以不能单独完成摘要。

| 环节 | 决策模型（Jev）负责 | 大模型负责 |
|---|---|---|
| 写入 | 这段会话值不值得记（是/否）、属于哪个主题（选项）、重要程度（分数）、和哪些已有记忆相关 | 写会话摘要、决定、待办，更新 `_index.md` 文字 |
| 读取 | 根据问题在 `_index.md` 的子目录里选下一步（选项）、给 grep 结果打相关分、判断是否已找够（是/否） | 用找到的记忆组织最终回答 |

`core/decide/` 只定义 `choose / score / judge` 三个方法，默认实现调用 Jev；Decisions API 在多家模型间兼容，换模型只改配置。

## 8. 代码目录树

```
Morphling/
├── README.md
├── AGENTS.md                     # AI 协作规则（含第 2 节原则）
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── .gitignore                    # node_modules、dist、数据目录、凭据
├── docs/
│   └── architecture.md           # 本文
├── packages/                     # ═══ 核心 ═══
│   ├── core/src/
│   │   ├── schema.ts             # 数据结构
│   │   ├── file-access.ts        # FileAccess / HostBridge 接口
│   │   ├── importers/            # dsh.ts / codex.ts / claude-code.ts / chatgpt.ts
│   │   ├── archive/              # 原始文件哈希存档
│   │   ├── store/                # 记忆树读写
│   │   ├── indexer/              # _index.md + manifest.json
│   │   ├── summarize/            # 归纳流水线 + prompts/
│   │   ├── llm/                  # 大模型统一接口（写摘要等需要生成文字的任务）
│   │   ├── decide/               # 决策模型接口（Jev：分类、打分、选目录、判断停止）
│   │   └── search/               # 关键词检索 + 决策模型导航
│   ├── cli/src/
│   │   ├── ingest.ts
│   │   └── search.ts
│   └── dashboard/src/
│       ├── pages/                # 概览 / 会话库 / 搜索 / 主题 / 设置
│       ├── components/
│       ├── theme/
│       └── index.tsx             # 导出 <Dashboard files={...} host={...} />
├── skills/                       # ═══ 给 AI 的说明 ═══
│   ├── memory-ingest/SKILL.md    # 怎么归纳、存哪里、什么格式
│   └── memory-recall/SKILL.md    # 怎么检索记忆
└── apps/                         # ═══ 壳 ═══
    ├── desktop/                  # Electron
    ├── dsh-plugin/               # DSH 插件
    ├── web/                      # 浏览器版
    └── codex/                    # AGENTS.md 片段 / MCP 配置示例（可选）
```

## 9. 与各 AI 工具的关系

| 工具 | 写入 | 读取 | Dashboard |
|---|---|---|---|
| DSH | 导入 `~/.dsh/sessions/**/session.v4.jsonl.zstd`；或在 DSH 中执行 ingest skill | 读文件 / 以后加 MCP | DSH 插件面板 |
| Codex | 导入 `~/.codex/sessions/` 下的 jsonl | `AGENTS.md` 引导直接读文件；以后可配 MCP | 无（用 Mac App 或网页） |
| Claude Code | 导入 `~/.claude/projects/**/*.jsonl` | skill 引导直接读文件 | 无（用 Mac App 或网页） |

开发 DSH 插件前，先读 DSH 插件 SDK 文档：Host 端怎么读文件、Client 端怎么挂载面板、是否必须用原生 React。

## 10. 开发路线（MVP 顺序）

| 阶段 | 内容 | 完成标志 |
|---|---|---|
| 1 | monorepo 骨架 + `core`（DSH / Claude Code 导入、原始存档、记忆树写入）+ `memory ingest` | 运行一条命令，`memory/` 里出现会话摘要 |
| 2 | 归纳完善（主题、决定、待办）+ `_index.md` 维护 + `skills/memory-recall` | 在 Codex / Claude Code 里问"我之前怎么决定的"能答上来 |
| 3 | `dashboard` + `apps/desktop`（Electron） | Mac 上打开 App 能浏览记忆 |
| 4 | `apps/dsh-plugin` | Dashboard 出现在 DSH 面板中 |
| 5 | `apps/web`、`memory search`、MCP、更多来源、周期汇总 | 按需 |

## 11. 待决定事项

| 问题 | 当前倾向 |
|---|---|
| 最先接入哪些来源 | DSH 会话 + Claude Code（Codex 紧随其后） |
| 归纳用哪个模型 | 待定（DeepSeek API / 本地 Ollama；敏感内容可走本地模型） |
| 归纳由 CLI 还是由对话 AI 执行 | 两者都支持；批量用 CLI，单次用 skill |
| 是否多设备同步 | 先只在本机；以后可用 Git / 云盘同步 `memory/` |
| Mac 壳 | Electron（本机已有 Node，无 Rust） |
| 决策模型 | Jev（TypeSafe AI），通过 `core/decide/` 接入；Decisions API 各家兼容，可换模型 |
| 项目目录 | `~/Desktop/zqy/Github/Morphling` |
