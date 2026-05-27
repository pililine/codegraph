# Fork 利用计划

> 目标：把这个 fork 项目吃透并转换利用，而不是重构原项目  
> 前提：已完成 `docs/project-analysis.md`（含修订版问题清单）+ `docs/design-rationale-tools.md`  
> 原则：尽量少动 core，从协议层 / 安装层 / 扩展点接入，保留 upstream 合并能力

---

## 1. 项目核心能力总结

### 1.1 真正有价值的核心能力

CodeGraph 的价值不在于"AI 能更快读代码"，而在于它把一个结构化问题（调用链路追踪、影响半径计算）从"运行时由 Agent 逐文件读"变成了"索引时预计算 + 查询时图遍历"。这个转换是根本性的，不是 prompt 工程能替代的。

**能力一：确定性语义图索引**  
tree-sitter → AST → Node/Edge → SQLite(FTS5)。提取结果是纯 AST 派生的，不依赖 LLM，因此对同一份代码是确定性的、可重现的、可差量更新的。这是整个系统的基础。25+ 语言的提取器和 20+ 框架的路由解析都建立在这之上。

**能力二：动态调度边合成**  
静态 AST 解析天然断在动态调度边界（回调、EventEmitter、React re-render、JSX child、接口分发、Django ORM descriptor）。项目的核心工程成就是通过启发式后处理（`callback-synthesizer.ts` + 各框架处理器）把这些"隐形跳跃"变成图里的 `calls` 边（标注 `provenance: 'heuristic'`），让调用链追踪能穿越动态边界。

**能力三：自适应输出预算**  
`getExploreBudget` / `getExploreOutputBudget` 根据项目文件数量（5 个档位）动态调整单次 explore 的输出量和调用次数上限。这解决了一个实际问题：小项目 dump 太多内容是浪费，大项目输出太少无法完成流追踪。这套参数经过真实 A/B 测试校准，不是拍脑袋的。

**能力四：充分性设计（Sufficiency Design）**  
`codegraph_trace` 把每跳函数体、调用站点源码、终点的下游调用全部 inline 输出，目标是让 Agent 一次调用就能完成整个流追踪，不需要后续 Read/Grep。这是经过多轮 A/B 验证的设计，不是单纯"多输出内容"。

**能力五：多 Agent 安装器**  
一套安装器支持 Claude Code、Cursor、Codex CLI、opencode、Hermes，每个 Agent 各自管理配置文件格式（JSON/JSONC/TOML）、MCP server 注册、使用说明文件。47 个参数化合约测试覆盖幂等安装/卸载。

---

### 1.2 可以直接复用的能力

以下能力不需要任何适配，拿来即用：

| 能力 | 入口 | 使用方式 |
|------|------|---------|
| 代码库索引 | `CodeGraph.indexAll()` | 实例化 `CodeGraph`，调用 `init()` + `indexAll()` |
| 增量同步 | `CodeGraph.sync()` | 文件变更后调用 `sync(changedPaths?)` |
| 符号搜索 | `CodeGraph.searchNodes()` | 传入关键词，返回 `SearchResult[]` |
| 调用链查询 | `CodeGraph.getCallers()` / `getCallees()` | 传入 nodeId，返回调用者/被调用者列表 |
| 路径查找 | `CodeGraph.findPath()` | 传入两个 nodeId，返回调用路径 |
| 影响半径 | `CodeGraph.getImpactRadius()` | 传入 nodeId，返回受影响的 Subgraph |
| AI 上下文构建 | `CodeGraph.buildContext()` | 传入任务描述，返回格式化的 Markdown 上下文 |
| MCP 服务器 | `MCPServer` | 直接启动，对接任意 MCP 兼容 Agent |
| 多 Agent 安装 | `codegraph install` | 自动检测并配置已安装的 Agent |

---

### 1.3 需要 wrapper/adapter 才适合复用的能力

