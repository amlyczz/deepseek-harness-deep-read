# 《DeepSeek Harness》深度解读 · 第 02 篇：核心 spine 与一轮 turn

- 仓库：deepseek-ai/deepseek-harness / 子类型：monorepo（pnpm workspaces，Node ^22.19||>=24，ESM，strict TS）
- 版本锁定：deepseek-ai/deepseek-harness **master 分支 tarball 快照，2026-08-14 下载**；快照不含 git 元数据，**无 commit hash 可记录**（材料降级，特此声明）；替代锚点为根 `package.json` 版本 `0.1.0-rc.5`。文中全部 `file:line` 锚点以该快照的文件系统状态为准。
- 版本锁定补充锚点：快照根 `package.json` 的 SHA-1 为 `16918dcd04afbb809d896844bd04c3cb3bd3d301`（在快照根执行 `shasum package.json` 可复算）。
- 材料范围：map+采样。本篇精读 `packages/core/` 六包（session、system-prompt、tools、agent、agent-loop、scope）的全部 src，扫读 `agent-default-model` 与 `agent-tool-presentation`，对照 `docs/architecture.md`、`docs/agent-lifecycle.md`、`docs/subsystems/` 五页与组合证据 `packages/examples/agent-spine-demo/`。
- 读者画像：已读第 01 篇（Cordis 垫层：ctx / service / effect / 事件 / waterfall / loader）的工程师；想弄清"一句话发出去之后，系统里到底发生了什么"。
- 本篇子锚点：你向一个 headless agent 发出一句「帮我跑下测试」——从这句话进 inbox，到 `turn/end` 落盘。

## 0. 阅读地图

core/ 六包是 dsh 的产品脊柱（spine）：session 管日志，system-prompt 管提示装配，tools 管工具，agent 管接口与注册，agent-loop 管驱动，scope 管 per-agent 作用域。本篇主线只有一个问题：一句话从进来到出去，每一步谁在干活、数据变成什么。叙事按主场景 S1（一次用户请求的完整生命周期）走双通道——控制流讲谁调用谁，数据流讲每步落进日志的是什么。读完你应能对着一条真实日志，逐个事件复述一轮 turn。

## 1. 背景与前置知识

上一篇（第 01 篇）建立了 Cordis 心智模型：插件往共享 ctx 上注册服务、类型化事件和可逆 effect；waterfall 监听器必须 `next()` 委托。本篇直接使用这些语义，不再重复。全书术语表里的词——turn / step、inbox、SessionEvent / SessionEventMap、Agent / AgentHandle / AgentRegistry、scope、system-prompt sections、scoped tool registry、runtime invariant、model-visible ⟺ logged——也直接引用，不重复定义。术语表没收、但本篇承重的四个新概念（surface、request/header、staged 工具管道、initiator）在各自首次出现处就地定义。

三个阅读约定：

- `file:line` 锚点全部相对仓库根，例如 `packages/core/agent-loop/src/agent.ts:255`。
- 行为论断分三层：**已验证（跑过）**、**静态推断**（带锚点）、**未验证**。本篇的"已验证"依据是两条真实跑过的命令：`pnpm vitest run packages/core/agent-loop packages/core/agent packages/core/session packages/core/system-prompt packages/core/scope`（46 个测试文件、815 个测试全过）与 `pnpm vitest run packages/core/tools`（12 个文件、386 个测试全过），合计 1201 个测试通过。
- 双通道叙事：每个场景段落先给控制流（谁调用谁、何时分支），再给数据流（什么数据进、变成什么出）。

锚点登场。你执行 `dsh --profile headless "帮我跑下测试"`。headless 组合把 `packages/examples/agent-spine-demo` 这棵插件树挂起来，里面有一个预先配好的 agent（通常叫 `main`）、它的 session、一条还空着的事件日志。你的这句话被包装成一个 `UserMessage`——带唯一 `id`、标记来源的 `source`、和文本 `content`——交给这个 agent。接下来的整篇，就是这句话的旅程。

## 2. 对象全景：core 六包与模块地图

第 1 节立了锚点，但还缺地图：这句话要经过哪几个包、按什么顺序。本节只到组件级，不进代码。

| 包 | `ctx` key | 一句话职责 | 依赖谁（import 证据） |
|---|---|---|---|
| `core/scope` | 无（纯库） | per-agent 作用域原语：`createScope`/`scopeOf`/`scopeTarget` | 无 harness 内依赖（`packages/core/scope/package.json` dependencies 为空） |
| `core/session` | `ctx.sessions` | append-only SessionEvent 日志 + 内存 store | scope、llm（`docs/module-graph.md:1473`） |
| `core/system-prompt` | `ctx.systemPrompt` | prompt 分段与工具 schema 装配 | scope、llm（`docs/module-graph.md:1474`） |
| `core/agent` | `ctx.agents` | Agent 接口、活注册表、`agent/*` 事件、inbox | scope、session、system-prompt、llm（`docs/module-graph.md:1478`） |
| `core/tools` | `ctx.tools` | scoped 工具注册表 + 带守卫的执行管道 | agent、scope、session、system-prompt、llm（`docs/module-graph.md:1517`） |
| `core/agent-loop` | `ctx.agentLoop` | 默认驱动器：实现 Agent 接口、跑 turn/step 循环 | agent、scope、session、system-prompt、tools、llm（`docs/module-graph.md:1539`） |

这张依赖边表有个值得停一秒的形状：**驱动器恰好是依赖图的顶点**。agent-loop 依赖所有注册表，因为它每跑一步都要摸一遍它们；而扩展插件（llm-retry、goal、commands、user-approval、jobs……）的依赖边里只有 `agent`，没有 `agent-loop`——`docs/subsystems/core.md:20` 把这条写成纪律："Extension plugins depend on `agent` … and never on `agent-loop` directly, so the loop stays swappable." loop 可替换不是口号，是依赖方向换来的：接口（Agent、AgentFactory、`agent/*` 事件）住在 `dsh-agent`，实现住在 `dsh-agent-loop`，换 loop 等于换一个实现 AgentFactory 的插件。

scope 在最底层且是六包里唯一不是服务的包：它不挂 `ctx` key，只导出函数。原因很实际——session 和 system-prompt 都要消费它，它若也是服务就会产生循环依赖；`docs/subsystems/core.md:20` 明说它"sits below `session/` and `system-prompt/` in the module graph precisely so they can consume it without a cycle"。

这六个包不会自己出现在你的进程里，是组合层把它们装配起来的。证据在 spine demo 的 `apply()`：`packages/examples/agent-spine-demo/src/index.ts:212-265` 依次 `ctx.plugin(...)` 挂上 LlmRuntime、SessionStore、SystemPrompt、ToolRuntime、AgentRegistry、四个 invariant companion，最后挂上 AgentLoop 并把 `agents` 列表转发给它（`packages/examples/agent-spine-demo/src/index.ts:261-264`）。它的 JSDoc 特意写清：挂载顺序无关，因为 Cordis 会把每个 fiber 挂起在它的 `inject` 声明的服务出现之前（`packages/examples/agent-spine-demo/src/index.ts:203-211`）。

## 3. 主线叙事 S1：「帮我跑下测试」的一轮 turn

地图有了，现在让锚点场景跑起来。先把整段旅程压缩成两行，然后逐段放大。

控制流总览：`agent.followup()` → `Inbox` 入队 → `wakeDriver()` → `kick()` → `turn()`（turn/start）→ `preStep()`（claim → assemble → pre-step 瀑布）→ `step()`（buildRequest → llm/stream → chunk 落盘 → 工具调度）→ `agent/turn-stopping` → turn/end。

数据流总览（即日志里会出现的事件序列）：`agent/inbox/spliced` → `turn/start` → `user/message` → `request/header` → `assistant/chunk`* → `assistant/message` → `tool/call` → `tool/result` → `step/end` → `turn/end`。

### 3.1 入口：followup 与 inbox

**chunk：`dsh-agent`——Agent 公共接口、活注册表与 `agent/*` 事件词汇的包。** inbox 就住在这里：它不是驱动器的私有队列，而是 agent 拥有的一份 **durable 投影**——待办消息列表的每一次变更先写进日志，再改内存。

