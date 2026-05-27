# tools.ts 设计理由分析

> 分析范围：`src/mcp/tools.ts`（2,469 行）及其上下游  
> 目标：理解现状的合理性，识别真实问题，而非把"行数多"等同于"设计差"  
> 方法：阅读全文 + 追 commit 历史 + 分析 import/export 关系 + 查测试覆盖 + 追调用链

---

## 1. 文件职责地图

tools.ts 内部分为四个层次，界限清晰：

```
tools.ts
├── A. 模块级常量与安全工具 (1–261)
│   ├── MAX_OUTPUT_LENGTH / MAX_INPUT_LENGTH / MAX_PATH_LENGTH
│   ├── RUST_PATH_PREFIXES / CONTAINER_NODE_KINDS
│   ├── lastQualifierPart()            ← 符号限定名解析，仅内部用
│   ├── exploreLineNumbersEnabled()    ← env flag 读取，仅 handleExplore 用
│   ├── numberSourceLines()            ← 行号标注，仅 handleExplore 用
│   └── markSessionConsulted()         ← 写 /tmp 的 session 标记，仅 handleContext 用
│
├── B. MCP schema 声明层 (264–521)
│   ├── ToolDefinition / ToolResult / PropertySchema 接口
│   ├── projectPathProperty (所有工具共用的可选参数)
│   └── tools: ToolDefinition[]        ← 10 个工具的 JSON Schema 声明
│
├── C. 自适应预算系统 (72–193)         ← 可导出，有独立测试
│   ├── ExploreOutputBudget 接口
│   ├── getExploreBudget(fileCount)    ← 调用次数预算
│   └── getExploreOutputBudget(fc)    ← 输出字符预算（按项目规模分 5 档）
│
└── D. ToolHandler 类 (529–2469)      ← 唯一的实现主体
    ├── 基础设施 (~180 行)
    │   ├── projectCache (跨项目连接池)
    │   ├── getCodeGraph()            ← 多级 path 解析 + 连接复用
    │   ├── getTools()                ← 动态注入预算信息到 schema
    │   ├── validateString() / validateOptionalPath()
    │   ├── toolAllowlist() / isToolAllowed()
    │   └── closeAll()
    ├── 主分发器 execute() (~55 行)
    ├── 10 个 handler 方法 (~1,400 行)
    │   ├── handleSearch    (806)
    │   ├── handleContext   (831)
    │   ├── handleCallers   (897)
    │   ├── handleCallees   (932)
    │   ├── handleImpact    (967)
    │   ├── handleTrace     (1019)    ← 最复杂，~115 行
    │   ├── handleExplore   (1365)    ← 最大，~510 行
    │   ├── handleNode      (1875)
    │   ├── handleStatus    (1953)
    │   └── handleFiles     (2005)
    └── 私有辅助方法 (~300 行)
        ├── synthEdgeNote()           ← 合成边的人类可读标注
        ├── sourceLineAt()            ← 读单行源码，带文件缓存
        ├── sourceRangeAt()           ← 读范围源码，inline 到 trace 输出
        ├── buildFlowFromNamedSymbols() ← explore 的流路径检测
        ├── formatTrail()             ← node 工具的调用链展示
        ├── matchesSymbol()           ← 符号名匹配（含限定名 + Rust 路径）
        ├── findSymbol() / findAllSymbols()
        ├── formatSearchResults() / formatNodeList() / formatImpact()
        ├── buildContainerOutline()   ← 容器节点成员摘要
        ├── formatNodeDetails()       ← 节点详情格式化
        ├── formatFilesFlat/Grouped/Tree()
        ├── truncateOutput()
        ├── textResult() / errorResult()
        └── looksLikeFeatureRequest()
```

**数字摘要：**

| 层次 | 行范围 | 行数 | 导出情况 |
|------|--------|------|---------|
| 常量与私有工具 | 1–261 | 261 | 无 |
| MCP schema 声明 | 264–521 | 258 | `tools[]`, `ToolDefinition`, `ToolResult` |
| 自适应预算系统 | 72–193 | 122 | `getExploreBudget`, `getExploreOutputBudget`, `ExploreOutputBudget` |
| ToolHandler 类 | 529–2469 | 1,941 | `ToolHandler` 类 |

---

## 2. 关键调用链路

### 2.1 MCP 请求路径