| 能力 | 现状 | 需要的适配 |
|------|------|-----------|
| MCP 工具集（非 MCP 场景） | 只支持 stdio MCP 协议 | 在 `ToolHandler.execute()` 外层包一个 HTTP/gRPC 路由层，输入是 JSON，输出是 `ToolResult`（已是 JSON 结构） |
| Markdown 格式输出 | 所有工具输出是 Markdown 字符串 | 如需 JSON 结构化输出，需在 `CodeGraph` API 层直接调用并自行格式化，绕过 `ToolHandler` |
| `buildContext` 集成进自定义 AI 流程 | 返回 Markdown 字符串，面向通用 Agent 消费 | 可直接用 `cg.findRelevantContext()` 获取原始 `Subgraph`，自行构造 prompt |
| 检索质量评估 | `__tests__/evaluation/` 需要真实代码库 | 需提供自己的测试代码库路径或预置 fixture |

---

## 2. 稳定边界与高风险区域

### 2.1 Core 层：不建议轻易修改

这些模块的改动代价最高，且最容易影响 upstream 合并：

```
src/types.ts                  ← NodeKind / EdgeKind 定义，所有层都依赖，改动是全局性的
src/db/schema.sql             ← 数据模型，改动需配套迁移脚本，且影响所有已存在的索引文件
src/db/migrations.ts          ← 版本迁移，必须与 schema.sql 联动
src/index.ts                  ← 公共 API 入口，破坏性变更影响所有消费者
src/extraction/               ← 25+ 语言提取器，每个提取器的细节都有测试覆盖
src/resolution/               ← 引用解析 + 动态边合成，有"端到端或不做"原则约束
```

**一条经验规则**：如果改动会影响 `nodes` / `edges` 表的内容，就属于 core 改动，需要极其谨慎。

---

### 2.2 各层特征与改动风险

| 层次 | 代表模块 | 改动风险 | 说明 |
|------|---------|---------|------|
| **Core 数据层** | `types.ts`, `schema.sql`, `index.ts` | 高 | 任何改动波及所有层 |
| **解析层** | `extraction/languages/*.ts` | 中 | 语言之间相互隔离，但改一个语言的提取器会影响该语言的所有用户 |
| **解析层** | `resolution/frameworks/*.ts` | 中 | 框架之间相互隔离；合成器改动须遵循"端到端"原则 |
| **协议层** | `mcp/tools.ts`, `mcp/index.ts` | 低-中 | 新增工具风险低；修改已有工具的参数名/类型会破坏已安装的 Agent 配置 |
| **CLI 层** | `bin/codegraph.ts` | 低 | 新增子命令不影响其他模块；修改已有命令的 flag 可能影响脚本用户 |
| **安装层** | `installer/targets/*.ts` | 低 | 每个 target 文件独立；新增 target 只需 1 文件 + 1 行 registry |
| **数据库层** | `db/queries.ts` | 低 | 预编译语句封装；改查询不影响 schema |
| **搜索层** | `search/query-parser.ts` | 低 | FTS5 查询构建，与图结构无关 |
| **UI 层** | `ui/shimmer-progress.ts` | 很低 | 纯展示，无功能耦合 |

---

### 2.3 哪些改动会影响 upstream 合并

**高冲突风险（建议放入 fork 专用目录或不做）：**
- `src/types.ts` 里新增 NodeKind / EdgeKind — upstream 随时可能也新增，导致类型冲突
- `src/db/schema.sql` 新增字段 — upstream 的 migrations.ts 版本号会冲突
- `src/mcp/tools.ts` 里修改已有工具的 inputSchema — 会导致 upstream 的更新难以合并

**低冲突风险（相对安全的扩展）：**
- 在 `src/installer/targets/` 新增 target 文件 — 只在 registry.ts 有 1 行新增
- 在 `src/extraction/languages/` 新增语言文件 — 只在 index.ts 有少量注册代码
- 在 `src/resolution/frameworks/` 新增框架文件 — 同上
- 在 `docs/` 新增文档 — 无代码冲突
- 在 `src/mcp/tools.ts` 新增工具（在 `tools[]` 末尾 + `execute()` 新 case）— 与 upstream 可能同时新增，但 case 之间不互相影响

**建议的 fork 隔离策略：**
将所有 fork 专有改动集中在两个位置，方便 rebase：
1. `src/extensions/` — 新的适配器、wrapper、业务逻辑（全新目录，upstream 不存在，无冲突）
2. 最小化对现有文件的侵入式修改，修改时加 `// FORK:` 注释标记，方便 rebase 时定位

---

## 3. 可扩展边界

### 3.1 最适合新增代码的位置

