# 011 相对上游 super-agent 的改动清单与升级/移植指南

> **基线**：上游 `deepseek-ai/dsh-superagent-prototype`，commit `6337fc8`（2026-10-31，`package.json` 版本 0.1.0）。
> **本 fork**：`D:\Deepseek_Harness\workspace\dsh-superagent-win`，版本 0.1.17，Windows 原生移植 + Router/Execute/Test 三角色 + 每角色模型端点 + 验证门禁 + 本地任务。**GitHub 默认关闭**：默认形态是纯本地任务，不需要 GitHub 账号、仓库、`gh` 或 git worktree；主 profile 的 embedded 模式仅支持本地任务。
> 运行形态：主 profile（3080 左侧栏插件图标）；用 `embedded: true` 防止污染主 Harness 的进程环境（见 §2.2）。
>
> 本文回答两件事：**相对原始 super-agent 改了哪些逻辑**（便于跟进上游更新），以及**怎么复制到另一台电脑**。
> 文件级差异由上游 checkout 与本 fork 逐文件比较生成（排除 `node_modules`、`lib`、`.artifacts`、`*.tgz`，见 §1）。

## 0. 一句话总览

改动分三层，新增文件为主、改动上游文件的集成点有限：

| 层 | 内容 | 主要新文件数 |
| --- | --- | --- |
| **A. Windows 平台层 + 本地模式** | 让上游原型在原生 Windows 上运行：盘符路径、`PATHEXT` 可执行查找、`.cmd` shim、PowerShell 服务脚本、`GH_CONFIG_DIR` 机器人账号、无 `env`/`sh` 的进程启动；**GitHub Source 默认关闭，关闭时不探测 `gh`、不需要仓库，生成纯本地 deployment，worker 的 git 没有 GitHub 凭据** | 6 |
| **B. 角色模型 + 验证门禁（P1/P2）** | 每角色独立 OpenAI 兼容端点与 API Key；独立只读 Test 角色；worker `ready` 报告先过验证，PASS 才交付、FAIL 打回最多 `maxRounds` 轮 | 7 |
| **C. 本地任务（P3）** | `local` Source：面板「新建任务」/本地收件箱目录、本地 JSONL 交付、挂载已有项目目录、默认允许本机操作者 | 3 |

## 1. 差异基线（可重复核对）

**当时的上游基线**：`https://github.com/deepseek-ai/dsh-superagent-prototype`，`main` @ `6337fc8`，版本 `0.1.0`。
本 fork 已初始化 git，便于以后对比：

- 分支 `windows-fork`，最新提交对应 v0.1.17，标签 `fork-v0.1.17`（历史标签 `fork-v0.1.14` / `fork-v0.1.16`）
- `git remote -v` 里有 `upstream → https://github.com/deepseek-ai/dsh-superagent-prototype.git`
- 已执行 `git fetch upstream main`，本地有 `upstream/main`


**当前统计**（v0.1.17，`git diff --stat upstream/main`）：**52 个文件，+3285 / −145 行**。

其中：**20 个新增文件、32 个上游文件被修改、0 个删除**（184 个上游文件保持原样）；
另外 `scripts/dsh-superagent` 仅有文件模式差异（上游 100755 → Windows checkout 100644），内容未改。

> 注意：fork 的 git 历史与上游**不共享祖先**（从上游源码快照开始）。因此直接 `git merge upstream/main` 会变成 unrelated histories 的大合并；推荐按 §6 的“按层对比、手动重放”流程。

## 2. A 层：Windows 平台层

目标：不依赖 WSL/Git Bash，让上游代码在原生 Windows + PowerShell 5.1+ 下运行。

