# 《DeepSeek Harness》深度解读 · 分篇 04：工具 seams 与执行管道

- 仓库：deepseek-ai/deepseek-harness / 子类型：monorepo（pnpm workspaces，ESM，strict TS）
- 版本锁定：deepseek-ai/deepseek-harness **master 分支 tarball 快照，2026-08-14 下载**；快照不含 git 元数据，**无 commit hash 可记录**（材料降级，特此声明）；替代锚点为根 `package.json` 版本 `0.1.0-rc.5`。文中全部 `file:line` 锚点以该快照的文件系统状态为准。
- 版本锁定补充锚点：快照根 `package.json` 的 SHA-1 为 `16918dcd04afbb809d896844bd04c3cb3bd3d301`（在快照根执行 `shasum package.json` 可复算）。
- 材料范围：map+采样。本篇主线链（shell → bash-local/bash-sandbox → subprocess → sandbox → sandbox-policy → interaction）精读至函数级；平行 seams（fs/terminal/lsp/skill/web/code-runtime/mcp/guard/spill/e2b）签名级扫读 + 各挑一处对照；`packages/core/tools` 的注册表内部在第 02 篇已拆，本篇只引用其管道段落。
- 读者画像：已读第 01 篇（Cordis 底座：ctx/service/effect/waterfall）与第 02 篇（核心 spine：turn/step、SessionEvent 日志、scoped 工具注册表）的工程师。

## 0. 阅读地图（本篇导览）

第 02 篇把一轮 turn 讲到了"模型在 assistant 消息里给出一个 tool_use 块"为止。本篇接住这个悬而未决的瞬间：这个 tool_use 块如何变成一个真实的操作系统进程，它的输出又如何变回一条模型可见的 `tool/result`。主锚点是一句话——**模型决定执行 `npm test`**。全文走两条路：先沿 bash 这条主链端到端走一遍（工具注册表的三条瀑布、approval 接缝、sandbox 包裹、subprocess 诞生进程），再退后一步，用一张对照表看清 shell/fs/lsp/web/skill 等十几个能力 seam 如何重复同一个"三角色"范式——这才是 dsh"本地换沙箱换远程"宣称的真正载体。

## 1. 背景与前置知识

第 01 篇给的 Cordis 词汇（`ctx`、service、effect、waterfall listener、注册即 effect）和第 02 篇给的核心词汇（turn/step、`tool/call` 与 `tool/result` 事件、scoped 工具注册表 `ctx.tools`、model-visible ⟺ logged）本篇直接使用，不再定义。本篇新增四个垫层概念，逐个进场。

**capability seam 的三角色**（第 01、02 篇已用，这里固定为本篇的组织轴）。dsh 把一个可替换能力拆成三个包角色：**Service Definition**（声明抽象服务与词汇，如 `ctx.shell` 的契约）、**Service Provider**（实现并注册该服务，如本地 bash 执行器）、**Consumer**（消费该服务的插件，通常是模型-facing 的工具）。仓库 AGENTS.md 把这条写成硬规则：一个 seam 永远三个角色齐全，"split only when roles evolve independently"。本篇的全部"对象全景"就是这条规则在十几个能力上的重复举证。

**request/spec split（请求/规格分离）**。dsh 的包边界纪律"Explicit > implicit"：默认值填充不许藏在执行函数里写成 `?? default`，必须是边界上一个显式的 `resolve(request): Spec` 步骤。调用方递一个字段可缺的 **request**， owning 实现把它解析成全字段必填的 **spec**，执行方法只收 spec。仓库根 AGENTS.md 点名 `dsh-shell` 是这个模板的出处（`packages/subprocess/subprocess/src/types.ts:69-74` 的注释也自称以它为模板）。第 3.7 节拆它的代码形态。

**SandboxMode（沙箱模式）**。dsh 的文件效果政策词汇，封闭三值：`read-only`（只允许 `/dev/null` 这类必需汇聚点）、`workspace-write`（可写工作区加平台临时区）、`danger-full-access`（不围栏）。注意它的管辖范围刻意很窄：网络与进程可见性不在这个词汇里（`packages/sandbox/sandbox/src/index.ts:23-29`）。一次调用跑在哪个模式下，由每次调用携带的 `SandboxExecutionPolicy` 决定——**per call，而非 provider 上固定**（`packages/sandbox/sandbox/src/index.ts:61-68`），所以同一时刻 bash 可以 confined 而一个子 agent 的状态目录可写。

**ApprovalOutcome（审批结果）**。审批接缝的封闭四值：`allowed-once`（唯一算"批准"的值）、`rejected`、`cancelled`、`unavailable`。后三个都导致拒绝，但文案各异，模型分得清"人说了不"和"没有审批通道"（`packages/interaction/user-approval/src/index.ts:81-82`、`packages/core/tools/src/index.ts:1686-1688`）。

**子锚点**。本篇在主锚点"模型决定执行 `npm test`"下设一个子锚点：**默认 CLI 组合**（`packages/bundle/base/cordis.patch.yml` 描述的那棵树）。每当叙事出现"这个插件在不在场"，默认答案以这个组合为准：POSIX 上挂 `bash-sandbox` + `tool-bash`，Windows 上挂 `pwsh-sandbox` + `tool-pwsh`，全平台共享 `sandbox-local` + `sandbox-policy` + `user-approval` + `permission-presets`（`packages/bundle/base/cordis.patch.yml:163-216`）。

## 2. 对象全景：执行类 seams 地图

上一篇的地图是 turn 的时间轴；本篇的地图是同一种空间结构的复制粘贴。先把 dsh 执行类能力按三角色摆成一张表——这是本篇的承重图，后面主线叙事里每个模块第一次出现时，回到这张表找它的格子。

| seam | ctx key | Service Definition | Service Provider | Consumer | 合并了角色的包 |
|---|---|---|---|---|---|
| shell（bash 执行） | `ctx.shell` | `shell/shell` | `shell/bash-local`、`shell/bash-sandbox`、`shell/pwsh-local`（+ pwsh-sandbox） | `shell/tool-bash`、`shell/tool-pwsh`、两个 hooks 桥 | — |
| subprocess（进程树） | `ctx.subprocess` | `subprocess/subprocess` | `subprocess/subprocess-local`、`e2b/subprocess-e2b` | bash-local、bash-sandbox、terminal-bash、lsp-stdio、三个进程外 subagent 后端 | — |
| sandbox（进程围栏） | `ctx.sandbox` | `sandbox/sandbox` | `sandbox/sandbox-local` | bash-sandbox、terminal-bash | — |
| sandbox 政策 | `ctx.sandboxPolicy` | `sandbox/sandbox-policy`（core 角色） | 同左（一个包） | bash-sandbox、fs-sandbox、terminal-bash | Definition=Provider=政策唯一 home |
| fs（文件系统） | `ctx.fs` | `fs/fs` | `fs/fs-local`、`fs/fs-sandbox`、`e2b/fs-e2b` | `fs/tool-fs` | — |
| terminal（持久 PTY） | `ctx.terminals` | `terminal/terminal` | `terminal/terminal-bash` | `terminal/tool-terminal`、`shell/tool-bash-persistent` | — |
| lsp | `ctx.lsp` | `lsp/lsp` | `lsp/lsp-stdio` | `lsp/tool-lsp` | — |
| skill | `ctx.skills` | `skill/skill` | `skill/skill-filesystem`、`skill/skill-badge` | `skill/tool-skill` | — |
| web | `ctx.web` | `web/web` | `web/web-search-*`（三个）、`web/web-fetch-http` | `web/tool-web` | — |
| code-runtime | `ctx.codeRuntime` | `code-runtime/code-runtime` | `code-runtime/code-runtime-worker-thread` | `core/tools`（Code Mode 的 run_code 传输） | — |
| approval | `ctx.approval` | `interaction/user-approval` | 同左（服务自带 answerer 瀑布；ACP 桥注册机器 answerer） | `core/tools`、`shell/tool-bash` | Definition=Provider 合一，answerer 靠 listener 扩展 |
| spill（结果外溢存储） | `ctx.spillStore` | `spill/spill` | `spill/spill-local` | `spill/spill-policy` | — |
| mcp（外部工具接入） | 无 | — | — | 桥接进 `ctx.tools` | 反例：三角色全不合，见 8.1 |

（来源：`docs/capability-seams.md:447-461` 各行 + 各包模块头；生成表自带完整性守卫。）

三个读图要点。第一，**Definition 包几乎不依赖任何人，Provider 依赖 Definition，Consumer 也依赖 Definition**——替换 Provider 时 Consumer 一行不改，这就是"seam"一词的字面意思：换 `bash-local` 为 `bash-sandbox`，`tool-bash` 无感知。第二，**有的 seam 把角色合进一个包**：`sandbox-policy` 是部署政策的唯一 home（core 角色而非可替换 seam），`approval` 的 Definition 与 Provider 合一、靠 `approval/request` 瀑布留扩展口。合并的理由都一样：这些角色没有独立演化的需求。第三，**consumer 列出现了别的 seam 的 provider**：bash-local 是 shell 的 Provider，同时是 subprocess 的 Consumer——seam 之间可以叠层，shell 叠在 subprocess 上，sandbox 包裹在最外层。这条叠层链就是主锚点 `npm test` 要走的路。