控制流。你的「帮我跑下测试」到达 `agent.followup(message)`。它是 `send(message, 'next-turn', true)` 的固定别名（`packages/core/agent-loop/src/agent.ts:122-124`）：先把消息 splice 进 `next-turn` 队列，然后因为 `wakeup=true` 调 `wakeDriver()`（`packages/core/agent-loop/src/agent.ts:113-120`）。驱动器此刻 idle，于是 `wakeDriver` 把相位（Phase）切成 `running`——相位是驱动器的状态机，只有 `idle` / `maintenance` / `running` 三种（`packages/core/agent-loop/src/agent.ts:38-46`）——发出 `agent/status` 事件，再在 initiator 边界里启动 `kick()`（`packages/core/agent-loop/src/agent.ts:172-193`）。

initiator 是这里第一个新概念：**进程内的"发起者"归因**。`ctx.agents.withInitiator(agent, op)` 用 `AsyncLocalStorage` 把"这条异步链是谁发起的"传到链的每一处（`packages/core/agent/src/index.ts:341-343`），工具调度器后面就靠 `requireInitiator()` 找回这个 agent（`packages/core/agent-loop/src/tool-calls.ts:67`）。文档对它很克制：ambient presence 既不是存活证明也不是授权（`packages/core/agent/src/index.ts:244-255`）。

数据流。消息进队列前，先落一条 `agent/inbox/spliced` 事件：`session.append(...)` 在前，`inbox.splice(...)` 改内存在后（`packages/core/agent/src/inbox.ts:186-187`）。这个顺序是刻意的——同步的 `session/event` 观察者能看到 splice 之前的队列，配合事件里的规范化坐标自己重建被移除的消息（`packages/core/agent/src/inbox.ts:129-138` 的 JSDoc）。进队成功后再发 `agent/inbox/inserted` 通知。也就是说，**inbox 的状态可以从日志重放**：构造函数把 `seedLength` 之后的所有 splice 事件逐个应用一遍（`packages/core/agent/src/inbox.ts:32-39`），进程重启后待办消息不丢。

三种输入方式的区别全在 `send` 的两个参数上（`packages/core/agent-loop/src/agent.ts:113-132`）：

| 方法 | 目标队列 | 唤醒驱动 | 语义 |
|---|---|---|---|
| `followup()` | `next-turn` | 是 | 排队一个普通后续 turn，它独占自己的一轮 turn |
| `steer()` | `next-step` | 是 | 打方向：运行中的驱动在下一个 step 边界消费它 |
| `inject()` | `next-step` | 否 | 注入模型可见上下文，排队等别的消息来唤醒 |

`inject` 不唤醒就是 `docs/architecture.md:86` 那句"injected context waits in the inbox until another message does"的落地（已验证（跑过）：`packages/core/agent-loop/tests/loop.spec.ts` 与 `interception.spec.ts` 覆盖这些路由）。

**chunk：`dsh-scope`——per-agent 作用域原语。** 它在场景里的第一次露面很隐蔽：`ReactLoopAgent` 构造时用 `agentEvents(loopCtx, this)` 建了一个"融合派发器"（`packages/core/agent-loop/src/agent.ts:86`），之后每次发 `agent/*` 事件都经过它。融合的意思是：事件的 scope 载体（一个只为路由存在的 `this`）和 payload 里的 agent 主体被绑死成同一个对象，两者不可能分叉（`packages/core/agent/src/dispatch.ts:107-149`）。scope 的完整机制在第 4.2 节拆。

### 3.2 turn 开启与 claim

**chunk：`dsh-agent-loop`——默认驱动器。** `kick()` 是个 `while (await this.turn()) {}` 循环（`packages/core/agent-loop/src/agent.ts:210-223`）：每跑一轮 turn，`turn()` 返回"还有没有排队工作"，有就立刻开下一轮。

**chunk：`dsh-session`——事件溯源日志与内存 store。** `turn()` 的第一件事是往日志追加 `turn/start`（`packages/core/agent-loop/src/agent.ts:255`）。注意次序：**turn 边界先于任何输入认领**——这对应 `docs/architecture.md:65` 的"a turn opens before its first input is claimed"。turn 号也不从内存计数，而是从日志里找最后一个 `turn/start` 续号（`packages/core/agent-loop/src/agent.ts:92`），所以 resume 之后编号连续（静态推断，锚点即此行；resume 路径已被 `packages/core/agent-loop/tests/resume.spec.ts` 覆盖）。

接着进入 step 循环，第一步是 `preStep()` 里的 **claim**（`packages/core/agent-loop/src/agent.ts:229`）：从 inbox 取走本 step 的输入批次。规则是"全部 next-step 输入，加上（仅在 turn 边界）恰好一条 next-turn 消息"——正是 `docs/architecture.md:69` 的"claim next-step input plus one queued message"。你的「帮我跑下测试」此刻离开 `next-turn` 队列，日志里多一条纯删除的 `agent/inbox/spliced`，并收到 `agent/inbox/claimed` 通知（`packages/core/agent/src/inbox.ts:71-78`）。claim 是纯删除：被取走的消息不算"丢弃"，不发 `discarded`——它即将被 turn 拥有。

数据流上，claim 把消息从"待办"变成"本 turn 的输入候选"，但还没有进模型历史。它能不能进，要过下一关。

### 3.3 prompt 装配与 pre-step

**chunk：`dsh-system-prompt`——prompt 分段与工具 schema 的装配器。** claim 之后、瀑布之前，驱动先调 `ctx.systemPrompt.assemble(...)`（`packages/core/agent-loop/src/agent.ts:230`）。装配是**每个 step 重做一次**的，没有跨 step 缓存：所有注册的 section、context、tool provider、variable 都在这一次调用里重新求值（`packages/core/system-prompt/src/index.ts:467-542`）。这回答了一个自然会问的问题：热插拔一个贡献 prompt 的插件，下一个 step 就生效——因为根本没有旧装配可复用。

装配产物是四样东西：排序后的 prompt sections（渲染成 system prompt）、动态 context sections（渲染成"运行时上下文快照"）、规范排序的工具 schemas、变量表。驱动把动态上下文交给 `RuntimeContextProjection.project()`（`packages/core/agent-loop/src/agent.ts:233`）：它记住上一次随请求发出的快照文本，**只有内容变了才生成一条新的快照消息**（`packages/core/agent-loop/src/runtime-context.ts:64-75`），没变就什么都不发——上下文快照是 user 角色的 durable 消息，发多了就是浪费 token。

然后轮到 `agent/pre-step` 瀑布（`packages/core/agent-loop/src/agent.ts:234-240`）。这是模型之前最后一个拦截点：监听器拿到 claimed 批次，可以改写它（`enter(messages)`），也可以整体拒绝（`reject`）。默认的 `next()` 返回 `enter`：claimed 消息加上（有变化时的）上下文快照。

拒绝语义是这个系统"日志即真相"原则的最好例证。看 `docs/architecture.md:88` 的宣称："a rejected or empty first claim still closes a durable turn that spent no step, so the log records the attempt." 代码逐字兑现：reject → `turnEnds = { kind: 'blocked' }`，直接收尾（`packages/core/agent-loop/src/agent.ts:267-269`）；首个 step 的 enter 被改写成空消息列表 → `turnEnds = { kind: 'completed' }`，同样收尾（`packages/core/agent-loop/src/agent.ts:274-277`）。两条路都不开 step、不发模型请求，但 `turn/start` 已经落盘、`turn/end` 在 `finally` 里必定补上（`packages/core/agent-loop/src/agent.ts:317-322`）——**这次尝试在日志里永远存在**（已验证（跑过）：`packages/core/agent-loop/tests/interception.spec.ts`）。后续还有一个配套账本：`foldConsumedWork()` 专门从日志折出"哪些输入被消费了、哪些被取消丢弃"（`packages/core/agent/src/consumed-work.ts:68-108`），因为光读 turn 边界分不清"被拒"和"白跑"。

回到锚点：如果你的组合里挂了审批或守卫类插件，`agent/pre-step` 是它对你的这句话说"不"的第一个机会。默认组合里没有人拒绝，于是 `step/start` 落盘（`packages/core/agent-loop/src/agent.ts:279`），你的消息作为 `user/message` 追加进日志——带 `surfaceOp: 'append'` 标记（`packages/core/agent-loop/src/agent.ts:282-284`）。从这里起，它正式进入模型可见历史。

### 3.4 step：一次模型请求

