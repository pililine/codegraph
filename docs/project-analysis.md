# CodeGraph 项目深度分析

> 生成时间：2026-05-27  
> 版本：0.9.4  
> 分析范围：全量源码 + 测试 + 文档 + 配置

---

## 1. 项目用途和技术栈

### 1.1 项目用途

**CodeGraph** 是一个**本地优先的代码语义知识图谱**工具，核心价值在于：

- 使用 tree-sitter 将任意代码库解析为**符号节点 + 关系边**组成的有向图
- 将图数据存储在 SQLite（FTS5 全文检索）中，索引目录为 `.codegraph/`
- 通过 **MCP（Model Context Protocol）服务器**将代码智能暴露给 AI Agent
- 支持的 Agent：Claude Code、Cursor、Codex CLI、opencode、Hermes Agent

**典型效果（跨 7 个真实代码库的 A/B 测试中位数）：**

| 指标 | 节省幅度 |
|------|---------|
| API 费用 | -35% |
| Token 用量 | -57% |
| 响应时间 | -46% |
| 工具调用次数 | -71% |

具体案例：Excalidraw（TS/React，643 文件），用 codegraph 回答"element 更新如何触发 canvas 重渲染"，从 115–139s + 9–11 次 Read/Grep 降至 51–74s + 0–2 次 Read。

### 1.2 技术栈

