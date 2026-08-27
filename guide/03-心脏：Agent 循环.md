# 03 · 心脏：Agent 循环

> 本篇目标：读懂 harness 真正的"主循环"——它如何开始一轮工作、如何排队与打断、如何取消、如何在每个关口给插件留门。读完并做完复刻清单，你将拥有一个**结构正确**的最小 agent 内核。

## 从一个问题开始

把模型接到工具上，最天真的想法是这样一个循环：

```python
# 天真版 agent loop（~30 行）
messages = [{"role": "user", "content": user_input}]
while True:
    reply = llm.chat(messages, tools=all_tools)
    messages.append(reply)
    if not reply.tool_calls:
        break                      # 模型不调工具了，收工
    results = [run_tool(t) for t in reply.tool_calls]
    messages.extend(results)       # 结果喂回去，再问一轮
```

三十行就能跑通 demo。但只要往产品方向走一步，它立刻四处漏风：

| 真实场景 | 天真版的下场 |
|---|---|
| 模型干活时用户又发了一条消息 | 要么丢掉，要么等这轮跑完才处理——无法"边干边补充指示" |
| 用户点"停止"按钮 | 没有 `stop` 的语义：命令已经 spawn 了怎么办？日志记了一半怎么办？ |
| 想注入一条上下文（"文件刚被外部改过"） | 只能混进下一条用户消息，污染对话语义 |
| 想让插件在请求前做点事（换模型、压缩历史） | 无处安放 |
| 崩溃后恢复 | `messages` 在内存里，全丢了 |

产品级的循环要回答的是一组**并发与记账**问题。DSH 的答案集中在两个文件里：

- `packages/core/agent-loop/src/agent.ts` —— 驱动器 `DriverMachine`（整个仓库唯一写循环逻辑的地方）
- `packages/core/agent/src/` —— `Agent` 接口、注册表、以及那只精巧的收件箱

先建立三个词汇（第 02 篇的 [6][7][15] 号驿站正式展开）：

- **步骤（step）** = 一次模型请求 + 它引发的工具执行；
- **轮次（turn）** = 零或多个步骤：从领取第一条输入前打开，到不再欠任何工作时关闭；
- **驱动器（driver）** = 一个 while 循环实例，管着一轮接一轮的 turn。

## 收件箱：两条队列定乾坤

一切输入（用户的、插件的、子代理的报告）都从同一个入口进来：

```ts
// packages/core/agent-loop/src/agent.ts:113-132
send(message, target, wakeup): void {
  // 唤醒型输入不能加入已被中止的活动，所以改道开新轮。
  const wakingAfterAbort = wakeup && this.phase.kind !== 'idle' && this.phase.abort.signal.aborted
  const resolvedTarget = wakingAfterAbort ? 'next-turn' : target
  this.inbox.splice(resolvedTarget, Infinity, 0, [message])
  if (wakeup) this.wakeDriver(wakingAfterAbort)
}
followup(input): void { this.send(input, 'next-turn', true) }   // 排队下一轮
steer(input): void    { this.send(input, 'next-step', true) }   // 插队进当前轮
inject(input): void   { this.send(input, 'next-step', false) }  // 注入上下文，不唤醒！
```

收件箱就是两个数组 `next-turn` 和 `next-step`（`packages/core/agent/src/inbox.ts:26`），但四个设计决定让它远不止于此：

**决定一：inject 不唤醒。** 注入类消息固定 `wakeup:false`——模型不需要立刻知道"文件变了"，它躺在 `next-step` 里，等驱动器下次经过步骤边界时被**顺手搭车认领**；若 agent 已空闲，就一直躺到你下次说话。一个布尔值，精确区分了"我催你"和"给你补个材料"。

**决定二：先记账后改内存。** 每次增删都先落持久事件 `agent/inbox/spliced` 再动内存数组（`inbox.ts:186-187`）。于是同步观察者看到的是变化前的完整列表，重启后靠重放日志事件也能原样重建收件箱——原则一的又一次执行。

**决定三：claim ≠ discard。** 循环用 `claim()` 取走整批输入走"纯删除"路径，并逐条广播 `agent/inbox/claimed { message, turn }`；而用户在 UI 删掉排队消息走 `remove()` 路径，产生 `outcome:'canceled'`。回放侧因此能分清**"活被认领了"和"活被扔掉了"**（`AG/consumed-work.ts:42-108` 用这两个信号审计整个生命周期）。

**决定四：竞态防护藏在两行注释里。** `send` 第一行就先捕获 `wakingAfterAbort` 再插入——注释写明这是为了防止 splice 观察者重入 cancel 导致消息被错误分类。这种"先拍快照再操作"的手法，后面还会反复见到。

## turn()：一轮的完整剧本