```
src/installer/targets/<new-agent>.ts    ← 接入新 AI Agent（最安全）
src/extraction/languages/<lang>.ts      ← 新语言支持
src/resolution/frameworks/<fw>.ts       ← 新框架路由解析
src/extensions/                         ← fork 专有：适配器、wrapper、业务集成（建议新建）
```

**新增 MCP 工具**（在 tools.ts 内，风险较低）：
1. 在 `tools[]` 数组末尾添加工具 schema 声明
2. 在 `execute()` 的 `switch` 里加一个 `case`
3. 在 `ToolHandler` 类里加一个 `private async handleXxx()` 方法
4. 注意：不修改已有工具的 schema，不修改 `getExploreBudget` / `getExploreOutputBudget`

---

### 3.2 接入自己的 AI Agent 或业务流程

#### 路径一：直接使用 `CodeGraph` 类（推荐）

最干净的接入方式，完全绕过 MCP 协议：

```typescript
import CodeGraph from '@colbymchenry/codegraph';

const cg = new CodeGraph('/path/to/your/project');
await cg.init();
await cg.indexAll();

// 符号搜索
const results = cg.searchNodes('handleRequest', { limit: 10 });

// 调用链追踪
const path = cg.findPath(fromNodeId, toNodeId, ['calls']);

// 影响半径
const impact = cg.getImpactRadius(nodeId, { maxDepth: 3 });

// 为 AI 构建上下文
const context = await cg.buildContext({ task: 'fix the login bug' });

// 获取原始子图（自行格式化）
const subgraph = await cg.findRelevantContext(query, { maxNodes: 50 });
```

适用场景：自定义 AI 工作流、内部工具、CI 脚本、代码审查自动化。

#### 路径二：复用 `ToolHandler` 作为执行引擎

如果想复用现有的 10 个工具逻辑（符号查找、流追踪、explore 等），但通过 HTTP 而非 stdio 暴露：

```typescript
import { ToolHandler } from '@colbymchenry/codegraph';

const handler = new ToolHandler(cg);

// 在自己的 HTTP server 里
app.post('/codegraph/:tool', async (req, res) => {
  const result = await handler.execute(req.params.tool, req.body);
  res.json(result);  // ToolResult: { content: [{type:'text', text:'...'}], isError?: boolean }
});
```

`ToolHandler.execute()` 已经做了：输入验证、路径安全检查、工具白名单过滤、错误包装。输出是 `ToolResult`（已是 JSON 结构），不需要再解析。

#### 路径三：接入新 MCP Agent

按安装层的扩展模式，在 `src/installer/targets/` 新增一个文件实现 `AgentTarget` 接口，然后在 `registry.ts` 注册。参考 `hermes.ts` 作为最近新增的 target。

---

## 4. 问题优先级矩阵

> 评级：evidence（H/M/L）+ risk of fix（高/中/低）+ business impact（高/中/低）  
> 只有 evidence=H 且 fix risk=低/中 的问题才推荐近期处理。

| # | 问题 | Evidence | Fix Risk | Business Impact | Upstream 冲突？ | 建议 |
|---|------|----------|----------|-----------------|----------------|------|
| Q1 | `markSessionConsulted` 平台耦合 | H | 低 | 中（非 Claude Agent 的正确性） | 否 | **近期处理** |
| Q2 | `handleStatus` 硬编码后端字符串 | H | 低 | 低（展示类问题） | 否 | 低优先 |
| Q3 | 三处指引须人工同步 | H | 低 | 中（指引过期会误导 Agent） | 否 | **近期处理** |
| Q4 | `looksLikeFeatureRequest` 关键词表 | H | 低 | 低（产品启发式） | 否 | 最简是删除，测评影响后做 |
| Q5 | evaluation/ 无 CI 保护 | H | 低 | 高（检索质量回归静默） | 否 | **近期处理** |
| Q6 | Windows 依赖 Parallels VM | H | 中（CI 配置） | 中（Windows 用户覆盖） | 否 | 中期处理 |
| Q7 | 无数据流边 | H | 高（节点爆炸风险） | 低（已知约束） | 是（upstream 也在探索） | 暂缓 |
| Q8 | 动态边须端到端桥接 | H | 高（需 A/B 验证） | 高（检索质量） | 是 | 仅在有完整设计时做 |
| Q9 | 发布依赖 CI | H | — | — | — | 无需处理（刻意设计） |
| Q10 | install.sh 无网络降级 | M | 低 | 低 | 否 | 低优先 |
| Q11 | parse-worker 错误处理 | L | — | — | — | 先读代码验证 |
| Q12 | 测试 I/O 竞争 | L | — | — | — | 先实测验证 |

