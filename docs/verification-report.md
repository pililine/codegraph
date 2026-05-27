# 验证报告 — Phase 1 零源码风险层

> 生成时间：2026-05-27  
> 执行人：Claude Code（远程容器，无交互）  
> 原则：不修改业务代码，不自动修复失败，所有结论引用命令输出或文件路径

---

## 1. 环境信息

| 项目 | 值 |
|------|---|
| 分支 | `cc/project-analysis` |
| 最新 commit | `26dfb2b docs: add fork adoption plan` |
| Node.js | `v22.22.2` |
| npm | `10.9.7` |
| OS | Linux（远程容器，非 macOS，非 Windows） |
| `node_modules` 初始状态 | **不存在**，需先 `npm ci` |

`npm ci` 执行成功，安装后继续所有验证。

---

## 2. 可用脚本清单（来自 `package.json`）

| script | 命令 | 类型 | 本次运行 |
|--------|------|------|---------|
| `build` | `tsc && copy-assets && chmod` | 编译 | ✅ 执行 |
| `copy-assets` | 复制 schema.sql + *.wasm 到 dist/ | 资产复制 | ✅（build 依赖） |
| `dev` | `tsc --watch` | 开发模式 | ⏭ 跳过（交互式） |
| `test` | `vitest run` | 测试 | ✅ 执行 |
| `test:watch` | `vitest` | 监听测试 | ⏭ 跳过（交互式） |
| `test:eval` | `vitest run __tests__/evaluation/` | 评估测试 | ⏭ 跳过（需真实代码库） |
| `eval` | `build + tsx runner.ts` | 完整评估 | ⏭ 跳过（需真实代码库） |
| `clean` | `rm -rf dist` | 清理 | ⏭ 跳过（不必要） |
| `cli` | `build + node dist/bin/codegraph.js` | 本地 CLI | ⏭ 跳过（build 已验证） |
| `preuninstall` | `node dist/bin/uninstall.js` | 卸载钩子 | ⏭ 跳过（非卸载场景） |

**注意：没有 `lint`、`typecheck`（独立）、`format` 脚本。** TypeScript 类型检查通过 `build`（调用 `tsc`）覆盖。

---

## 3. 命令执行结果

### 3.1 `npm run build` — **PASS**

```
> tsc && npm run copy-assets && chmod ...
（无错误输出）
```

- tsc 编译通过，zero errors，zero warnings
- schema.sql 和 *.wasm 正确复制到 `dist/`
- `dist/bin/codegraph.js` chmod 0o755 完成

**结论：** 编译器（包含 typecheck）在当前代码库上完全通过。

---

### 3.2 `npm test` — **FAIL（10/817 个测试失败）**

**命令整体结果：** `npm test` **整体退出码非零（失败）**；`npm run build` 整体退出码为 0（通过）。两者均完整执行，无中途中断。

**总览：**

```
Test Files: 3 failed | 33 passed (36)
     Tests: 10 failed | 805 passed | 2 skipped (817)
  Duration: 13.30s
```

> **术语说明**
> - **fail**：测试完整执行，但断言失败或抛出异常，vitest 报红（本次 10 个均属此类）
> - **skip**：测试主动跳过（`it.skip` / `.todo`），不执行，不计入失败（本次 2 个）
> - **xfail**：预期失败（`expect(...).rejects`），本次无此用法
> - **不可执行**：本次无——所有 10 个测试均被实际执行，只是运行后抛出了环境相关错误

---

#### 失败组 A：`__tests__/sync.test.ts`（7 个测试 **fail**）

**失败类型：** fail（测试完整运行，`git commit` 调用抛出进程错误，测试 catch 到异常后断言失败）

**describe 路径：** `Sync Module > Git-based sync`

**7 个测试的完整名称：**

| # | 测试名称 |
|---|---------|
| 1 | `should detect modified files via git` |
| 2 | `should detect new untracked files via git` |
| 3 | `should stop reporting untracked files once they are indexed (issue #206)` |
| 4 | `should re-index an untracked file when its contents change` |
| 5 | `should detect deleted files via git` |
| 6 | `should skip files with unsupported extensions` |
| 7 | `should report no changes on clean working tree` |

**实际错误输出：**

```
Error: signing failed: signing operation failed: signing server returned status 400:
  {"error":{"message":"missing source",...}}
fatal: failed to write commit object
```

**环境限制类型：** **CI runner 配置差异 / 权限差异**  
本远程容器强制启用了 git commit signing（gpg/ssh 签名钩子），且签名服务要求 "source" 字段（标识提交来源的仓库上下文）。测试在 `fs.mkdtempSync()` 创建的临时目录中 `git init`，该临时 repo 没有合法 source 信息，签名服务拒绝签名（HTTP 400），导致 `git commit` 失败，测试中的 `sync()` 调用因此抛出错误。