```
AI Agent (Claude Code / Cursor / Codex 等)
  │ JSON-RPC over stdio
  ▼
src/mcp/index.ts :: MCPServer
  ├── handleToolsList()
  │     └── toolHandler.getTools()     ← 动态注入 budget 信息
  │           └── 返回 tools[] (schema)
  └── handleToolsCall()
        └── toolHandler.execute(toolName, args)
              ├── isToolAllowed() 检查
              ├── validateOptionalPath() 检查
              └── switch(toolName) → handleXxx(args)
                    ├── validateString()/validateOptionalPath()
                    ├── getCodeGraph(projectPath) → CodeGraph 实例
                    ├── cg.searchNodes() / cg.getCallers() / ... 等
                    └── textResult() / errorResult()
```

### 2.2 handleTrace 的内部调用链

```
handleTrace(args)
  ├── validateString("from"), validateString("to")
  ├── getCodeGraph()
  ├── findAllSymbols(cg, from)       ← 模糊 → 精确匹配
  │     └── cg.searchNodes() + matchesSymbol()
  ├── findAllSymbols(cg, to)
  ├── cg.findPath(fromId, toId, ['calls'])
  │
  ├── [for each hop]
  │     ├── synthEdgeNote(edge)           ← 合成边描述
  │     ├── sourceLineAt(cg, ref, cache)  ← 调用站点源码
  │     └── sourceRangeAt(cg, filePath, start, end, cache)  ← hop 函数体
  │
  └── [destination callees]
        └── sourceRangeAt() × 6 (last-mile 信息)
```

注意：`cache` 是 `Map<string, string[]>`，在 handleTrace 内创建，传给 `sourceLineAt` 和 `sourceRangeAt`。这意味着整个 trace 对每个文件只读一次磁盘，即使同一文件有多个 hop。这个共享状态无法在不引入参数传递的情况下跨文件拆分。

### 2.3 handleExplore 的内部调用链

```
handleExplore(args)
  ├── getExploreOutputBudget(cg.getStats().fileCount)
  ├── cg.findRelevantContext(query)  ← 返回 Subgraph
  │
  ├── buildFlowFromNamedSymbols(cg, query)   ← "流路径先行"优化
  │     ├── findAllSymbols() × N tokens
  │     └── cg.getCallees() BFS (MAX_BRIDGE=1 约束)
  │
  ├── [file clustering loop]
  │     ├── readFileSync(absPath) 
  │     ├── numberSourceLines()
  │     └── [contiguous range merging]
  │
  └── [synth edge在 relationship map 中]
        └── synthEdgeNote() (被 explore 和 trace 共用)
```

---

## 3. 与各层的关系

### 3.1 与 MCP 协议层（`index.ts`）

`index.ts` 对 `tools.ts` 的消费极为克制：

```typescript
// index.ts 中仅两处
import { tools, ToolHandler } from './tools';
this.toolHandler = new ToolHandler(null);
export { tools, ToolHandler } from './tools';
```

`MCPServer` 只负责：协议握手、stdin/stdout 传输、PPID watchdog、project 路径初始化。  
它不知道任何工具的具体逻辑，只知道 `toolHandler.getTools()` 和 `toolHandler.execute()`。这是一条清晰的界面。

### 3.2 与 CodeGraph 核心类（`src/index.ts`）

ToolHandler 持有 `CodeGraph` 实例作为私有成员，通过 `getCodeGraph()` 解析后调用 CodeGraph 的公开 API：

```
searchNodes(), getCallers(), getCallees(), findPath()
getImpactRadius(), buildContext(), findRelevantContext()
getStats(), getCode(), getChildren(), getFiles()
getJournalMode(), getProjectRoot()
```

ToolHandler 不直接碰 SQLite、不直接碰 tree-sitter。CodeGraph 是工具层和底层的唯一边界。这是正确的。

### 3.3 与 CLI 层（`src/bin/codegraph.ts`）

CLI 完全不使用 `tools.ts`。CLI 直接实例化 `CodeGraph` 并调用其公开方法。tools.ts 是纯 MCP 层的概念，CLI 的 `callers`/`callees`/`impact` 子命令并行实现了相似的格式化逻辑，两者不共享代码。

### 3.4 与数据库层

tools.ts 不直接访问 SQLite。所有数据库操作都通过 CodeGraph 类中介，tools.ts 中唯一的直接 I/O 是：
- `readFileSync(absPath)` — 读取源文件内联到 trace/explore 输出
- `writeSync(fd, ...)` — 写 /tmp session 标记（`markSessionConsulted`）

