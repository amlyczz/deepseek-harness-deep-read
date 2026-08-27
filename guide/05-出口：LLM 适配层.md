# 05 · 出口：LLM 适配层

> 本篇目标：搞懂 DSH 怎么跟大模型对话。表面看这是"调个 API"的事，做深了全是学问：怎么同时支持多个供应商而不写出 if 地狱？流式方言怎么翻译成统一词汇？错误怎么分类才不会散落一地字符串比较？重试为什么必须在 agent 层而不是 SDK 层做？

## 从一个问题开始

天真写法一定是这样的：

```ts
// 天真版：模型调用与业务逻辑焊死
import OpenAI from 'openai'
const openai = new OpenAI()
const res = await openai.chat.completions.create({ model: 'deepseek-chat', messages })
// ...别家模型？再 import 一个 SDK，复制一遍，if (provider === 'x') 分支 ×100
```

三个必然爆发的痛点：

1. **供应商锁定**：换模型 = 全文改造，业务代码里到处是 SDK 类型；
2. **不可拦截**：调用直接从你的代码飞向外部 HTTP，任何中间环节（审计、降级、切流、缓存）都插不进去；
3. **错误是字符串**：`"rate limit exceeded"`、`429 Too Many Requests`、SDK 自定义异常……每个错误都要 if-else 猜语义，重试逻辑没法写。

DSH 的解法是一套经典但执行得极其彻底的分层：**先发明自己的词汇表（中立协议），再把所有翻译推到适配器边缘**。相关包一张表看完：

| 包 | 角色 |
|---|---|
| `packages/llm/llm` | 词汇表 + `LlmAdapter` 抽象类 + `ctx.llm` 服务 + 共享聚合器 |
| `packages/llm/llm-deepseek` | DeepSeek 官方 API 直连适配器（原生 fetch + SSE） |
| `packages/llm/llm-pi-ai` | 基于第三方库的多供应商通用适配器 |
| `packages/llm/llm-retry` | 重试策略执行器（agent 层插件） |
| `packages/llm/token-meter` | token 计量服务 |

包首注释自我定位就一句话："**面向循环、会话日志和插件的中立消息与流式词汇表；只有适配器负责翻译供应商线上报文**"（`packages/llm/llm/src/types.ts:1-5`）。这句话就是全篇大纲。

## 词汇表：三张族谱

### 族谱 A：消息 Message

```ts
// packages/llm/llm/src/message.ts:129-138
export interface Message {
  readonly id: MessageId
  readonly role: 'system' | 'user' | 'assistant'
  readonly content: ContentBlock[]
  readonly source: MessageSource    // ← 谁生产的这条消息，必须声明
}
```

三点说明：

- **内容不是 string 而是块数组**。`ContentBlockMap` 五种块（`types.ts:54-110`）：`text` / `reasoning`（思考链，与可见文本严格分账）/ `image` / `tool-call` / `tool-result`。联合类型由 map 派生、可声明合并扩展；
- **tool-call 的 arguments 是原始 JSON 字符串**，注释原文"Raw JSON string as produced by the model"。这是全书出现次数最多的纪律之一：解析是消费方的事，存储永远保真——转义差异、注释残缺这些信息丢了就再也回不来；
- **工具结果以 user 角色回传**（内含单个 tool-result 块），这与 OpenAI 风格不同但更贴近"历史里都是消息"的单一性；适配器负责在出口把它拆回各家协议的形状。

另外 assistant 消息带**出处**（provenance）：生成它的 provider/model 回放所需的无损私有状态（`message.ts:8-19`）。每条 AI 的话都能回答"谁家的哪个模型说的"，且重启后能原样重建。

### 族谱 B：流 StreamChunk

```ts
// packages/llm/llm/src/types.ts:312-324
export type StreamChunk =
  | { type: 'block-start'; index; blockType }
  | { type: 'text-delta'; index; text }          // 一段可见文本
  | { type: 'reasoning-delta'; index; text }     // 一段思考文本
  | { type: 'tool-call-delta'; index; id; name?; argumentsDelta }
  | { type: 'block-end'; index; block }          // 关块时直接携带组装好的完整块
  | { type: 'usage'; usage: TokenUsage }
  | { type: 'finish'; reason; replayState? }
```

三条契约刻进类型注释里（305-311 行）：`index` 把交错的 delta 关联到所属块；**usage 必须出现在 finish 之前**；finish 之后不得再有 chunk。`FinishReason` 也是闭集 map：`stop / tool-calls / max-tokens / aborted(failure) / error(failure)`——还记得第 03 篇 turn/end 的理由闭集吗？同一气质。

`TokenUsage` 计数互不重叠：`inputTokens` 只算未命中缓存的输入，缓存读写单列两桶（135-141 行）。计费口径先在词表层面掰干净。

### 族谱 C：失败 LlmFailure

```ts
// types.ts:40-51
interface LlmFailure { message; code; status?; providerRetryAfterMs?; requestId? }
```