| 文件 | 改动要点 | 为什么 | 上游更新时注意 |
| --- | --- | --- | --- |
| `src/deployment.ts` | ① 绝对路径校验从 `/^\//` 放宽为盘符/UNC/POSIX；② `executable()` 按 `PATHEXT` 枚举 `.exe/.cmd/.bat`；③ `workerProbe()` 用 subprocess 的显式 `PATH` 环境变量代替 `env` 命令；④ git credential helper 的 CLI 路径转成 shell 友好路径；⑤ Windows 上强制要求 `GH_CONFIG_DIR`（否则 `gh` 会以操作者身份行动）；⑥ Feishu bot 解析在 Windows 上直接返回“不支持 Feishu”的 setup 错误 | Windows 没有 `env`、`sh`、`X_OK`；盘符/UNC 不是 `/` 开头；git `!` helper 仍会经过 shell | 这是冲突高发文件。保留：盘符/UNC 正则、PATHEXT 查找、`GH_CONFIG_DIR` 门禁、Feishu 禁用、credential helper 路径转换。上游若重构 Setup/Deployment，先跑 `tests/win-env.spec.ts` 与启动自检 |
| `src/infra/environment.ts` | ① worker shim 由 `#!/bin/sh` 脚本改为 `.cmd` 批处理（`REM` 标记、CRLF、`cmdPath`/`cmdText` 转义）；② `gh` shim 要求并透传 `GH_CONFIG_DIR`；③ `lark-cli`/被禁命令生成批处理 fail shim；④ `verifyEnvironment` 比较 git config 时把 Windows 路径统一成小写正斜杠再比较 | Windows 没有 POSIX shell；`git config` 输出经过 `GIT_CONFIG_GLOBAL` 后路径形式不一致 | 保留 `.cmd` shim 写法和 `GH_CONFIG_DIR` 逻辑；上游若改环境/shim，必须同步改 `.cmd` 分支 |
| `src/executor/dsh.ts` | prepare 命令在 Windows 用 `powershell.exe -NoProfile -NonInteractive -Command`，其他平台用 `sh -c`（其余为扩展改动，见 B/C 层） | Windows 无 `sh`；执行设置里的准备命令按 PowerShell 文档化 | 若上游改 prepare 命令的执行方式，保留平台分支 |
| `src/app/sessions.ts` | ① `SHELL_TOOLS` 按平台取 `pwsh` 或 `bash`，限制工具时只命名本平台有的 shell 工具；② `noWait()` 增加 PowerShell 的 `Start-Sleep`/`sleep` 识别；③ worker 工具白名单同时含两种 shell 名称，由 `workOnly()` 按实际可用过滤 | DSH presets 在 Windows 只装配 `pwsh`；`tools.restrict()` 拒绝不存在的工具名 | 保留平台 shell 常量与 `SLEEP_PWSH`。上游若改注册/限制工具逻辑，重点回归“主 Session 与 worker 能正常启动、不因 restrict 报错” |
| `scripts/dsh-superagent.ps1`（新增） | 服务管理器：start/stop/restart/status、pid 文件、`launch.cmd`、日志与 `.err.log` 写到 `.artifacts/`；用 `Invoke-CimMethod Win32_Process.Create` 启动，以脱离 DSH 工具调用的 Windows Job Object | 上游的 `scripts/dsh-superagent` 是 Python + tmux + fcntl，Windows 不可用；从 DSH 会话里普通 spawn 的常驻进程会被 Job Object 连坐杀掉 | 这是 fork 自有文件，通常不用跟上游；注意 DSH 升级后 Win32_Process/launch 方式仍有效 |
| `docs/009-windows.md`（新增） | Windows 与上游差异、依赖、账号模型、服务脚本、`GH_CONFIG_DIR` | 文档 | 上游更新后补“新版本差异”段 |
| `tests/win-env.spec.ts`（新增） | Windows 环境适配的单元测试 | 防止路径/PATHEXT/显式 env 回归 | 保持通过 |

### 2.1 本地模式（GitHub 关闭时的默认形态）

v0.1.16 起，这个 fork 默认不做 GitHub 操作。对应改动：