**近期推荐处理：Q1、Q3、Q5**（三者均 evidence=H、fix risk=低、不影响 upstream）

---

## 5. 七条手术级建议排序

### 建议 1（对应 Q5）：evaluation/ 纳入可选 CI job

**为什么值得做：** 这是检索质量的最后一道防线。Q5 的 business impact 是"高"，因为检索质量回归不会触发任何现有 CI 报警——测试只验证代码正确性，不验证"Agent 减少了多少 Read 调用"。这是这个项目最核心的价值指标，却没有自动保护。

**改动范围：**
- 新建 `.github/workflows/eval.yml`（约 40 行 YAML）
- 不改任何源码或测试文件

**验证方式：** 手动触发 workflow，观察 `npm run eval` 能否正常运行并输出结果。

**是否影响 upstream 合并：** 否，新文件，无冲突。

---

### 建议 2（对应 Q3）：自动验证三处使用指引一致性

**为什么值得做：** 三处指引（`server-instructions.ts` / `instructions-template.ts` / `codegraph.mdc`）不同步时，Agent 接收到的使用指导会相互矛盾，CLAUDE.md 明确记录这是历史上出现过的问题。手动同步是维护摩擦，自动化成本低。

**改动范围：**
- 新建 `__tests__/docs-sync.test.ts`（约 40 行）
- 提取两者共同包含的关键字符串（工具名列表、关键参数）做 `expect().toContain()` 断言
- 不改任何源文件

**验证方式：** `npx vitest run __tests__/docs-sync.test.ts`

**是否影响 upstream 合并：** 否，新测试文件，且如果 upstream 同步了三处指引，测试仍通过。

---

### 建议 3（对应 Q1）：将 `markSessionConsulted` 上移至 `MCPServer` 层

**为什么值得做：** `tools.ts` 是协议实现层，不应该知道 `CLAUDE_SESSION_ID`。目前其他 Agent（Cursor、Codex）使用这个 MCP 服务器时，这段代码静默跳过，但职责越界已经成立。更重要的是：如果未来想让这个 session 机制可配置（或对不同 Agent 有不同行为），现在的位置让它无法被独立测试。

**改动范围：**
- `src/mcp/tools.ts`：删除 `markSessionConsulted` 调用（约 4 行）
- `src/mcp/index.ts`：在 `handleToolsCall` 里，tool 调用成功后触发 session 标记（约 10 行）
- `markSessionConsulted` 函数本体可以保留在 `tools.ts`（仍在同文件内），或移到 `index.ts`，或提取为独立模块
- 相关测试（`security.test.ts` 里的 session marker 测试）需小调整

**验证方式：** `npx vitest run __tests__/security.test.ts` 通过；手动用 Claude Code 调用 `codegraph_context` 后确认 `/tmp/codegraph-consulted-*` 文件仍被创建。

**是否影响 upstream 合并：** 低风险。`tools.ts` 的 session 逻辑是一小块，upstream 不太可能同时在这里有大改动。

---

### 建议 4（对应 Q2）：`handleStatus` 后端信息动态化

**为什么值得做：** 目前 `codegraph status` 总是输出 "node:sqlite (Node built-in) — full WAL + FTS5"，不反映运行时真实状态。这是"代码作文档"的反模式。影响虽小，但每次后端变更时需要记得同步这个字符串，是不必要的维护负担。

**改动范围：**
- `src/index.ts`（或 `src/db/index.ts`）：新增 `getBackendInfo(): string` 方法（约 5 行）
- `src/mcp/tools.ts`：`handleStatus` 调用 `cg.getBackendInfo()` 替换硬编码字符串（约 2 行）

**验证方式：** `npx vitest run __tests__/mcp-initialize.test.ts`；手动运行 `codegraph status` 确认输出正确。

**是否影响 upstream 合并：** 极低。是纯新增 API，不修改任何接口。

---

### 建议 5（对应 Q4）：评估并简化 `looksLikeFeatureRequest`