surface 是这里必须先定义的概念。**surface（表面）是日志上的一层有序视图**：只有三种产生消息的事件（`user/message`、`assistant/message`、`tool/result`）能上去，每个上去的事件都必须声明自己怎么上去——尾部追加（`append`）或遮蔽替换（`replace`）（`packages/core/session/src/types.ts:343-389`）。turn/step 边界、chunk、log-only 记录都留在日志里但不上 surface。模型历史不从日志直接推，而是从 surface 推——这样 compaction 才能用一次 `replace` 把旧节点遮蔽掉，而日志本身一字不动。

`Session.deriveMessages()` 就是 surface 的折叠：按 surface 节点顺序，逐节点套投影规则 `deriveEventMessage()`——user/message 原样投影、空内容的 assistant/message 跳过（它只为承载 usage 而存在）、tool/result 投影其 message、其余事件一律 `null`（`packages/core/session/src/surface.ts:83-114`）。结果带增量缓存：每个节点只投影一次，发生 `replace`（代际号变化）才整体重建（`packages/core/session/src/index.ts:726-747`）。

step 的控制流从 `buildRequest()` 开始（`packages/core/agent-loop/src/agent.ts:407-495`）。它以日志里折叠出的上一个 `request/header` 为种子配置——这是第二个新概念：**`request/header` 事件记录一次请求的完整头部**（call config、渲染后的 system prompt、工具 schemas），最新一份快照就能重建任一请求是在什么头部下发出的（`packages/core/session/src/request-header.ts:65-71`）。种子配置过 `agent/request` 瀑布，插件可以整体换配置（比如换模型）；然后 `ctx.llm.prepareCall()` 把路由绑到具体 adapter；头部有变化就追加一条新的 `request/header`（initial / resume / change 三种原因，`packages/core/agent-loop/src/agent.ts:465-470`）。最终请求被 `deepFreeze` 并打上 loop 标记（`packages/core/agent-loop/src/agent.ts:486-493`）——冻结不是洁癖，是不变量的前提。

请求发出后，`ctx.llm.stream(request)` 返回异步 chunk 流。驱动逐 chunk 做两件事：追加 `assistant/chunk` 事件、喂给 `BlockAssembler`（`packages/core/agent-loop/src/agent.ts:347-351`）。**每个原始 chunk 都落盘**——这是 replay 和 UI 保真的来源：`docs/architecture.md:94` 说"raw `assistant/chunk` events preserve replay and UI fidelity"，代价是日志体积，收益是 token 级的时间线可以逐帧重放。流结束后，组装出的 assistant 消息连同全部 chunk 的 seq 一起作为 `assistant/message` 落盘（`sourceEventSeqs: chunkSeqs`，`packages/core/agent-loop/src/agent.ts:381-390`）——消息和它的原材料在日志里显式相连。

失败路径同样是结构化的。流以 error/aborted 收尾时，驱动先发 `agent/request-error` 瀑布：有监听器返回 `{ kind: 'retry' }` 就原地重试（`continue`，同一 step 号），否则把失败包装成 `LlmError` 抛出（`packages/core/agent-loop/src/agent.ts:354-370`）。抛到 `turn()` 边界后，任何非 `LlmError` 被压平成 `{ message: errorChain(error), code: 'UNKNOWN' }`，写进 `turn/end` 的 reason（`packages/core/agent-loop/src/agent.ts:307-315`）。llm-retry 插件就是挂在 `agent/request-error` 上的（`packages/llm/llm-retry/src/index.ts:210`）——重试是一个插件，不是 loop 的内建行为。

model-visible ⟺ logged 在哪里断言？就在请求离开进程前的最后一厘米。agent-loop 的 invariant companion 以 `{ global: true, prepend: true }` 挂在 `llm/stream` 瀑布上（`packages/core/agent-loop/src/invariant.ts:21`），对每个 loop 发出的请求检查：请求对象已冻结、携带活跃 sessionId、日志里有 `step/start` 和 `request/header`、**`options.messages` 与 `session.deriveMessages()` 的 JSON 逐字节相等**、请求各字段与折叠出的 header 一致（`packages/core/agent-loop/src/invariant.ts:23-52`）。任何一项不符即判 invariant 失败。这条 prepend 注释说明了为什么放最前："Prepend prevents a short-circuiting replay listener from silencing the check"（`packages/core/agent-loop/src/invariant.ts:20`）——重放类监听器会短路瀑布，检查必须比它还先。

### 3.5 工具调用瀑布

**chunk：`dsh-tools`——scoped 工具注册表与带守卫的执行管道。** 模型在 assistant 消息里回了 `bash` 的 tool-call 块（「我帮你跑 `pnpm test`」），驱动把本 step 的全部 tool-call 块交给 `executeToolCalls()`（`packages/core/agent-loop/src/agent.ts:393-398`）。

这里出现第三个新概念：**staged（分段）工具管道**。agent-loop 的调度器和 tools 注册表之间不是一次 `execute()` 调用，而是一个四段内部接口 `ToolRuntimeScheduler`：`prepare → dispatch → finalize / finish`（`packages/core/tools/src/index.ts:451-464`，符号 key `TOOL_RUNTIME_SCHEDULER` 在 `packages/core/tools/src/index.ts:466`）。分段的意义在于让调度器把"策略"和"并发"分开：pre-execute 策略按模型顺序串行等，工具体按并发模式重叠跑，结果再按模型顺序提交。

并发模式由工具自己声明：定义里的 `isConcurrencySafe(args)` 返回严格的 `true` 才进 parallel 组，其余一律 exclusive（fail-closed，`packages/core/tools/src/index.ts:1276-1285`）。调度器按模式分组：exclusive 调用是屏障，parallel 组进一个有界滚动池（上限默认 10，`packages/core/agent-loop/src/constants.ts:6`），且每提交完一批都重新分类未启动的调用——注册表在运行中被热改也不会冲破屏障（`packages/core/agent-loop/src/tool-calls.ts:84-99`、`198-213`）。

单个调用穿过注册表的旅程（对照 `docs/architecture.md:77`，逐步都有代码）：

1. `tool/call` 落盘，记下事件 seq（`packages/core/agent-loop/src/tool-calls.ts:167`、`262-265`）。模型的原始 arguments 字符串原样进日志；解析失败时保留原文本而不是报错（`packages/core/agent-loop/src/tool-calls.ts:104-110`）。
2. `prepare` 段：先跑 `tools/pre-execute` 瀑布——allow / deny / ask（`packages/core/tools/src/index.ts:1475-1481`）。ask 会转给 approval seam 问人；deny 或守卫（guard）拒绝则物化成一条错误结果，不再进 dispatch（`packages/core/tools/src/index.ts:1486-1499`）。审批就缝在这里，不需要改 loop。
3. `dispatch` 段：`tools/execute` 瀑布包裹工具体（`packages/core/tools/src/index.ts:1573-1575`），超时、重试、指标都挂这一层。包装器只能替换 `exec.signal`，注册表在进工具体前把调用方原始信号重新融合回去（`packages/core/tools/src/index.ts:1532-1560`、`1889-1916`）——取消不可能被包装器"弄丢"。
4. `finalize` 段：`tools/post-execute` 瀑布可以接受、替换或拦截结果（`packages/core/tools/src/index.ts:1742-1781`）；然后工具自有的 `finalizeContent` 做最后一公里内容变换，物化深冻，发 `tools/result` 通知（`packages/core/tools/src/index.ts:1631-1676`）。
5. 调度器按模型顺序提交：`tool/result` 落盘，`sourceEventSeqs` 指回它的 `tool/call`（`packages/core/agent-loop/src/tool-calls.ts:146-160`、`268-289`）。

两个值得知道的收尾语义。其一，工具可以宣布"这轮够了"：工具体里调 `exec.concludeTurn()`，结果带上 `concludesTurn: true`，step 就以 completed 结束而不再续模型请求（`packages/core/tools/src/index.ts:1394-1396`、`packages/core/agent-loop/src/agent.ts:395-399`）。其二，取消不打烂日志：abort 时已启动的调用正常结算，未启动的每个调用补一条合成的 `tool/call` + 错误 `tool/result`（`packages/core/agent-loop/src/tool-calls.ts:237-259`）——replay 时每个 call 都有 result，transcript 永远合法。

### 3.6 turn 收尾

`step/end` 在 `finally` 里落盘（`packages/core/agent-loop/src/agent.ts:291-293`）——不管 step 成功、失败还是被取消，边界一定闭合。