| 文件 | 改动要点 |
| --- | --- |
| `src/sources/registry.ts` | Source 实例没有显式 `enabled` 时，GitHub 默认 **off**（其他 Source 默认 on）。新装即本地模式；已有 stateDir 会保留之前存下的开关，可在「设置 → 来源」里手动关掉 GitHub |
| `src/deployment.ts` | `Deployment` 增加 `localOnly?: true`，`localDeployment()` 生成空仓库/空账号的 deployment |
| `src/app/setup.ts` | GitHub Source 关闭时走 **本地分支**：不 probe `gh`、不要求仓库/`GH_CONFIG_DIR`；仍写 worker shim 和隔离 git config，再应用 worker 环境；Feishu 仅在显式启用时验证 |
| `src/infra/environment.ts` | 无 GitHub 时 `gh` shim 直接失败（“GitHub is off in local mode”），被禁命令提示改成不带账号版本，worker 的 `worker-gitconfig` **没有 GitHub credential helper**（git 不能推送/拉取私有仓库），`verifyEnvironment` 跳过凭据检查；有 GitHub 时行为与之前一致 |
| `src/app/http.ts` + `client/deployment.tsx` | `/state` 的 `accounts` 增加 `localOnly`；面板「部署」页显示“本地模式（GitHub 关闭）”及说明，不再显示空仓库/空账号 |
| 执行设置（数据） | 不再需要 prepare 命令；本机已把它清空。工单目录就是普通文件夹，不再运行 git worktree/clone，也没有 PR/branch 交付 |
| `src/app/sessions.ts` | 本地任务 worker 提示词追加 `LOCAL_TASK_NOTE`（无 GitHub/PR/远端会话；直接改目录并报告） |
| `src/deployment.ts` + `src/app/setup.ts`（v0.1.17） | 新增机器配置 `embedded: true`：插件跑在主 DSH profile 内时，**不把 worker shim/agent git config 应用到共享进程环境**（否则主 Harness 所有 Session 的 `gh`/git 都会变成 agent 的隔离环境）；embedded 模式只支持本地任务，GitHub Source 打开会得到 setup 错误。新增测试 `tests/setup-local.spec.ts` |

**本地模式下的行为**
- 任务创建：面板「新建任务」（可选已有项目目录）或收件箱目录；全部走 Router → Ticket → Worker → Test。
- 交付：写到本地 JSONL outbox（`<stateDir>/local/outbox.jsonl` 或 `<inboxDir>/outbox.jsonl`）。
- 工作区：给了 `directory` 就直接用它（`execution.mounted`，不运行 prepare/git 检查）；没给就在执行设置的目录根下新建普通文件夹。
- 不需要 `gh` 登录、不需要 `GH_CONFIG_DIR`、不需要仓库；服务启动脚本的 `-GhConfigDir` 可继续传（本地模式下不用它）。
- 当前主 profile 使用 `embedded: true`，只支持本地任务；不要打开 GitHub Source。

**本机验证（v0.1.16）**
- `GET /state`：`accounts.localOnly=true`、`runtime/local/model-endpoints=ready`、`github=paused Off`。
- worker 环境：`gh.cmd` 提示 “GitHub is off in local mode”；`worker-gitconfig` 无 `[credential "https://github.com"]`。
- 本地任务 ticket `65093167…`（**不指定项目目录**）：worker 在普通目录创建 `local-ok.txt` → `working/testing` → PASS → ended；报告写“无 GitHub 仓库/PR，无需 PR”，outbox 收到 ack+reply。这验证了不再需要 git worktree/prepare/PR。

### 2.2 主 Harness 插件模式（本机当前）

dsh-superagent 是标准 DSH bundle/插件：服务端注册 orchestrator，客户端在主 Harness 左侧栏注册 `sidebar.panellist` 图标（锤子）和面板页。插件运行在主 Harness 进程内，入口为 `http://127.0.0.1:3080` 左侧栏的 Superagent 图标。

**本机安装与配置（2026-11-01 切换并已热加载生效）**：

1. `dsh plugin --profile web add file:...dsh-superagent-0.1.16.tgz` —— 主 profile 的 `package.json` 增加依赖，并把 `dsh-superagent` 加进 `dsh.profile.bundles`；
2. `%USERPROFILE%\.dsh\profiles\web\cordis.patch.yml` 末尾追加：
   ```yaml
   - id: dsh-superagent
     config:
       stateDir: C:/Users/hangi/.dsh/state/superagent
       embedded: true
   ```
   `stateDir` 保存历史工单、设置与模型端点。`embedded: true` 是 v0.1.17 新增：主 profile 与其它插件共享同一个 DSH 进程，**不能再把 worker shim/agent git config 写进进程环境**，否则主 Harness 的所有会话都会被换成 agent 的 `gh`/git；embedded 模式因此只做本地任务，不支持 GitHub Source。

**生效方式与结果（v0.1.17 已在本机生效，无需再重启）**：
- 安装 bundle + 修改 profile patch 后，运行中的主 Harness **热加载**了插件；`http://127.0.0.1:3080/orchestrator/api/state` 返回 200；
- `embedded: true` + 新版已生效：主进程环境已恢复干净（`GIT_CONFIG_GLOBAL` 为空，`PATH` 不再包含 superagent 的 `worker-bin`）；
- 通过 3080 API 跑了一个本地任务 ticket `f86a3779…`（不填项目目录）：`working/testing → PASS → ended`，`main-ok.txt` 创建成功，outbox 收到 ack+reply——主界面模式端到端可用；
- **入口**：浏览器打开/刷新主 Harness（`http://127.0.0.1:3080`）后，左侧栏应出现 Superagent 图标（锤子）；若页面是插件热加载前打开的，按 F5 刷新一次。若刷新后仍未出现，关闭并重新运行 `start_dsh.bat` 一次（bundle 图在进程启动时必定加载）。