| 层次 | 技术 |
|------|------|
| 语言 | TypeScript（strict 模式，ES2022 target，CommonJS 输出） |
| 解析器 | [web-tree-sitter](https://github.com/tree-sitter/tree-sitter) + WASM grammars |
| 数据库 | SQLite via `node:sqlite`（Node.js 内置，v0.9.4 起；原 `better-sqlite3` + WASM 降级路径已在 commit `ac52fd7` 中移除） |
| 全文检索 | SQLite FTS5 virtual table |
| CLI 框架 | [commander](https://github.com/tj/commander.js) |
| 测试框架 | [vitest](https://vitest.dev/) |
| 交互式提示 | [@clack/prompts](https://github.com/bombshell-dev/clack) |
| 文件监听 | 原生 FSEvents（macOS）/ inotify（Linux）/ RDCW（Windows） |
| 协议 | MCP（Model Context Protocol）—— stdio transport |
| 包管理 | npm，发布为 `@colbymchenry/codegraph` |
| 运行环境 | Node.js >=20.0.0 <25.0.0（硬卡 25.x，原因：V8 turboshaft WASM JIT bug） |

---

## 2. 目录结构说明

```
codegraph/
├── src/                          # 全量 TypeScript 源码（100 个文件，~21,500 行）
│   ├── index.ts                  # 公共 API 入口：CodeGraph 类
│   ├── types.ts                  # 核心类型（NodeKind / EdgeKind / 接口定义）
│   ├── errors.ts                 # 结构化错误类 + Logger 系统
│   ├── utils.ts                  # Mutex / FileLock / 批处理 / 防抖
│   ├── directory.ts              # .codegraph/ 目录生命周期管理
│   │
│   ├── bin/                      # CLI 可执行文件
│   │   ├── codegraph.ts          # 主 CLI（commander），含所有子命令
│   │   ├── node-version-check.ts # Node.js 版本检查
│   │   └── uninstall.ts          # 卸载辅助
│   │
│   ├── db/                       # 数据库层
│   │   ├── index.ts              # DatabaseConnection（连接管理 + WAL + 事务）
│   │   ├── schema.sql            # SQLite schema（152 行，含 FTS5 + triggers）
│   │   ├── queries.ts            # QueryBuilder（预编译语句封装）
│   │   ├── migrations.ts         # Schema 版本迁移
│   │   └── sqlite-adapter.ts     # 原生 / WASM 后端选择适配器
│   │
│   ├── extraction/               # 代码解析与符号提取
│   │   ├── index.ts              # ExtractionOrchestrator（协调并行解析）
│   │   ├── parse-worker.ts       # Worker 线程，解析任务移出主线程
│   │   ├── tree-sitter.ts        # tree-sitter 封装
│   │   ├── grammars.ts           # Grammar 加载 + 缓存
│   │   ├── svelte-extractor.ts   # Svelte SFC 独立解析器
│   │   ├── vue-extractor.ts      # Vue SFC 独立解析器
│   │   ├── liquid-extractor.ts   # Liquid 模板解析器
│   │   ├── dfm-extractor.ts      # Delphi DFM 格式解析器
│   │   ├── wasm/                 # 额外 WASM grammar 文件
│   │   │   ├── tree-sitter-lua.wasm
│   │   │   ├── tree-sitter-luau.wasm
│   │   │   ├── tree-sitter-pascal.wasm
│   │   │   └── tree-sitter-scala.wasm
│   │   └── languages/            # 各语言提取器（19 个文件）
│   │       ├── typescript.ts     # TS / JS / JSX / TSX
│   │       ├── python.ts
│   │       ├── go.ts
│   │       ├── rust.ts
│   │       ├── java.ts
│   │       ├── c-cpp.ts
│   │       ├── csharp.ts
│   │       ├── php.ts
│   │       ├── ruby.ts
│   │       ├── swift.ts
│   │       ├── kotlin.ts
│   │       ├── dart.ts
│   │       ├── scala.ts
│   │       ├── lua.ts / luau.ts
│   │       └── pascal.ts
│   │
│   ├── resolution/               # 引用解析 + 动态调度合成
│   │   ├── index.ts              # ReferenceResolver 协调器
│   │   ├── import-resolver.ts    # 导入路径解析
│   │   ├── path-aliases.ts       # tsconfig paths + cargo workspace glob
│   │   ├── name-matcher.ts       # 符号名匹配（限定名 / 简短名）
│   │   ├── callback-synthesizer.ts # 回调 / observer / EventEmitter 边合成
│   │   ├── lru-cache.ts          # 解析缓存（有界 LRU）
│   │   └── frameworks/           # 框架路由处理器（20 个文件）
│   │       ├── express.ts / nestjs.ts / react.ts / vue.ts / svelte.ts
│   │       ├── django.ts / python.ts（Flask/FastAPI）
│   │       ├── laravel.ts / drupal.ts / ruby.ts（Rails）
│   │       ├── go.ts（Gin/chi）/ rust.ts（Axum/actix）
│   │       ├── java.ts（Spring）/ csharp.ts（ASP.NET）
│   │       ├── swift.ts（Vapor）/ play.ts（Play Framework）
│   │       └── cargo-workspace.ts
│   │
│   ├── graph/                    # 图查询与遍历
│   │   ├── traversal.ts          # GraphTraverser（BFS/DFS、影响半径）
│   │   └── queries.ts            # GraphQueryManager（高阶查询接口）
│   │
│   ├── context/                  # AI Agent 上下文构建
│   │   ├── index.ts              # ContextBuilder（symbol → context block）
│   │   └── formatter.ts          # Markdown / JSON 格式化输出
│   │
│   ├── search/                   # 全文检索
│   │   ├── query-parser.ts       # 搜索语法解析（FTS5 query DSL）
│   │   └── query-utils.ts        # 评分 / 词干处理 / 工具方法
│   │
│   ├── sync/                     # 文件变更监听 + git hook
│   │   ├── index.ts              # FileWatcher（带防抖过滤器）
│   │   ├── watcher.ts            # 原生 FS 监听（FSEvents/inotify/RDCW）
│   │   ├── watch-policy.ts       # 包含 / 排除规则
│   │   └── git-hooks.ts          # git pre-commit / post-commit hook
│   │
│   ├── mcp/                      # MCP 服务器
│   │   ├── index.ts              # MCPServer 类
│   │   ├── tools.ts              # 15+ 工具定义实现（~15,000 行）
│   │   ├── server-instructions.ts # MCP initialize 响应中的使用指引
│   │   └── transport.ts          # stdio transport 层
│   │
│   ├── installer/                # 多 Agent 安装器
│   │   ├── index.ts              # 安装器协调器
│   │   ├── instructions-template.ts # 通用 Agent 说明文件模板
│   │   ├── claude-md-template.ts # Claude Code 专用模板（向后兼容）
│   │   └── targets/              # Agent 安装目标（9 个文件）
│   │       ├── registry.ts       # 目标注册表
│   │       ├── types.ts          # AgentTarget 接口定义
│   │       ├── claude.ts / cursor.ts / codex.ts / opencode.ts / hermes.ts
│   │       ├── shared.ts         # 共享工具方法
│   │       └── toml.ts           # 自研 TOML 序列化器（无外部依赖）
│   │
│   └── ui/                       # 终端 UI
│       ├── shimmer-progress.ts   # 动态进度条（shimmer 动画）
│       ├── shimmer-worker.ts     # 进度渲染 Worker 线程
│       └── glyphs.ts             # Unicode 符号
│
├── __tests__/                    # 测试套件（42 个文件，~13,500 行）
│   ├── extraction.test.ts        # 语言提取测试（127K，最大单测文件）
│   ├── frameworks.test.ts        # 框架路由测试（46K）
│   ├── installer-targets.test.ts # 安装器合约测试（36K，47 参数化用例）
│   ├── security.test.ts          # 安全路径验证（22K）
│   ├── pr19-improvements.test.ts # 历史回归防护
│   └── evaluation/               # 检索质量评估（不纳入 npm test）
│
├── scripts/                      # 构建 + 发布脚本
│   ├── build-bundle.sh           # 多平台 Node 运行时打包
│   ├── pack-npm.sh               # npm 瘦安装包准备
│   ├── extract-release-notes.mjs # changelog 提取脚本
│   └── agent-eval/               # Agent 评估工具
│
├── docs/                         # 设计文档 + 基准测试报告
│   ├── design/                   # 架构设计（dynamic-dispatch-playbook 等）
│   ├── benchmarks/               # A/B 测试记录
│   └── plans/                    # 重构计划文档
│
├── .github/                      # GitHub Actions（release workflow）
├── .claude/                      # Claude Code 本地配置
├── .cursor/rules/                # Cursor IDE 规则（codegraph.mdc）
├── package.json                  # npm 包元数据
├── tsconfig.json                 # TypeScript 配置（strict 全开）
├── vitest.config.ts              # 测试配置
├── CLAUDE.md                     # 项目对 Claude Code 的核心指导
├── CHANGELOG.md                  # 发版记录（source of truth）
├── install.sh / install.ps1      # 零依赖安装脚本
└── README.md
```

---

## 3. 启动方式

### 3.1 开发模式

```bash
# 安装依赖
npm install

# 一次性构建（含 schema.sql 和 *.wasm 资产复制）
npm run build

# 监听模式构建（开发时用）
npm run dev

# 运行本地 CLI（自动 build 后执行）
npm run cli

# 清理构建产物
npm run clean
```

### 3.2 CLI 子命令

```bash
# 在当前项目初始化 codegraph
codegraph init

# 安装到 AI Agent（交互式选择 Claude Code / Cursor / Codex / opencode）
codegraph install

# 全量索引
codegraph index

# 增量同步（仅处理变化文件）
codegraph sync

# 查看索引状态
codegraph status

# 搜索符号
codegraph query <keyword>

# 列出文件
codegraph files

# 构建任务上下文
codegraph context <task-description>

# 查找受影响文件
codegraph affected <file-path>

# 启动 MCP 服务器
codegraph serve --mcp

# 查看调用者 / 被调用者 / 影响半径
codegraph callers <symbol>
codegraph callees <symbol>
codegraph impact <symbol>
```

### 3.3 作为库使用

```typescript
import { CodeGraph } from '@colbymchenry/codegraph';

const cg = new CodeGraph('/path/to/project');
await cg.init();
await cg.indexAll();

const results = await cg.searchNodes('MyComponent');
const context = await cg.buildContext({ task: 'fix the login bug' });

cg.close();
```

### 3.4 零安装运行（通过发布包）

```bash
# Unix/macOS（下载预编译 bundle，不需要 Node）
curl -fsSL https://raw.githubusercontent.com/.../install.sh | sh

# Windows
irm https://raw.githubusercontent.com/.../install.ps1 | iex

# npm 全局安装
npm i -g @colbymchenry/codegraph
```

### 3.5 测试

```bash
npm test                  # vitest run（全量，不含 evaluation/）
npm run test:watch        # 监听模式
npm run test:eval         # 仅运行 __tests__/evaluation/
npm run eval              # build 后运行 evaluation runner（实测评估）

# 单文件 / 单 pattern
npx vitest run __tests__/extraction.test.ts
npx vitest run __tests__/extraction.test.ts -t "TypeScript"
```

---

## 4. 核心模块说明

### 4.1 数据流全景

```
源码文件
  ↓  (文件系统遍历 + .gitignore 过滤)
ExtractionOrchestrator
  ↓  (tree-sitter 解析 → AST)
  ↓  (语言提取器 → Node / Edge 列表)
DatabaseConnection  (SQLite，WAL 模式)
  ↓  (写入 nodes / edges / files 表)
  ↓  (FTS5 触发器同步全文索引)
ReferenceResolver
  ↓  (import-resolver：解析导入路径)
  ↓  (name-matcher：跨文件符号链接)
  ↓  (框架处理器：路由节点 + synthesized edges)
  ↓  (callback-synthesizer：动态调度边合成)
GraphTraverser / GraphQueryManager
  ↓  (BFS/DFS 遍历、影响半径计算)
ContextBuilder
  ↓  (Markdown / JSON 格式化)
MCP Server / CLI 输出
```

### 4.2 核心模块详解

#### `src/index.ts` — CodeGraph 主类

对外暴露的唯一公共 API 入口。聚合所有子系统：

| 方法 | 说明 |
|------|------|
| `init()` | 创建 .codegraph/ 目录，初始化 DB |
| `open()` | 打开已有索引（不重新解析） |
| `close()` | 释放 DB 连接和 Worker |
| `indexAll(opts?)` | 全量重索引（并行 Worker 解析） |
| `sync(files?)` | 增量同步（仅处理变更文件） |
| `searchNodes(query, opts?)` | FTS5 全文搜索符号 |
| `getCallers(nodeId)` | 查找所有调用者 |
| `getCallees(nodeId)` | 查找所有被调用者 |
| `getImpactRadius(nodeId)` | 计算影响半径（BFS） |
| `buildContext(task)` | 为任务构建 AI 上下文 |
| `watch(opts)` / `unwatch()` | 启动 / 停止文件监听 |

#### `src/db/schema.sql` — 数据模型

5 张核心表：

```sql
nodes           -- 符号节点（kind, name, qualified_name, file_path, metadata JSON）
edges           -- 关系边（source_id, target_id, kind, provenance, metadata JSON）
files           -- 已索引文件（path, hash, language, indexed_at, node_count）
unresolved_refs -- 待解析引用（from_node_id, reference_name, candidates JSON）
project_metadata -- 键值对元数据（项目根目录、索引统计等）
```

FTS5 虚拟表 `nodes_fts`（字段：name, qualified_name, docstring, signature）由 trigger 自动维护。

#### `src/extraction/` — 语言提取器

**支持 25+ 语言**，分两类：
1. **tree-sitter 提取器**（`languages/`）：TypeScript/JS、Python、Go、Rust、Java、C/C++、C#、PHP、Ruby、Swift、Kotlin、Dart、Scala、Lua、Luau、Pascal
2. **独立解析器**（非 tree-sitter）：Svelte SFC、Vue SFC、Liquid 模板、Delphi DFM

所有提取器产出统一的 `Node[]` + `Edge[]`，通过 `ExtractionOrchestrator` 批量写入 SQLite。重解析在 `parse-worker.ts`（Worker 线程）中运行，不阻塞主线程。

#### `src/resolution/` — 引用解析 + 边合成

解析 `unresolved_refs` 表中的悬挂引用，生成 `calls / imports / references` 边：

- **import-resolver**：解析模块导入路径（支持 tsconfig paths alias、cargo workspace glob）
- **name-matcher**：跨文件符号名匹配（限定名 `Class.method` / 简短名 `method`）
- **callback-synthesizer**：合成动态调度边：
  - 回调 / observer 模式
  - EventEmitter `.on` / `.emit`
  - React `setState → render`（react-render edge）
  - JSX child `render → <ChildComponent>`（jsx-render edge）
  - Flutter `setState`
  - C++ 虚函数覆盖
  - Java/Kotlin 接口派发
  - Django ORM descriptor
- **框架处理器（20 个）**：识别路由注解 / 装饰器，生成 `route` 节点和完整请求链路边

所有合成边标注 `provenance: 'heuristic'` + `metadata.synthesizedBy`，可溯源区分。

#### `src/mcp/tools.ts` — MCP 工具层

暴露给 AI Agent 的 15+ 个工具，关键工具：

| 工具 | 说明 |
|------|------|
| `codegraph_search` | FTS5 全文符号搜索 |
| `codegraph_node` | 获取符号详情 + 调用者 / 被调用者 |
| `codegraph_trace` | 调用链追踪（inline 源码 + 每跳调用站点） |
| `codegraph_explore` | 区域探索（自动流探测，基于命名符号集合） |
| `codegraph_context` | 为任务构建 AI 上下文块 |
| `codegraph_files` | 项目文件树 |
| `codegraph_affected` | 给定文件的影响范围（测试推荐） |
| `codegraph_status` | 索引状态、DB 后端、节点数统计 |

**自适应输出预算**（避免 Agent 上下文膨胀）：

| 项目规模 | explore 调用次数上限 | 单次字符上限 |
|---------|---------------------|------------|
| <500 文件 | 1 | ~18K |
| <5,000 文件 | 2 | ~28K |
| <15,000 文件 | 3 | ~35K |
| <25,000 文件 | 4 | ~38K |
| ≥25,000 文件 | 5 | ~38K |

#### `src/installer/` — 多 Agent 安装器

每个 Agent target 实现 `AgentTarget` 接口，负责：
- 配置文件写入（JSON / JSONC / TOML）
- MCP server 注册
- 使用说明文件（CLAUDE.md / .cursor/rules/codegraph.mdc 等）写入
- 幂等安装 / 卸载（47 个参数化合约测试覆盖）

添加新 Agent 只需：1 个新 target 文件 + 1 行 registry.ts 注册。

---

## 5. 依赖和环境变量

### 5.1 生产依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `web-tree-sitter` | ^0.25.3 | 核心 AST 解析库（WASM） |
| `tree-sitter-wasms` | ^0.1.11 | 主要语言 WASM grammar 集合 |
| `commander` | ^14.0.2 | CLI 命令框架 |
| `@clack/prompts` | ^1.3.0 | 交互式终端提示 |
| `jsonc-parser` | ^3.3.1 | JSONC 文件手术级编辑（保留注释） |
| `ignore` | ^7.0.5 | .gitignore 规则解析 |
| `picomatch` | ^4.0.3 | Glob 模式匹配 |
| `fast-string-width` | ^3.0.2 | 终端宽度计算（含全角字符） |
| `fast-wrap-ansi` | ^0.2.0 | ANSI 转义序列感知换行 |
| `sisteransi` | ^1.0.5 | ANSI 检测与处理 |

**内置 SQLite（v0.9.4 起）：**  
使用 Node.js `node:sqlite` 内置模块，无需外部绑定。原 `better-sqlite3`（原生 Node 绑定）和 `node-sqlite3-wasm`（WASM 降级）均已在 commit `ac52fd7` 中移除。

### 5.2 开发依赖

| 包名 | 用途 |
|------|------|
| `typescript` ^5.0.0 | 编译器 |
| `vitest` ^2.1.9 | 测试框架 |
| `@types/node` ^20.19.30 | Node.js 类型 |
| `@types/better-sqlite3` ^7.6.0 | SQLite 绑定类型 |
| `@types/picomatch` ^4.0.2 | Glob 类型 |

### 5.3 环境变量

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `CODEGRAPH_ALLOW_UNSAFE_NODE` | boolean | 跳过 Node.js 版本检查（≥25.x 禁止） |
| `CODEGRAPH_MCP_TOOLS` | string | 逗号分隔的工具白名单（如 `trace,search,node`） |
| `CODEGRAPH_PPID_POLL_MS` | number | PPID watchdog 轮询间隔（默认 5000ms） |
| `CODEGRAPH_DEBUG` | boolean | 开启详细调试日志 |

**无 `.env` 文件**，所有环境变量通过运行时 shell 注入。

### 5.4 Node.js 版本要求

- **最低：** Node.js 20.0.0
- **上限：** <25.0.0（Node 25.x 有 V8 turboshaft WASM JIT bug，强制退出）
- **推荐：** Node 20 LTS 或 22 LTS

---

## 6. 当前潜在问题（含证据评级）

> **评级说明**
> - `evidence: high` — 有具体代码行、commit 或文档直接证明
> - `evidence: medium` — 观察到现象，但后果未在代码中完整验证
> - `evidence: low` — 推断性判断，缺乏直接代码证据，需进一步读代码确认
>
> **判断标准**：行数多不等于设计差。问题的有效依据是：维护摩擦（重复改多处）、测试困难（无法自动验证）、隐式状态（跨模块副作用）、认知负担（读者无法从结构中推断职责）。

---

### 6.1 有代码证据的真实问题

**Q1 — `markSessionConsulted` 将 Claude 特有会话机制耦合进 MCP 工具层**  
`evidence: high`  
代码证据：`tools.ts:836–838`，直接读取 `process.env.CLAUDE_SESSION_ID` 并向 `/tmp` 写 session 标记，触发 Claude Code 特有 hook（解锁 Grep/Glob/Bash）。这是 Claude Code 平台行为，与 MCP 协议无关，但混入了工具实现层。Cursor、Codex、opencode 使用同一 MCP 服务器时，这段代码静默跳过但职责越界已成立。修复成本低：将 session 标记逻辑上移至 `MCPServer`（`index.ts`）层或设计成可注入的 hook。

**Q2 — `handleStatus` 硬编码 SQLite 后端字符串，不反映运行时状态**  
`evidence: high`  
代码证据：`tools.ts:1968` 硬编码 `"node:sqlite (Node built-in) — full WAL + FTS5"`，未动态查询运行时后端。若未来后端再次变更，`codegraph status` 的输出将无声失实。正确做法是从 `DatabaseConnection` 暴露 `getBackendInfo()` 方法并动态调用，而非在工具层编写注释性字符串。

**Q3 — 三处使用指引需人工同步，无自动化验证**  
`evidence: high`  
文档证据：CLAUDE.md 明确要求修改工具行为时同步更新 `server-instructions.ts`、`instructions-template.ts`、`.cursor/rules/codegraph.mdc`（"update all three"）。该要求说明三者在历史上曾出现不同步。这是维护摩擦，且无 CI 检测。

**Q4 — `looksLikeFeatureRequest` 关键词表硬编码在工具实现中，无测试和验证**  
`evidence: high`  
代码证据：`tools.ts:869–892`，约 24 个硬编码的 feature/bug/exploration 关键词，无对应测试用例，无 A/B 效果验证。这是产品层启发式逻辑混入协议实现层，行为对外不透明，边界情况（如 "swap" 归为 feature）静默误判。

**Q5 — evaluation/ 退出 `npm test`，检索质量回归无 CI 保护**  
`evidence: high`  
代码证据：`package.json` 中 `"test": "vitest run"` 不含 `__tests__/evaluation/`；CLAUDE.md 明确说 evaluation 需手动运行。后果：新增语言提取器或框架合成器破坏检索质量时，CI 不报警，只有手动执行 `npm run eval` 才能发现。

**Q6 — Windows 验证依赖人工 Parallels VM，CI 无 Windows runner**  
`evidence: high`  
文档证据：CLAUDE.md 专有"Windows validation"章节，记录了 VM 连接方式（SSH + PowerShell）。检查 `.github/workflows/` 无 `windows-latest` runner。Windows 特有行为（drive letter、`%APPDATA%`、CRLF、敏感路径 `SENSITIVE_PATHS`）通过 `it.runIf(process.platform === 'win32')` 门控但未在 CI 中运行，依赖维护者人工介入。

---

### 6.2 有设计理由的已知约束（不是"问题"）

**Q7 — 无数据流（def-use）边，局部变量传递不在图中**  
`evidence: high`  
文档证据：CLAUDE.md 明确说"tracking every local would explode the graph"，这是刻意的设计边界，不是疏忽。已知后果（`canvasNonce` 等局部 nonce 需 Agent 手动 Read）已被记录并接受。只有在准备设计探索性实现（需严格控制覆盖范围、避免节点爆炸）时才需要重新评估。

**Q8 — 动态边合成须端到端桥接，半成品会使 Agent Read 次数上升**  
`evidence: high`  
文档证据：CLAUDE.md 用 Excalidraw 实测数据记录：只合成 react-render 边（未合成 jsx-child 边）时，Read 次数从 9–10 反而升至 5–10。这是未来贡献者需遵循的操作原则，并非当前 bug，但违反原则的 PR 可能造成性能回归。

**Q9 — 发布依赖 GitHub Actions，无本地多平台 bundle 验证**  
`evidence: high`  
文档证据：CLAUDE.md 明确说"Publishing manually is wrong now"，集中化发布是刻意设计选择。`scripts/build-bundle.sh` 在 CI 多 runner 上并行构建多平台包，本地无法完整复现，但这与主流多平台 npm 包的发布方式相同，属于已接受的工程权衡。

**Q10 — 安装脚本依赖 GitHub Releases URL，网络隔离环境无降级**  
`evidence: medium`  
代码证据：`install.sh` 使用 `curl` 从 GitHub Releases 下载 bundle；网络中断时无自动降级到 `npm install`。`npm i -g @colbymchenry/codegraph` 是手动替代，但脚本本身不提示。真实影响范围：仅限完全无法访问 GitHub 的网络环境，属于部署场景限制，非通用问题。

---

### 6.3 证据不足、需进一步验证的推断

**Q11 — `parse-worker.ts` 跨线程错误处理可能损失上下文**  
`evidence: low`  
推断：Worker 线程解析失败时，错误需经序列化传回主线程，tree-sitter WASM 的 OOM/段错误类错误可能被降级为字符串。**未读 `parse-worker.ts` 全文，未验证实际错误传递路径**，不宜列为已知问题。

**Q12 — 大量磁盘临时文件写入可能在 CI 中产生 I/O 竞争**  
`evidence: low`  
观察：42 个测试文件均用 `fs.mkdtempSync` + 真实 SQLite，理论上存在并发 I/O 竞争。**未实测 CI 环境下的耗时分布，未观察到实际超时或失败**。这是常见 SQLite 测试模式，作为已知问题需要实证，不应凭推断列出。

---

### 关于原 P1 的更正

~~**原 P1 — `src/mcp/tools.ts` 体积过大（已撤销）**~~

原始分析将行数等同于设计问题，这是错误的判断依据。经深度阅读（全文 2,469 行，不是原分析误写的"15,000 行"），tools.ts 是**集中式、有意识的单文件设计**，有以下具体理由支撑：

- **schema 与 handler 共站**：`getTools()` 基于运行时项目大小动态注入 `codegraph_explore` 的预算说明，schema 和实现层的耦合是刻意的，拆文件不能消除它，只能让它变成跨文件隐式依赖
- **私有辅助函数均为"本地密封"**：`lastQualifierPart`、`numberSourceLines`、`markSessionConsulted` 等模块级函数无导出，各只有单一调用点，移出文件无设计收益
- **共享状态有功能性理由**：`handleTrace` 内的 `fileCache`（`Map<string, string[]>`）被 `sourceLineAt`/`sourceRangeAt` 共用，保证单次 trace 只读每个文件一次磁盘，这一生命周期语义无法通过跨文件拆分保留
- **`synthEdgeNote` 是跨工具格式一致性保障**：trace、node、explore 三个工具共用，保证合成边在所有输出中格式相同
- **公开接口极小**：7 个导出，`ToolHandler` 仅 6 个公开方法，所有 handler 均私有；测试通过 `execute()` 和 `getTools()` 访问，内部组织对测试完全透明

详细分析见 `docs/design-rationale-tools.md`。

---

## 7. 建议的后续修改计划

> 此处仅列出有代码证据支持（evidence: high/medium）的改进方向。推断性问题（Q11/Q12）不纳入，待读代码验证后再决定。

### 优先级 1：局部手术，成本低、收益明确

**7.1 将 `markSessionConsulted` 移出工具层**（对应 Q1）  
将 session 标记逻辑从 `tools.ts` 上移至 `MCPServer`（`src/mcp/index.ts`）层，或设计成可注入的 hook 函数。工具层不再直接读取 `CLAUDE_SESSION_ID`。改动范围：`tools.ts` ~10 行，`index.ts` ~10 行，相关测试小调整。

**7.2 自动验证三处使用指引一致性**（对应 Q3）  
新增测试或独立检查脚本，对比 `server-instructions.ts`、`instructions-template.ts`、`codegraph.mdc` 中的工具列表和关键参数，CI 不一致即失败。消除人工同步负担，防止三者静默漂移。

**7.3 从 `DatabaseConnection` 动态暴露后端信息供 `handleStatus` 调用**（对应 Q2）  
`CodeGraph` 类新增 `getBackendInfo(): string` 方法，`handleStatus` 动态调用，不再硬编码字符串。

### 优先级 2：流程改善，需协调 CI 配置

**7.4 evaluation/ 纳入可选 CI job**（对应 Q5）  
`.github/workflows/` 新增 `eval.yml`，在特定标签（如 `eval` label 或 `[run-eval]` commit message）触发时，从 GitHub 克隆测试代码库后运行 `npm run eval`，结果输出为 Check 注释。防止检索质量回归静默入库。

**7.5 CI 添加 Windows runner**（对应 Q6）  
在测试 workflow 加入 `windows-latest` runner，移除对 Parallels VM 的人工依赖。已知失败项（symlink 权限）用 `it.runIf` 合理标注，不作为 blocker。

### 优先级 3：探索性功能，需充分设计再动手

**7.6 响应式运行时合成器**（对应 Q8 的前置约束）  
为 Vue Proxy、MobX 等静默无结果的动态边补充合成器。须遵循 Q8 描述的操作原则：端到端桥接，否则不做。在提交前必须通过 probe 脚本 + 至少 4 run A/B 测试验证。

**7.7 `install.sh` 增加 npm 降级路径**（对应 Q10）  
网络隔离场景的低优先级改善，增加 `--from-npm` 标志或安装失败时自动提示 npm 替代命令。

### 暂缓或存疑

- **`looksLikeFeatureRequest` 关键词表**（Q4）：最简单的处理是删除该逻辑，将 UX 提示改为 `codegraph_context` 的固定后缀；但需要先评估对 Agent 行为的影响，避免无意中降低 context 工具的有效性。
- **API 版本兼容性策略**（原 P5）：`src/index.ts` 确实有大量 re-export，但无证据表明当前有 breaking change 已伤害下游用户，需先确认实际 npm 库使用场景再决定投入。
- **拆分 `tools.ts`**：已撤销，详见上方"关于原 P1 的更正"。

---

## 附录：关键文件速查

| 场景 | 文件 |
|------|------|
| 新增语言支持 | `src/extraction/languages/<lang>.ts` + `src/extraction/index.ts` 注册 |
| 新增框架路由 | `src/resolution/frameworks/<framework>.ts` + `index.ts` 注册 |
| 新增 MCP 工具 | `src/mcp/tools.ts`（定义 + 实现）+ `server-instructions.ts`（文档） |
| 新增 Agent 安装目标 | `src/installer/targets/<agent>.ts` + `registry.ts` 注册 |
| 修改数据库 schema | `src/db/schema.sql` + `src/db/migrations.ts` 加版本 |
| 修改工具使用说明 | `server-instructions.ts` + `instructions-template.ts` + `.cursor/rules/codegraph.mdc`（三处同步） |
| 发布新版本 | `CHANGELOG.md` + `package.json` 版本号 → commit → GitHub Actions Release workflow |