## 3. 主线叙事 S2：一次 `npm test` 从 tool_use 到进程诞生

第 2 节给了空间地图，但一张表看不出旅程。本节沿主锚点走时间轴：模型在某个 step 的 assistant 消息里给出了 `bash` 工具的 tool_use 块，参数是 `{command: "npm test", description: "Run package test suite"}`。控制流与数据流双通道并行，沿途每个 seam 第一次登场时给 chunk 标题。

### 3.1 入口：tool/call 已落日志，执行尚未开始

第 02 篇讲过：agent-loop 在解析 assistant 消息时先向会话日志 append `tool/call`（**先于任何执行**），然后把调用交给 `ctx.tools` 的执行调度器。本篇从这里接手。调度器的第一步是 `prepareExecution`（`packages/core/tools/src/index.ts:1463-1507`）：为这次调用造出执行上下文 `exec`（含参数、`exec.agent`、`exec.signal`），然后进入三条瀑布的第一条。

### 3.2 tools/pre-execute：政策瀑布（chunk：什么都还没执行，政策先表态）

`tools/pre-execute` 是一条 waterfall（`packages/core/tools/src/index.ts:1475-1478`）：每个 listener 拿到 `exec`，必须 `next()` 委托，最终汇成一个 `PreToolDecision`——`allow`、`deny`（带理由）或 `ask`（带理由）。它是**可扩展政策层**：谁想在执行前说话就挂在这里。全仓生产代码里的注册点（静态推断，grep 注册拓扑确证）：`packages/hooks/hooks-claude-code/src/index.ts:238`、`packages/hooks/hooks-codex/src/index.ts:225`（两个 hooks 桥把外部钩子协议接进来）、`packages/jobs/tool-jobs/src/index.ts:233`（job 工具的守卫）。默认 CLI 组合里如果没有配置 hooks，这条瀑布直接落到默认的 `{kind:'allow'}`——我们的 `npm test` 在多数部署下在这里无人阻拦。

listener 之间的相对顺序由 Cordis waterfall 语义决定（注册顺序，`prepend` 可插队，见第 01 篇）；哪些 listener 在场由组合层决定。所以对"approval 在第几个"这类问题，代码给出的答案不是一个数字，而是一个拓扑——这是本篇对导读问题 1 的直接回答。

### 3.3 ask 分支：approval 不是瀑布上的 listener（chunk：唯一能在执行前问人的接缝）

如果某个 pre-execute listener 回了 `ask`，注册表并不自己去弹窗，而是调用 `serviceAsk`（`packages/core/tools/src/index.ts:1689-1721`）：`ctx.get('approval')` 拿不到服务就直接 deny；拿到就调 `ctx.approval.request(...)`，把四值结果映射回 allow/deny。注意这里的结构：**approval 是一个独立 seam，不是挂在 `tools/*` 事件上的 listener**——`docs/tool-execution-pipeline.md` 的 mermaid 图把 approval 画成管线中的一站，读图容易以为它是个 listener，代码实证它是被注册表主动调用的服务（静态推断，证据：`packages/core/tools/src/index.ts:1479-1481` 与 `1689-1721`；第二条调用路径在工具体内部，见 3.6）。

`ApprovalService.request` 的行为（`packages/interaction/user-approval/src/index.ts:257-276`）：先检查会话日志当前处于一个 open turn 内（审计事件对必须被 turn 边界包住，否则崩溃恢复后无法与崩溃尾巴区分），然后 append `approval/asked`，再走 `decide`，最后 append `approval/decided`——**每一问一答都在日志里留下审计对**，这正是 model-visible ⟺ logged 纪律在审批上的投影。`decide` 内部（同文件 `304-344`）：会话政策为 `never` 时在服务自有路径上直接 `rejected`（注释解释了为什么这个判定不能做成 listener——prepend 的 listener 会绕过它）；政策为 `ask` 时派发到 `approval/request` 瀑布，由组合的 answerer（ACP 桥的机器应答、UI 的人工应答）认领；没有 answerer、answerer 抛错、answerer 返回词汇表外的值，全部归一到 fail-closed 的 `unavailable`；调用方 abort 则竞速出 `cancelled`。

### 3.4 单调 guards：只能踩刹车的检查器（chunk：owner 政策的最后 veto 点）

pre-execute 判定为 allow 之后，执行前还有一道闸：单调 guards（`packages/core/tools/src/index.ts:1486-1488` 调用 `guardReason`，注册接口在同文件 `1101-1124`）。guard 是一个同步函数，返回一个字符串即拒绝，返回 `undefined` 即弃权——**它能拒不能放**：没有任何 guard 可以强行放行另一个 guard 拒绝的调用（"monotonic"的含义）。全局层先查，再沿 agent scope 链查。这是给"不得被重新排序的 owner 政策"留的位置；与 pre-execute 的分工是：可扩展、可协商的政策走瀑布，一刀切的安全闸走 guard。

### 3.5 tools/execute：围绕派发的外层包装（chunk：deadline 在这里套上）

过了闸，`dispatchScheduledExecution` 发起第二条瀑布 `tools/execute`（`packages/core/tools/src/index.ts:1573-1576`），它的 `next()` 末端是真正调用工具 `execute()` 的 `dispatchToolBody`。这条瀑布是 **around 语义**的经典位置：listener 可以在 `next()` 前后包东西。生产注册点：`packages/guard/timeout-policy/src/index.ts:56`（工具超时）与 `packages/session/session-checkpoint-policy/src/index.ts:70`。

timeout-policy 值得停一秒（`packages/guard/timeout-policy/src/index.ts:56-78`）：它读出工具定义上声明的 `timeoutMs`（**超时是部署策略，不是模型参数**——`packages/web/tool-web/src/fetch.ts:1-5` 的模块头把这条写明了），用 deadline 库把超时与上游取消融合成新 signal 临时换到 `exec.signal` 上，`next()` 后恢复；如果响的是自己的计时器，就把工具返回的任何结果替换成结构化的 `TOOL_TIMEOUT` 错误。工具协作契约是：声明 `timeoutMs` 的工具必须承诺尊重 `exec.signal`。bash 工具不在这条路径上声明超时——bash 的超时由执行器自己在更底层实现（3.9），两者不冲突。

### 3.6 工具体：tool-bash 的 execute（chunk：模型参数在此变成执行请求）

瀑布末端进入 `bash` 工具的 `execute`（`packages/shell/tool-bash/src/index.ts:330-390`）。tool-bash 是 `ctx.shell` seam 的 **Consumer**：它不认识 bash-local 也不认识 bash-sandbox，只面对 `ctx.shell` 的契约。它依次做四件事：

1. **校验模型参数**（同文件 `55-68`）：command/description 非空、timeoutMs 为正数、escalation 参数成对（`sandbox_permissions` 与 `justification` 必须同来同去——schema 表达不了这条，所以在 execute 里查）。
2. **解析本调用的沙箱政策**（`199-200`）：如果挂载的执行器会围栏（`ctx.shell.sandboxMode` 非 `undefined`），就调 `ctx.sandboxPolicy.resolve({ session })` 拿到这次调用的完整政策——模式优先级是"显式审批的模式 > 会话日志里最后一条 `sandbox/mode` 事件 > 部署默认"，工作区根取会话的不可变 cwd（`packages/sandbox/sandbox-policy/src/index.ts:135-142`）。这里有个加载期保险：执行器声称围栏但 `ctx.sandboxPolicy` 缺席时，tool-bash 在插件加载时就抛错（`packages/shell/tool-bash/src/index.ts:195-197`）——拼错的组合**死在启动，不死在第一次执行**。
3. **可选的 escalation 审批**（`334-336` → `213-233`）：如果模型带了 `sandbox_permissions`（比如上一次 `npm test` 因写 `node_modules/.cache` 被拒，模型按工具描述里的指引原样重试并申请更宽模式），在**任何东西执行之前**调共享的 `approveEscalation`——严格更宽校验、approval 通道存在性、`ctx.approval.request` 问人，只有 `allowed-once` 返回目标模式，其余路径各抛一句固定文案的错误，这个错误直接成为本调用的 isError 结果。这个制品在第 5.2 节全文拆。
4. **组装 request 并交给 seam**（`341-380`）：`ctx.shellEnv.collect(exec)` 收一份受信的 `DSH_*` 环境快照，然后 `ctx.shell.run(ctx.shell.resolve(request))`——先 resolve 成 spec，再 run。