错误码闭集：`AUTH / RATE_LIMIT / SERVER / TIMEOUT / TRANSPORT / EMPTY_RESPONSE / CONTEXT_WINDOW_EXCEEDED / INVALID_REQUEST ...`。HTTP 状态 → 错误码的映射固定一处（401/403→AUTH、429→RATE_LIMIT、≥500→SERVER 等，`adapter.ts:333-345`）。

> 三张族谱的共同气质：**一切联合类型皆来自 keyed map、皆可扩展、扩展时编译器强制你更新所有消费方**（漏一个 switch 就编译不过）。词汇表的演进成本被 TS 类型系统兜住了。

## ctx.llm：注册制适配器中枢

```ts
// packages/llm/llm/src/index.ts:191-260（节选）
abstract class LlmAdapter {
  abstract stream(options: GenerateOptions): AsyncIterable<StreamChunk>   // 唯一必选方法
  providerInfo(provider)            // 名称、上下文窗口等元数据
  providerRetryPolicy?(provider)    // 该家的重试策略
  listModels? / resolveModel? / prepareCall?
}
```

服务侧提供 `registerAdapter(providers, adapter)`（365-394 行）：一次注册一组路由（如 `['deepseek-official']`），重复注册同名路由抛错；返回句柄支持原子换路（热更新安全）。启动时 base bundle 挂了两个适配器行——DeepSeek 直连与 pi-ai 多供应商——要接入新的服务商，你只需要写一个新的 `LlmAdapter` 子类并注册，**系统其他部分一行不改**。

`prepareCall(config, signal)`（824-869 行）值得单独讲，它解决一个阴险的问题：请求准备、头部记录、实际派发如果发生在不同时刻，中间恰好插件热更新换了适配器实现，你就会用 A 家的配置调 B 家的端点。prepareCall 把三件事绑定到**同一份适配器注册代**上；代际不符直接抛 `INVALID_PREPARED_CALL`。

## 解剖一只真实适配器：DeepSeek 直连

抽象说完了，看具体翻译怎么做。文件 `packages/llm/llm-deepseek/src/adapter.ts`：

**发送方向**（内部词汇 → OpenAI 兼容线上格式）：

1. assistant 消息的 text 块拼成 `content` 字段——注意这条注释（`serialize.ts:213-219`）：纯工具调用轮 content 发 **空串而绝不发 null**，因为 reasoning-only 的轮次官方 API 直接以 400 拒绝 null，而且"这条消息已经持久化在会话日志里，一个 null 会 brick 掉这个会话后续每一轮"。一个字段值的选择牵动着磁盘上几千条记录的命运——这就是把保真当信仰的写法；
2. reasoning 块整体回传为 `reasoning_content` 字段（CoT passback：给网关恢复上游思考签名留的唯一凭据）;
3. tool-call 块拼成 `tool_calls:[{id,type:'function',function:{name,arguments}}]`;
4. harness 中 user 角色的工具结果，拆回独立的 `{role:'tool', tool_call_id, content}` 线上消息。

**连接管理**：每条流开始时快照连接事实与密钥（"进行中的流永不观察配置变更"）；`AbortSignal.any([调用方信号, 看门狗信号])` 加一个 300 秒空闲看门狗——上游断流不吐字节时不会永久挂起，超时映射为 TIMEOUT 错误码（462-498 行）。

**接收方向**（SSE 字节 → 词汇流）：`EventSourceParserStream` 解码 → `translate()` 状态机（`translate.ts:86-185`）：首块规则很讲究——第一个**非空** delta 才开块（空串开场不产生空气块）；`tool_calls[].index` 增量拼接：按 index 取开放块、id/name 覆盖写、arguments 片段字符串累加；finish_reason 与 usage 都推迟到 `[DONE]` 才 flush——一次性满足"usage 在 finish 前"+"finish 后无 chunk"两条契约。

pi-ai 通用适配器同构，但多了一处必须提的神来之笔：它刻意设置库级 `maxRetries: 0`（`adapter.ts:127`，注释"The agent recovery layer owns visible attempts; one adapter call is one SDK attempt"）——**一次适配器调用等于一次上游尝试，重试的全部主权收归 agent 层**。下面马上看到为什么。

## llm/stream：最后一个 waterfall

还记得第 03 篇 buildRequest 里 `agent/request` 是"改配置的最后机会"吗？到了真正发流的时刻还有一道闸：

```ts
// packages/llm/llm/src/index.ts:989-999（意译）
return this.ctx.waterfall(this, 'llm/stream', options,
  () => this.adapterStream(options, prepared))       // next() = 真正的适配器流
```

谁能干什么，规则分得极清：

- 循环发起的请求带着进程内标记且深度冻结，监听者**只能读不能改**——配置修改请去上一道闸 agent/request；
- 监听者可以**包装整个 AsyncIterable**：逐 chunk 观测（遥测）、穿透 yield（持久化检查点插件就在放行前把已记账的前缀 flush 落盘——刷不动不放行）、甚至完全不调 next() 自己编造响应（测试支持的 llm-replay 就是这么做的）。

一道瀑布，三种正当用途，边界靠类型与标记硬约束。

## BlockAssembler：第二级状态机