之后驱动判断"还欠不欠东西"：工具有没有要求续请求（step 返回 null）、next-step 队列空不空。都不欠了，先跑 `agent/turn-stopping`——serial 事件，没有 `next()`（`packages/core/agent-loop/src/agent.ts:295-298`）。它是 turn 关上前的最后一个检查点：反对的监听器可以当场 `agent.steer(...)` 塞新输入，跑完后驱动**重读 inbox** 再决定（`packages/core/agent-loop/src/agent.ts:299`）。事件 JSDoc 把设计原则写明了："Data decides, so listener order cannot change the outcome"（`packages/core/agent/src/runtime-types.ts:261-278`）——结果由队列里有没有数据决定，不由谁先说话决定。

最后 `turn/end` 落盘，reason 是闭合枚举：`completed` / `blocked` / `aborted` / `error` / `max-tokens` / `interrupted`（`packages/core/session/src/types.ts:155-174`；`interrupted` 是持久化层崩溃修复专用，loop 自己从不发出）。`max-tokens` 是粘性的：任何一个 step 撞过输出上限，后续 step 正常完成也不能把 turn 结果降级回 completed（`packages/core/agent-loop/src/agent.ts:285-290`）。

驱动回到 idle：如果排队里还有工作，`turn()` 返回 true，`kick()` 立刻开下一轮 turn；没有则相位切回 idle，期间被闩住的唤醒在这里重放（`packages/core/agent-loop/src/agent.ts:210-223`、`324-329`）。

复盘锚点。你的「帮我跑下测试」走完一轮（模型调了一次 bash 工具然后总结），日志里的事件序列是：

1. `agent/inbox/spliced`（进 next-turn 队列）
2. `turn/start { turn: 1 }`
3. `agent/inbox/spliced`（claim 纯删除）→ 你的消息被 `agent/inbox/claimed`
4. `step/start { turn: 1, step: 1 }`
5. `user/message`（你的这句话，surfaceOp: append）
6. `request/header { reason: 'initial' }`（+ 可能的 `request/context`）
7. `assistant/chunk` × N（模型的每个流式片段）
8. `assistant/message`（含 bash tool-call 块，sourceEventSeqs 指回全部 chunk）
9. `tool/call { name: 'bash', … }`
10. `tool/result`（测试输出，sourceEventSeqs 指回 9）
11. `step/end` → 工具欠一次续请求 → `step/start { step: 2 }` → 又一轮 chunk/message → 模型给出总结、无 tool-call
12. `step/end`，`agent/turn-stopping`（无人反对）
13. `turn/end { reason: { kind: 'completed' } }`

拿着这份序列对照任何一条真实 session 日志，你就能逐事件讲清系统在干什么——这就是本篇的验收标准。

## 4. 模块详解：注册表、装配与所有权

主线跑完了，但有四个机制在场景里只露了一面：agent 怎么被创建和拥有、scoped 注册怎么工作、注册表内部长什么样、日志的存储与修复。本节补齐。

### 4.1 Agent 所有权与创建事务

`ctx.agents.get(id)` 返回的是裸 `Agent`；真正危险的东西——`dispose()`——只通过 `AgentHandle` 交给创建者。JSDoc 用大写强调："The disposer is a CAPABILITY: among consumers, only the holder can tear this agent down"（`packages/core/agent/src/index.ts:158-175`）。注册表本身不创建 agent：创建能力由 `AgentFactory` 接口定义，留在 `dsh-agent`（`packages/core/agent/src/index.ts:183-214`），agent-loop 在构造时 `ctx.agents.setFactory(this)` 注册实现（`packages/core/agent-loop/src/index.ts:350`）。消费者（比如 ACP 桥）只对 `ctx.agents` 编程，编译期完全不依赖 agent-loop——这就是"loop 可替换"在 import 上的形状。

创建是一桩事务，核心在 `prepare()`（`packages/core/agent-loop/src/index.ts:459-578`）。它先融合三路取消（调用方 signal、owner fiber 卸载、工厂 teardown），再构造 `ReactLoopAgent`，然后返回一个 `PreparedAgent`：`publish()` 负责按序进注册表（session 先入 agent 的 scope ctx，agent 再入全局注册表）、announce、发 `agent/session-start`；`dispose()` 是记忆化的逆向拆除——停机器、等静默、退注册、解开 scope。关键点在时序：**teardown 在任何资源存在之前就注册好了**，所以 setup 中途插件卸载也能完整回滚，不留半个 agent。

`CreateAgentOptions.setup` 回调在"两个 id 都未发布"的状态下组合 agent 的 scoped 世界（注册 scoped 工具、prompt section、监听器）；它抛错或它返回的 `commit()` 抛错，`setupAndPublish` 直接 `await prepared.dispose()` 然后重抛——外部永远看不到一个配置了一半的 agent（`packages/core/agent-loop/src/index.ts:625-645`，已验证（跑过）：`packages/core/agent-loop/tests/agent.spec.ts`、`scope-lifecycle.spec.ts`）。resume 多一层持久化加载屏障，加载与三路取消 race，加载完还要复检工厂仍活跃（`packages/core/agent-loop/src/index.ts:662-710`）。

工厂自己也管一本子账：`FactoryOwnership` 跟踪所有活 agent 的 teardown 与启动期任务，工厂卸载时全部一起拆（`packages/core/agent-loop/src/index.ts:40-90`）。

### 4.2 scope 原语：一个 ctx 标签如何变成 per-agent 世界

scope 的全部家当是三个函数加一组存储类。`createScope(ctx, key)` 在调用方 ctx 下挂一个空插件 fiber，把不透明 key 写进 ctx 的 symbol 属性（`packages/core/scope/src/index.ts:137-147`）；`scopeOf(ctx)` 读最近的标签（`packages/core/scope/src/index.ts:154-156`）。ReactLoopAgent 用 agent 对象自己当 key（`packages/core/agent-loop/src/agent.ts:94`），于是"这个 agent 的 scope"就是"以这个 agent 为 key 的注册上下文"——`agent.ctx` 上的所有注册（scoped 工具、prompt section、监听器）都跟着 agent 的处置一起回滚。

事件侧由 `scopeTarget(base, key)` 完成：它造一个只有路由功能的载体对象，Cordis 派发时问它"这个监听器该不该收"。许可规则是：**未打标签的监听器全收；打了标签的监听器，只当它的 key 是派发 key 或其祖先时收**（`packages/core/scope/src/index.ts:170-185`）。事件沿 scope 链向上流、不向下流——这就是"一个 standing 组合可以观察它名下的每个 agent"的机制（父子 scope 由 `bindScopeParent` 连边，带环检查，`packages/core/scope/src/index.ts:72-82`）。

注册侧由 `ScopedLayers` 完成：一个急切创建的全局层 + 按需创建的 scoped 层；读操作沿 scope 链合并，**近层同名遮蔽远层**（`packages/core/scope/src/store.ts:208-217`）。注册动作同时决定可见性和 effect 所有权，返回 Cordis 的精确 disposer（`packages/core/scope/src/store.ts:226-266`）。system-prompt 的 section 遮蔽、tools 的 scoped 注册与 restrict，全部建立在这一个原语上。还有一道保险：scope 包的 invariant 会在每次 scope 事件派发时检查载体与 payload 主体同键（`packages/core/scope/src/invariant.ts:16-33`）。

### 4.3 system-prompt 注册表

`ctx.systemPrompt` 收四类贡献，都按 scope 分层：section（prompt 分段，按 `order` 升序拼接，约定 -100 是 harness 身份、0 是部署 persona、100–199 是工具引导）、context（动态运行时上下文）、tools（工具 schema provider）、variable（`{{name}}` 插值变量）（`packages/core/system-prompt/src/index.ts:53-85`）。装配时 scoped 遮蔽全局、sections 稳定排序、工具 schema 深拷贝后按 `toolOrder` 或字典序排列（`packages/core/system-prompt/src/index.ts:467-542`、`164-178`）。

两个容易踩的硬规则。变量插值是严格的：引用一个没注册或本次装配无值的变量直接抛错（`packages/core/system-prompt/src/index.ts:258-295`）——拼错变量名在装配期爆炸，不会静默变成空串进 prompt。`complete: true` 的 section 是"完整 prompt"声明：装配照常跑协作瀑布（工具、变量要解析），但最后把这个 section 恢复为唯一的 prompt 段，且同时存在两个 complete 就直接失败（`packages/core/system-prompt/src/index.ts:505-541`）。装配结果本身也有一条 invariant 兜底：section/context 名非空不重复、变量名合法（`packages/core/system-prompt/src/invariant.ts:16-52`）。