后台分支（`run_in_background: true`）在同文件 `349-378`：不直接跑，而是把 `ctx.shell.start(...)` 包进 `ctx.jobs` 的一个 job，立即返回 jobId；输出以后用 `job_output` 增量读。这是 bash"一次性执行"与 terminal"持久会话"之间的第三态——有生命周期托管的后台进程。

### 3.7 resolve(request)：request/spec split 的代码形态（chunk：默认值只许活在这里）

`ctx.shell.resolve` 是 shell seam 三抽象方法的第一个（`packages/shell/shell/src/index.ts:85`）。`ShellExecRequest` 的字段几乎全可选（command 之外：workdir、timeoutMs、stdoutMaxBytes、signal、stdin、env、dshEnv、sandboxPolicy），`ShellExecSpec` 则把 workdir/timeoutMs/stdoutMaxBytes/sandboxPolicy 全部变为必填（`packages/shell/shell/src/types.ts:38-110`）。bash-local 的 resolve 实现（`packages/shell/bash-local/src/index.ts:146-171`）：`clampTimeout` 把请求超时夹在配置默认 120 秒与上限 600 秒之间（两个值都是 cordis.yml 可改的 Config 字段，`105-112`），workdir 回填 `config.cwd ?? process.cwd()`，stdin/env/dshEnv 原样透传，sandboxPolicy 也原样透传——**bash-local 自己不围栏，所以政策字段对它是惰性的**；bash-sandbox 覆盖这个方法，在请求没带政策时盖上 `ctx.sandboxPolicy.resolve()` 的部署政策（`packages/shell/bash-sandbox/src/index.ts:84-86`）。为什么值得把"填默认值"单独提成一步？因为默认值是**实现的配置**，不是 seam 的语义——把它留在边界上显式发生，subprocess 这样的下游 seam 才能理直气壮地声明"我不应用任何默认"（`packages/subprocess/subprocess/src/types.ts:69-74`）。第 5.1 节把这对类型连实现一起拆。

### 3.8 sandbox 包裹：argv 在此易容（chunk：bash -c 被包进 runner）

默认 POSIX 组合里，`ctx.shell` 的实现是 `SandboxBashExecutor`——它**继承** `LocalBashExecutor`，注入多了 `sandbox` 与 `sandboxPolicy` 两个服务（`packages/shell/bash-sandbox/src/index.ts:44-45`）。`run` 的路径（同文件 `88-114`）：政策模式是 `danger-full-access` 就直接走父类；否则调 `this.ctx.sandbox.confine(['bash', '-c', command], policy)`——**把即将 spawn 的精确 argv 交给 sandbox seam，拿回一个包裹后的 argv**（`177-179`）。`confine` 的返回不只是 argv：还有 enforcement（`full`/`partial`，本宿主对这次模式的执行完整度）、denialSignatures（**这个后端**的拒绝方言——bwrap 的 EROFS、Landlock 的 EACCES、Seatbelt 的 EPERM，消费者只匹配所选后端的方言，而不是跨后端并集）和 runnerFailureRules（区分"runner 自己没能启动"与"围栏生效拒绝了命令"的结构化证据规则）（`packages/sandbox/sandbox/src/index.ts:90-116`）。

sandbox-local 这个 provider 按平台选 runner 链（`packages/sandbox/sandbox-local/src/index.ts:150-166`）：Linux 是 `[bwrap, landlock]`（有多个候选才跑功能探针仲裁，bwrap 优先），darwin 是 `[seatbelt]`（唯一候选免探针），win32 是 `[windows-acl]`；平台没有链时 `confine` 失败即 fail closed，抛带 `SANDBOX_UNAVAILABLE` 错误码的 `SandboxUnavailableError`（`packages/sandbox/sandbox/src/index.ts:124-144`）——**宁可不跑，不可不围栏地跑**，seam 契约明文禁止"静默不围栏直传"（同文件 `152-157`）。这些平台行为本篇只能静态推断（本机 macOS，无法全平台运行验证）；windows-acl 的 enforcement 自报 `partial`（ACL 表达力有残余面，代码注释自证 `177-187`）——这里只描述机制，不做"安全/不安全"的价值断言。 runner 分类这套结构化规则是有血的教训的：postmortem 0004（`docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md`）记载旧 ABI Landlock 内核会在每个子进程前打印一行良性 partial-enforcement 提示，旧代码用宽泛的子串匹配把它误判成 runner 失败，ripgrep 的"无匹配 exit 1"都被报成 `SANDBOX_UNAVAILABLE`。

回到主锚点：包裹后的 argv 交给从父类继承的 `runArgv`，`npm test` 此时已易容为类似 `sandbox-exec -p <profile> -- bash -c 'npm test'`（macOS 上）的形态。

### 3.9 进程诞生：subprocess 是唯一的进程 substrate（chunk：全仓库只有一个地方 fork 进程）

`runArgv`（`packages/shell/bash-local/src/index.ts:223-240`）做三件事：用 `using d = deadline(spec.signal, spec.timeoutMs, 'BASH_TIMEOUT')` 把执行器自己的超时与调用方取消融合成一个 deadline；调 `this.ctx.subprocess.spawn(spawnSpec)`；等 `handle.done` 后做**第一因分类**——`timedOut` 与 `aborted` 互斥，只有执行器自己的超时理由算 timedOut，外层 deadline 一律算 aborted（`229-231`）。

`spawnSpec`（同文件 `175-198`）是把 shell spec 翻译成 subprocess 词汇的地方：stdin 有数据就 `{data}` 否则 `'ignore'`；stdout/stderr 都是 collect 模式（内存上限溢出后保留尾部，完整流写入上限 64MB 的 spill 文件）；env 三层叠加 `{...ENV_OVERRIDES, ...spec.env, ...spec.dshEnv}`——`NO_COLOR=1`、`TERM=dumb`、`PAGER=cat`、`GIT_PAGER=cat` 垫底（给模型看的输出不要颜色和分页器），调用方普通环境居中，受信的 `DSH_*` 快照永远最后合并、不可被顶掉（`27-32`）。

subprocess seam 自己的词汇（`packages/subprocess/subprocess/src/types.ts`）刻意"无默认、无 shell、无超时分类"：`SubprocessSpawnSpec` 全字段显式（`75-104`），argv 永不 shell 解释（`76`），`terminate()` 是唯一的终止动词——POSIX 信号整个 detached 进程组、SIGTERM 后过 `graceMs` 升级 SIGKILL，Windows 用 `taskkill /T` 杀整棵树（`158-194`）。本地实现 `LocalSubprocessRuntime` 还持有所有活句柄，组合卸载时 terminate+等整树退出，进程退出阶段同步强杀残留（`packages/subprocess/subprocess-local/src/index.ts:49-102`）。

为什么整个仓库要把进程诞生收敛到一个 seam？subprocess 组 README 一句话点题：它是"**one execution world** 的共享进程 substrate"（`packages/subprocess/README.md:5`）。bash 执行器、PTY 终端后端、LSP 宿主、三个进程外 subagent 后端全部经 `ctx.subprocess` 派生进程（`docs/capability-seams.md:447`）——**换成远程沙箱时，把 subprocess-local 换成 subprocess-e2b、把 fs-local 换成 fs-e2b（两者共享 `ctx.e2b` 持有的同一个 E2B SDK 句柄与远程运行时，`docs/capability-seams.md:446`），bash、PTY、LSP 这些 consumer 一行不改，整体迁进远程 Linux**。这就是导读问题 5 的答案：耦合点只有一个，就是 `ctx.subprocess` 这个服务边界。sandbox seam 的模块头也说了同一件事的另一面：容器、微 VM、远程执行不是这个 seam 的事——"replace the surrounding capability seam instead"（`packages/sandbox/sandbox/src/index.ts:1-5`），同世界围栏包 argv，换世界换 seam。

至此 `npm test` 已经在围栏内的一个受管进程树里跑起来了。

### 3.10 回程：post-execute、finalize 与落日志

进程结束，`ShellRunResult` 沿来路返回：tool-bash 把它整理成结构化输出（退出码、信号、超时/中止标志、截断的 stdout/stderr、沙箱事实），渲染成模型可读文本（非零退出带 `[exit code: N]` 标记）。`dispatchToolBody` 把它包成 `ToolExecutionResult` 后，进入第三条瀑布 `tools/post-execute`（`packages/core/tools/src/index.ts:1743-1744`）——**accept / block / replace / 追加上下文**四种决定。挂在这里的生产 listener 正好展示这条瀑布的两种典型用法：

