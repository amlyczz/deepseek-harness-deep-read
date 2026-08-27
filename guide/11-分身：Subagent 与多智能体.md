# 11 · 分身：Subagent 与多智能体

> 本篇目标：搞懂 DSH 怎么把"一个 agent"扩展成"一支队伍"。核心难题三个：子代理的上下文怎么隔离？多轮任务怎么续命不失控？父子之间怎么安全通信？读完你将得到一套可以直接照抄的多智能体纪律。

## 从一个问题开始

让主 agent 干大活，最先死掉的不是它，是它的**上下文窗口**。搜代码、读文件、跑测试的中间过程一条条堆进历史，真正重要的结论反而被挤没。天真方案是提示词工程："请先规划再分步执行，保持简洁"——模型多半答应，然后照旧膨胀。

真正的解法只有一个：**把过程外包给独立上下文，只让结论回家**。这就是 subagent 的存在理由。但做出正确的 subagent 有三个隐藏深坑：

1. **隔离与继承的度**：完全新生（什么都不给）则无法委派；全量继承（复制父对话）则隐私和窗口双双爆炸；
2. **生命周期管理**：子代理是谁的所有物？死了算谁的？能复活吗？
3. **通信纪律**：谁能给谁发消息？怎么防止子代理 A 冒充 B 说话？

## 服务即注册表：spawn 与 fork 的统一

DSH 把 subagent 做成一个具名 provider 注册表服务 `ctx.subagents`（`packages/subagent/subagent/src/index.ts:171`）——注意这个形状选择：不是写死一个"执行器"，而是像 LLM 适配器一样开放注册（文档原话：镜像 LLM adapter 注册表形状）。内置两个 provider：

| provider | inheritsParentContext | 种子内容 |
|---|---|---|
| spawn-in-process | `false`（全新头脑） | 无——只有你给的 prompt |
| fork-in-process | `true`（知道前情） | 父日志截到**最后一个 turn/end 的平衡已完成轮次前缀**（`completedTurnPrefix()`） |

这个表值得盯着看半分钟：**fork 与 spawn 的全部区别，最终只是一个 `seed: SessionEvent[]` 数组的长度**。fork 不是什么神秘机制——就是用第 04 篇的血统切片作为种子事件创建子会话，且只吃已完成轮次（进行中的轮次不可回放，硬拒绝）。

能力协商发生在 start 时、fail loud（`assertCapabilities()`，497-512 行）：outputSchema / depthLimit / toolFilter / persona 任一不被 provider 支持，立刻抛 `UNSUPPORTED_CAPABILITY`，注释明言"绝不会被接受后静默忽略"。能力不支持就说清楚，比装作支持坑人诚实一万倍。

## 委派一圈的完整旅程

```
模型调用 subagent(description, prompt)
   │
   ▼
ctx.subagents.start(provider, descriptor)     ← 快照描述符（此后改参数无效）
   │
   ▼
provider 经 parent.ctx.agents.create() 创建子 agent
   │  此刻发生四件安全事：
   │  · 子加入父所在 preset；作用域事件天然只向上冒泡
   │  · 注入 order=120 固定委派宣言 SUBAGENT_DELEGATION_CONTEXT：
   │    "你的权限范围在你启动时就已固定，无法从会话内部放宽"
   │  · tools.restrict() 按 descriptor 收窄工具面
   │  · approval 钉死 'never'（子代理不许弹审批打扰人！），并作为
   │    source:'delegation' 策略事实写入子日志 → 重启后策略可重建
   │
   ▼
首个 agent/pre-step 时 append 持久 subagent/descriptor 事件
   │
   ▼
发送初始消息 → await child.whenIdle()
   │
   ▼
readResult：从 activation 边界切子日志，取 finalAssistantOutput 与 stopReason
   （stopReason 由 turn-end 词表映射；pre-step blocked 映射为 refusal）
```

结果返回模型的是三选一形态：前台 `{runId, output}`、后台 `{jobId}`（接 jobs 运行时）、可继续 `{subagentId}`。

几个设计点单独咀嚼：

- **子代深度有持久下界**：`SessionHeader.delegationDepth` 记录在 header 里，派生值恒为 `delegationDepthOf(parent)+1` 再对比可选绝对上限——想用重放伪造浅层级？日志先过不去；
- **审批钉 never 的哲学**：委派出去的工作不该反过来打断人类工作流；真需要授权的活要么父代批好权限再委派，要么干脆别委派；
- **一次性的归一次性**：前台 run 用 `Promise.allSettled` 同时隔离结果失败与 dispose 失败成 AggregateError——拆家失败不会吞掉干活成果。

## 可继续子代理：驻留、路由与冷恢复

one-shot 只解决了外卖问题；长期合作还需要"招个常驻员工"。这是本篇技术含量最高的部分（`continuation.ts`）：

**一条持久会话至多一个驻留 Activation**。所谓 running/waiting/settled 三相**不是新状态机**——它们是从"Agent 是否静稳 + ownedChildren"实时推导的视图（165 行附近）。记住这个倾向：**能用推导就不存储**，又见原则一。

续聊的路由按驻留态分流：

```
followup(subagentId, message) ┐
                              ├─ running  → 入队 inbox FIFO
interrupt(subagentId)         ├─ waiting  → 唤醒当轮消费
                              └─ 缺席(进程重启) → coldResume：
                                   不经 provider！管理器折叠持久描述符，
                                   直接 ctx.agents.resume() —— 零成本复活
```

冷恢复这条尤其值得复刻：由于描述符、血统、策略都以事件形式持久化在双方日志里（`subagent/descriptor` 等），"恢复一个子代理"缩减为"按账重放一遍再拉起 loop"。没有第二个真相源要同步。