**重启后验证清单**（若执行了重启）：
- 主 Harness 左侧栏出现 Superagent 图标；点开概览看到「本地任务 / 三个 Agent 的模型」；
- `http://127.0.0.1:3080/orchestrator/api/state`（带有效登录）返回 `runtime=ready`、`accounts.localOnly=true`；
- 设置 → 部署显示「本地模式（GitHub 关闭）」；设置 → 模型三张卡正常。

## 3. B 层 P1：每角色模型端点

**目的**：Router / Execute / Test 三个 Session 可以分别接不同 URL、模型、API Key（例如便宜模型跑执行、强模型做验证）。

**新增**
- `src/infra/openai-adapter.ts`：OpenAI 兼容 `/chat/completions` 适配器（流式文本/推理/工具调用，无第三方依赖），provider 前缀 `dsh-superagent-`。
- `src/app/model-endpoints.ts`：`Secrets`（`<stateDir>/secrets.json`，0600，只暴露 `hasKey`）、`ModelEndpoints`（保存时注册/注销 adapter、连接测试）、`roleModelFor(settings, role)`、`MODEL_ROLES`。
- `src/client/models.tsx`：面板「设置 → 模型」三张卡（Router / Execute / Test）：URL、模型、服务商、推理强度、API Key、测试连接、清除 Key。

**修改上游文件的集成点**
- `src/policy.ts`：`ModelEndpointSchema`、`ModelsConfigSchema`、`SettingsSchema.models`、`SettingsBlock` 增加 `'models' | 'test'`；`TestConfigSchema`。
- `src/app/settings.ts`：`BLOCKS` 加 `models`/`test`；`saveModels()`；`syncModels()`；`panelOnlyModels` 校验；settings view。
- `src/app/http.ts`：`SettingsBlockSchema` 加 `models`/`test`；PUT `settings/:block` 路由到 `saveModels`；`POST models/test`；`HttpHost.modelEndpoints`。
- `src/app/api.ts`：`POST models/test` 契约。
- `src/app/runtime.ts`：创建 `ModelEndpoints`，注入 Settings/httpHost，`initialize()` 调 `syncModels`，shutdown 释放。
- `src/app/sessions.ts`：`ensure()` 用 `roleModelFor(settings, role)` 初始化角色模型。
- `src/app/views/settings.ts`、`src/app/settings-history.ts`、`src/core/errors.ts`（`panelOnlyModels`）。
- `src/client/settings.tsx`、`route.ts`、`locales.ts`：设置分组「模型」与文案。
- `tests/fixtures/panel-server.ts`：settings view/路由 mock。

**行为/不变量**
- URL 非空 → 注册 provider `dsh-superagent-<role>`；URL 为空 → 沿用 `provider`+`model` 覆盖或 DSH 默认模型。
- Key 只存 `secrets.json`，API 只返回 `hasKey`；不写日志、不进提示词。
- 改配置只影响**之后新建**的 Session；健康项 `model-endpoints` 汇总状态。
- 面板入口：**设置 → 模型**；概览页也有「三个 Agent 的模型」摘要卡（v0.1.14 增加）。

## 4. B 层 P2：独立 Test 角色与自动验证门禁

**目的**：执行器自报完成不算数，由独立只读的 Test Session 检查；不通过打回去改，最多 N 轮，避免“假验证”。

**新增**
- `src/app/tester.ts`：验证者系统提示词、`testerBrief`、失败反馈文本。
- `src/app/test-gate.ts`：`TestGate`：`shouldVerify` / `deferReady` / `start` / `resume` / `submit` / `cancelAll`。
- `src/app/tools/tester.ts`：Test Session 的只读工具 + `tester_verdict {pass, summary, evidence, issues}`。
- `src/client/test.tsx`：面板「设置 → 验证」页（开关、最多轮数、单轮超时、团队验证说明）。
- `tests/verification.spec.ts`：PASS 放行、FAIL 打回、轮数用尽如实交付。