- **spill-policy**（`packages/spill/spill-policy/src/index.ts:190-209`，`prepend: true` 注册）：如果最终结果是超过 `maxInlineBytes` 的纯文本，把**完整文本**存进 `ctx.spillStore`（默认实现存 0700 私有目录的会话作用域文件，`packages/spill/spill-local/src/index.ts:1-8`），把模型-facing 结果换成有界的 head/tail 预览 + 定位符 + 取回指引（本地实现给的是 path + read/grep 提示）。两个细节：它跳过 `read` 工具防止 read→spill→read 循环（`196`）；配置里省略 `maxInlineBytes` 时插件根本不注册，是真 no-op（`20`）。
- **repeat-tool-reminder**（`packages/guard/repeat-tool-reminder/src/index.ts:213-231`）：per-agent 连续重复调用计数器，默认阈值 `[3, 5, 8]`（`46`）。它是**纯劝告**——先计数，再 `next()` 委托，然后把提醒折进 `additionalContexts`，绝不否决、不改写；用户在 step 间插话会重置链条（`agent/pre-step` 上的另一个 listener，`228-231`）。这就是 guard 组的"loop-hygiene"：不拦你，但提醒你"这个调用你已经连着跑 5 次了"。

瀑布之后是注册表的归一化（快照失败的异常降级为 isError）、定义自有的 `finalizeContent`、然后 `tools/result` 同步通知冻结的权威结果（`docs/tool-execution-pipeline.md:6` 与 `packages/core/tools/src/index.ts:1602-1640` 的收尾路径，静态推断）。agent-loop 在 `tools/result` 上把单条 `tool/result` 事件 append 进会话日志——第 02 篇讲的 model-visible ⟺ logged 闭环在此合拢：`npm test` 这一问一答，从 `tool/call` 到 `tool/result`，全程可从日志重建。

整条链的时序压缩成一张图（listener 注册点为静态推断，相对顺序由 cordis waterfall 注册拓扑决定）：

```text
tool_use(npm test)
  │  agent-loop: append tool/call（第 02 篇）
  ▼
tools/pre-execute 瀑布 ← hooks-claude-code:238 / hooks-codex:225 / tool-jobs:233
  │ allow                 │ ask → serviceAsk → ctx.approval.request
  ▼                       │      （approval/asked → approval/request 瀑布 → approval/decided）
单调 guards（deny-only）  │ rejected/cancelled/unavailable → deny
  ▼
tools/execute 瀑布 ← timeout-policy:56（ToolDefinition.timeoutMs → TOOL_TIMEOUT）
  ▼
tool-bash.execute：校验 → ctx.sandboxPolicy.resolve → [escalation: approveEscalation]
  → ctx.shellEnv.collect → ctx.shell.resolve(request) → ctx.shell.run(spec)
  ▼
bash-sandbox.run：ctx.sandbox.confine(['bash','-c',cmd], policy)
  │  sandbox-local 按平台选 runner：linux[bwrap,landlock] / darwin[seatbelt] / win32[windows-acl]
  ▼
bash-local.runArgv：deadline 融合 → ctx.subprocess.spawn(spawnSpec)
  │  subprocess-local：detached 进程树、env 擦洗、collect+spill、SIGTERM→grace→SIGKILL
  ▼
进程（npm test）→ ShellRunResult → 工具渲染
  ▼
tools/post-execute 瀑布 ← spill-policy:190(prepend，超限外溢) / repeat-tool-reminder:213(劝告)
  ▼
finalizeContent → tools/result 通知 → agent-loop append tool/result
```

## 4. 模块详解：平行 seams 的同一范式

主锚点走完了 bash 一条链，但第 2 节那张对照表还有大半格子没被旅程经过。本节把它们逐个补上——不按包名字母序流水账，而是按“三角色怎么拆”归成四种结构形态加一个反例：每个 seam 给 chunk 标题、签名级证据与替换点，guard 组与 spill 这类纯 listener 插件在末尾归位。读完这节，“dsh 的能力都是同一范式”不再是一句宣称，而是一张可逐格验证的表。

### 4.1 注册表型：fs 与 web（chunk：服务只管登记与选择，语义全在 Provider）

最纯正的 seam 形态是**服务本体即注册表/契约，不自带任何语义**。fs 的 Service Definition 模块头自述：目标身份、路径与 URI 处理、containment、文本读、解码、二进制拒绝、原子变更全部归后端所有（`packages/fs/fs/src/index.ts:1-9`）。服务自己定义的只有契约与三个自有事件门：`fs/write-intent`（写前决策）、`fs/edit-intent`（编辑前版本闸）、`fs/observed`（读后同步记录）（`packages/fs/fs/src/index.ts:58-76`）。注意这两个 waterfall 与第 3.2 节的 `tools/pre-execute` 语义不同：它们是 **single-slot decision**——第一个返回意图的 listener 独占决定，不与同伴组合（JSDoc 原文：the first listener that returns an intent owns the decision rather than composing with peers，`packages/fs/fs/src/index.ts:58-60`）。文件世界的“政策闸”因此不在工具层，而在 fs seam 自己身上。fs 的三种 Provider 对应三种执行世界：`fs-local`（本机直写）、`fs-sandbox`（继承 LocalFileSystem，只在两个变更方法上加 per-call 政策围栏，读路径全模式放行，模块头自证 `packages/fs/fs-sandbox/src/index.ts:1-8`）、`fs-e2b`（远程）。消费端除了 read/write 的 `tool-fs`，还有 `tool-fs-search`（grep/glob）与 `tool-str-replace-editor`——三个工具插件共享同一个 `ctx.fs`。

web 同构但多一层“选择”：`ctx.web` 内置 search/fetch 两个 provider 注册表，重复 id 拒绝；执行时若配置了 provider 用配置的，没配置则要求**恰好一个**可用 provider——“选择永不依赖注册顺序”（`packages/web/web/src/index.ts:1-7` 模块头）。它的消费端 tool-web 也贡献了本篇 3.5 节已用过的那条纪律：fetch 的超时是部署策略（config → `ToolDefinition.timeoutMs` → timeout-policy 执行），不是模型参数（`packages/web/tool-web/src/fetch.ts:1-5`）。**替换点**：fs 换后端 = 组合层换 Provider 行；web 换搜索引擎 = 换挂 `web-search-exa` / `web-search-perplexity` / `web-search-deepseek`，或在 config 里点名。

### 4.2 会话型：terminal（chunk：owner-scoped 的持久 PTY）

terminal 与 bash 的分工在 3.6 已见（一次性执行 vs 后台 job），这里补第三态：**持久 PTY 会话**。六个模型工具 `terminal_open/send/read/signal/close/list`（`packages/terminal/tool-terminal/src/index.ts:163/198/297/330/355/386`）操作的不是进程而是会话对象；模块头一句话立身份：会话归属来自工具执行时的那个 Agent 实例（Owner identity comes from the exact tool execution Agent，`packages/terminal/tool-terminal/src/index.ts:1-4`）——同一 agent 的会话跨调用存活，输出有界滚动。Provider 是 `terminal-bash`：PTY 后端跑在 subprocess 的 terminal 原语上，共享沙箱政策、有界输出、provider 自有的会话清理（`packages/terminal/terminal-bash/src/index.ts:1-5`）——它同时是 subprocess 与 sandbox 的 Consumer，第 2 节表里的叠层关系在这里落成代码。另一个 Consumer `tool-bash-persistent` 把“持久 bash 工具”也建在这条 seam 上（模块头：over the owner-scoped PTY seam，`packages/shell/tool-bash-persistent/src/index.ts:1-3`，用 `ctx.terminals.read/list` 翻页）。**替换点**：换 shell 后端（如远程 PTY）只动 Provider；六个工具与持久 bash 工具对后端无感知。

### 4.3 封闭词汇型：lsp（chunk：协议面收缩到四个语义操作）

lsp seam 是“词汇表收紧”的极端样本：Service Definition 只暴露四个语义操作 `goToDefinition / findReferences / goToImplementation / hover`——封闭 union，加一个操作是跨 seam/provider/tool 的编译期强制变更；不暴露任何协议类型、进程或文档控制、通用 JSON-RPC 逃逸口（`packages/lsp/lsp/src/types.ts:8-17`）。LSP 线协议的复杂性被整体吸收在 Provider `lsp-stdio` 一侧：一个插件实例配置一张命名 server 命令表，每个条目注册一个隔离 provider，provider 对每个规范化工作区目标惰性单飞一个 server 进程，读源码走 `ctx.fs`、拉进程走 `ctx.subprocess`——“本地与远程实现共享同一个宿主”（`packages/lsp/lsp-stdio/src/index.ts:1-6`）。另一个值得记住的事实：lsp 栈不在默认 base 组合里（`packages/bundle/` 目录无 lsp 条目，静态推断；`docs/subsystems/lsp.md:5` 自述 one optional capability）——它是按需挂载的可选能力。**替换点**：换 LSP 宿主实现只动 Provider；四个操作词汇表不动，模型问导航的方式不变。

### 4.4 目录合并型：skill（chunk：多来源目录的裁决权在服务）