agent-loop 往这里注册的三个变量——`provider`、`model`、`cwd`（`packages/core/agent-loop/src/index.ts:351-353`）——是 persona 模板里 `{{model}}` 这类引用的来源。

### 4.4 tools 注册表：注册、限制与视图

`ctx.tools.register(definition)` 全局或 scoped 注册工具；`ToolDefinition` 在 schema 之外必须带 `output` 声明（输出 schema + 两个纯投影 `render`/`presentationMeta`），还可以带超时预算、并发分类器、UI 投影（`packages/core/tools/src/index.ts:212-288`）。`run_code` 是保留名，任何注册或遮蔽都被拒（`packages/core/tools/src/index.ts:1051-1056`）——它是 Code Mode 的传输工具，属于展示基础设施，不在能力过滤范围内（Code Mode 的工具管道属第 04 篇范围，本篇不展开）。

`restrict({ allow, deny })` 只能 scoped 调用：给一个 agent 的继承工具面做减法，全局限额会遮蔽所有 agent 所以被显式拒绝（`packages/core/tools/src/index.ts:1071-1098`）。`guard(fn)` 是 pre-execute 之后的单调守卫：任何守卫可以拒绝，没有守卫能翻案别人的拒绝（`packages/core/tools/src/index.ts:1110-1128`）。

一个 scope 实际看到什么工具，由 `view()` 一次遍历算出：继承面（全局 + 祖先链，近层遮蔽远层）先被各层 restriction 求交集过滤，再叠加本 scope 自己的注册（豁免过滤——否则子 agent 的能力过滤会把它的汇报工具也滤掉），最后按展示模式决定要不要附上保留传输工具（`packages/core/tools/src/index.ts:1152-1193`）。这段 JSDoc 里记着一次真实的教训：把"豁免集"误读成"全局层"在 preset 把工具搬上 agent 平面后曾让过滤静默失效——值得作为"继承语义"复杂性的标本一读。

### 4.5 session 存储、fork 与崩溃修复

`SessionStore`（`ctx.sessions`）只管内存：持久化是插件的事——订阅 `session/event` 流，在 `session/flush` 检查点落盘（`packages/core/session/src/index.ts:786-791` 的类注释）。创建有两条路径：`create()` 一步进 store 并 announce；`prepare()` + `enter()` + `announce()` 三步拆开，给 agent 工厂把 session 生命周期折进自己那一个 effect 里、保证拆除顺序的机会（`packages/core/session/src/index.ts:830-861`）。

`fork(source, boundary?)` 从活 session 的稳定前缀造子 session：boundary 可以指向任何 between-turn 位置，落在打开中的 turn 里就拒绝而不是悄悄截断（`packages/core/session/src/index.ts:1081-1138`）。日志头部 `SessionHeader` 存的是存储元数据（格式版本、cwd、fork 谱系、seed 边界、委派深度），刻意不进事件日志（`packages/core/session/src/types.ts:58-99`）；格式版本 `SESSION_FORMAT_VERSION` 预发布期间钉死在 0，不兼容即拒（`packages/core/session/src/types.ts:56`）。

进程崩溃在半截 turn 上怎么办？`interruptedTurnClosers()` 扫一遍加载出的日志，为悬空的 tool-call 合成错误结果（区分"记录后失联"与"从未开始"，前者警告模型不要盲目重试有副作用的操作）、补上 `step/end`、以 `{ kind: 'interrupted' }` 关闭 `turn/end`（`packages/core/session/src/repair.ts:27-133`）。合成事件的 timestamp 复用最后一个真实事件——确定性，也绝不"发明未来"。

### 4.6 invariant 体系：五道机器执行的防线

每个包带一个 `./invariant` companion，由 spine 统一挂载（`packages/examples/agent-spine-demo/src/index.ts:246-249`）。它们断言的不是"服务在不在"，而是**事件流与数据之间的关系**：

- `dsh-session`：turn/step 编号单调、闭合配对；step 内事件归属当前打开的 turn/step；tool/result 必有本 step 的 tool/call 在先（`packages/core/session/src/invariant.ts:55-166`）。
- `dsh-agent-loop`：model-visible ⟺ logged（第 3.4 节已拆，`packages/core/agent-loop/src/invariant.ts:19-55`）。
- `dsh-tools`：管道阶段单调（pre → execute → post 不重复不倒序）、最终快照冻结、code-dispatch 事件只在打开的 turn 内（`packages/core/tools/src/invariant.ts:84-119`）。
- `dsh-agent`：`agent/status` 不出现无变化迁移（`packages/core/agent/src/invariant.ts:15-24`）。
- `dsh-scope`：scope 事件派发必须带载体且载体与主体同键（`packages/core/scope/src/invariant.ts:16-33`）。

这些检查是运行时行为，不是类型体操：它们挂在事件流上，关系破了就上报。约 57 万行 monorepo 的纪律有一部分就是这样机械执行的。

### 4.7 两个扫读包

`agent-default-model`（`packages/core/agent-default-model/src/index.ts:64-105`）：一个极薄的 settings 持有者，给"创建时没指定模型的 agent"提供默认 provider/model/effort，用户层 settings 实时可读。`agent-tool-presentation`（`packages/core/agent-tool-presentation/src/index.ts:59-72`）：preset 组合里的一行，声明"这个组合下所有 agent 的模型看到哪种工具形态"（native / code / both），code 模式会显式等待 `codeRuntime` 服务存在——配置错误在挂载时爆炸，而不是在第一次 prompt 时。两者都是"组合层怎么不改代码重塑产品"的小标本。

## 5. 关键制品拆解链

模块都见了，最后把五个枢纽制品全文贴出逐行拆：turn 驱动器、step 驱动器、inbox 认领、历史投影、工具调度器，外加那道请求重建不变量。

### 5.1 `ReactLoopAgent.turn()`——一轮 turn 的状态机

来源：原文，`packages/core/agent-loop/src/agent.ts:246-330`。

```ts
  /** Open one turn before claiming its first proposed step. */
  private async turn(): Promise<boolean> {
    if (this.phase.kind !== 'running') {
      this.throwError(new Error(`agent "${this.id}": turn without driver reservation`))
    }
    const phase = this.phase
    const { signal } = phase.abort
    signal.throwIfAborted()
    const turn = phase.turn + 1
    try {
      this.session.append('turn/start', { turn })
    } catch (error: unknown) {
      this.throwError(error)
    }
    phase.turn = turn
    let turnEnds: TurnEndReason | null = null
    let target: InboxTarget = 'next-turn'
    try {
      while (true) {
        signal.throwIfAborted()
        const step = phase.step + 1
        const decision = await this.preStep(target, { turn, step })
        if (decision.kind === 'reject') {
          turnEnds = { kind: 'blocked' }
          return false
        }
        if (turnEnds && decision.messages.length === 0) break
        // A removed waking message or an enter decision rewritten to empty
        // still owns the initial turn boundary, but it spends no model call.
        if (phase.step === 0 && decision.messages.length === 0) {
          turnEnds = { kind: 'completed' }
          return false
        }
        signal.throwIfAborted()
        this.session.append('step/start', { turn, step })
        phase.step = step
        try {
          for (const message of decision.messages) {
            this.session.append('user/message', message, { surfaceOp: 'append' })
          }
          // max-tokens is sticky: once any step hits the ceiling, later steps
          // that complete normally must not downgrade the turn outcome.
          const stepEnd = await this.step(decision.assembly)
          // max-tokens stays sticky: a later completed step must not
          // downgrade the turn outcome.
          if (turnEnds === null || turnEnds.kind !== 'max-tokens') turnEnds = stepEnd
        } finally {
          this.session.append('step/end', { turn, step })
        }
        signal.throwIfAborted()
        if (turnEnds && this.inbox.nextStep.length === 0) {
          await this.dispatch.serial('agent/turn-stopping', { turn, signal })
          signal.throwIfAborted()
        }
        if (turnEnds && this.inbox.nextStep.length === 0) break
        target = 'next-step'
      }
    } catch (error: unknown) {
      if (signal.aborted) {
        turnEnds = { kind: 'aborted', reason: signal.reason as AgentCancelCause }
        throw error
      }
      // Every failure is structured: an `LlmError` keeps its facts, anything
      // else flattens to `errorChain` text under the `UNKNOWN` code.
      turnEnds = {
        kind: 'error',
        error: error instanceof LlmError
          ? error.failure
          : { message: errorChain(error), code: 'UNKNOWN' },
      }
      this.throwError(error)
    } finally {
      try {
        // oxlint-disable-next-line typescript/no-non-null-assertion -- every exit assigns a turn ending
        this.session.append('turn/end', { turn, reason: turnEnds! })
      } catch (error: unknown) {
        this.throwError(error)
      }
    }
    if (!this.inbox.hasPending) return false
    phase.abort = new AbortController()
    // A fresh controller makes a latch set on the old one stale: the live driver claims the queue itself.
    phase.wakeRequested = false
    phase.step = 0
    return true
  }
```