**修改上游文件的集成点**
- `src/app/tools/executor.ts`：`orchestrator_report state=ready` 时先 `testGate.deferReady()`；返回 `verification round` 并异步 `start()`，不再直接 relay。
- `src/app/runtime.ts`：创建 `TestGate`、`testerToolHost`、tester 工具注入；启动时 `testGate.resume()`；停机 `cancelAll()`；shutdown 包含 `tester-<ticket>` Session。
- `src/app/sessions.ts`：`SessionRole` 增加 `tester`；`TESTER_PRESET`；`ensure()` 选 preset/模型；`decorate()` 识别 `tester-<ticketId>`、用 `testerSystemPrompt()`、注册 `host.tools(agent, ticket, role)`、限制为只读工具（`read/read_image/glob/grep/pwsh|bash/job_*/skill`，**无 write/edit**）。
- `src/core/model.ts`：`Ticket.testing { round, startedAt, report }`、`Ticket.testRounds`。
- `src/core/stage.ts`：`working` action 增加 `testing`。
- `src/policy.ts`、`src/app/settings.ts`、`http.ts`、`api.ts`、`views/settings.ts`、`settings-history.ts`：`test` 设置块与保存/历史。
- `cordis.patch.yml`：新增 agent preset `dsh-superagent-tester`（复制 worker preset，只读由代码限制）。
- `src/client/marks.ts`、`route.ts`、`settings.tsx`、`locales.ts`：`stage.testing` 显示「验证中」、设置页路由与文案。
- `src/app/http.ts`：`SettingsBlockSchema` 必须加 `test`（否则 `PUT /settings/test` 会 400，本机踩过）。

**门禁流程与不变量**
1. worker 调 `orchestrator_report state=ready` → 门禁在 `store.atomic` 内记录 `ticket.testing`、`testRounds`，**不**调用 `engine.report`。
2. 异步启动 `tester-<ticketId>` Session，prompt 包含 request、环境、团队验证说明与轮次。
3. `tester_verdict` → `TestGate.submit`：
   - PASS：把 worker 原报告交给 `engine.report`（ready 交付），清 `testing`。
   - FAIL 且 `round < maxRounds`：清 `testing`，向 worker Session 发失败反馈（队列消息），worker 继续改。
   - FAIL 且轮数用尽：仍按 ready 交付，但 summary 明确写 `Verification failed after N round(s): ...`。
   - 超时/验证者启动失败按 FAIL 处理。
4. `ticket.testing` 持久化；重启后 `TestGate.resume()` 重发该轮；停机 `cancelAll()`。
5. Test Session 工具只读；tester 会话 id 固定复用；prompt 不接受团队直接覆盖（团队说明走 `test.instructions`）。

**本机实测证据**：本地任务 `27dc5da3…` PASS 交付；`ce9063c7…` 第 1 轮 FAIL → worker 补 `verified.txt` → 第 2 轮 PASS；`56878607…` 本地提示词回归 PASS。

## 5. C 层 P3：本地模式（`local` Source）

**目的**：不依赖 GitHub 评论也能派活：面板「新建任务」或本地收件箱；结果写本地 outbox；可挂已有项目目录。

**新增**
- `src/sources/local.ts`：Source kind（`inboxDir`/`pollSeconds` 配置、收件箱轮询、本地 JSONL Channel、threadContext、Policy 默认 `operator`、环境变量 `DSH_SUPERAGENT_LOCAL_TASK`）+ `localTaskTrigger()` + 插件入口 `dsh-superagent/local`。
- `src/client/local.tsx`：概览「新建任务」对话框（标题/任务内容/可选项目目录）。
- `tests/local.spec.ts`：Trigger 形状、默认策略、outbox 写入。

**修改上游文件的集成点**
- `cordis.patch.yml`：新增 `dsh-superagent-local`（`dsh-superagent/local`）。
- `package.json`：`exports["./local"]`；`tsdown.config.ts`：增加 `local` 入口。
- `src/app/http.ts`：`POST local/tasks` 路由；`HttpHost.receive()` 与 `HttpHost.localContext()`；首次创建任务时自动把 `operator` 补进 local Policy。
- `src/app/api.ts`：路由契约。
- `src/app/runtime.ts`：httpHost 注入 `receive`/`localContext`；`passes.receive` 复用现有 Eval→Ticket 流程。
- `src/executor/dsh.ts`：`ticketExecution(..., trigger?)` 读取 `trigger.meta.directory` → `execution.directory` + `execution.mounted`；`prepareDirectory` 对 mounted 目录跳过 prepare 与 git 检查。
- `src/core/model.ts`：`Execution.mounted?: true`。
- `src/app/sessions.ts`：本地任务 worker 提示词追加 `LOCAL_TASK_NOTE`（无 GitHub/PR/远端会话，直接改目录并报告）。
- `src/client/overview.tsx`、`locales.ts`：概览「本地任务」入口。