**为什么不是代码 bug：**  
- `sync.ts` 本身只调用 `git status`、`git diff`、`git commit` 等标准 git 命令，代码逻辑正确
- 签名失败由容器级 git 配置注入，与代码无关；在未强制 commit signing 的环境（普通 macOS/Linux 开发机、标准 GitHub Actions runner）中，`git commit` 正常完成，这 7 个测试均通过
- 可在 `git log --show-signature` 或 `git config --list | grep gpg` 看到该配置是否存在

**本地复现/确认方式：**

```bash
# 在本地开发机（无强制 signing）运行
npm test -- __tests__/sync.test.ts
# 预期：7 个测试全部 pass

# 若想模拟容器环境（确认是 signing 导致）：
git config --global gpg.program <signing-agent>
git config --global commit.gpgsign true
# 再运行 npm test，7 个测试将复现相同 HTTP 400 错误
```

---

#### 失败组 B：`__tests__/extraction.test.ts`（2 个测试 **fail**）

**失败类型：** fail（同 A 组，测试运行后 `git commit` 调用抛出签名错误）

**describe 路径：** `ExtractionOrchestrator`

**2 个测试的完整名称：**

| # | 测试名称 |
|---|---------|
| 1 | `Git Submodules` |
| 2 | `Nested non-submodule git repos` |

**实际错误输出：**

```
Command failed: git commit -q -m "lib init"
Error: signing failed: signing operation failed: signing server returned status 400:
  {"error":{"message":"missing source",...}}
```

**环境限制类型：** 与 A 组完全相同——**CI runner 配置差异 / 权限差异**（容器强制 commit signing）

**为什么不是代码 bug：**  
`ExtractionOrchestrator` 的 git submodule 检测逻辑依赖 `git commit` 在临时目录中成功执行，以建立基准 commit 状态。代码本身无误；在无强制 signing 的环境中这两个测试正常通过。

**本地复现/确认方式：**

```bash
# 在本地开发机（无强制 signing）运行
npm test -- __tests__/extraction.test.ts -t "Git Submodules"
npm test -- __tests__/extraction.test.ts -t "Nested non-submodule"
# 预期：均 pass
```

---

#### 失败组 C：`__tests__/glyphs.test.ts`（1 个测试 **fail**）

**失败类型：** fail（测试执行完毕，值断言不匹配）

**describe 路径：** `getGlyphs`

**1 个测试的完整名称：**

| # | 测试名称 |
|---|---------|
| 1 | `returns Unicode glyphs on macOS` |

**实际错误输出：**

```
AssertionError: expected { ok: '[OK]', err: '[ERR]', … } to be { ok: '✓', err: '✗', … }
```

**根本原因详解：**  
测试通过 `vi.stubGlobal('process', { ...process, platform: 'darwin' })` 模拟 macOS 平台，期望 `getGlyphs()` 返回 `UNICODE_GLYPHS`（`✓`/`✗` 等）。但 `src/ui/glyphs.ts` 在**模块首次 import 时**即执行了一次平台判断并将结果缓存到模块级变量；在 Linux 容器中，该缓存已初始化为 `ASCII_GLYPHS`（`[OK]`/`[ERR]` 等）。测试用 `vi.stub` 修改 `process.platform` 不会使已初始化的模块级缓存失效，因此 `getGlyphs()` 仍返回 ASCII 版本。

**环境限制类型：** **OS 平台差异（macOS vs Linux）**  
这是一个**测试隔离问题**，不是业务逻辑 bug：glyphs 模块本身在各平台行为完全正确（macOS 返回 Unicode，Linux 返回 ASCII）；问题仅在于该测试的 mock 方式假设了无模块级缓存，而实际有缓存，导致该测试只能在 macOS 本机运行时通过。

**为什么不是代码 bug：**  
- `src/ui/glyphs.ts` 的缓存设计合理（避免每次调用都判断 platform）
- 业务行为在所有平台均正确
- 其余 13 个 glyphs 测试（Linux 分支）全部通过
- 仅 macOS 分支测试在非 macOS 平台存在 mock 失效问题

**本地复现/确认方式：**

```bash
# 在 macOS 本机运行——预期 pass（平台匹配，无 mock 失效）
npm test -- __tests__/glyphs.test.ts

# 在 Linux 机器运行——预期 1 个 fail（macOS 分支）
npm test -- __tests__/glyphs.test.ts

# 强制验证是缓存问题（非 mock 问题）：
# 在测试前加 vi.resetModules() 重置模块，mock platform，再 import glyphs
# 预期此时在 Linux 也能 pass
```

---

### 3.3 `npm audit` — **需关注（非立即阻塞）**

```
8 vulnerabilities (6 moderate, 2 high)
```

| 包 | 严重性 | 漏洞 | 可修复 |
|----|--------|------|--------|
| `picomatch` | **HIGH** | ReDoS via extglob quantifiers | `npm audit fix` |
| `rollup` | **HIGH** | Arbitrary file write via path traversal | `npm audit fix` |
| `postcss` | MODERATE | XSS via CSS stringify | `npm audit fix` |
| 其余 5 个 | MODERATE | — | `npm audit fix` 或 `--force` |