**回答什么问题**：一轮 turn 的完整生命周期——边界何时开、step 何时续、结果怎么定、取消和异常怎么记录。

**逐项解释**：`turn = phase.turn + 1` 续号；`turn/start` 先于一切（连 append 失败都要经 `throwError` 上报 `agent/error`）。`turnEnds` 是整轮的"结果槽"，`null` 表示还欠活。循环里：`preStep` 返回 reject → `blocked` 收尾；首个 step 空 enter → `completed` 收尾（注释明说"仍拥有 turn 边界但不花模型调用"）；后续 step 空 enter → 直接 break。每个进入的 step：`step/start` → entered 消息逐条 `user/message` → `this.step()` → `finally` 里 `step/end`。`turnEnds` 的赋值条件是 max-tokens 粘性的实现。step 结束后：不欠活且 inbox 空 → `agent/turn-stopping`；再读一次 inbox 仍空 → break。catch 分支区分取消（`aborted`，带原因重抛）与失败（结构化 error，`LlmError` 保 facts）。`finally` 里 `turn/end` 必定落盘——注释"every exit assigns a turn ending"是这个非空断言合法的原因。尾部：无排队工作返回 false（驱动收工）；否则换新 AbortController、清闩锁、step 归零，返回 true 让 `kick()` 再开一轮。

**直觉**：整个函数是"每个出口都有结局"的纪律的物化。`turnEnds` 只有一个赋值槽，三条提前 return、一个 break、两个 catch 分支都必须先给它一个值——所以 `finally` 里的非空断言从不撒谎。

**反事实**：若 `turn/end` 不在 `finally` 而在各分支里写，任何一条新加的异常路径都会留下一个永不关闭的 turn，fork 和崩溃修复全都会误判。若 turn-stopping 后不重读 inbox，监听器塞进的 steering 就会被吞掉，"data decides"变成"order decides"。

**白话翻译**：开个 turn 就意味着你一定会在日志里看到它怎么结束的——被拒、白跑、跑完、出错、被掐，五种结局必居其一，且只有一种。

### 5.2 `ReactLoopAgent.step()`——一次模型请求

来源：原文，`packages/core/agent-loop/src/agent.ts:332-401`。

```ts
  private async step(assembly: PromptAssembly): Promise<StepEndReason | null> {
    /* v8 ignore next -- private callers establish the running phase before executing a step */
    if (this.phase.kind !== 'running') throw new Error(`agent "${this.id}": step outside running phase`)
    const { turn, step, abort: { signal } } = this.phase
    signal.throwIfAborted()
    const system = renderPrompt(assembly)

    while (true) {
      const { request, preparedCall } = await this.buildRequest(
        turn, step, assembly.tools, system, this.session.deriveMessages(), signal,
      )
      const assembler = new BlockAssembler()
      const chunkSeqs: number[] = []
      const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
      signal.throwIfAborted()
      for await (const chunk of stream) {
        signal.throwIfAborted()
        chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
        assembler.push(chunk)
      }
      signal.throwIfAborted()
      const finish = assembler.finish
      if (finish.kind === 'error' || finish.kind === 'aborted') {
        const action = await this.dispatch.waterfall(
          'agent/request-error', {
            turn,
            step,
            provider: request.provider,
            failure: finish.failure,
            retryPolicy: preparedCall?.retryPolicy,
            signal,
          },
          () => Promise.resolve<RequestErrorAction>(undefined),
        )
        signal.throwIfAborted()
        if (action?.kind !== 'retry') {
          throw new LlmError(finish.failure.message, finish.failure.code, finish.failure)
        }
        continue
      }

      const message = createAssistantMessage({
        content: assembler.blocks(),
        source: {
          provider: request.provider,
          model: request.model,
          ...assembler.replayState !== undefined ? { replayState: assembler.replayState } : {},
        },
      })
      this.session.append(
        'assistant/message',
        {
          turn,
          step,
          message,
          ...assembler.usage === undefined ? {} : { usage: assembler.usage },
        },
        { surfaceOp: 'append', sourceEventSeqs: chunkSeqs },
      )
      if (finish.kind === 'max-tokens') return { kind: 'max-tokens' }

      const toolCalls = message.content.filter(block => block.type === 'tool-call')
      if (toolCalls.length === 0) return { kind: 'completed' }
      const { concluded } = await executeToolCalls(
        this.loopCtx, turn, step, toolCalls, signal,
        context => this.inbox.splice('next-step', this.inbox.nextStep.length, 0, [context]),
      )
      return concluded ? { kind: 'completed' } : null
    }
  }
```

**回答什么问题**：一次模型请求从构造到落盘的全过程，含重试与工具续命。

**逐项解释**：返回 `null` 是"工具欠一次续请求"的信号，turn 循环靠它决定再来一个 step。`while (true)` 是重试循环：每次迭代先用**当时的日志** `deriveMessages()` 重建历史（重试带着最新状态重发）。chunk 循环里 `throwIfAborted` 出现四次——取消随时可打断，但打断点都在事件边界上。`chunkSeqs` 收集每个 chunk 的事件 seq，最后变成 `assistant/message` 的 `sourceEventSeqs`。失败分支先发 `agent/request-error`：默认动作 `undefined` 即终局；`retry` 则 `continue`——**同 step 号**重试。成功分支组装消息、落盘（usage 随行）；`max-tokens` 直接返回（粘性由 turn() 保管）。有 tool-call 就进调度器，工具结果里带的 `additionalContexts` 经回调 splice 进 next-step 队列，等下一个 step 边界被 claim。

**直觉**：历史永远在请求前一刻从日志重建——内存里没有第二份"对话状态"需要维护一致性。这让 fork、resume、replay 全部退化成"换个前缀重放日志"。

**反事实**：若 chunk 不落盘（只存最终消息），replay 保真和流式 UI 回放就没了；若重试换新 step 号，`request/header` 折叠和计费归因都会把一次逻辑请求算成两次。

**白话翻译**：发请求前先看日志，听到什么记什么，失败了问插件要不要重来，成了就把成品和原材料的对应关系一起存档。

### 5.3 `Inbox.claim()`——step 边界的认领

来源：原文，`packages/core/agent/src/inbox.ts:71-78`。

```ts
  claim(target: InboxTarget, turn: number): UserMessage[] {
    const claimed = this.mutate('next-step', 0, this.nextStep.length, [], false)
    if (target === 'next-turn') {
      claimed.push(...this.mutate('next-turn', 0, 1, [], false))
    }
    for (const message of claimed) this.notifications.claimed(message, turn)
    return claimed
  }
```

**回答什么问题**：一个 step 的输入批次从哪来、来多少。

**逐项解释**：先清空整个 `next-step` 队列（打方向与注入的上下文都归本 step）；仅当 `target === 'next-turn'`（turn 的第一个 step）再额外取**恰好一条** `next-turn`——`splice(target, 0, 1, [])` 的 `1` 就是"plus one queued message"的那个 one。`false` 表示被移除的消息不算 discarded；逐条发 `claimed` 通知并带上 owning turn 号。

**直觉**：turn 边界一次只放一条普通 prompt 进来——排队的五句话是五轮 turn，不是一轮 turn 的五条输入（已验证（跑过）：`packages/core/agent-loop/tests/properties.spec.ts` 的"sequential sends each get their own turn with increasing numbers"）。

**反事实**：若一次吞掉整个 next-turn 队列，用户连发的三条消息会被揉进同一轮，取消和归因都说不清每条消息的结局。

**白话翻译**：插队的话全收，排队的话一轮只叫一个号。

### 5.4 `Session.deriveMessages()`——从日志投影历史

来源：原文，`packages/core/session/src/index.ts:726-747`。

