# 本地接手指南

> 生成时间：2026-05-27  
> 分支：`cc/project-analysis`  
> 最新 commit：`ed29e16 docs: clarify environment-limited tests`  
> 状态：working tree clean，所有修改均已 push

---

## 1. 当前文档清单

### 本 Fork 新增文档（`cc/project-analysis` 分支）

| 文件 | 内容 | commit |
|------|------|--------|
| `docs/project-analysis.md` | 项目全面分析：技术栈、模块架构、问题清单 Q1–Q12（含证据评级） | `25fb07a` → 修订 `d8ed385` |
| `docs/design-rationale-tools.md` | `tools.ts` 设计理由分析；含 P1 撤销说明与三个实际问题定位 | `04827d6` |
| `docs/fork-adoption-plan.md` | Fork 利用计划：核心能力、稳定边界、四阶段任务拆分、优先级矩阵 | `26dfb2b` |
| `docs/working-conventions.md` | 本 Fork 工作约定：文档优先原则、分支约定、commit 规范、FORK 标记 | `26dfb2b` |
| `docs/verification-report.md` | Phase 1 零源码风险验证报告：build/test/audit 结果、10 个失败测试详解 | `7ebdaf3` → 修订 `ed29e16` |
| `docs/local-handoff.md` | 本文件：接手步骤、Phase 2 建议 | 本次 commit |

### 上游原有文档（未修改）

| 路径 | 说明 |
|------|------|
| `docs/benchmarks/` | 检索性能基准数据 |
| `docs/design/` | 动态分发覆盖 playbook、callback edge 合成设计 |
| `docs/plans/` | 上游规划文档 |
| `docs/SEARCH_QUALITY_LOOP.md` | 搜索质量迭代记录 |

---

## 2. 本地接手步骤

### 2.1 克隆与切换分支

```bash
git clone https://github.com/pililine/codegraph.git
cd codegraph
git checkout cc/project-analysis
```

### 2.2 安装依赖

```bash
npm ci
# 注意：node_modules 不提交；必须先 npm ci 再做其他操作
# 要求：Node.js >= 18.0.0 < 25.0.0（推荐 v22.x）
```

### 2.3 构建与类型检查

```bash
npm run build
# 预期：tsc 零 error，schema.sql + *.wasm 复制到 dist/，dist/bin/codegraph.js chmod 755
# TypeScript 类型检查包含在 build 步骤中（tsc），无独立 typecheck 脚本
```

### 2.4 运行测试套件

```bash
npm test
# 在无强制 git commit signing 的本地机器上：预期 807/817 通过，2 skip，8 fail（见下）
# 8 个 fail = 7（sync.test.ts）+ 2（extraction.test.ts）中的 git submodule 组
# 注：glyphs.test.ts 的 macOS 测试在 macOS 本机直接通过
```

### 2.5 单独运行特定测试文件

```bash
npx vitest run __tests__/sync.test.ts
npx vitest run __tests__/extraction.test.ts
npx vitest run __tests__/glyphs.test.ts
npx vitest run __tests__/installer-targets.test.ts
```

### 2.6 安全审计

```bash
npm audit
# 当前已知：8 个漏洞（picomatch HIGH 为生产依赖，rollup HIGH 为 devDep）
# 不要直接 npm audit fix，见 Phase 2 建议
```

---

## 3. 如何复现 verification-report 中的环境限制测试

### 组 A + B：git commit signing 导致的 9 个失败

**这些测试在本地无强制 signing 时正常通过。**

如需确认本地无签名强制：

```bash
git config --list | grep -E 'gpg|signing'
# 若无输出 → 本地无强制 signing → 9 个测试应通过
```

如需模拟容器环境（复现失败）：

```bash
git config --global commit.gpgsign true
npm test -- __tests__/sync.test.ts
# 预期：7 个 fail，错误信息含 "signing failed"
# 复原：git config --global commit.gpgsign false
```

### 组 C：glyphs macOS 测试（1 个失败，仅在非 macOS 平台）

```bash
# macOS 本机：
npm test -- __tests__/glyphs.test.ts   # 预期全部 pass（含 macOS 分支）

# Linux 机器：
npm test -- __tests__/glyphs.test.ts   # 预期 1 个 fail（macOS 分支模块缓存未失效）
```

---

## 4. Phase 2 建议（不执行，供本地 review 决策）

### 4.1 推荐优先开始的问题

**第一优先：`picomatch` 安全升级（独立 commit，可单独回滚）**

- 背景：`picomatch ^4.0.3` 是生产依赖，存在 HIGH 级 ReDoS 漏洞（extglob quantifiers）
- 操作：`npm audit fix`（仅升级 picomatch），然后运行 `npm test` 确认无回归
- 风险：低（picomatch patch/minor 升级，API 稳定）
- 文件变更：仅 `package-lock.json`，可能 `package.json`
- 建议：单独一个 commit `chore: upgrade picomatch to fix ReDoS (HIGH)`

**第二优先：`glyphs.test.ts` platform 门控（最小侵入性测试修改）**

- 背景：`returns Unicode glyphs on macOS` 在 Linux 上因模块缓存问题失败
- 操作：在测试中加 `it.runIf(process.platform === 'darwin')(...)`，或在 beforeEach 加 `vi.resetModules()`
- 风险：极低（只改测试文件，不改业务代码）
- 参考文件：`__tests__/glyphs.test.ts`

### 4.2 必须等本地 review 后再做的改动

以下问题涉及业务代码修改，需在本地充分测试后再实施，不建议直接在远程容器中操作：

| 问题 | 文件 | 改动性质 | 为什么需要本地 review |
|------|------|---------|-------------------|
| Q3：`markSessionConsulted` 与 Claude 平台耦合 | `src/mcp/tools.ts:836` | 将调用移至 `MCPServer` 层 | 涉及 MCP 请求生命周期，需验证 session 文件写入时机不变 |
| Q4：`handleStatus` 硬编码后端字符串 | `src/mcp/tools.ts:1968` | 调用 `CodeGraph.getBackendInfo()` | 需在 `src/index.ts` 新增方法，影响公共 API |
| Q5：`looksLikeFeatureRequest` 关键词列表 | `src/mcp/tools.ts:869` | 评估是否可简化或外置 | 影响 MCP context 工具的行为分支，需 eval 验证 |
| CI 覆盖：新增 push/PR 触发的测试 workflow | `.github/workflows/` | 新增 `ci.yml` | 需确认 runner 无强制 signing，否则 CI 会持续红 |
| Windows CI runner | `.github/workflows/` | 新增 `windows-latest` | 需先在 Parallels VM 本地验证全部测试通过 |

### 4.3 可安全新增（零上游冲突）的扩展

- 新建 `src/extensions/` 目录（Fork 专属功能入口，upstream rebase 无冲突）
- 新增 `__tests__/docs-sync.test.ts`（验证三处文档版本号一致性，Phase 1 Task 1.1）
- 新增 `.github/workflows/eval.yml`（可选评估 CI，Phase 1 Task 1.2）

---

## 5. 快速状态核查命令

```bash
# 确认在正确分支
git branch --show-current         # 应输出 cc/project-analysis

# 确认所有文档已 push
git status                        # 应输出 nothing to commit, working tree clean
git log --oneline -6              # 应看到 ed29e16 docs: clarify environment-limited tests 在顶部

# 确认 build 通过
npm run build && echo "BUILD OK"

# 确认测试通过数
npm test 2>&1 | tail -5
```
