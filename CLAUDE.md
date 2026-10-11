# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目定位

REA(Reverse Engineer Anything)——本地化逆向工程工具,通过 CLI 与 MCP server 暴露统一的逆向能力,支持原生二进制(Hopper / Ghidra / IDA)、JavaScript/Electron 应用、.NET 程序集、固件、Android APK 与网站运行时。
版本 `6.3.0`,包名 `rea-agents`(MCP 名 `io.github.morluto/rea`)。授权 MIT。

## 工具链

- Node.js:`.nvmrc` 锁定 `24.18.0`(同时支持 `^22.19.0 || ^24.11.0 || >=26.0.0`)
- 包管理器:`npm@11.16.0`(`package.json#packageManager`)
- TypeScript 7 / Vitest 5 / Turborepo 2 / oxlint + oxfmt / VitePress 1
- 安装前先 `nvm use`;Cursor Cloud 上旧版 Node 可能在 `PATH` 前,环境安装会把固定工具链放到 `/usr/local/bin`
- Husky pre-commit 跑 `oxfmt` + `oxlint --fix`;pre-push 跑 `check:fast`

## 常用命令

```bash
# 安装与基础检查
npm ci                                  # 锁定依赖 + 准备 husky 钩子
npm run check:fast                      # 缓存的 typecheck + lint(类似 pre-push)

# 构建
npm run build:cached                    # 编译运行时并打包 skill(日常够用)
npm run build                           # 强制重跑 build:cached 同集任务
npm run metadata:generate               # 包/锁文件/托管 skill 版本变更后必须先跑

# 测试(按场景选)
npm run test:local -- <path...>         # 不构建跑源测试
npm run test:focused -- <path...>       # 精确测试及其依赖产物
npm run test:changed                    # 相对 origin/main 受影响测试
npm run test:boundary                   # 跨生产边界的集成测试
npm run test:mcp                        # MCP transport / 工具契约
npm run test:acceptance                 # 编译后 CLI / MCP 旅程
npm run test:coverage                   # v8 覆盖率
npm run test:ci:shard                   # CI 分片

# 全量本地门(可选)
npm run check                           # typecheck+lint+format+knip+metadata+architecture-guards
npm run check:pr                        # 上述 + 测试 + docs:check + jscpd + scan:todos

# 真实引擎验证(需对应环境)
npm run verify:hopper                   # macOS Hopper
npm run verify:hopper:linux             # Linux Hopper demo runner
npm run verify:ghidra / verify:ida      # Ghidra / IDA 真实通道
npm run verify:browser / verify:electron
```

构建/生成命令在 Turbo 图上持有 SQLite 事务;`.cache/rea-command-locks/` 是锁库,**命令运行时禁止删除**。

## 架构(分层 + 依赖向内)

依赖流向:**contracts → domain → application → adapters → entry points**。Provider 协议/解析只能放在 adapter 层;CLI/MCP 共享工作流放在 application 层。修改组合前先看 `docs/architecture.mermaid`。

```
入口: src/cli.ts + src/cli/      CLI 一次性适配器
      src/main.ts + src/main/    MCP stdio server 适配器
      scripts/rea.mjs            执行分发器,路由 mcp / one-shot

组合: src/composition/           全新 typed provider 与 session 工厂,不启动引擎、不获取目标

适配: src/server/                MCP 请求翻译、createServer、工具注册
      bridge/                    引擎桥接(Hopper Python / Ghidra Java / Swift LLDB / JADX / mitmproxy / pwntools / pwndbg)
      src/cli/+src/application/Setup*   主机安装适配器

应用: src/application/           共享 CLI/MCP 工作流;investigation/InvestigationRecords、EvidenceLedger、Unknown 归属
      SessionProviderRouter + BinarySession:每目标一条不可变深度绑定
      AnalysisProviderRegistry:排序后的深度候选,确定性选择,无回退

Provider: src/hopper, src/ghidra, src/ida, src/browser, src/javascript,
         src/dotnet, src/firmware, src/android, src/evm,
         src/native(pwntools/pwndbg), src/reference, src/artifacts,
         src/inspector(V8 Inspector)

进程底座: src/process/            自有进程组、私有根目录、截止时间、清理
         src/windows/             Windows 主机边界(Job Object、保护 DACL)
         native/windows/          Node-API 实现(NTFS 准入、原子 Job 分配)

契约/域/配置/评估:
  src/contracts/      caller 可见 schema(analysis / managed / browser / Electron / application / session)
  src/contracts/errorSchemas.ts   共享分析错误 schema + CLI 诊断
  src/domain/         纯无副作用模块(evidence、comparisons、application graphs)
  src/config/         Zod 校验的环境解析(provider 设置与来源)
  src/evaluation/     agent 发布评估、会话转录评分
```