入口 `kick()` 只有一行：`while (await this.turn()) {}`——turn 返回 true 说明收件箱还有活，接着来。整个 `turn()`（`AL/agent.ts:246-330`）是一个精心编排的状态机，逐段看：

### 第一步：先记账，再领工

```ts
// AL/agent.ts:247-258（节选）
signal.throwIfAborted()
const turn = phase.turn + 1
this.session.append('turn/start', { turn })   // ← 第一件事永远是记账
phase.turn = turn
let target: InboxTarget = 'next-turn'
```

注意顺序：**轮次的打开先于任何领取动作**。哪怕这一轮最后什么都没干，日志里也有它的生卒记录——审计链条无缺口。

### 第二步：preStep —— 三件事一次做完

```ts
// AL/agent.ts:225-243（节选+意译）
const claimed = this.inbox.claim(target, position.turn)        // a. 原子领走整批
const assembly = await this.loopCtx.systemPrompt.assemble(...) // b. 组装提示词+工具表
const decision = await this.dispatch.waterfall('agent/pre-step', {
  messages: claimed, turn, step, signal,
})                                                              // c. 全体监听者表决
```

`agent/pre-step` 是整个循环最重要的扩展点，监听者可以：

- **放行**（调 `next()`）：默认行为是把认领的消息作为 user/message 进入本步；
- **改写**（返回 `{kind:'enter', messages}`）：全量替换进入本步的内容——上下文压缩插件用这个能力把老历史折成摘要；
- **拒绝**（返回 `{kind:'reject'}`）：本轮以 blocked 结束，不开步骤。有趣的细节：首次领取被拒时仍会关闭一个不含步骤的持久轮次——**"尝试过"本身就是需要记录的事实**。

这里有个反直觉的设计值得停下想清楚：**claim 发生在 pre-step 表决之前**。消息已经从收件箱删掉了，如果表决结果把它们拒绝了、或者 enter 决策没包含它们，它们不会退回收件箱，也不会作为 user/message 出现——永久消失（文档原文："既不丢弃也不作为 user/message 再现"，`packages/core/agent/src/runtime-types.ts:186-197`）。这是一个"先-commit-后-决策"的取舍：换取 claim 的原子性，牺牲了可撤回性。第 15 篇会把它列进弱点清单。

### 第三步：step() —— 与模型的完整往返

```ts
// AL/agent.ts:279-292（节选）+ step 内部要点
this.session.append('step/start', { turn, step })
for (const m of claimed.messages)
  this.session.append('user/message', m)          // 认领的消息逐一落库
try { turnEnds = await this.step(assembly) }
finally { this.session.append('step/end', ...) }  // 无论成败，括号必须闭合
```

`step()` 内部是本篇技术含量最高的三段：

**A. 请求构造：derive-then-send**

```ts
// AL/agent.ts:426-514（buildRequest，节选意译）
await waterfall('agent/request', config, next)     // 插件可整体替换调用配置
// 校验 provider/model 存在 → llm.prepareCall 绑定确切适配器
const request = markAgentLoopRequest(deepFreeze({
  messages: session.deriveMessages(),              // ← 历史不是攒出来的，是从日志推出来的
  sessionId, signal, ...
}))
```

三道机关：
1. `agent/request` waterfall 让插件能在最后一刻换模型、调采样参数（内置的模型选择功能 `AG/model-selection.ts:39-75` 就是一个普通监听者）；
2. `deepFreeze` + 进程内标记 `markAgentLoopRequest`，从此下游只能读不能改；
3. **最关键的一行是 `session.deriveMessages()`**——发给模型的历史永远是当前时刻从持久日志重新推导的结果，而不是某个内存数组的累加。配合一个全局不变量（拦截 `llm/stream`，逐字节比对请求与日志重推结果，`AL/invariant.ts:21-54`），"模型看见什么"永远可以审计。这是原则一在系统最热路径上的落地。

**B. 流式消费：chunk 双写**

```ts
// AL/agent.ts:346-351
for await (const chunk of preparedCall?.stream(request) ?? ctx.llm.stream(request)) {
  signal.throwIfAborted()                          // 每个 chunk 都检查取消
  assembler.feed(chunk)                            // 聚合成块
  this.session.append('assistant/chunk', { turn, step, chunk })  // 原样入库
}
```

逐 token 保真。为什么不只存聚合后的完整消息？因为回放 UI 需要"打字机效果"，遥测需要真实时序——原始流就是最低层的感官记录。

**C. 收束与续命：欠活机制**

```ts
// 成功：恰好一条 assistant/message，引用全部 chunk seq（sourceEventSeqs）
// 若带 tool-call → executeToolCalls(...) 后返回 null：
//   工具干完把结果送回收件箱旁路，while(true) 继续下一步 —— 这就是"欠活"
// max-tokens 结束 → 标记 {kind:'max-tokens'}（粘性：后续正常完成也不能降级它）
```