两者都有明确的安全约束（`validatePathWithinRoot`、`O_NOFOLLOW` flag）。

---

## 4. 为什么它被设计成单文件

### 4.1 schema 和 handler 必须共站

`getTools()` 在运行时动态修改 `codegraph_explore` 的描述：

```typescript
getTools(): ToolDefinition[] {
  // ...
  return visible.map(tool => {
    if (tool.name === 'codegraph_explore') {
      return {
        ...tool,
        description: `${tool.description} Budget: make at most ${budget} calls for this project (${stats.fileCount.toLocaleString()} files indexed).`,
      };
    }
    return tool;
  });
}
```

这意味着 MCP schema（`tools[]`）和实现类（`ToolHandler`）之间存在设计上的刻意耦合：schema 声明描述了 Agent 看到的工具表面，而 `getTools()` 基于运行时的项目大小动态注入信息。将 schema 移到另一个文件并不会消除这个耦合，只会使其变成跨文件的隐式依赖。

### 4.2 私有辅助函数都是单一用途的"本地密封"

模块级函数全部没有导出，全部只有一个调用点：

| 函数 | 唯一调用点 |
|------|-----------|
| `lastQualifierPart` | `findSymbol()` 内 |
| `exploreLineNumbersEnabled` | `handleExplore()` 内 |
| `numberSourceLines` | `handleExplore()` 内 |
| `markSessionConsulted` | `handleContext()` 内 |

这些函数本质上是对应 handler 的一个被分离出来的子步骤，从"隐式内联"变成"命名函数"的唯一理由是代码可读性和可测试性，而非模块解耦。将它们移到另一个文件不能提升任何设计属性，只会增加一个文件引用。

### 4.3 共享辅助方法存在真实的状态共享

`handleTrace` 内的 `fileCache`：

```typescript
const fileCache = new Map<string, string[]>();
// 被 sourceLineAt 和 sourceRangeAt 共用
```

这是在一次 trace 调用中实现"每个文件只读一次磁盘"的关键。这个 cache 的生命周期是单次 tool 调用，不是进程级。如果把 `sourceLineAt`/`sourceRangeAt` 移出 ToolHandler，调用方就需要在每次 trace 时创建 cache 并作为参数传入，增加了公开接口的复杂度，却不改善任何实际问题。

### 4.4 `synthEdgeNote` 是跨工具的格式一致性保障

三个工具都调用 `synthEdgeNote()`：
- `handleTrace` — 在 hop 之间显示 `↓ callback — registered via ...`
- `handleNode` 的 `formatTrail()` — 在 trail 中显示 `[dynamic: callback via ...]`
- `buildFlowFromNamedSymbols` — 在 explore 的流路径中显示 `↓ dynamic: jsx-render`

这三处用同一个函数保证了合成边在所有工具输出中的格式一致。如果拆到不同文件，一致性就变成了一个需要通过导入来维护的约定，而不是代码本身保证的属性。

### 4.5 成长路径是功能累积，不是随机蔓延

从 commit 历史可以清楚地看到文件体积增长的模式：

| Commit | 描述 | tools.ts 变化量 |
|--------|------|----------------|
| `f5bbc26` | perf: answer-directly steering | +554 行 |
| `025ebc8` | Release 0.9.4: dynamic dispatch coverage | +大量（含 formatTrail, 新合成边注解） |
| `23ad4ea` | fix: cap context output | 小修 |
| `7340892` | fix: bound caches + input validation | 中量 |

每次大增量都对应一个完整的性能特性（trace 内联源码、explore 流路径检测、合成边可见性），而不是"往一堆乱代码里堆"的模式。`handleExplore` 从 ~100 行增长到 ~510 行，是因为它的"整个小文件直接返回"逻辑、文件聚类算法、关系图、自适应预算、行号注解等都在同一次设计决策中被确立，共同服务于"一次调用完成流探索"的目标。

---

## 5. 体现成熟设计的地方

### 5.1 最小公开接口面

7 个导出：`getExploreBudget`、`getExploreOutputBudget`、`ExploreOutputBudget`、`ToolDefinition`、`ToolResult`、`tools`、`ToolHandler`。`ToolHandler` 的公开方法只有 6 个：`execute`、`getTools`、`setDefaultCodeGraph`、`setDefaultProjectHint`、`hasDefaultCodeGraph`、`closeAll`。所有 handler 方法都是私有的。