`src/composition/` 工厂必须**全新构造 provider,不启动引擎、不获取目标**;可选适配器失败时**保留已成功的同伴**。

## 测试分层

详见 `docs/testing.md`。深度从浅到深:

| 深度        | 位置                         | 作用 |
|-------------|------------------------------|------|
| Module      | `src/**/*.test.ts`           | 单域/服务/适配器,只读自己的公开面 |
| Composition | `tests/composition/**`       | provider 中立的 session/registry 装配;不跨生产边界 |
| Boundary    | `tests/boundary/**`          | 恰好一条生产边界(文件系统/进程/网络/浏览器/CLI/provider) |
| MCP         | `tests/boundary/mcp/**`      | MCP transport + 工具契约 |
| Acceptance  | `tests/acceptance/**`        | 编译后 CLI / MCP 旅程(注入 provider 仍是集成) |
| Conformance | `tests/conformance/**`       | 共享 provider 契约,按声明能力参数化 |
| Evaluation  | `tests/evaluation/**`        | 评估器解析、评分、报告生成 |

约束:
- 共置 domain 测试**只能导入自有域与向内依赖**,不能构造 session、启动子进程或开 socket
- 共置 application 测试**通过显式 port + 录制 adapter**,不走 CLI/MCP 入口
- 仅 acceptance 能装配完整运行时或调用编译后产品面
- 临时目录用 `tests/fixtures/temporaryDirectory.ts` 的 `createTestTempDirectory`,失败/超时也要精确 await 清理
- `npm run verify:test-temp-hygiene` 在新 `TMPDIR` 下跑全套,禁止 glob 清 `shared /tmp/rea-*`
- Vitest worker 拆解时统一释放原生进程 inspector;fork 终止不跑 Node 退出钩子

## 工具设计(新增/修改 MCP tool 时)

- 先看 caller 提问,再扫现有 canonical 契约、最近工具、provider 能力;能扩展就别新增
- 命名按"动作 + 对象",避免在一把工具里混发现/执行/变更
- Provider 中立的公开名与语义;每个 provider 报告**精确覆盖**
- 输入要严格(target identity + 必填字段);保留 caller 选定的 target/action/capture/output,删掉被忽略的选项
- 返回**内联完整证据**:artifact/provider identity、source locations、Evidence 引用、confidence、limitations
- Observed/derived/inferred 必须区分;missing = unknown ≠ absent
- 限制只能是真实格式/协议/权限/已测资源约束;如实报告分页、截断、取消、部分失败
- Provider 协议放 adapter;CLI/MCP 共享工作流放 application;Evidence provenance、unknown 投影、session 保留语义必须跨适配器保持
- 真实引擎声明必须跑对应 provider lane;模拟 provider 不能建立真实行为

## 关键约定(摘自 AGENTS.md,必须遵守)

**安全/权限**
- 配置改动**必须可加、可幂等**,带备份;安装器不得安装/升级无关软件(Homebrew、Node、npm、Java、Ghidra 等)
- Ghidra/IDA 是 bring-your-own;`rea setup` 必须先展示计划并获批准
- Ghidra 使用临时数据库,**不修改可执行字节、不控 GUI**;Windows Ghidra P0 无变更权限
- IDA 分析只读;绝不在附加 GUI 数据库上 save/close
- 保留 caller 选定的输入与本地证据;**显式标为敏感的才脱敏**,不要从名字/文本推断机密
- 不要因为子进程继承了环境就持久化环境变量

**数据/所有权**
- 每个解释与资源有清晰 owner;跨边界保留 identity、omission、uncertainty、source evidence
- 清理失败的资源**保留所有权**;仅 PID + 可执行路径**不足以**在退出/复用后认定所有权
- 用 `src/process/` 的 supervision + identity 原语,别自己另起炉灶
- 解析未知边界值 + 建模预期失败:**tagged error + `Result`**;保留部分事实与有意义的失败原因
- 身份与展示格式分离;绝对路径/file URL/HTTP 路径语义不同,不要混
- 一次归一化,在所属边界用**格式感知**解析器
- 抽象只在移除已观察复杂度时引入;不再需要的表示、别名、包装、冗余测试在确认消费方后清除
- 评估触及代码的脆性;先修产生方或所有权规则,别加下游修补