**为什么值得做（以及为什么要先评估）：** 关键词表是硬编码的产品启发式，无测试、无 A/B 验证。最简处理是删除，把 `codegraph_context` 的 UX 提示改成固定后缀（始终添加）。但删除之前需要评估：这个提示对 Agent 行为有多大影响？如果 Agent 在这个提示下会有实质不同的响应，删除可能降低工具的有效性。

**建议操作：**
1. 在本地用几个典型的 feature request 和 bug fix query 测试，记录 `looksLikeFeatureRequest` 的分类是否符合预期
2. 若分类质量差，删除该方法，改为在 `handleContext` 末尾无条件添加 UX 提示
3. 若分类质量尚可，为该方法补充测试用例，让行为可验证

**改动范围：** 仅 `tools.ts` 内，约 25 行，`handleContext` 逻辑小调整。

**是否影响 upstream 合并：** 否。

---

### 建议 6（对应 Q6）：CI 添加 Windows runner

**为什么值得做：** Windows 验证目前依赖人工 SSH 到 Parallels VM，维护者变动或 VM 不可用时 Windows 验证即中断。Windows 用户占 npm 下载量的重要份额，无自动验证是隐患。

**改动范围：**
- `.github/workflows/` 里的测试 workflow：添加 `windows-latest` 矩阵项（约 5 行 YAML）
- 已知失败项（`security.test.ts > symlink resistance`）用 `it.runIf(process.platform !== 'win32')` 门控，防止误报

**验证方式：** Push 后观察 Actions tab 里的 Windows job 是否通过。

**是否影响 upstream 合并：** 否，YAML 文件改动独立。

---

### 建议 7（对应 Q10）：`install.sh` 添加 npm 降级提示

**为什么值得做：** 目前安装脚本在无法访问 GitHub Releases 时直接报错退出，没有提示替代方案。只需在 `curl` 失败的 `else` 分支添加一行提示即可，成本极低。

**改动范围：** `install.sh` 约 3 行，`install.ps1` 约 3 行。

**是否影响 upstream 合并：** 低风险，是纯新增的 else 分支错误提示。

---

## 6. 下一阶段任务拆分

> 原则：每个任务单独 commit，可独立验证，可独立回滚，不修改 core 架构，不重构 tools.ts

### Phase 1：零风险基础建设（建议第一周）

每个任务独立，互相无依赖，可并行。

---

**Task 1.1 — 新建 `docs-sync.test.ts`，验证三处指引一致性**

```
目标：防止 server-instructions / instructions-template / codegraph.mdc 静默漂移
文件：__tests__/docs-sync.test.ts（新建）
范围：约 40 行测试代码，不改任何源文件
验证：npx vitest run __tests__/docs-sync.test.ts
回滚：git revert 该 commit
upstream 影响：无
```

实现思路：读取三个文件内容，提取工具名列表（`codegraph_search`、`codegraph_trace` 等），断言每个文件都包含相同的工具名集合。失败信息明确说明哪个文件缺少哪个工具名。

---

**Task 1.2 — 新建 `.github/workflows/eval.yml`**

```
目标：检索质量回归有 CI 保护
文件：.github/workflows/eval.yml（新建）
范围：约 40 行 YAML，不改任何源文件
触发条件：PR 带 eval label，或 commit message 含 [run-eval]
验证：手动触发，观察 workflow 是否能运行 npm run eval
回滚：git revert 该 commit
upstream 影响：无
```

注意：`npm run eval` 依赖真实代码库，CI 需要先克隆测试用代码库（或使用 `__tests__/evaluation/` 里已有的 fixture）。首次运行可以只跑 fixture，不跑真实 repo 克隆。

---

**Task 1.3 — 修正 `project-analysis.md` 中的技术栈描述（已完成）**

此任务已在上一阶段完成（SQLite 后端更正为 `node:sqlite`），无需再做。

---

### Phase 2：小范围代码手术（建议第二周，按顺序）

每个任务需要读完相关代码再动手，不要批量合并成一个大 commit。

---

**Task 2.1 — 将 `markSessionConsulted` 调用从 `tools.ts` 上移至 `index.ts`**