这是刻意的：Agent 调用的是 `execute()` 分发器，而不是直接调用 `handleExplore()`。外部测试也只通过 `execute()` 或 `getTools()` 来访问工具行为，这意味着内部重组不会破坏测试合约。

### 5.2 跨工具一致的输入验证

```typescript
// execute() 中的集中验证
const pathCheck = this.validateOptionalPath(args.projectPath, 'projectPath');
if (typeof pathCheck === 'object' && pathCheck !== undefined) return pathCheck;
```

输入边界检查（MAX_INPUT_LENGTH 10,000字符、MAX_PATH_LENGTH 4,096字符）集中在 `execute()` 和各 handler 开头，而不是分散在各处。这是防御式设计，保证了安全约束不会因新增工具而被遗漏。

### 5.3 跨项目连接池的 double-close 防御

```typescript
// 如果 projectPath 解析到和默认项目相同的根，直接返回默认实例
// 而不是把它放入 projectCache（避免 stop() 时 double-close）
if (this.cg && this.cg.getProjectRoot() === resolvedRoot) {
  return this.cg;
}
```

这段注释里明确引用了 issue #238（"database is locked" 的根因），是在真实 bug 修复过程中沉淀的防御性设计，放在 getCodeGraph() 里是合适的位置，因为连接池管理的语义就在这里。

### 5.4 自适应预算是有测试保护的不变量

`getExploreBudget` 和 `getExploreOutputBudget` 被独立导出并有专属测试文件 `__tests__/explore-output-budget.test.ts`，测试覆盖了：
- 小项目输出上限必须严格小于大项目
- 各档位之间不能出现非单调（回归 case：`<5000` 档的 per-file 预算曾小于 `<500` 档）
- 档位边界两侧共享同一参数（一致性）

这些不是通过文档约定，而是通过测试保证的。这是成熟设计。

### 5.5 "充分性"而非"指令"的输出策略

trace 的输出结尾：

```typescript
lines.push('> Full path + every hop body + the destination\'s calls are inlined above — the complete flow. Answer from it; a Read is only needed to chase a specific local variable\'s data-flow.');
```

这不是"告诉 Agent 不要 Read"的指令（CLAUDE.md 明确说这类指令无效），而是通过内联足够的信息来让 Agent 没有理由去 Read。这是经过 A/B 测试验证的设计选择，不是随意的字符串。

---

## 6. 真正的问题，而不是表面上的"大"

### 6.1 问题一：`markSessionConsulted` 是泄漏到 MCP 层的 Claude 特有行为

```typescript
// handleContext 中
const sessionId = process.env.CLAUDE_SESSION_ID;
if (sessionId) {
  markSessionConsulted(sessionId);
}
```

`CLAUDE_SESSION_ID` 是 Claude Code 特有的环境变量，与 MCP 协议无关。这段逻辑的目的是触发 Claude Code 的一个 hook（`codegraph-consulted-{hash}` 标记文件，使 Grep/Glob/Bash 得以解锁）。这是 **平台特有行为混入了协议实现层**。

当其他 Agent（Cursor、Codex、opencode）使用相同的 MCP 服务器时，这段代码无害地静默跳过（因为 `CLAUDE_SESSION_ID` 未设置），但它的存在说明工具层和 Claude Code 特有行为之间没有明确的分层。

这是文件中**唯一一处真正违反单一职责的地方**：MCP 工具逻辑不应该知道任何特定 Agent 的内部会话机制。

### 6.2 问题二：`handleStatus` 中硬编码的后端描述已过时

```typescript
// 硬编码，不再动态检查
lines.push(`**Backend:** node:sqlite (Node built-in) — full WAL + FTS5`);
```

在 `ac52fd7` 提交（0.9.4 之前）将 `better-sqlite3` 替换为 `node:sqlite` 之后，这行代码被写成了硬编码字符串，不再实际检查运行时使用的后端。`codegraph status` 命令现在总是输出 "node:sqlite"，即使未来再次切换后端也不会更新。这是**代码作为文档的反模式** — 正确的来源在 `DatabaseConnection` 内部，不应该在 tool 输出中硬写。

### 6.3 问题三：`looksLikeFeatureRequest` 是硬编码关键词表