**Schema/契约**
- SDK 转换后用声明的方言和真实客户端**校验广告的 JSON Schema**
- handler 绑定到命名契约 + 精确类型;目录顺序**永远不能**选择操作或 schema

## 桥接与第三方(本地携带)

```
bridge/
  hopper_bridge.py             Hopper 内运行,适配 Hopper Python API
  ghidra/                      Headless 脚本入口 + 扩展(NativeAot*)
  native/                      Swift UI 子树探针 + LLDB tracer
  android/                     JADX 桥
  mitmproxy/                   网络抓取
  pwntools/, pwndbg/           ELF / core / 调试器适配
  process/                     Swift 进程运行 token 读取
third_party/                   binwalk / evmole / ghidra-nativeaot /
                               jadx-headless-mcp / pwndbg / pwntools /
                               unblob / wakaru / web-network-captures
```

桥接是宿主引擎外的子进程;超时、清理、私有根目录走 `src/process/`。

## 文件所有权与忽略

**禁止提交**:`dist/`、`node_modules/`、`native/windows/build/rea-windows-x64.node`、`build/conformance/manifest.json`(除非显式指定)、二进制、provider 项目文档、凭据、本地规划工件。

**已忽略输出**:`docs/public/product-catalog.json`、`docs/verification/managed-conformance-*.json`、`skills/`、`.cache/mcp-tool-catalog.json`(测试元数据,运行时与源检查不消费)。

修改契约:`docs/error-contract.schema.json` 与 `src/generatedPackageMetadata.ts` 保持受版本控制,需运行 `npm run metadata:generate`。

修改 skill:改 `skill-src/`,再跑 `npm run metadata:generate`(生成 `skills/`,不提交)。

## 日志

- `REA_LOG_LEVEL`: `trace | debug | info | warn | error | fatal | silent`
- MCP 默认 `info` 到 stderr;一次性 CLI 日志默认不开,需显式开启走 stdout
- 请求参数、bridge token、环境数据**自动脱敏**

## 提交与发布

- 提交标题与 PR 标题用 Conventional Commits;breaking change 加 `!` 或 `BREAKING CHANGE:` footer
- PR 描述中说明行为变更、相关校验、真实 provider 验证范围
- 发布流程见 `docs/releasing.md`;主分支推送触发 Release Please PR,审核迁移说明、CI 通过后合并即发布
- 包/锁文件/托管 skill 版本变更 → `npm run metadata:generate` → 再 build
- 添加/移动测试后跑 `npm run verify:test-discovery`;`verify:module-boundaries` 包含在 lint 中

## 文档站点

- VitePress 源在 `docs/`;导航在 `docs/.vitepress/config.ts`
- `docs/` 内链接用相对路径(在 GitHub 与站点都生效);`docs/` 外文件用完整 GitHub URL
- `npm run docs:dev` 开发预览,`docs:build` 校验生产构建+链接,`docs:preview` 在 `/rea/` 预览
- `npm run docs:check` 校验生成的元数据与 schema;`docs/public/...` 与 `docs/verification/...` 是 CI 产物,不提交

## Turborepo 特别提示(自动注入)

> 这是你认识的 Turborepo **不是**同一个。turbo 的版本、任务行为、CLI 命令因安装版本而异,可能与训练数据不同。
> 在 monorepo 中从相关 workspace 解析 `turbo` 包(根目录不一定可见);例如 `node -p "require.resolve('turbo/package.json')"`。
> 改 turbo 配置或命令前,先读该安装包里的 `docs/README.md`,再读其 `docs/` 下相关页面;注意弃用说明。这些打包的文档与安装版本一致,且无需联网。
> 块由 turbo 在检测到 AI agent 时自动添加并重写;在 turbo 源码中其模板在 `crates/turborepo-cli/src/cli/agent_guidance.rs`。
> 在根 `turbo.json` / `turbo.jsonc` 设 `"agentGuidance": false` 可退出(不会移除已存在的块)。提交时保留此块,避免下次 agent 调用触发 uncommitted change。