```ts
  deriveMessages(): Message[] {
    const surface = this.surface
    const nodes = surface.nodes
    const generation = surface.replaceGeneration
    if (generation !== this.derivedGeneration) {
      this.derived = []
      this.derivedNodes = 0
      this.derivedGeneration = generation
    }
    for (const seq of nodes.slice(this.derivedNodes)) {
      // Surface sequences are built from this.log — seq is always a valid
      // index by construction. The non-null assertion expresses that invariant.
      // oxlint-disable-next-line typescript/no-non-null-assertion
      const msg = this.deriveEventMessage(this.log[seq]!)
      // A surface node is one of the five message-producing types, but an
      // empty-content assistant/message (a max-tokens step that hosts only
      // usage) derives to null and must not enter the transcript.
      if (msg) this.derived.push(msg)
    }
    this.derivedNodes = nodes.length
    return [...this.derived]
  }
```

**回答什么问题**：模型历史从哪来——以及为什么 compaction 改历史不用改日志。

**逐项解释**：`surface.nodes` 是模型可见顺序的事件 seq 列表。`replaceGeneration` 是遮蔽替换的代际号：一次 compaction `replace` 让它 +1，缓存整体作废重建（前三行 if）。否则增量：只投影新出现的节点，每个节点恰好投影一次。`deriveEventMessage` 是第 3.4 节的逐节点规则；返回的数组是快照（已持有的数组不会被后续 append 撑长），里面的 `Message` 对象共享且深冻——缓存不需要第二次深拷贝，消费者也改不动日志。

**直觉**：缓存键不是"日志长度"而是"代际 + 节点数"——append 便宜（增量），replace 昂贵（全量重建），而这恰好符合两者真实的频率与语义。

**反事实**：若没有 surface 层、直接按日志顺序投影，compaction 想改模型可见历史就只能物理删除日志事件——fork、审计、replay 全部陪葬。

**白话翻译**：历史是日志上的一个视图，改视图不动账本；视图被重写过才重算，否则只补差额。

### 5.5 `executeToolCalls()`——工具调度器入口

来源：原文，`packages/core/agent-loop/src/tool-calls.ts:59-101`。

```ts
export async function executeToolCalls(
  ctx: Context,
  turn: number,
  step: number,
  toolCalls: ToolCallBlock[],
  signal: AbortSignal,
  acceptContext: (context: UserMessage) => void,
): Promise<{ concluded: boolean }> {
  const agent = ctx.agents.requireInitiator()
  const { session } = agent

  // Inputs are distinct because tools/execute wrappers may replace `exec.signal`.
  const planned: PlannedCall[] = toolCalls.map(block => ({
    block,
    exec: {
      callId: block.id,
      name: block.name,
      arguments: parseArguments(block.arguments),
      agent,
      signal,
    },
  }))

  let next = 0
  let concluded = false
  while (next < planned.length) {
    // Commit before classifying again so registry changes affect unstarted calls.
    // oxlint-disable-next-line typescript/no-non-null-assertion -- bounded by the loop condition
    const first = planned[next]!
    const mode = ctx.tools.executionMode(first.exec).kind
    const group = mode === 'parallel' ? planned.slice(next) : [first]
    const outcome = await runGroup(
      ctx, turn, step, group, mode, signal, acceptContext,
    )
    next += outcome.consumed
    concluded ||= outcome.concluded
    if (outcome.aborted) {
      for (const call of planned.slice(next)) appendSkippedToolCall(session, turn, step, call.block)
      return { concluded }
    }
  }
  return { concluded }
}
```

**回答什么问题**：一个 step 的若干 tool-call 按什么顺序、什么并发度执行，结果按什么顺序落盘。

**逐项解释**：`requireInitiator()` 从 initiator 边界找回本 agent（它的 session 是落盘目标）。每个 call 打包成 `PlannedCall`：arguments 此刻解析（坏 JSON 保留原文）。主循环按组推进：取下一个调用问注册表要并发模式——`exclusive` 自成一组（屏障），`parallel` 把剩余全部候选成一组（`runGroup` 内部还会边提交边重新分类，遇到屏障就截断）。`runGroup` 返回消费了几个、是否 abort、有没有 `concludesTurn`。abort 时给剩余未启动的调用补合成结果再收工（replay 合法性的来源）。

**直觉**：分类发生在**启动前**且**提交后重新分类**——热插拔改了工具的并发声明，影响的是还没启动的调用；已启动的按启动时的承诺跑完。注册表是活的，调度器的应对是"每次只相信当下"。

**反事实**：若一次性按初始模式切好组再跑，运行中把某工具换成 exclusive 也无法形成屏障，写操作就可能和读操作重叠；若不按模型序提交结果，日志里的 call/result 顺序就和模型的意图顺序脱节。

**白话翻译**：能并行的进池子，不能并行的当栅栏；一边跑一边回头看规则有没有变，出事也要给没排上队的留个交代。

### 5.6 agent-loop invariant——model-visible ⟺ logged 的断言

来源：原文，`packages/core/agent-loop/src/invariant.ts:19-55`。

```ts
const install: InvariantInstaller = Object.assign((ctx: Context, fail: InvariantFailure) => {
  // Prepend prevents a short-circuiting replay listener from silencing the check.
  ctx.on('llm/stream', (options: GenerateOptions, next) => {
    if (!isAgentLoopRequest(options)) return next()
    if (!Object.isFrozen(options)) fail('a loop-built request must be frozen')
    if (options.sessionId === undefined) fail('a loop-built request must carry a session id')
    const session = ctx.sessions.get(options.sessionId)
    if (!session) fail(`a loop-built request must carry a live session id, got "${String(options.sessionId)}"`)
    if (!Object.isFrozen(options.messages)) {
      fail('a loop-built request must carry a frozen messages array')
    }

    const events = session.events
    if (!events.some(event => event.type === 'step/start')) {
      return fail('a loop-built request with no step/start in its session log')
    }
    const header = foldRequestHeader(events)
    if (header === undefined) {
      return fail('a loop-built request with no request/header event in its session log')
    }
    const expected = session.deriveMessages()
    if (JSON.stringify(options.messages) !== JSON.stringify(expected)) {
      fail(`llm request for session "${String(session.id)}" diverges from the dispatch-time durable derivation (log-reconstruction desync)`)
    }

    const headerMatches = options.model === header.config.model
      && options.system === header.system
      && options.temperature === header.config.temperature
      && options.maxTokens === header.config.maxTokens
      && JSON.stringify(options.stop) === JSON.stringify(header.config.stop)
      && JSON.stringify(options.tools ?? []) === JSON.stringify(header.tools ?? [])
    if (!headerMatches) {
      fail(`llm request for session "${String(session.id)}" diverges from the folded request header`)
    }
    return next()
  }, { global: true, prepend: true })
}, { inject: ['sessions'] })
```

**回答什么问题**："进模型请求的必须能从日志重建"这条硬不变量，在哪、以什么强度被机器执行。

**逐项解释**：挂在 `llm/stream` 瀑布上、`global` 且 `prepend`——所有 loop 请求必经，且排在任何可能短路的监听器之前。只查 loop 自己造的请求（`isAgentLoopRequest`，对象级 WeakSet 标记）。六项检查：请求冻结、带 sessionId、session 活着、messages 冻结；日志里必须有 `step/start` 和 `request/header`（请求必须在 turn/step 结构内、有头部快照）；然后两条重头——`options.messages` 与当场 `deriveMessages()` 的结果做 JSON 逐字节比较，请求的 model/system/temperature/maxTokens/stop/tools 与日志折叠出的 header 逐项比较。任何不符调 `fail(...)` 上报，但**不拦截**（最后仍 `next()`）——不变量是报警器不是闸门。

**直觉**：断言点选在请求离开进程前的最后一个瀑布，而不是构造请求的地方——因为"模型实际看到什么"只有在这里才算数。检查做全等比较而非抽样：desync 没有"轻微"一说。

**反事实**：若没有这道检查，某个插件在 `agent/request` 瀑布里改了 messages（绕过日志渠道），模型看到的就是日志里不存在的内容——fork 之后对话会"失忆"，而且没有任何信号告诉你为什么。

**白话翻译**：每个发给模型的请求出门前都要过一次安检——你说的每句话，账本上必须找得到。

## 6. 局限与偏差（意图 vs 现实）

本篇验证了全书首轮架构假设：**`docs/architecture.md:65-84` 的 turn flow 与代码一致，假设成立**（逐行对照证据见第 3 节各处锚点）。但对照中也发现三处文档与代码的偏差，按"文档声明 / 代码实际 / 推断"四层列出：