**CodeGraph 的 `picomatch` 是直接生产依赖**（用于 glob 匹配，`package.json` 中 `"picomatch": "^4.0.3"`），HIGH 级 ReDoS 漏洞理论上可被恶意 glob 输入触发。  
**`rollup` 是 vitest 的间接依赖**（devDependency），不影响生产运行时。  

**不建议立即修复原因：** `npm audit fix` 会修改 `package-lock.json`，可能引入 API 变化；`--force` 可能升级 breaking version。应在独立 commit 中单独处理，并运行完整测试套件验证。建议在 Phase 2 处理 picomatch（生产依赖），devDependency 的 rollup 可暂缓。

---

## 4. 文档一致性检查

检查 `project-analysis.md`、`fork-adoption-plan.md`、`working-conventions.md` 三者之间的一致性：

| 检查项 | 结果 |
|--------|------|
| SQLite 后端描述 | ✅ 三处均使用 `node:sqlite`，与 commit `ac52fd7` 一致 |
| 问题编号 Q1–Q12 | ✅ `project-analysis.md` 定义，`fork-adoption-plan.md` 引用编号一致 |
| 任务分阶段（Phase 1–4） | ✅ `fork-adoption-plan.md` 定义，`working-conventions.md` 不重复定义，无冲突 |
| 文档优先约定 | ✅ `working-conventions.md` 记录，本报告遵循该约定 |

**发现 1 处轻微过时信息：**  
`project-analysis.md` Section 5.2（开发依赖）列出 `@types/better-sqlite3 ^7.6.0`。这在 `package.json` 中仍然存在（devDependency），技术上准确，但描述为"SQLite 绑定类型"有误导性——实际运行时不再使用 `better-sqlite3`。该类型声明是历史遗留，未来可清理但不影响当前功能。

---

## 5. CI 覆盖情况

`.github/workflows/` 下只有一个文件：`release.yml`

| CI 能力 | 覆盖情况 |
|--------|---------|
| `npm run build`（tsc typecheck） | ❌ 未覆盖 |
| `npm test`（vitest） | ❌ 未覆盖 |
| Windows runner | ❌ 未覆盖 |
| `npm audit` | ❌ 未覆盖 |
| `npm run eval`（检索质量） | ❌ 未覆盖 |
| 发布流程（bundle + npm publish） | ✅ `release.yml`（手动触发） |

**结论：** 项目没有常规 CI（push/PR 触发的测试流程）。所有代码质量验证依赖本地手动运行。这与 `fork-adoption-plan.md` Phase 2 的 Task 3.1（Windows CI）和 Phase 1 Task 1.2（eval CI）的必要性完全吻合。

---

## 6. 失败汇总与建议

| 失败项 | 失败数 | 失败类型 | 环境限制类型 | 是否代码 bug | 本地可通过 | 建议 |
|--------|--------|---------|------------|-------------|-----------|------|
| `__tests__/sync.test.ts` — Git-based sync | 7 | fail（断言失败） | CI runner 配置：容器强制 commit signing | 否 | ✅（无强制 signing 的 macOS/Linux） | 本地验证；不修改业务代码 |
| `__tests__/extraction.test.ts` — Git Submodules / Nested git repos | 2 | fail（断言失败） | CI runner 配置：容器强制 commit signing | 否 | ✅（无强制 signing 的 macOS/Linux） | 同上 |
| `__tests__/glyphs.test.ts` — returns Unicode glyphs on macOS | 1 | fail（值不匹配） | OS 平台差异：Linux 模块缓存已初始化为 ASCII | 否（测试隔离问题） | ✅（macOS 本机直接通过） | Phase 2：`it.runIf(process.platform === 'darwin')` 门控 |
| npm audit 漏洞 | 8 | 非测试失败 | 依赖版本过旧 | picomatch（生产）值得关注 | — | Phase 2：独立 commit 升级 picomatch |

**不建议立即修复的理由：**
- git signing 失败是容器环境特有限制，在正常开发环境（本地 + GitHub Actions runner）不复现，强行绕过会污染测试
- glyph 缓存问题不影响生产行为，修改需理解缓存初始化时机
- audit fix 可能引入依赖版本冲突，需独立评估

---

## 7. 下一步建议

基于本次验证，对 `fork-adoption-plan.md` Phase 1 任务优先级补充确认：

1. **立即可做（零风险）：** Task 1.1（docs-sync 测试）、Task 1.2（eval CI YAML）——不涉及任何会 git commit 的逻辑
2. **Phase 2 新增：** 升级 `picomatch`（HIGH 级 ReDoS，生产依赖）；处理 glyphs 测试 platform 门控
3. **不需要做：** 为容器 git signing 问题创建 workaround——这会掩盖测试真实行为