适配器的 translate 已经聚过一次块，为什么还要共享聚合器？因为**消费方不止一种**：循环要拿到最终 assistant/message（喂第 03 篇的收束流程），而日志侧想存原始 chunk（原则一）。两者的桥就是 `BlockAssembler`（`assembler.ts:37`）：text/reasoning 按 index 追加；已被 block-end 关闭的 index 再收到迟到 delta 直接忽略（防内存膨胀）；无 block-start 的简化协议也兼容。

它还有两个防御性细节值得抄作业：

1. **max-tokens 时丢弃全部 tool-call 块**，并同步裁剪 ReplayEnvelope 的逐块条目，长度对不上就整包弃用（134-149 行）——截断的工具调用参数执行起来是定时炸弹，且"存的块"与"存的元数据"永不失配；
2. **中断抢救** `interruptedBlocks()`：取出已完成或虽有残缺但有内容的 text/reasoning 块，供第 03 篇说的 interrupted anchor 使用；工具调用一律不抢救（没发出去的东西不该像发出去过）。

## 错误与重试：主权的归宿

**错误的两种形态统一归一**：适配器可以 throw（传输层错误），也可以用 `finish{kind:'error'}` 带内终止（如空 completion）。两者都归一为 `LlmFailure`，由 `normalizeLlmFailure()` 只信任自有数据属性快照、逐一校验类型——注释原话："third-party SDK codes are not our taxonomy"（第三方 SDK 的错误码不是我们的词表）。供应商的怪话在这里被彻底格式化。

**重试作为纯数据策略**（`retry-policy.ts`）：

```ts
DEFAULT_MAX_RETRIES = 5          // normal 策略默认
DEFAULT_RETRYABLE_CODES = [EMPTY_RESPONSE, RATE_LIMIT, SERVER, TIMEOUT, TRANSPORT]
指数退避 500ms → 封顶 10s，±10% 对称抖动
```

**重试作为普通插件执行**：`dsh-llm-retry` 挂在第 03 篇见过的 `agent/request-error` 上，返回 `{kind:'retry'}` 让循环回到 step 顶部。执行器三个亮点：

1. **尊重 Retry-After**：只有超过本地上限且 non-always 模式才放弃本次尝试（194-205 行）——对限流最礼貌的做法；
2. **重试有账本**：每次等待前先 append 日志事件 `llm/retry`、开始时 `llm/retry-started`；崩溃后恢复计数靠反查日志中同 turn/step/provider/policyKey 的既有事件（150-192 行）——**重启丢不掉重试进度**，因为账本就在事实源里；
3. 重试粒度是"整步重建请求"：由于请求本来就是 deriveMessages 现推的（第 03 篇），重试天然使用最新历史——压缩插件若在中途瘦身了上下文，下一次重试立刻受益。

现在你能回答"为什么 maxRetries:0"了：适配器层的隐形重试会让"第几次尝试"这件事变得不可观测、不可记账、不可干预。可见的努力必须全部经过 agent 层这本账。

## 复刻清单

- [ ] 定义你的三张词汇表：Message+BlockMap、StreamChunk+FinishReasonMap、LlmFailure 错误码闭集；
- [ ] `LlmAdapter` 抽象类：只强制 stream()，其余可选；`registerAdapter` 注册制 + 原子替换句柄；
- [ ] `prepareCall` 式代际绑定：适配器解析、头记录、派发三者锁同一代；
- [ ] 一只最小适配器：SSE 解析 → translate 状态机（非空首 delta 开块 / index 拼 tool-call / finish 延迟 flush）；
- [ ] `BlockAssembler`：迟到 delta 忽略、max-tokens 裁剪 tool-call、interruptedBlocks 抢救；
- [ ] 归一化的失败通道：throw 与 in-band finish 都折算成 LlmFailure；
- [ ] 重试插件：数据化策略 + Retry-After 礼让 + 日志账本驱动的计数恢复。

验收标准：
1. 你的适配器收到中断流时会不会永久挂起？（idle watchdog 应该救场）
2. 把一条纯 tool-call 的 assistant 消息 replay 给一个"content:null 会炸"的网关——你的序列化安全吗？
3. kill -9 在第 2 次退避等待中，重启后重试计数是否正确续上而非从头再来？

## 超越思考

- **估算太粗**：token 启发式是"4 字符/token"密度估计。接各家的 tokenizer 或增量精确 usage，压力预测能从保守估计升级为精确仪表。
- **传输层债**：直连适配器自己留着 TODO 要迁移到统一的 Cordis HTTP 服务；pi-ai 侧因上游压扁 Error 只能用文本模式匹配分类错误。一个共享 transport 层（超时/重试接线/错误结构一体化）能让未来每只适配器免费获得这些东西。
- （留给你的更大题目：适配器层做 KV-cache 友好的 prompt 前缀重排、多路由成本加权调度……都是在这一层长出来的产品能力。）

---

出口打通了，手也已经伸向工具（第 06 篇）。但在那之前回头想想：这三章我们其实一直在见证同一个模式的复利——**词汇先行、边缘翻译、一切入账**。[06 · 双手：工具系统](./06-双手：工具系统.md)会把这套路数推到第三个战场。