### 第四步：数据裁决的停止协议

每步结束都要回答"轮次该关了吗"：

```ts
// AL/agent.ts:295-300（节选）
if (turnEnds && this.inbox.nextStep.length === 0) {
  await this.dispatch.serial('agent/turn-stopping', { turn, signal })  // 给机会补活
  signal.throwIfAborted()
}
if (turnEnds && this.inbox.nextStep.length === 0) break               // 重读数据定夺
target = 'next-step'
```

注意这个两段式：先广播 serial 事件 `agent/turn-stopping`（serial = 监听者依次执行，返回非空即短路），任何监听者都可以在这最后关头 `steer()` 一条消息进去挽留轮次——但机器**不信任何人的口头表态，广播完重新读一遍收件箱**才决定是否关闭。"数据说了算"是这个代码库处理并发意见分歧的一贯哲学。

### 第五步：finally 里的仪式感

```ts
} finally {
  this.session.append('turn/end', { turn, reason: turnEnds ?? ... })
}
```

`turn/end` 必带 `reason`，词表是闭集（`packages/core/session/src/types.ts:155-177`）：

| reason | 含义 | 备注 |
|---|---|---|
| `completed` | 干完了 | 空批次直接完成也算（被删掉的唤醒消息仍拥有它开的轮次） |
| `blocked` | 输入被 pre-step 拒绝 | 尝试本身也留痕 |
| `aborted(reason)` | 被取消 | cause 见下文四值清单 |
| `error(failure)` | 结构化失败 | 带 LlmFailure 分类 |
| `max-tokens` | 长度截断 | 粘性标记 |
| `interrupted` | 中断后由崩溃恢复补写 | 循环自己永不发出此值 |

每一个出口都有名字。排查问题的第一步永远是 grep 日志里的 `turn/end` 理由分布。

## 取消：一根信号线的旅程

点"停止"按钮之后发生了什么？DSH 把取消做成了一套分层协议：

**cause 只有四种稳定取值**（`packages/core/session/src/types.ts:143-147`）：

```ts
export type AgentCancelCause =
  | { kind: 'user' }      // 人点的停止
  | { kind: 'parent' }    // 父 agent 连坐
  | { kind: 'hook'; reason: string }  // 插件裁决
  | { kind: 'disposed' }  // 整个 agent 销毁
```

**传播路径**：`cancel()` 清收件箱（除非 keepInbox）、熄灭唤醒锁存、abort 当前相位的 `AbortController`。signal 的检查点遍布热路径——每个 chunk、每组工具前后都有 `throwIfAborted()`。

**中间态也有账**。如果流式正打到一半被取消，但模型已经吐出了一段有意义的前缀，循环会抢救这些块追加成一条带 `interrupted:true` 的 assistant/message，照样引用已发生的 chunk seqs（`AL/agent.ts:354-370`）；还没派发的工具调用则拿到一对合成的 `tool/call` + `ABORTED_BEFORE_DISPATCH` 结果（`AL/tool-calls.ts:237-259`）。**宁可造假事件的"合成事实"，也不留下对不上的半截账**——这是回放平衡的铁律：每条 call 都必须有自己的 result。

## 相位机：idle / maintenance / running

驱动器内存状态只有三相（`AL/agent.ts:38-46`）：

```
idle ──有唤醒输入──▶ running ──kick finally──▶ idle
  │                    │
  │ runMaintenance()   │ cancel/maintenance 完成
  ▼                    ▼
maintenance ◀──────────┘
```

- `running`：驱动器活着，turn 一个接一个；
- `maintenance`：后台维护任务（如手动 /compact）独占的间隙，**对外状态仍显示 idle**（用户不需要知道你在打扫房间），期间到达的唤醒通过 `wakeRequested` 锁存，维护结束后补放行（142-162 行）；
- 所有跨相位唤醒统一经 `wakeDriver()` 判定， disposed 状态下的取消永不锁存。

还有一个容易忽略的小机关：驱动器整个跑在 `ctx.agents.withInitiator(this, …)` 里——用 AsyncLocalStorage 建立"发起者上下文"，于是工具执行的深层代码随时可以用 `ctx.agents.requireInitiator()` 反查"现在是哪个 agent 在调用我"。一个进程内隐式通道，省掉了把 agent 引用层层传参的灾难。

## Agent 接口：句柄上有什么

外层世界看到的 Agent 是个很小的只读面 + 六个方法（`packages/core/agent/src/runtime-types.ts:64-144`）：