**行为/不变量**
- 面板任务 outbox：`<stateDir>/local/outbox.jsonl`；收件箱任务 outbox：`<inboxDir>/outbox.jsonl`。
- 收件箱：`*.task.json`（`{title,request,directory}`）或 `*.md` → 接收后改名 `*.done`。
- local Policy 默认 `allowed: [{id:'operator'}]`；面板入口首次调用会补齐。
- 本地任务仍走 Router→Ticket→Worker→Test；默认不带 GitHub 交付。
- **限制**：当前若 GitHub Source 保持启用，服务整体仍要求 GitHub setup（仓库 + CLI）；纯本地无 GitHub 的 setup 解耦尚未实现。

## 6. 上游更新（同步/重放）操作手册

**推荐流程**：以上游为参照做“分层对比 + 手动重放”，而不是直接 merge 未共享历史的两棵树。

1. **取上游**：
   ```powershell
   cd D:\Deepseek_Harness\workspace\dsh-superagent-win
   git fetch upstream main
   git diff --stat upstream/main HEAD
   git diff --name-status upstream/main HEAD
   ```
   需要逐文件看时：`git diff upstream/main HEAD -- <path>`；上游侧内容也可 `git show upstream/main:<path>`。
2. **先处理上游改动，再处理 fork 改动**：把上游新版本拷贝/checkout 到临时目录，按 §6.1 的分类逐文件判断。
3. **按层处理冲突**：
   - **A 层（Windows 平台 + 本地模式）**：保留 fork 的平台实现（盘符/PATHEXT/`.cmd` shim/`GH_CONFIG_DIR`/PowerShell prepare/平台 shell 工具/Win32_Process 服务脚本）以及本地分支（GitHub 关闭时无 probe、无凭据的 worker git、`localOnly` deployment）。若上游重构了相同函数，把上游新语义移植进 fork 的平台/本地分支。
   - **B/C 层新增文件**：大多与上游无关，直接保留；仅在 import/类型签名变化时适配。
   - **B/C 层对上游文件的集成点**：按 §3–§5 的“修改上游文件的集成点”重新应用；这些点很小但漏一个就编译不过或门禁失效。
4. **每次必看的文件清单（冲突热点）**：
   `src/deployment.ts`、`src/app/setup.ts`、`src/sources/registry.ts`、`src/infra/environment.ts`、`src/executor/dsh.ts`、`src/app/sessions.ts`、
   `src/policy.ts`、`src/app/settings.ts`、`src/app/http.ts`、`src/app/runtime.ts`、`src/app/tools/executor.ts`、
   `src/core/model.ts`、`src/core/stage.ts`、`cordis.patch.yml`、`package.json`、`tsdown.config.ts`、`src/client/deployment.tsx`、
   `tests/fixtures/panel-server.ts`、`tests/sources.spec.ts`、`tests/win-env.spec.ts`。
5. **重放后的验收门槛**：
   ```powershell
   pnpm install
   pnpm typecheck
   pnpm exec vitest run tests/local.spec.ts tests/verification.spec.ts tests/github-read.spec.ts tests/win-env.spec.ts
   pnpm build
   pnpm pack
   dsh plugin --profile web add file:.\dsh-superagent-<ver>.tgz
   ```
   重启主 Harness，然后人工/脚本验证：`runtime=ready`、`local=ready`、概览三个模型卡、设置→模型三张卡、设置→验证、本地任务跑一次 PASS。
6. **回滚/对照**：每次升级前 `git tag fork-vX.Y.Z`；出问题 `git switch windows-fork && git reset --hard fork-vX.Y.Z`（注意 stateDir 数据库独立于代码，回退代码不会自动回退 settings schema；本次扩展字段均可选、无 schema 版本变化，见 §7）。
7. **版本号**：每次打包前改 `package.json` 的 `version`（DSH plugin manager 按包版本识别更新）。