skill seam 的服务本体不执行任何东西，它**合并多个 provider 的技能目录，并对同名技能做裁决**——merge provider catalogs, resolve the winning skill for a name（`packages/skill/skill/src/index.ts:1-11` 模块头）。两个 Provider：`skill-filesystem`（从文件系统读技能）、`skill-badge`（徽章来源）；消费端 `tool-skill` 把获胜目录与定义暴露给模型。它与 fs/web 的差别在于：注册表型的 provider 各管各的调用，而 skill 的 provider 是**同一份数据的多个来源**，冲突必须裁决而不是并存。**替换点**：换技能来源（如远程目录）= 新 Provider 注册进 `ctx.skills`，裁决规则不动。

### 4.5 code-runtime：Consumer 在核心里的 seam（chunk：containment，不是安全边界）

code-runtime 的三角色位置特殊：它的 Consumer 不是某个工具插件，而是 `core/tools` 的 Code Mode（run_code 传输）。Provider 是 `code-runtime-worker-thread`：每个去类型化的 TypeScript 程序跑在一个新 worker 里、绑定经 message port 桥接（`packages/code-runtime/code-runtime-worker-thread/src/index.ts:8-9` 的 import 侧自证）。模块头把承诺写得极克制：**这是容纳，不是安全边界**（containment, not a security boundary）——空环境、heap 上限、事件循环忙时/墙钟预算、能终止同步死循环，但模型代码仍具 bash 等效信任（`packages/code-runtime/code-runtime-worker-thread/src/index.ts:1-7`）。**替换点**：换运行时（如独立进程隔离器）只动 Provider；Code Mode 传输不动。

### 4.6 反例：mcp-client（chunk：外部协议适配器不需要 seam）

mcp-client 是全套执行类能力里唯一不符合三角色范式的包，机制事实如下：它是 namespace 函数插件，每个实例连一台外部 MCP server，把对方工具以 `mcp__<serverName>__<rawName>` 公共名注册进 `ctx.tools`（`packages/mcp/mcp-client/src/index.ts:1-9`）；公共名过长或含非法字符时规范化并追加内容 hash 后缀（`packages/mcp/mcp-client/src/tools.ts:96-97`）。断线由 connection supervisor 管：一次 outage 共享一个尝试预算（maxAttempts 连败、指数退避封顶），连接稳定超过窗口则重置预算——崩环 server 即使偶尔连上也会耗尽上限；耗尽即注销该 server 全部工具并停手，只有 dispose/HMR 能回到可连状态（`packages/mcp/mcp-client/src/connection.ts:1-12`）。transport 在 stdio（经凭证擦洗）与 Streamable HTTP 之间二选一（`packages/mcp/mcp-client/src/transport.ts:1-5`）。为什么它不做成三角色 seam？可替换性已经在组合层解决：换 server 是换 cordis.yml 里一条插件配置，不需要服务边界。三角色范式解决的是“能力语义可替换”，外部协议适配的替换点是实例配置——这正是范式的适用边界，与 4.3 的词汇闭包互为两极：lsp 把外部协议收缩到四个词，mcp 把外部协议原样透传。

### 4.7 e2b：两个 Provider 共享一个世界（chunk：POC，一句话带过）

e2b seam 只有一件事：共享持有一个 E2B SDK 句柄，让 `fs-e2b` 与 `subprocess-e2b` 的文件与进程操作落进**同一个远程 Linux 世界**（模块头原话：filesystem and process operations inhabit one remote Linux world，`packages/e2b/e2b/src/index.ts:1-5`）。第 3.9 节“换执行世界”的完整拼图，最后一块就是它——POC 状态，一句话带过。

### 4.8 收束：形态跟着“角色是否需要独立演化”走

四种形态加一个反例看下来，第 2 节那张表可以再读出一层：**形态的选择跟着角色演化需求走**。fs/web 的来源会替换（注册表）；terminal 的会话要跨调用存活（owner 裁定）；lsp 的协议面要封死（词汇闭包）；skill 的来源要裁决（合并目录）；mcp 的替换点是实例本身（不要 seam）。guard 组与 spill-policy 这类“纯 listener 插件”则是范式的退化形态——连服务都不注册，只挂事件：spill 模块头自述“policy 只决定何时外溢”（it registers NO service and owns NO storage，`packages/spill/spill-policy/src/index.ts:1-10`）。它们已在第 3.5 与 3.10 节随管道出场，此处归位即可。

## 5. 关键制品详解

主线走完了。本节把四处最值得逐行读的制品拿出来走拆解链五连——它们是这条链上"设计决策密度"最高的四个点。

### 5.1 ShellExecRequest / ShellExecSpec 与 bash-local 的 resolve

**回答什么问题**：默认值填充放在哪、以什么形态发生，才能让"下游 seam 无默认"与"执行方法不重新默认"同时成立。

制品全文，先是类型对（`packages/shell/shell/src/types.ts:32-110`，原文）：

```ts
/**
 * A caller's execution REQUEST: `workdir` and `timeoutMs` are optional and
 * filled by {@link ShellExecutor.resolve} from the implementation's config.
 * This is the model-/plugin-facing shape; pass it to `resolve()` to obtain a
 * fully-resolved {@link ShellExecSpec}.
 */
export interface ShellExecRequest {
  command: string
  /** Working directory override (default: implementation-configured). */
  workdir?: string | undefined
  /** Timeout override in milliseconds (implementations cap it). */
  timeoutMs?: number | undefined
  /** Foreground stdout capture budget in bytes. … */
  stdoutMaxBytes?: number | undefined
  /** Abort signal — implementations kill the command when it fires. */
  signal?: AbortSignal | undefined
  /** Bytes to write to the command's stdin, then close it. … */
  stdin?: string | undefined
  /** Ordinary environment entries for the command, merged after the credential scrub. … */
  env?: Record<string, string> | undefined
  /** Harness-owned `DSH_*` variables for this execution (typed to managed keys). … */
  dshEnv?: DshEnvironment | undefined
  /** Fully resolved per-call sandbox policy; sandboxing executors default it. */
  sandboxPolicy?: SandboxExecutionPolicy | undefined
}

/** A resolved execution spec. {@link ShellExecutor.resolve} fills and caps the
 * required fields; {@link ShellExecutor.start} ignores `timeoutMs` … */
export interface ShellExecSpec {
  command: string
  workdir: string
  timeoutMs: number
  stdoutMaxBytes: number
  signal?: AbortSignal | undefined
  stdin?: string | undefined
  env?: Record<string, string> | undefined
  dshEnv?: DshEnvironment | undefined
  sandboxPolicy: SandboxExecutionPolicy | undefined
}
```

再是实现（`packages/shell/bash-local/src/index.ts:146-171`，原文）：

```ts
resolve(request: ShellExecRequest): ShellExecSpec {
  const timeoutMs = clampTimeout(
    request.timeoutMs,
    this.config.timeoutMs,
    this.config.maxTimeoutMs,
    'bash-local: request.timeoutMs',
  )
  const stdoutMaxBytes = request.stdoutMaxBytes ?? this.config.maxOutputBytes
  assertPositiveFinite('request.stdoutMaxBytes', stdoutMaxBytes)
  return {
    command: request.command,
    workdir: request.workdir ?? this.config.cwd ?? process.cwd(),
    timeoutMs,
    stdoutMaxBytes,
    ...request.signal ? { signal: request.signal } : {},
    // Carry stdin/ordinary env/trusted dshEnv through verbatim — optional,
    // no config default. The subprocess service owns the scrub and merge order.
    ...request.stdin !== undefined ? { stdin: request.stdin } : {},
    ...request.env !== undefined ? { env: request.env } : {},
    ...request.dshEnv !== undefined ? { dshEnv: request.dshEnv } : {},
    // Carry a sandbox policy through verbatim: this executor never
    // confines, so the field is inert here (the seam contract) — a
    // sandboxing subclass overrides resolve() to stamp its default instead.
    sandboxPolicy: request.sandboxPolicy,
  }
}
```

**逐项解释**：request 侧，只有 `command` 必填；`workdir`/`timeoutMs`/`stdoutMaxBytes` 是"实现配置填充"的三个可缺字段；`signal`/`stdin`/`env`/`dshEnv` 是**透传型**可选字段——它们没有默认可言，缺席本身就是语义（无 stdin、无额外环境）；`sandboxPolicy` 特殊：请求方可带全解析政策，不带则由"会围栏的执行器"在 resolve 里盖部署默认。spec 侧四个字段变必填，其余保持可选。实现里 `clampTimeout` 不是简单 `??`：请求值要被夹进 `[config.timeoutMs 的语义默认, config.maxTimeoutMs 硬上限]`，越界会被夹或报错；三个 `...cond ? {x} : {}` 展开是 dsh 全仓的可选字段习惯写法——缺席时键本身不出现，而不是出现为 `undefined`。

**直觉**：关键设计在"**谁拥有默认值**"。timeout 的默认 120s/上限 600s 是 bash-local 的 Config（cordis.yml 可改），不是 seam 的常量，更不是 subprocess 的兜底——所以 subprocess 可以宣布"我无默认"，spec 上的每个数字都能追溯到某个具体部署的配置。这让"策略在哪改"有了唯一答案：改实现插件的配置，而不是改调用方。