| 成员 | 说明 |
|---|---|
| `id` | 与所属会话共享同一 SessionId |
| `status` | `'idle' \| 'running'` 二元 |
| `inbox` | 只读查看两条队列 |
| `session` | 会话日志引用 |
| `ctx` | **agent 私有作用域上下文**（工具/提示词可按 agent 定制的关键） |
| `send/followup/steer/inject` | 上文四件套 |
| `cancel(cause, opts?)` | keepInbox 可保留排队任务 |
| `whenIdle()` | Promise：等整个 agent 彻底静稳（含换代驱动器） |
| `runMaintenance(task)` | 占用 maintenance 相位跑后台作业 |

创建与销毁是一对原子事务：`agents.create()/resume()` 走"私有准备→setup→commit"流程，setup 未发布前无人可见；而销毁权限被做成了**能力**——只有持有 `AgentHandle.dispose()` 的人才能拆掉这个 agent，teardown 固定为"停机排空→出注册表→移除会话→解作用域"（`AG/index.ts:163-175`）。生命周期的每一环都有唯一责任人与固定顺序。

## 错误恢复：外包给监听者

基础循环本身**不做重试**——请求失败抛出的错误会被翻译成结构化 `LlmFailure`，然后交给 waterfall 事件 `agent/request-error` 处置（373-390 行）：监听者返回 `{kind:'retry'}` 就回到 step 顶部重建请求重来，否则 LlmError 终结本轮。内置的自动重试其实是仓储里的另一个普通插件 `dsh-llm-retry`（指数退避、尊重 Retry-After、**重试次数自己也记进日志**——细节在第 05 篇）。同时有条硬规则防止循环自我腐蚀：**middleware 或工具失败不会进入 request-error，而是直接终结当前轮次**——"插件失败结束轮次而不是循环"。

## 三原则对账

| 原则 | 本章证据 |
|---|---|
| 一 · 记一次 | turn/start 先于领取；chunk 逐条入库；claim/cancel 分事件；turn/end 理由闭集 |
| 二 · 组合 | pre-step 改写输入、request 换配置、turn-stopping 补活、request-error 自持恢复——四个拦截点全是普通监听者 |
| 三 · 不说谎 | deriveMessages+冻结+不变量断言；interrupted 合成锚点保回放平衡；粘性 max-tokens 不许降级 |

## 复刻清单

现在合上文章，你的最小内核应该长这样（建议 ~600 行 TypeScript）：

- [ ] `Inbox` 类：两个数组 + `splice/claim/remove`，每次变更发 `spliced` 通知（先通知后改内存）；
- [ ] `send(target, wakeup)` + `followup/steer/inject` 三别名，实现 inject-不唤醒语义；
- [ ] `DriverMachine`：idle/running 两相位起步（maintenance 可后补）；`wakeDriver` 里正确处理 wakeAfterAbort 改道；
- [ ] `runTurn()`：按序 append `turn/start` → claim → 交给上层组装 → `step/start` → user/messages 落库 → 模型往返 → `step/end` → stop 协议（serial 回调 + 重读队列）→ finally 写 `turn/end{reason}`；
- [ ] reason 闭集枚举 + 每个 catch 分支映射到正确的理由；
- [ ] 取消：每轮一个 AbortController；chunk 级 throwIfAborted；中断时抢救已完成块为 interrupted 消息；未派发调用配合成 result 对；
- [ ] 四值 CancelCause + whenIdle（do/while 跟随换代驱动器）。

验收标准（自测题）：
1. 模型响应途中按下取消，之后重放日志——你能看到完整的转/步括号、被打断的消息锚点、每个未派发调用的合成结果吗？
2. 在 pre-step 监听者里 reject 一次首条领取——日志是否留下了一个零步骤的 turn？
3. busy 时 steer 一条消息，它是否在下一工具边界就被消费而不是等到轮次结束？

## 超越思考

- **没有轮次预算**。steer 可以无限续命 turn，循环原生不知道"最多 N 步/M 个 token 该停"；失控防护完全依赖外挂 turn-stopping 监听者。你的版本可以在状态机里内建预算槽位。
- **claim 不可撤销**。被 pre-step 拒绝的消息永久消失。可以做"决策后才删除"或提供 `dead-letter` 退路。
- **串行执行**。每 agent 单驱动器、单 AbortController，exclusive 工具形成全局屏障，步骤间零流水线重叠。保守正确，但多核时代可挖掘的空间明显。
- **取消粒度粗**。不能"取消本步保留后续排队"。多级 signal 树（per-call level）是现成的改进方向。

---

循环讲完了，但它旋转的每一圈都在读写一样东西——我们反复提到的那本"账"。下一篇就把这本账彻底摊开：[04 · 真相：会话日志](./04-真相：会话日志.md)。