```
目标：tools.ts 不再直接读取 CLAUDE_SESSION_ID
文件改动：
  src/mcp/tools.ts —— 删除 handleContext 里的 3 行 session 调用
  src/mcp/index.ts —— 在 handleToolsCall 成功后调用 markSessionConsulted
  （markSessionConsulted 函数体可以保留在 tools.ts 或移入 index.ts，视具体实现）
验证：npx vitest run __tests__/security.test.ts
      手动：调用 codegraph_context 后确认 /tmp/codegraph-consulted-* 仍被创建
回滚：git revert 该 commit
upstream 影响：低（session 逻辑改动小，不影响工具行为）
```

**先读代码再做**：完整读 `src/mcp/index.ts:handleToolsCall()` 和 `src/mcp/tools.ts:handleContext()` 的当前实现，确认移动逻辑后 session 标记在正确时机写入。

---

**Task 2.2 — `handleStatus` 动态读取后端信息**

```
目标：codegraph status 输出反映真实运行时后端
文件改动：
  src/index.ts 或 src/db/index.ts —— 新增 getBackendInfo(): string（约 5 行）
  src/mcp/tools.ts:handleStatus —— 调用 cg.getBackendInfo() 替换硬编码字符串（约 2 行）
验证：npx vitest run __tests__/mcp-initialize.test.ts
      手动：codegraph status 输出无硬编码字符串
回滚：git revert 该 commit
upstream 影响：极低（纯新增 API）
```

---

**Task 2.3 — 评估并处理 `looksLikeFeatureRequest`**

```
目标：明确关键词表行为，补充测试或删除
先做：用 3-5 个典型 query 手动测试分类结果，判断质量
如果质量差：删除该方法，handleContext 末尾无条件添加 UX 提示
如果质量可接受：新建 __tests__/context-feature-heuristic.test.ts 覆盖边界情况
文件改动：仅 src/mcp/tools.ts 内，约 25 行
回滚：git revert 该 commit
upstream 影响：无
```

**注意**：这个任务在评估结果出来之前不要合并，需要人工判断。

---

### Phase 3：CI 基础设施（建议第三周）

**Task 3.1 — CI 添加 Windows runner**

```
目标：Windows 测试自动化，不依赖 Parallels VM
文件改动：
  .github/workflows/<test-workflow>.yml —— 添加 windows-latest 矩阵（约 5 行）
  __tests__/security.test.ts —— 确认 symlink 测试已有 it.runIf 门控
验证：Push 后观察 Actions tab
回滚：git revert 该 commit
upstream 影响：低（YAML 矩阵新增独立于 upstream 逻辑）
```

先读 `.github/workflows/` 下的现有 workflow 文件，理解当前 matrix 配置再做修改。

---

### Phase 4：探索性扩展（按需，无时间约束）

这些任务视业务需要再启动，不列入近期计划：

- **Task 4.1**：在 `src/extensions/` 新建 HTTP wrapper，将 `ToolHandler.execute()` 暴露为 REST API
- **Task 4.2**：新增自定义 Agent 安装 target（如 Zed、Windsurf、Continue）
- **Task 4.3**：响应式运行时合成器（Vue Proxy、MobX），须完整读 dynamic-dispatch-playbook 后设计
- **Task 4.4**：`install.sh` 添加 npm 降级提示

---

## 附录：fork 工作流建议

```bash
# 日常开发
git fetch upstream main               # 同步 upstream
git rebase upstream/main              # 基于最新 upstream 重放 fork 改动

# 每次改动
git checkout -b task/<task-name>      # 独立分支
# 做改动
git commit -m "feat/fix/docs: ..."
git push origin task/<task-name>

# 合并到 fork 的开发分支
git checkout cc/project-analysis      # 或 fork 的主开发分支
git merge task/<task-name>

# 标记 fork 专有改动（方便 rebase 时识别）
# 在改动行附近加注释：// FORK: <原因>
# 这样 upstream rebase 时可以快速定位需要手工处理的行
```

**关键约定**：
- `src/extensions/` 目录全部是 fork 专有代码，upstream 不存在，rebase 时零冲突
- 对现有文件的侵入式改动，用 `// FORK:` 注释标记，每次 rebase 后 `grep -r "FORK:" src/` 快速检查
- Phase 1 和 Phase 2 的任务属于"改善 fork 后维护体验"，对 upstream 也有贡献价值，可考虑发 PR
- Phase 4 的扩展任务属于"fork 专有功能"，不建议提 PR 到 upstream（除非完全满足 upstream 的验证要求）