**反事实**：如果允许 `run()` 内部写 `timeoutMs = spec.timeoutMs ?? 120_000`，那么默认值会随实现复制进每个执行方法；subprocess  seam 也得各自发明兜底，"这次执行到底用了什么超时"就得读实现源码才能回答——spec 失去"完全解析"的承诺，日志与重放也无法只靠类型信任它。

**白话翻译**：调用方说"我要跑这个，其他你看着办"；resolve 把"看着办"一次性办成白纸黑字，执行器只认白纸黑字。

### 5.2 approveEscalation：执行前的审批编排

**回答什么问题**：模型申请临时放宽沙箱时，"校验→找通道→问人→映射结果"这套顺序怎样做到两个执行家族（bash/fs）一字不差地共享，且每一步都 fail closed。

制品全文（`packages/sandbox/sandbox/src/escalation.ts:157-189`，原文）：

```ts
export async function approveEscalation<A, C>(request: EscalationRequest, approval: EscalationApproval<A, C>): Promise<SandboxMode> {
  const { requestedMode: mode, effectiveMode, justification, subject } = request
  // Strict widening is an EXECUTION check against the call's effective mode —
  // deliberately not a schema constraint (the enum is the closed target
  // vocabulary; the effective mode is per-call truth).
  if (!(WIDER_MODES[effectiveMode] ?? []).includes(mode as SandboxMode)) {
    throw new Error(`sandbox escalation to "${mode}" is not strictly wider than this call's current "${effectiveMode}" mode`)
  }
  if (approval.approver === undefined) {
    throw new Error(`sandbox escalation to "${mode}" requires approval, but no approval service is composed`)
  }
  if (approval.agent === undefined) {
    throw new Error(`sandbox escalation to "${mode}" requires approval, but the call has no agent to route it through`)
  }
  // Self-contained for the audit trail: approval/asked stores this reason,
  // and the target mode is part of the grant's identity.
  const outcome = await approval.approver.request({
    agent: approval.agent,
    toolName: approval.toolName,
    callId: approval.callId,
    reason: `escalate sandbox to ${mode}: ${justification}`,
    ...approval.signal ? { signal: approval.signal } : {},
  })
  switch (outcome) {
    // The schema enum already pinned `mode` to the closed target vocabulary;
    // the check above proved it is strictly wider.
    case 'allowed-once': return mode as SandboxMode
    case 'rejected': throw new Error(`the user rejected escalating this ${subject} to "${mode}"`)
    case 'cancelled': throw new Error(`approval for escalating to "${mode}" was cancelled`)
    case 'unavailable': throw new Error(`sandbox escalation to "${mode}" requires approval, but no approval channel is available`)
    default: return assertNever(outcome, 'EscalationOutcome')
  }
}
```

**逐项解释**：`request` 携带申请目标模式、模型的理由原文、**本调用的**有效模式（会话覆盖 ?? 组合默认——注释强调这是 per-call 真相）；`approval` 是工具层持有的三件料：approver（`ctx.approval` 或缺席）、agent、调用身份。函数体是严格顺序的四道门：①严格更宽校验——`WIDER_MODES` 表（同文件 `28-31`：read-only 可升两档，workspace-write 只能升 danger-full-access），不更宽直接抛，**不打扰人**；②无审批服务抛；③无 agent 可路由抛；④问人，reason 里嵌目标模式与模型的 justification（审计自足）；⑤switch 映射：只有 `allowed-once` 活着出去并带回目标模式，其余三个各抛一句互不相同、逐字固定的文案。`assertNever` 守住封闭 union 的穷尽性。

**直觉**：两个设计决定值得记住。其一，严格更宽校验放在**执行时**而非 schema 里——schema 的 enum 是封闭目标词汇表（注册表全局），而有效模式是 per-call 的，schema 层根本算不出"对这个调用而言什么算更宽"。其二，这个包**不依赖 approval 包**：`EscalationApprover` 是结构类型，工具层闭包 `ctx.approval.request` 递下来——sandbox 定义包保持零下游依赖。

**反事实**：若把 widening 校验做成 schema enum 的动态裁剪，组合默认 danger-full-access 时会广告空枚举，而实际被切到更窄模式的会话就失去了升级杠杆（`packages/sandbox/sandbox/src/escalation.ts:33-41` 的注释正好讲了这个坑）。若缺了②③两道门，无审批通道的部署会把"问不了人"静默吞成别的错误类。

**白话翻译**：先自己查清楚"配不配问"，配，再问人；人说行，就只放行这一次。

### 5.3 ApprovalService.request 与 decide：每一问都要留痕

**回答什么问题**：审批如何做到"不问人时结果也可判定"、"问人的全过程可审计可重放"、"任何失败都倒向拒绝"。

制品全文（`packages/interaction/user-approval/src/index.ts:257-276` 与 `304-344`，原文，删节号处为 JSDoc）：

```ts
async request(req: ApprovalRequest): Promise<ApprovalOutcome> {
  const session = req.agent.session
  if (!hasOpenTurn(session.events)) {
    throw new Error(
      'approval.request() outside an open turn: the approval/asked + approval/decided audit pair '
      + 'must be turn-enclosed (a bare event between turns is crash-tail garbage on reload). '
      + 'Ask from inside the turn that needs the decision.',
    )
  }
  const id = ApprovalRequestId(randomUUID())
  session.append('approval/asked', {
    id,
    toolName: req.toolName,
    ...req.callId !== undefined ? { callId: req.callId } : {},
    ...req.reason !== undefined ? { reason: req.reason } : {},
  })
  const outcome = await this.decide(req, session)
  session.append('approval/decided', { id, outcome })
  return outcome
}

private async decide(req: ApprovalRequest, session: Session): Promise<ApprovalOutcome> {
  const signal = req.signal
  if (signal?.aborted) return 'cancelled'
  // The 'never' policy is decided HERE, before any dispatch: a listener
  // registered with `prepend: true` after this service mounts would sit
  // ahead of any gate LISTENER, so a listener-shaped gate cannot keep the
  // documented promise that 'never' rejects deterministically regardless
  // of registration order — only the service's own request path can.
  if (this.effectivePolicy(session) === 'never') return 'rejected'
  // Enter the promise chain BEFORE dispatching: a listener that throws
  // SYNCHRONOUSLY (before its first await) must land in the same rejection
  // path as an async one — `Promise.resolve(call())` would let it escape
  // the containment into the caller.
  const answer: Promise<ApprovalOutcome> = Promise.resolve().then(
    () => this.ctx.waterfall(
      scopeTarget(this, req.agent), 'approval/request', req,
      () => Promise.resolve<ApprovalOutcome>('unavailable'),
    ),
  ).then(
    // Normalize a rogue (non-vocabulary) answerer return to the fail-closed
    // outcome instead of leaking it into callers' closed-union switches.
    outcome => OUTCOMES.includes(outcome) ? outcome : 'unavailable',
    // A throwing answerer must fail the QUESTION closed, not the caller's
    // tool call open — the seam contains its callbacks.
    () => 'unavailable',
  )
  if (signal === undefined) return answer
  return await new Promise<ApprovalOutcome>((resolve) => {
    const onAbort = () => {
      signal.removeEventListener('abort', onAbort)
      resolve('cancelled')
    }
    signal.addEventListener('abort', onAbort, { once: true })
    void answer.then((outcome) => {
      signal.removeEventListener('abort', onAbort)
      // After an abort won the race this resolve is a settled-promise no-op:
      // the late answer is discarded by construction.
      resolve(outcome)
    })
  })
}
```

**逐项解释**：`request` 三段式——open-turn 前置检查（`hasOpenTurn` 倒扫日志找未闭合的 `turn/start`）、`approval/asked` 落日志（branded `ApprovalRequestId` 配对）、`decide` 之后 `approval/decided` 落日志。`decide` 五层：已 abort → `cancelled`；政策 `never` → 就地 `rejected`（长注释解释了为什么不能做成 listener）；`Promise.resolve().then(...)` 包瀑布调用，把同步抛错的 listener 也收进同一个拒绝路径；两个 `.then` 分别把"词汇表外返回值"和"answerer 抛错"归一到 `unavailable`；最后与 signal 竞速，abort 先到即 `cancelled`，晚到的回答被构造性地丢弃。

**直觉**：整段代码的重心不在"问"，在**兜底**。四条独立路径（无 answerer、answerer 抛、answerer 乱回、调用方取消）全部有显式归宿，且归宿都是拒绝家族。审计对必须 turn-enclosed 这条前置条件也值得品味：durable 日志以 turn 为提交/重放边界，turn 外的裸事件在重载后与崩溃尾巴无法区分——审批留痕因此被迫"只能从 turn 里问"。

**反事实**：若 `never` 判定做成瀑布 listener，后挂载的 prepend listener 会排到它前面，`never` 的"无论注册顺序都确定性拒绝"承诺就守不住；若 rogue 返回值不归一化，它会漏进调用方的封闭 union switch，编译期类型安全在运行时被一个坏 answerer 击穿。

**白话翻译**：问之前先留名，答完立刻画押；问不成、答得怪、人不在，一律当作没批。

### 5.4 cordis.patch.yml 的平台门控段：一份配置两个 shell 栈

**回答什么问题**：POSIX 与 Windows 共享一份组合文件时，如何保证每个宿主恰好挂载一个 shell 栈。

制品全文（`packages/bundle/base/cordis.patch.yml:178-216`，原文配置，quote-first）：

```yaml
    - id: bash-sandbox
      name: '@deepseek-ai/dsh-bash-sandbox'
      disabled: !!js process.platform === 'win32'
      config:
        timeoutMs: 60000

    - id: pwsh-sandbox
      name: '@deepseek-ai/dsh-pwsh-sandbox'
      disabled: !!js process.platform !== 'win32'

    - id: approval
      name: '@deepseek-ai/dsh-user-approval'
      config:
        policy: !!js "(process.env.DSH_PERMISSION_MODE ?? 'workspace-write') === 'danger-full-access' ? 'never' : 'ask'"

    - id: permission
      name: '@deepseek-ai/dsh-permission-presets'
      config:
        presets:
          read-only:
            sandbox: read-only
            approval: ask
          workspace-write:
            sandbox: workspace-write
            approval: ask
          danger-full-access:
            sandbox: danger-full-access
            approval: never

    - id: shell-env
      name: '@deepseek-ai/dsh-shell-env'

    - id: tool-bash
      name: '@deepseek-ai/dsh-tool-bash'
      disabled: !!js process.platform === 'win32'

    - id: tool-pwsh
      name: '@deepseek-ai/dsh-tool-pwsh'
      disabled: !!js process.platform !== 'win32'
