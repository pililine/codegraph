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
| 数据库 | SQLite via `better-sqlite3`（原生）/ `node-sqlite3-wasm`（WASM 降级） |
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

**可选原生依赖（运行时动态加载）：**
- `better-sqlite3`（原生 Node 绑定，性能优先）
- 降级：`node-sqlite3-wasm`（纯 WASM，零编译，性能略低）

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

## 6. 当前潜在问题

### 6.1 架构层面

**P1 — `src/mcp/tools.ts` 体积过大**  
单文件 ~15,000 行，包含所有 MCP 工具实现。随功能增长，已超出合理单文件范围，代码导航、测试覆盖、代码审查都变得困难。每次新增工具都需要修改同一个大文件，合并冲突概率高。

**P2 — 无数据流（data-flow）边**  
文档明确标注：局部变量传递（如 `canvasNonce`）不在图中，属于"已知空白"。对于响应式运行时（Vue Proxy、MobX、Halo `ReactiveExtensionClient`、MediatR）等无静态边的场景，探索直接失效（静默无结果，非报错），Agent 仍需手动 Read。

**P3 — 动态边覆盖原则"部分桥接比不桥接更差"**  
CLAUDE.md 明确指出：Excalidraw 实测中，只合成 react-render 边（未合成 jsx-child 边）时，Read 次数反而从 9–10 升至 5–10。如果未来有新的框架合成器只覆盖一跳，可能造成 Agent 工具调用数上升而非下降。

### 6.2 代码质量层面

**P4 — 缺少全局 eslint / prettier 配置**  
项目使用 TypeScript strict 模式，但无 eslint 或 prettier。代码风格一致性依赖人工约定，PR review 中可能产生风格分歧。

**P5 — 无 API 版本兼容性策略**  
`src/index.ts` 作为库入口，随着功能增加已有大量 re-export，但无 semver-protected 稳定 API 列表。破坏性变更（如修改 `TraversalOptions` 字段）对库的下游用户是无声的。

**P6 — `parse-worker.ts` 错误处理粒度粗**  
Worker 线程解析失败时，错误信息经序列化传回主线程，部分 tree-sitter 解析错误（如 WASM OOM）可能被转换为不明确的错误信息，排查困难。

### 6.3 测试层面

**P7 — evaluation/ 不纳入 `npm test`**  
`__tests__/evaluation/` 需要真实代码库（需网络下载或预置），手动运行 `npm run eval`。新贡献者可能忽略评估步骤，导致检索质量回归未被 CI 发现。

**P8 — Windows 测试门控依赖人工 Parallels VM**  
Windows 特有行为（drive letter、`%APPDATA%`、CRLF、敏感路径）通过 `it.runIf(process.platform === 'win32')` 门控，但 CI 不跑 Windows runner，实际验证依赖人工连接 Parallels VM。维护成本高，易被遗忘。

**P9 — 测试中大量写临时文件到 `fs.mkdtempSync`**  
每个测试创建真实文件系统 + 真实 SQLite，测试速度受磁盘 I/O 影响明显。在 CI 环境中，42 个测试文件并行跑可能产生资源竞争（特别是 `db-perf.test.ts`）。

### 6.4 发布流程层面

**P10 — 发布完全依赖 GitHub Actions，无本地验证手段**  
`scripts/build-bundle.sh` 打包逻辑在 CI 上运行，本地无法完整模拟多平台 bundle 构建。如果 CI 配置有问题，只能等到触发 Release workflow 后才能发现。

**P11 — `install.sh` 依赖外部 URL（GitHub Releases）**  
安装脚本从 GitHub Releases 下载预编译 bundle。如果用户网络无法访问 GitHub（如企业内网），安装完全失败，无降级路径（npm 安装是独立流程）。

### 6.5 文档层面

**P12 — 三处使用指引需手动同步**  
`src/mcp/server-instructions.ts`、`src/installer/instructions-template.ts`、`.cursor/rules/codegraph.mdc` 内容应保持一致，但无自动化验证。历史上已出现三者不同步的情况。

---

## 7. 建议的后续修改计划

以下计划按优先级从高到低排列，均为局部改动，不影响核心逻辑。

### 优先级 1：可维护性提升（低风险，高收益）

**7.1 拆分 `src/mcp/tools.ts`**  
将 15,000+ 行按工具分组拆分为多个文件：`tools/search.ts`、`tools/trace.ts`、`tools/explore.ts`、`tools/node.ts`、`tools/context.ts` 等，`tools/index.ts` 统一 re-export。不改变任何对外行为，只改文件组织。

**7.2 添加 eslint + prettier**  
加入 `@typescript-eslint/eslint-plugin`（严格规则集）和 prettier，在 `package.json` 加 `"lint"` 脚本，纳入 CI check。可选：添加 `husky` + `lint-staged` pre-commit hook。

**7.3 自动验证三处使用指引一致性**  
新增测试用例（或独立脚本），对比 `server-instructions.ts`、`instructions-template.ts`、`codegraph.mdc` 中的关键段落（工具列表、参数格式），CI 失败即报警，消除人工同步负担。

### 优先级 2：测试与 CI 改善（中等风险）

**7.4 evaluation/ 纳入可选 CI job**  
在 `.github/workflows/` 新增一个 `eval.yml` workflow，在有标记（如 `[run-eval]` commit message 或 `eval` label）时触发，从 GitHub 克隆测试代码库后运行 `npm run eval`，结果作为 Check 注释输出。

**7.5 Windows CI Runner**  
在 `.github/workflows/` 的测试 job 中加入 `windows-latest` runner，移除对 Parallels VM 的人工依赖。需要处理已知失败项（symlink 权限问题）做合理标注。

**7.6 轻量级数据库 Mock 用于部分测试**  
对不依赖真实 SQLite 特性的单元测试（如 `query-parser.test.ts`、`search-query-parser.test.ts`），引入内存 SQLite（`:memory:` 模式），减少磁盘 I/O，加快测试速度。

### 优先级 3：功能补全（中等风险）

**7.7 数据流（def-use）边的探索性实现**  
对局部变量的简单传递（`const x = foo(); bar(x)`），在 TypeScript 提取器中试点提取 `type_of` 边。需严格控制覆盖范围，避免节点爆炸（参考 CLAUDE.md 的"不跟踪每个局部变量"原则）。先做为可选实验特性，用 `--experimental-defuse` flag 控制。

**7.8 响应式运行时合成器**  
为 Vue Proxy、MobX、Halo `ReactiveExtensionClient` 添加合成器，补全"静默无结果"的动态边缺口。遵循 CLAUDE.md 要求：端到端合成，不桥接一半；先用 probe 脚本验证，再提交。

**7.9 npm 离线安装模式**  
为 `install.sh` 增加 `--from-npm` 标志，在无法访问 GitHub Releases 时自动切换为 `npm i -g @colbymchenry/codegraph`，解决企业内网安装问题。

### 优先级 4：长期架构（高风险，需充分讨论）

**7.10 稳定公共 API 面**  
在 `src/index.ts` 中显式标注 `@public` / `@internal` 方法，配合 `tsconfig` 生成只包含公共 API 的 `.d.ts`，便于下游库用户升级。

**7.11 MCP 工具版本化**  
随着工具数量增长，考虑在 MCP `initialize` 响应中加入工具版本号，让 Agent 可以适配不同版本的 codegraph，减少跨版本兼容问题。

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