**为什么不直接 merge**：fork 从上游源码快照开始，没有共同祖先；`git merge --allow-unrelated-histories` 会把整个文件树当冲突处理。若坚持 merge，建议先用上游历史建一个 `upstream-main` 分支（`git fetch upstream main` 已有 `upstream/main`），再在该分支上新建“干净上游”提交做三方合并；对本 fork 当前规模，逐文件重放更可控。

## 7. 移植到另一台 Windows 电脑（从零）

**前提**：Windows 10/11、Node.js 22.19+ 或 24+、pnpm 11+、PowerShell 5.1+。git 2.32+ 与 GitHub CLI 2.48+ **只在需要 GitHub Source 时才需要**；纯本地模式不需要仓库、`gh` 或机器人账号（Feishu 不支持）。

1. **复制代码并安装依赖**：
   ```powershell
   cd D:\Deepseek_Harness\workspace\dsh-superagent-win
   pnpm install
   pnpm typecheck
   pnpm build
   pnpm pack
   ```
2. **安装 bundle 到主 profile**（配置详见 §2.2）：
   - 安装到主 profile（`web`；profile 名以 `%USERPROFILE%\.dsh\profiles\` 下实际为准）：
     `dsh plugin --profile web add file:.\dsh-superagent-<ver>.tgz`
   - 在 `...\profiles\web\cordis.patch.yml` 末尾加：
     ```yaml
     - id: dsh-superagent
       config:
         stateDir: C:/Users/<me>/.dsh/state/superagent
         embedded: true
     ```
3. **启动主 Harness**：关闭并重新运行 `start_dsh.bat`，或按现有方式重启主 Harness；左侧栏出现 Superagent 图标即成功。
4. **GitHub 说明**：主 profile 的 embedded 模式只支持本地任务，不要启用 GitHub Source；无需配置 `GH_CONFIG_DIR`、仓库或 GitHub CLI。
5. **面板配置（本地模式）**：
   - 设置 → 模型：需要独立端点的角色填 URL/模型/Key，点「测试连接」。
   - 设置 → 验证：开关、最多轮数、超时、团队验证说明。
   - 概览 → 新建任务（可填已有项目目录）；设置 → 来源 → 本地任务：收件箱目录、轮询间隔。
   - 设置 → 部署：应显示 **本地模式（GitHub 关闭）**。
6. **仓库外的机器级集成**：主 Harness 继续通过现有 `start_dsh.bat` 启动，无需额外的 superagent 服务或面板快捷方式；入口是 3080 左侧栏的 Superagent 图标。
7. **smoke 验收**（在主 profile 的 3080 UI/API 上进行）：
   - `GET /orchestrator/api/state`：`runtime=ready`、`local=ready`；`github=paused Off`（本地模式）。
   - `setup.accounts.localOnly=true`；面板「部署」页显示本地模式。
   - 概览：三个模型摘要卡可见；设置 → 模型三张卡可见；设置 → 验证可保存。
   - 新建一个本地任务（**不填项目目录**也可以），确认 `working → testing → ended` 与 outbox 回复，worker 报告里没有 PR 操作。
   - 收件箱：把 `demo.task.json` 放进 `inboxDir`，确认改名 `.done` 并生成 outbox。
8. **迁移注意**：
   - `stateDir` 里是 SQLite/会话/密钥；迁移整个目录等于迁移历史与 API Key（secrets.json）。跨机器可复制，但路径配置仍需改。
   - **不要**复制 `node_modules`、`lib`、`.artifacts`、`*.tgz` 到新机器；重新 `pnpm install`/build/pack。
   - 代码基线用 git：新机器 `git clone` 或复制整个目录后 `git remote set-url upstream ...`，再 `git checkout windows-fork`。

## 8. 本机验证记录（截至 v0.1.16）

| 项 | 结果 |
| --- | --- |
| `pnpm typecheck` / `pnpm build` | 通过 |
| 定向测试 | `tests/local.spec.ts` 3、`tests/verification.spec.ts` 3、`tests/github-read.spec.ts` 7、`tests/win-env.spec.ts` 2、`tests/setup-local.spec.ts` 1，全部通过；`tests/sources.spec.ts` 中 GitHub 用例 65 通过（6 项 Feishu 消费者失败为未移植的 Windows 既存问题） |
| 本地任务 PASS | ticket `27dc5da3…`（hello.txt）、`56878607…`（prompt-ok.txt）：`working → testing → PASS → completed`，outbox ack+reply |
| 验证门禁 FAIL→返修→PASS | ticket `ce9063c7…`：round1 FAIL → worker 补 verified.txt → round2 PASS |
| **纯本地模式（GitHub 关闭）** | `accounts.localOnly=true`、`github=paused Off`；worker 的 `gh.cmd` 失败、`worker-gitconfig` 无 GitHub credential helper；ticket `65093167…` **不填项目目录**：普通目录创建 `local-ok.txt` → `working/testing → PASS → ended`，报告写“无 GitHub 仓库/PR，无需 PR”，outbox ack+reply |
| 面板 | 概览「三个 Agent 的模型」、设置→模型三张卡、设置→验证、概览「本地任务」、设置→部署「本地模式（GitHub 关闭）」；headless Chrome 无 console error；截图 `.artifacts/overview-role-models.png`、`.artifacts/models-three-roles-014.png`、`.artifacts/test-settings-page.png` |
| 服务 | **v0.1.17 本机已切换主界面插件模式并生效**：bundle 装进 `web` profile，`cordis.patch.yml` 配 `stateDir` + `embedded: true`，运行中的主 Harness 热加载插件；3080 `/orchestrator/api/state` 返回 200、runtime ready、localOnly=true、github Off，主进程环境干净（无 worker-bin/GIT_CONFIG_GLOBAL 污染）；通过 3080 API 的本地任务 `f86a3779…` 完成 `working/testing → PASS → ended` + outbox reply |

**已知限制**
- **GitHub 默认关闭**。本地模式下没有仓库/PR/`gh` 操作。
- GitHub 模式仍会跳过与 agent 同账号的消息（`sameName(item.user.login, login)`），所以机器人无法给自己发 GitHub 触发；完整 GitHub E2E 需操作者用主账号发 mention。
- Feishu Source 未移植（上游依赖 `lark-cli` + POSIX shell）。
- 全量 `pnpm test` 在 Windows 有既存 POSIX 夹具失败（service 脚本/chmod/temp 权限/Feishu 消费者），定向测试为准。
- 模型 endpoint 的 adapter 只覆盖 OpenAI 兼容 `/chat/completions`；其他协议需另写适配器。
- 本地模式仍要求 git 2.32+ 吗？**不要求**：GitHub 关闭时 setup 不再运行 `gh`/仓库检查；worker git config 仍会写（隔离），但机器上没有 git 也不影响本地任务（worker 不应调用 git）。
- **主 profile（embedded）模式只支持本地任务**：为了不污染主 Harness 进程环境，embedded 模式不应用 worker shim/git config；因此 worker 会继承主进程的 git/gh 环境，不要启用 GitHub Source。

## 附录 A：本机 profile 配置（可复制后改路径）

**主 profile 插件模式**（本机当前；`%USERPROFILE%\.dsh\profiles\web\cordis.patch.yml` **末尾追加**，不要写 webserver 端口）：

```yaml
- id: dsh-superagent
  config:
    stateDir: C:/Users/<me>/.dsh/state/superagent
    embedded: true
```

安装：`dsh plugin --profile web add file:.\dsh-superagent-<ver>.tgz`；重启主 Harness 后图标出现在 3080 左侧栏。`embedded: true` 让插件不修改主进程环境（主 profile 模式只支持本地任务，不支持 GitHub Source）。配置细节见 §2.2。

## 附录 B：配置入口速查（都在面板 UI，不用改 YAML）

| 想改什么 | 入口 |
| --- | --- |
| Router/Executor/Tester 的 URL、模型、API Key、测试连接 | 设置 → 模型（概览也有摘要与「配置模型」按钮） |
| 验证门禁开关、最多轮数、超时、团队验证说明 | 设置 → 验证 |
| **GitHub 状态（默认关 = 纯本地模式）** | 设置 → 来源 → GitHub；主 profile 的 embedded 模式不要开启 |
| 查看当前是本地模式还是 GitHub 模式 | 设置 → 部署（仓库与账号 一栏） |
| 本地任务创建 | 概览 → 本地任务 → 新建任务 |
| 本地收件箱目录/轮询间隔 | 设置 → 来源 → 本地任务 |
| 谁能发起工作、评估模式、接收矩阵 | 设置 → 评估 |
| 团队提示词分节 | 设置 → 提示词 |
