# 14 · 复刻路线图：从零造一个 Harness

> 本篇目标：把前十三章的所有知识压缩成一条**可以直接开工的建造路线**——M0 到 M5 六个里程碑，每个都有明确的产物、验收标准和工作量预估。照着走完，你将得到一个架构上与 DSH 同构（但代码全部属于你）的 agent harness。

## 总原则

动工之前立三条军规，它们是全书三原则的施工版：

1. **每个里程碑结束都必须可用**。没有"写完半年后才能跑"的阶段——每一步收工时你都有一个能演示的东西；
2. **先记后做**。任何子系统动工前，先把它的账目事件定义出来；
3. **闭集优先**。凡是能枚举的语义（错误码、理由、决策、状态相）一律用封闭联合 + map 派生，拒绝裸字符串。

## 里程碑总览

| 里程碑 | 内容 | 预估规模 | 对应篇章 |
|---|---|---|---|
| M0 | 心跳：最小 agent 循环 | ~500 行 | 03 |
| M1 | 账本：会话日志 + 投影 + 崩溃安全 | +~1500 行 | 04 |
| M2 | 手脚：工具运行时 + 安全缰绳 | +~2500 行 | 06、07 |
| M3 | 出口：LLM 词汇表 + 多适配器 + 重试 | +~800 行 | 05 |
| M4 | 骨架：mini-cordis + 启动装配 | +~600 行 | 08、09 |
| M5 | 认知：分身 + 自律五件套 + 管家四件套 | +~3000 行 | 11、12、13 |

合计约 9000 行 TypeScript——数量级上远小于 DSH 的实际体量（它还有前端、多平台沙箱、大量产品打磨），但**架构要素一个不少**。

---

## M0 · 心跳（第一个周末）

**目标**：模型可以连续调工具直到任务完成。

```
DriverMachine ─ while(runTurn)
turn: append turn/start → claim → step → turn/end{reason}
step: renderPrompt → llm.chat → [tool calls?] → execute → loop
```

- 内存 messages 数组暂可接受（M1 会消灭它）；
- 工具先用两个硬编码的 echo/read_file；
- LLM 调用直接 fetch 任意 OpenAI 兼容端点（M3 才正规化）。

**验收**：
1. "读取这两个文件并总结差异"能在 ≤6 步内收敛结束；
2. 中途 Ctrl+C 后进程干净退出且不留半条日志记录（哪怕日志还只是 stdout）；
3. reason 闭集六种值在单元测试里各命中一次。

---

## M1 · 账本（第二周）

**目标**：历史成为 append-only 事件流；崩溃无损。

按第 04 篇复刻清单执行：SessionEventMap ≥8 种事件、append 五道门卫、seq=log.length、关系不变量（括号配对、call/result 成对）、deriveMessages 投影（generation 缓存增量折叠）、JSONL 追加 + link 发布 + 截断回滚 + 撕裂尾修复。

**验收**：
1. M0 的循环改为从 deriveMessages() 取历史——行为不变，这证明"投影驱动请求"成立；
2. append 到一半 kill -9，重启加载无重复 seq 且转录完整（合成 interrupted 收尾是否合理交代）；
3. 日志里手工混入非法序列（result 先于 call）→ 加载器拒绝。

---

## M2 · 手脚（第三至四周）

**目标**：任意能力以标准形状插入；危险的活有兜底。

1. `ToolDefinition` + `defineTool` 工厂 + output 三件套（先只实现 schema/render，finalize 可缓）；
2. 九站流水线全量：pre-execute（禁改参）→ ask 降级 deny → **单调 guard** → 双重取消检查 → around+signal 熔合 → body+输出校验 → post-execute → finalize 冻结广播；
3. 三个内置工具：read/write/bash；bash 接入你平台的真实沙箱（macOS seatbelt 或 Linux bwrap），denial signature 渲染成 `[sandbox: …]` 文案；
4. 并行调度 maxParallelToolCalls + exclusive 屏障（fail-closed 分类）。

**验收**：第 06、07 两篇的全部自测题（恒为 deny 的守卫、INVALID_TOOL_OUTPUT、schema 不显示不存在的能力等）。

---

## M3 · 出口（第五周）

**目标**：换供应商不换系统。