```

**逐项解释**：四行 `disabled: !!js ...` 是 cordis.yml 允许的 JS 表达式求值（仅 `config` 与 `disabled` 两处放行，`AGENTS.md` 的 Secrets/.env 节写明）：bash 族两行在 win32 上禁用，pwsh 族两行用取反表达式只在 win32 挂载。`approval` 的默认政策从环境变量派生：部署模式是 danger-full-access 时审批政策默认 `never`（全自动部署不弹窗）。`permission` 的 presets 表把三档权限预设各绑一对 `(sandbox 模式, approval 政策)`——用户切预设时一次写三个日志事件（`permission/preset` 记意图 + `sandbox/mode` + `approval/policy` 两个旋钮事件，`packages/interaction/permission-presets/src/index.ts:376-428`）。

**直觉**：平台门控放在**组合层而非代码里**，是这个制品的核心决策。bash 没有 Windows runner 这件事不该由 TypeScript 里的 `if (platform)` 表达——那是部署事实，归 cordis.yml 管。而"恰好一个栈"的保证不是靠配置纪律，是靠机制兜底：两族执行器注册的是**同一个** `shell` 服务，如果哪个 Windows 用户的自定义 patch 把两边都启用或都禁用得不干净，cordis 的重复服务注册直接在加载时抛错（`packages/shell/shell/src/index.ts:16-20` 的注释与 `46-50` 的契约；`packages/bundle/base/README.md:7` 把这个"恢复配方必须完整"的陷阱写给用户）。

**反事实**：若门控写进代码（插件内部自检平台后跳过注册），配置目录里就再也看不出"这个宿主到底挂哪个栈"，loader 的组合视图失真；若两族注册不同服务名，双挂载不会报错，但 tool-bash/tool-pwsh 会面对两个执行器无所适从——失败从"加载期响亮"退化为"运行期暧昧"。

**白话翻译**：同一张配方，两个平台各吃一半；吃错了，开不了机而不是跑错饭。

## 6. 非文字内容讲解

本篇两张承重图已随文讲解：第 2 节的**seam 范式对照表**（空间结构：谁替换谁、谁消费谁，读法是"按列换 Provider，按行看 Consumer 无感"）与第 3.10 节末尾的**S2 时序图**（时间结构：三条瀑布 + guards + approval 旁路的相对位置，读法是"每条 `←` 都是组合层可挂插件的扩展点"）。两者互补：表告诉你 `npm test` 路上每个模块的"格子"，图告诉你它路过的"顺序"。`docs/tool-execution-pipeline.md` 的官方 mermaid 图是第三视角（含 fs 写意图闸门、Code Mode 子调用等本篇未展开的分支），与 3.10 的图对照时记住一处读图陷阱：官方图把 approval 画成管线一站，代码里它是被调用的服务而非 listener（3.3 节已证）。

## 7. 价值与贡献

本篇覆盖的部分集中体现了 dsh 三个真正的取舍点。第一，**三角色 seam 作为机械纪律**：十几个能力共享同一个包结构模板，换来的是"替换点"可以被枚举——第 2 节的表本身就是证据，每个 seam 的最小替换操作都是"组合层换 Provider 行"。这比"约定俗成的插件接口"强在可机械检查（`docs/capability-seams.md` 是带完整性守卫的生成物）。第二，**政策与机制的三层分离**：机制在 Provider（怎么围栏）、部署政策在 sandbox-policy（默认模式与根）、会话政策在日志事件（`sandbox/mode` fold）—— bash 与 fs 两个执行家族读同一个 `ctx.sandboxPolicy`，从结构上保证"bash 能写 /tmp 而 write 工具不能"这类不对称不会出现（`packages/sandbox/sandbox/src/roots.ts:1-13` 把可写根推导收敛到一个函数也是同一意图）。第三，**fail closed 是一等设计值**：sandbox 缺 runner 抛 `SANDBOX_UNAVAILABLE` 而非裸跑，approval 缺 answerer 归一 `unavailable`，escalation 在每个岔口抛不同文案——"拒绝"被细化成一组可诊断的封闭词汇，而不是一个布尔。

## 8. 局限与批判

### 8.1 意图 vs 现实偏差

| 宣称（文档/命名给人的预期） | 代码实际 | 出处 |
|---|---|---|
| `docs/tool-execution-pipeline.md` 的 mermaid 把 approval 画成 pre-execute 与 guards 之间的一站，读感像瀑布 listener | approval 是独立 seam：registry 的 `serviceAsk` 与工具体的 `approveEscalation` 两处**主动调用** `ctx.approval.request`；`approval/request` 瀑布只服务 answerer 注册 | `packages/core/tools/src/index.ts:1689-1721`；`packages/shell/tool-bash/src/index.ts:213-233` |
| "一切皆三角色 seam"（仓库 AGENTS.md 的 seam 规则） | mcp-client 不符合：无 Service Definition/Provider/Consumer 拆分，是 namespace 函数插件直接把外部 MCP 工具桥接进 `ctx.tools`（公共名 `mcp__<serverName>__<rawName>`，断线重连带 bounded backoff） | `packages/mcp/mcp-client/src/index.ts:1-9`、`packages/mcp/mcp-client/src/tools.ts:96-97`、`packages/mcp/mcp-client/src/connection.ts:1-12` |
| spill 对 read 工具豁免（防 read→spill→read 循环） | 豁免只在 model-facing 臂；`tools/code-dispatch-log` 臂**不豁免** read（日志副本不是模型上下文，循环不会发生）——同一插件两臂规则不同，代码注释自证，文档未强调 | `packages/spill/spill-policy/src/index.ts:196` 与 `216-219` |
| `docs/capability-seams.md` 的 seam 图（生成物）把 lsp provider 标为 `lsp-local` | 仓库实际包是 `lsp/lsp-stdio`（无 lsp-local 包；`docs/subsystems/lsp.md:5` 写的也是 dsh-lsp-stdio）——生成图漂移未重跑（静态推断） | `docs/capability-seams.md:190`；`packages/lsp/lsp-stdio/src/index.ts:1-2` |
| windows-acl 与其余后端并列在 runner 链里 | 它的 enforcement 自报 `partial`（ACL 表达力残余面：Everyone 写授权、硬链接别名）——并列不代表同等承诺，消费者必须把 `partial` 与 `full` 区分对待 | `packages/sandbox/sandbox-local/src/index.ts:177-187` |

### 8.2 行为论断标注汇总

本篇的行为论断绝大多数是**静态推断**（读代码推得，带 `file:line` 锚点）：管道阶段顺序、listener 注册拓扑、resolve/confine/spawn 的调用关系、approval 的兜底路径均属此类——未实际组装运行 harness。平台相关行为（bwrap probe、Seatbelt、windows-acl 的真实执行）在本机无法验证，维持静态推断并已在 3.8 节声明。postmortem 0004 是文档记载的已解决历史事故，作机制设计的风险佐证引用，非本篇复现。本篇全部 `file:line` 锚点的文件存在性由 deep-read 的 lint 脚本机械校验（`python3 scripts/lint_note.py … --repo-root …` 通过，0 error），此项为已验证（跑过）。

### 8.3 开放问题

- [ASK USER] 为什么 bash 的围栏走"executor 子类"（bash-sandbox 继承 bash-local）而 fs 的围栏走"provider 内围栏"（fs-sandbox 实现 `ctx.fs` 时自带 fence）？两种叠加结构不对称；`packages/sandbox/sandbox/src/roots.ts:1-13` 只解释了两者共享可写根推导，没解释为什么不统一走同一条路线。
- [ASK USER] escalation 的"拒绝即终局"目前靠工具描述约束模型（`packages/shell/tool-bash/src/index.ts:82-92` 的长段提示词），没有机制级强制（同 turn 内重复 escalation 不会被 guard 拦下）——这是有意留给模型的柔性问题，还是尚未补的闸？

## 9. 怎么用

**换 provider 的最小操作**（每个 seam 的替换点，都是组合层一行改动）：

- bash 换远程：cordis.yml 里把 `subprocess-local` 换成 `subprocess-e2b`、加挂 `e2b` 与 `fs-e2b`——bash/terminal/lsp 随之整体迁移（3.9 节）；只换执行器（如不要围栏）则把 `bash-sandbox` 行换成 `bash-local`。
- 换沙箱 runner：Linux 上无需操作（bwrap→landlock 自动探针仲裁）；自定义 runner 用 `sandbox-local` 的 runner argv 覆盖配置。
- 换 fs 后端：`fs-local` ↔ `fs-sandbox` ↔ `fs-e2b` 三选一，注意 `fs-sandbox` 与 `fs-local` 并存会重复注册 `ctx.fs` 在加载时失败（`packages/bundle/base/README.md:7`）。
- 接外部工具：cordis.yml 加一条 `mcp-client` 插件实例，配 `serverName` 与 transport（stdio/HTTP），工具自动以 `mcp__<serverName>__*` 出现。
- 挂新工具：`defineTool` + `ctx.tools.register`（tool-bash 的 `packages/shell/tool-bash/src/index.ts:242-393` 是最完整的样板：schema、output render、presentCall/presentResult、approval、后台 job 全覆盖）。

**拉线头入口**：想验证管道行为，从 `packages/core/tools/tests/tools.spec.ts`（pre/post-execute 决策语义）与 `packages/shell/tool-bash` 的测试入手；想看真实组合下的端到端，`pnpm dsh --profile headless "task"`（需 `DEEPSEEK_API_KEY`）或 `packages/examples/` 的 keyless 快照回放。**坑清单**：①escalation 参数成对出现否则 execute 抛错；②`sandbox_permissions` 只在挂载了围栏执行器时才广告，但 schema 校验只查广告键，未广告时带了也会在 execute 被拦（`packages/shell/tool-bash/src/index.ts:350-352` 同款逻辑）；③listener 返回前忘了 `next()` 会短路整条瀑布（第 01 篇的 cordis waterfall 语义）。

## 10. 总结与自测

三句话总结：bash 工具的旅程是"三条瀑布 + 单调 guards + 一个被主动调用的 approval 服务"串起的管道，政策在 pre-execute 表态、deadline 在 execute 外套上、结果在 post-execute 被 spill 与 loop-hygiene 塑形；shell seam 用 request/spec split 把默认值收敛到边界，sandbox 用 argv 包裹实现同世界围栏，subprocess 作为唯一进程 substrate 让"换执行世界"成为一行组合层改动；这套三角色 seam 范式在 fs/terminal/lsp/web/skill/spill 上机械重复，构成了 dsh"能力可替换"宣称的实体。

自测 Q&A：

1. 模型给的 tool_use 块在变成进程前要经过哪些"可以说不"的点，各自的决定词汇是什么？（提示：pre-execute 的 allow/deny/ask、approval 的四值、guard 的拒绝理由、escalation 的抛错。）
2. `ctx.shell.resolve()` 与 `run()` 的分工是什么，为什么默认值不许出现在 `run()` 里？
3. 默认 POSIX 组合里 `npm test` 的 argv 在 spawn 前被谁、以什么形式改写？换成 Windows 宿主这条链哪几环不同？
4. 为什么把 subprocess-local 换成 subprocess-e2b 就能让 bash、PTY、LSP 整体迁进远程沙箱？耦合点在哪？
5. spill-policy 为什么跳过 `read` 工具，又为什么在 code-dispatch-log 臂不跳过？

延伸阅读：`docs/tool-execution-pipeline.md`（官方管道图，含本篇未展开的 fs 闸门与 Code Mode 分支）、`docs/subsystems/shell.md` 与 `sandbox.md`（子系统参考页）、`docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md`（runner 分类的事故复盘）、`.agents/notes/implemented/architecture/2026-07-26-subprocess-seam.md`（subprocess seam 的决策记录）。

## 附录：覆盖报告

**图表索引**（本篇图表均为内嵌 markdown 表格与文中图，无外置物化文件）：

| 图/表 | 位置 | 类型 | 覆盖级 |
|---|---|---|---|
| seam 范式对照表（三角色 × ctx key） | 第 2 节 | 空间结构 | 承重 |
| S2 时序压缩图（npm test 全链） | 第 3.10 节 | 时间结构 | 承重 |
| ShellExecRequest/Spec 类型对 + resolve 实现 | 第 5.1 节 | 代码制品 | 承重 |
| approveEscalation 编排 | 第 5.2 节 | 代码制品 | 承重 |
| ApprovalService.request/decide | 第 5.3 节 | 代码制品 | 支撑 |
| cordis.patch.yml 平台门控段 | 第 5.4 节 | 配置制品 | 支撑 |

**覆盖矩阵**（本篇范围 × 覆盖深度；“模块头级/签名级”= 只读模块头、类型与注册点，未逐函数）：

| 组 | 包 | 覆盖深度 | 出现位置 |
|---|---|---|---|
| shell | shell、bash-local、bash-sandbox、tool-bash | 函数级 | 第 3、5 节 |
| subprocess | subprocess、subprocess-local | 类型 + 关键路径 | 3.9 |
| sandbox | sandbox、sandbox-local、sandbox-policy、escalation、roots | 函数级 | 3.7/3.8、5.2、8.1 |
| interaction | user-approval、permission-presets | 函数级 | 3.3、5.3、5.4 |
| guard | timeout-policy、repeat-tool-reminder | 注册点级 | 3.5/3.10 |
| spill | spill-policy、spill-local | 注册点级 | 3.10、8.1 |
| fs | fs、fs-local、fs-sandbox、fs-e2b | 模块头级 + 事件门 | 4.1 |
| web | web、tool-web/fetch、web-search-*（三选一示意） | 模块头级 | 4.1 |
| terminal | terminal、terminal-bash、tool-terminal、tool-bash-persistent | 签名级 | 4.2 |
| lsp | lsp、lsp-stdio | 类型级 + 模块头 | 4.3 |
| skill | skill、skill-filesystem、skill-badge | 模块头级 | 4.4 |
| code-runtime | code-runtime、code-runtime-worker-thread | 模块头级 | 4.5 |
| mcp | mcp-client（index/tools/connection/transport） | 模块头级 | 4.6、8.1 |
| e2b | e2b、fs-e2b、subprocess-e2b | 模块头级（POC 一句话） | 4.7 |
| core/tools | （第 02 篇已拆，本篇只引用管道段） | 引用级 | 3.1–3.5、3.10 |

**未覆盖/从简清单**（稳定模块或过渡胶水，主线不承重）：`pwsh-local`/`pwsh-sandbox`/`tool-pwsh`（Windows 孪生，行为对称，未逐行）；`shell-env` 包内部（只用其 collect 语义）；hooks 桥（hooks-claude-code/hooks-codex）内部实现（只确证注册点）；`web-search-*` 三个 provider 内部；`tool-fs` 读写工具细节与 `fs-observation-policy`、`tool-str-replace-editor` 内部；`session-checkpoint-policy`（只确证 tools/execute 注册点）；spill 的 `output-retention` 库内部；terminal 的 PTY 原语实现；`sandbox-windows-acl` 包内部（只经 runner 链引用）；`interaction/tool-ask-user`、`interaction/user-questions`、`interaction/commands`。

**开放问题**：①[ASK USER] bash 围栏走 executor 子类而 fs 走 provider 内围栏，结构不对称的原因（详 8.3 第一条）；②[ASK USER] escalation“拒绝即终局”仅靠提示词约束、无机制闸，是否有意为之（详 8.3 第二条）；③[未验证] 各平台沙箱 runner（bwrap probe、Seatbelt、windows-acl）真实运行行为，本机 macOS 仅静态可读，全部标静态推断；④[未验证] `docs/capability-seams.md` 生成图的 lsp-local 漂移是否为重生成遗漏（未运行 `pnpm run gen-doc-graphs`）。