| # | 文档声明（出处） | 代码实际（锚点） | 判断 |
|---|---|---|---|
| D1 | 时序图把 `step/end` 画在 `agent/request-error` 之前（`docs/agent-lifecycle.md:41-43`）；core.md 亦称 request-error "runs after a failed model step closes"（`docs/subsystems/core.md:228`） | request-error 在 `step()` 的 while 循环内 dispatch（`packages/core/agent-loop/src/agent.ts:355-365`），`step/end` 在 `turn()` 的 `finally`（`packages/core/agent-loop/src/agent.ts:292`）——request-error **先于** step/end | 文档过期。时序先后影响"监听器能不能在 step 闭合前修复 durable 状态"的理解，属实质偏差 |
| D2 | core.md 称 SessionEventMap 有"twelve event variants"且含 `steering/message`（`docs/subsystems/core.md:248`）；同页表格与 subsystems 索引提到 `TurnTrigger`/`TurnTriggerMap`（`docs/subsystems/core.md:290`、`docs/subsystems/README.md:17`） | SessionEventMap 实际 13 个成员（`packages/core/session/src/types.ts:236-333`），无 `steering/message`；全库不存在 `TurnTrigger` 类型，只有 `TurnEndReasonMap`（`packages/core/session/src/types.ts:155-174`） | 文档过期（疑似早期设计的残留）。注意 session.md 本身是准的，漂移集中在 core.md 的转述与索引 |
| D3 | spine README 称"The retry policy may repeat a failed request in a new numbered step"（`packages/examples/agent-spine-demo/README.md:70`） | `agent/request-error` 返回 `retry` 后 `continue`，**同一 step 号**重发（`packages/core/agent-loop/src/agent.ts:367-370`）；llm-retry 正是挂这个事件（`packages/llm/llm-retry/src/index.ts:210`） | 措辞与代码不符。存疑保留：compaction 等插件另有"开新 turn 重试"的路径，但那发生在 turn 粒度而非 step 粒度 |

其他批判性观察：

- **重试不拦截、只报警的 invariant 哲学**意味着关系破坏不会阻断产品，只靠诊断通道曝光——对生产是韧性，对开发期则可能让真问题静默。这是取舍不是缺陷，但读者应知道 invariant 失败不等于请求失败。
- 主线的行为论断绝大多数已升级为已验证（跑过）（1201 个测试，命令见第 1 节）；仍为静态推断的主要有：turn 号续号在"日志被外部工具修剪过"的边界情形、以及 `FactoryOwnership` 三路取消在全厂 teardown 与单 agent dispose 竞争下的精确时序（代码读得通，未构造并发实验）。未验证项：无——本篇范围内没有条件不足的论断。
- 本篇未进 `tools/src/code-mode.ts`、`schema.ts`、`json-schema.ts`、`py-types.ts`、`ts-types.ts`、`presentation.ts` 的内部（Code Mode 与 schema DSL 属第 04 篇范围），未进 `session/src/chunk-rows.ts` 与 `json.ts` 的编解码细节（属第 05 篇持久化范围）。这些都是刻意的不平均用力。

`[ASK USER]`（代码读不出，不编造）：

1. request-error 时序（D1）是文档没跟上代码，还是代码有意改序后文档未更新？决策史在 Agent Notes 之外无从确认。
2. D2 的 `steering/message` 与 `TurnTrigger` 是已被删除的早期设计，还是规划中的设计？快照里只有 turn 边界与 inbox splice 两种现存机制能对上号。
3. turn 号从 1 开始而非 0、resume 从最后一个 `turn/start` 续号（`packages/core/agent-loop/src/agent.ts:92`）——惯例可推断，但从 1 起始的理由无记录。（低优先）

## 7. 总结与自测

三句话总结：

1. 一轮 turn 是 `turn()` 一个函数里的状态机：turn/start 先行、每个出口都有 turn/end，step 之间靠 inbox 与工具结果续命，拒绝和空跑同样留下 durable 记录。
2. 模型可见的一切都在日志里：`deriveMessages()` 从 surface 投影历史，chunk 全量落盘保 replay，agent-loop 的 invariant 在 `llm/stream` 上逐字节断言"请求 ⟺ 日志"——fork、resume、replay 都是这条不变量的红利。
3. loop 可替换不是口号：接口与工厂住在 `dsh-agent`，驱动器是依赖图顶点，扩展插件（含重试）只挂 `agent/*` 与 `tools/*` 事件，从不 import `dsh-agent-loop`。

自测 Q&A：

- Q：用户连发三句话，会产生几轮 turn？A：三轮。turn 边界的 claim 只取一条 next-turn（`packages/core/agent/src/inbox.ts:71-78`）。
- Q：`agent/pre-step` 拒绝后，被 claim 的消息去哪了？A：随 turn 以 `blocked` 关闭而终结——既不回 inbox 也不成 user/message（`packages/core/agent/src/runtime-types.ts:188-197`）；`foldConsumedWork` 能把这笔账从日志里折出来。
- Q：模型历史存在哪个字段/表里？A：哪都不在。它由 surface 节点逐节点投影而来，只有增量缓存；surface 被 replace 就整体重建（`packages/core/session/src/index.ts:726-747`）。
- Q：想给某类工具加全局超时，改 loop 吗？A：不改。挂 `tools/execute` 瀑布即可（`packages/core/tools/src/index.ts:163`）；loop 只负责调度顺序。
- Q：想验证"我的插件没破坏 turn 结构"，从哪拉线头？A：跑 `pnpm vitest run packages/core/agent-loop`；运行时则靠 `dsh-session` invariant 的 turn/step 闭合断言（`packages/core/session/src/invariant.ts:55-166`）。

延伸阅读：`docs/architecture.md` 的 turn flow 与 capability seams 两节（本篇已逐行对照）；`docs/agent-lifecycle.md` 时序图（注意 D1 偏差）；`packages/core/agent-loop/tests/` 的 spec 文件名就是一份行为清单。

## 附录：覆盖报告

导读问题 → 章节映射：Q1（turn/step 状态机）→ 3.2、3.6、5.1；Q2（inbox 语义）→ 3.1、5.3；Q3（pre-step 拒绝）→ 3.3、5.1；Q4（prompt 装配）→ 3.3、4.3；Q5（deriveMessages）→ 3.4、5.4；Q6（工具瀑布）→ 3.5、5.5；Q7（Agent 所有权）→ 4.1；Q8（scope 原语）→ 4.2；Q9（model-visible ⟺ logged）→ 3.4、5.6；Q10（loop 可替换）→ 2、4.1。

覆盖矩阵（包 × 维度）：

| 包 | 入口/装配 | 核心类型 | 主线数据流 | 测试佐证 | 文档对照 |
|---|---|---|---|---|---|
| scope | 4.2 | 4.2 | 3.1（载体） | scope.spec、store.spec（跑过） | scope.md 一致 |
| session | 4.5 | 3.4（surface）、4.5 | 3.2–3.6 | 46 文件内全部 session spec（跑过） | session.md 一致；core.md 有 D2 |
| system-prompt | 4.3 | 4.3 | 3.3 | scoped、tool-order、invariant spec（跑过） | system-prompt.md 一致 |
| agent | 4.1 | 3.1、4.1 | 3.1–3.3 | inbox/dispatch/initiator/consumed-work spec（跑过） | core.md 大部一致，有 D1/D2 |
| tools | 4.4 | 3.5、4.4 | 3.5 | 12 文件 386 测试（跑过） | tools.md 一致 |
| agent-loop | 2、4.1 | 3.2–3.6、5.1/5.2/5.5/5.6 | 全主线 | 815 测试中的 loop 全部 spec（跑过） | architecture.md 一致；时序图有 D1 |

本篇未覆盖（已声明，非遗漏）：`tools/src/code-mode.ts`、`schema.ts`、`json-schema.ts`、`py-types.ts`、`ts-types.ts`、`presentation.ts`（Code Mode 与 schema DSL，第 04 篇）；`session/src/chunk-rows.ts`、`json.ts`（持久化编解码，第 05 篇）；`llm` 包的 Message/StreamChunk 词汇与 BlockAssembler 内部（第 03 篇）；`scope/src/scoped-events.generated.ts`（生成物，一句话：scope 不变量查 subject 用的生成解析表）。

`[ASK USER]` 汇总：见第 6 节末尾三条。