```typescript
const featureKeywords = ['add', 'create', 'implement', 'build', 'enable', 'allow', ...];
const bugKeywords = ['fix', 'bug', 'error', 'broken', ...];
```

这个启发式方法的目的是在用户发出"功能请求类"查询时提示 UX 注意事项。关键词硬编码在工具代码里，既不可配置，也不可测试（没有对应的测试用例），而且其有效性从未经过 A/B 验证。这是 **一个未经验证的产品逻辑混入了工具实现**，容易在边界情况下误判（例如 "swap" 被归为 feature，但它也可能是 bug fix 的一部分）。

### 6.4 问题四：`getTools()` 的动态描述注入不可扩展

当前只有 `codegraph_explore` 的描述是动态注入的。如果未来有第二个工具也需要动态描述（例如 `codegraph_trace` 需要注入最大跳数），现有的 `tool.name === 'codegraph_explore'` 模式会变成一个 if-else 链。这不是紧急问题，但是一个设计上的脆弱点。

---

## 7. 是否建议暂时保持现状

**建议：保持单文件结构，针对性处理三个具体问题，不做大规模拆分。**

### 理由

**反对拆分的实质性论据：**

1. **内聚性是真实的**：`synthEdgeNote` 被 trace/node/explore 三个工具共用来保证输出格式一致；`findSymbol`/`findAllSymbols`/`matchesSymbol` 被所有需要符号查找的工具共用；`sourceLineAt`/`sourceRangeAt` 与 trace 的 `fileCache` 共享生命周期。这些是功能性耦合，不是意外耦合。拆分不消除它们，只是把它们变成跨文件导入。

2. **公开接口已经足够细**：外部只看到 `execute()` 和 `getTools()`。内部有多少文件对外部（测试、MCP server）完全透明。

3. **测试通过接口而非实现**：现有测试通过 `ToolHandler.execute()` 调用工具，不依赖任何私有 handler 方法的存在。拆分不改变测试覆盖。

4. **每个 handler 方法是独立可读的**：`handleTrace`（~115 行）、`handleExplore`（~510 行）的逻辑各自完整，可以在文件中定位后独立阅读，无需在多个文件之间跳转。

5. **拆分的真实收益低**：CI 不会因文件大变慢；语言服务器（tsc，IDE）在 2,400 行文件上不会出问题；git blame/log 在单文件上比跨文件更容易追溯每个特性的引入。

**值得针对性修复的三件事：**

| # | 问题 | 严重性 | 建议处理方式 |
|---|------|--------|------------|
| 1 | `markSessionConsulted` 耦合 Claude 特有 session 机制 | 中 | 将 session marker 逻辑移到 `MCPServer`（`index.ts`）层，或抽为可注入的 hook，tools.ts 不再直接读 `CLAUDE_SESSION_ID` |
| 2 | `handleStatus` 硬编码 backend 字符串 | 低 | 从 `CodeGraph` 暴露一个 `getBackendInfo()` 方法，status 工具动态调用 |
| 3 | `looksLikeFeatureRequest` 关键词表硬编码 | 低 | 删除该方法，将 UX 提示作为 `codegraph_context` 固定后缀（无条件添加），或移到 `server-instructions.ts` 作为通用指导 |

这三个改动都是局部手术，不影响文件整体结构，也不需要大范围测试更新。

---

## 附录：导出关系图

```
src/mcp/tools.ts
  ├── export getExploreBudget          → __tests__/explore-output-budget.test.ts
  ├── export getExploreOutputBudget    → __tests__/explore-output-budget.test.ts
  ├── export ExploreOutputBudget       → __tests__/explore-output-budget.test.ts
  ├── export ToolDefinition            → src/mcp/index.ts (类型使用)
  ├── export ToolResult                → src/mcp/index.ts (类型使用)
  ├── export tools[]                   → src/mcp/index.ts (handleToolsList 间接用)
  └── export ToolHandler               → src/mcp/index.ts
                                       → __tests__/concurrent-locking.test.ts
                                       → __tests__/explore-output-budget.test.ts
                                       → __tests__/symbol-lookup.test.ts
                                       → __tests__/security.test.ts
                                       → __tests__/mcp-tool-allowlist.test.ts

唯一的生产消费者：src/mcp/index.ts
```

所有测试都通过 `ToolHandler` 的公开接口（`execute`, `getTools`）访问工具行为，没有任何测试直接访问私有 handler 方法。这证明内部组织可以自由调整，现有测试套件会继续有效。