**授权模型简单而严密**：

- 子代理向父说话只有一条合法通道：专用 `report` 工具，凭证是自己的 `exec.agent`；接收方不可指定——永远是从持久 `parentSession` 推导的直接父（`resolveReportParent`）。防伪的代价几乎为零；
- 父发消息走 `followup()`/`interrupt()`，后者要先过**谱系鉴权**：user 身份核对权威 `header.parentSession`；祖先身份验证调用者确实在目标的在线祖先链内，过期句柄与自我指涉先行拒绝（554-594 行）。
- 结算通知故意用两个不同的 MessageSource kind：`subagent-report`（孩子说的话）与 `subagent-settled`（runtime 记的账）。注释解释了为什么不能合并："合并会让孩子被记上它没写过的话"。transcript 的诚实从数据建模做起。

## 事件上行：谱系可见性

subagent/start 与 subagent/end 是仅供观察的事件，发射时以 `scopeTarget(this, parent)` 把**委派父绑定为分发载体**（`lifecycle.ts:100-123`），于是借助第 08 篇讲过的 locality 过滤——父作用域的监听者只听见自己谱系里的动静。A 家的分身完不成任务，B 家的老板不会收到通知；每个监听者的异常都被逐个捕获为 warn 日志，一个坏观察者炸不掉广播总线。

## Agent Teams：把协作账本也塞进同一本日志

实验包 `packages/experimental/agent-team` 展示了下一步野心，而且答案仍然惊人地"原则一"：**整支队伍的名册、任务板、留言板，全是折叠自 Lead 根会话日志的四种 log-only 事件**（`team/member`、`team/task`、`team/message/queued`、`team/message/delivered`，`fold.ts:157-160`）。

- 信箱的崩溃恢复语义免费获得：queued − delivered 就是待投递队列（投递确认在目标侧落定后才记账）；
- 共享任务板是整快照存储 + CAS revision（每次 +1），`blockedBy` 必须指向存活任务且无环;writeScopes 只是提示性路径前缀不是锁——约束靠规范文档与评审，不靠分布式锁；
- 工具面十件套：Lead 独占 `spawn_teammate`/`interrupt_agent`；全员可用 `send_message`（静默不唤醒）/`followup_task`(唤醒)/`wait_agent`/`team_task_*`（update 必须带 expected_revision 做 CAS）。

团队场景需要的所有状态，居然没有一行专门的存储代码。

## Workflow：让模型写脚本编排自己

fan-out N 个子代理再 join，如果让模型一步步"我来调 subagent，我再调一个……"，既慢又容易漏。workflow 引擎换了个抽象层级：`ctx.workflowEngine` 单引擎 seam，shipped 实现每 run 开一个 node worker thread 跑模型提交的 JS 脚本（顶层 await、以 return 收尾），脚本里每个 `agent()` 调用归属必填 parent 并走 subagent seam。

两条纪律决定这东西不会被玩坏：

1. **meta 是 JSON 由引擎校验，绝不对脚本文本求值获取**；
2. **失败两级分流**：钩子误用（坏参数、未知选项、超上限）→ `WorkflowError(fatal)` 直抛绝不吞；只有子代理非 completed 才映射为逐项 null——**拼写错误必须响彻整个脚本**，沉默的部分失败留给业务错误。

此外还有固定脚本的 ralph 工具：循环 N 轮、每轮全新 fresh 子代理、轮间只传不变 objective + 上轮有界 handoff（16KiB 上限、256 轮封顶）——一个哲学问题的工程化回答："迭代改进到底需要多少自由度？"

## 复刻清单

- [ ] provider 注册表形状的 `subagents.start(descriptor)`：能力前置校验 fail loud；
- [ ] spawn/fork 统一到 seed 数组；fork 切平衡已完成轮次前缀（拒绝敞开轮次）；
- [ ] 子组合四件套：委派宣言（含"权限不可内部放宽"）、restrict、approval=never、深度单调界+header 血统；
- [ ] one-shot 全链路：descriptor 入账 → whenIdle → 从激活边界切日志取结论与 stopReason；
- [ ] ContinuationManager：≤1 Activation、三相推导、followup 路由、不经 provider 的 coldResume；
- [ ] report 授权 + 双 kind 结算通知 + interrupt 谱系鉴权；
- [ ] scopeTarget 式事件上行过滤。

验收标准：
1. 让子代理尝试调 restrict 外的工具——schema 里看得见这个工具吗？（看不见才是对的）
2. 杀进程后 followup 一个 waiting 子代理——它是否携带完整记忆醒来且成本接近零？
3. 伪造成"孙代理"直接向祖父 report——能否通过？

## 超越思考

- **并行度还太保守**：一次委派一条任务的对话式模型适合人机节奏，但 DAG 并行需要 workflow 手写脚本。更高层可做"目标驱动自动扇出"（引擎根据任务图自动 spawn/join/retry）；
- **跨产品委派还没打通**：subagent seam 的接口注释放着口风——"把一个轮次委派给另一个产品"是既定方向；协议化的跨 harness 委派（带权限转移的任务描述符标准）会是生态级机会；
- **预算经济学缺位**：子代理烧 token 没有"外汇额度"概念。任务级 cost budget + 自动降档（fork→spawn、16k→4k 输出）是下一个明显战场。

---

分身有了，但你会注意到它们的记性都寄生在第 04 篇那本日志上。接下来两篇处理"agent 的自我修养"：先看它给自己发的工牌与便签（[12](./12-自律：模型的自我记账工具族.md)），再看管家如何让任何疯狂的长对话都不撑爆上下文（13）。