1. 三张词汇表落型：Message+BlockMap、StreamChunk+FinishReasonMap、LlmFailure 错误码闭集；
2. `LlmAdapter` 抽象类 + 注册表（原子替换句柄）；prepareCall 式代际绑定；
3. 一只真适配器：SSE 解析、translate 状态机（非空首 delta 开块/index 拼 tool-call/finish 延迟 flush）、content 空串纪律、usage 相减成不重叠桶；
4. retry 插件：数据化策略 + Retry-After 礼让 + 日志账本跨重启恢复计数；
5. usage 随 assistant/message 入账。

**验收**：第 05 篇的三道自测题（看门狗防挂起、null-content 炸网关安全、kill -9 后重试计数续接）。

---

## M4 · 骨架（第六周）

**目标**：插件化重组整个系统，热插拔成立。

按第 08 篇复刻 mini-cordis（proxy ctx、provide/effect、十行 waterfall、五种分发、isolate 标签）；然后把 M0-M3 的硬编码接线全部改造为插件注册：工具注册表、事件监听、适配器、甚至你的循环本身都变成树上一行 entry。配上第 09 篇的三层 patch 栈与 dump=mount 同函数原则。

**验收**：
1. 写一个第三方插件在你的循环 pre-step 处 reject 全部输入并注入自定义欢迎语——不改一行内核代码；
2. 运行中卸载该插件，注册项无泄漏（句柄计数不涨）；
3. dump 输出喂回系统能重建完全相同的树。

---

## M5 · 认知层（第七周起，可裁剪）

按需选装，每个都是独立增量：

| 选装件 | 最小版本要点 | 出处 |
|---|---|---|
| subagent 注册表 | spawn/fork 统一 seed 数组 + report 授权 + 描述符入账 | 11 |
| 自律三件套 | todo 整表 / goal 三元组续轮协议 / schedule followup 即投递 | 12 |
| 管家四件套 | meter 锚点判定 / spill 双端预览 / pruner 打点 / compaction 八拍 | 13 |
| Code Mode | worker 隔离 + 分阶段回调桥（复用 M2 的九站） | 06 |
| Web 门面 | POST+downlink WS + seq 窗口（前端可选任意栈） | 10 |

---

## 十条最值得抄写的代码级技巧

从源码里析出的可直接搬运的小件，按出现顺序：

1. **先拍快照再操作**（send 里 wakingAfterAbort 先捕获再 splice）——一切竞态防护的第一 reflex；
2. **信封带 ignorable 字段**——忘打标记的代价是过度拒绝而非静默误读；
3. **chunk 级原样入库 + sourceEventSeqs 因果引用**——回放保真的双子星；
4. **guard 无 allow 结果**——把权限翻转问题从逻辑层面消除；
5. **包装信号要熔合不要替换**——caller 取消权不可被中间层吞掉；
6. **决策闭集 {allow,deny,ask} + 结构化降级文案**——ask 无人应答也要说清"没渠道"≠"被拒绝"；
7. **介质决定发布协议**——追加型用 link+fsync+截断回滚，净态用 rename 原子替换；
8. **adapter maxRetries:0**——可见的努力必须经过账本；
9. **种子数组表达继承语义**——fork/spawn/前情摘要全是 seed 长度的函数；
10. **dump 与 mount 共用一个 compose 函数**——文档即真相的技术兑现。

## 常见坑清单（前辈们的血泪位）

- Loader 有配置写回习惯：启动时必须重写根配置，否则叠加爆炸（09）;
- 广播给同步观察者看的是入账后状态——设计 mutation 顺序要先记账后改内存（03）;
- 逐 token chunk 信封开销约 56× 载荷：不做打包运营成本会教训你（04）;
- content:null 在部分兼容网关会 brick 整个会话——空串纪律要刻进序列化层（05）;
- pre-step 表决前已 claim：想撤回就得另设 dead-letter，事后补不如一开始定契约（03）;
- compaction 切口若不在乎 tool 配对平衡，下一次请求就是协议违规现场（13）。

## 最后的建议

复刻不是抄代码，是**借结构**。上面每个里程碑刻意留了自由度：语言可以是 TS/Go/Rust/Python；存储可以是 JSONL/SQLite;前端可以根本没有。真正必须对齐的是那三条架构原则和这套事件词汇的设计品味。当你第一次在自己的循环里看到"发给模型的 == 从日志重推的"断言变绿时，这门课就毕业了。

最后一站 [15 · 超越 DSH](./15-超越：DSH%20的弱点与你的机会.md)——弱点清单与创新提案，送你出发。
