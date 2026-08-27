# 10 · 脸面：Web 前端与协议

> 本篇目标：搞懂浏览器里的像素如何与本地进程里的事件对话。协议怎么设计才能既实时又断线不乱？流式渲染怎么做才不掉帧？第三方前端想接入，最小接口面是什么？

## 从一个问题开始

agent harness 的前端有个独有难点：它要渲染的不是静态页面，而是一条**永续追加的事件流**——token 在流动、工具在跑、审批在等人、随时可能断线重连。天真做法会产生经典 bug 集合：

| 天真做法 | 你将收获的 bug |
|---|---|
| WebSocket 双向随便发 | 服务器成了黑箱 RPC 端点，审计困难 |
| 断线后重新拉全量历史 | 大会话秒级卡死；并发写期间还会撕裂 |
| 收到 chunk 就 setState 重渲 | 打字机效果掉帧、markdown 反复闪烁重排 |
| 工具卡片由前端解析输出文本 | 服务器改个格式前端就崩 |

DSH Web 栈的正确打开方式分四步看。

## 第一步：信任围栏，而不是登录墙

本地服务面向本机浏览器开放，DSH 刻意**不做账号密码**，取而代之的是一套"可达性围栏"（`packages/client/connection/src/api-request-trust.ts:96-123`）：

- Host 必须是 loopback 或命中 `trustedHosts` 配置；
- `sec-fetch-site: cross-site` 一律拒绝；带 Origin 时必须与 Host 同 authority（防 DNS 重绑定）；
- **特权方法表**（改设置、读凭证、选目录等管理面操作）就算你配置了 trustedHosts 也钉死 loopback——实现方式很妙：对这类方法用一张空信任表把围栏再过一遍。

注释明言这是刻意的："围栏是可达性策略，不是鉴权"。不是偷懒，而是对本机工具类产品威胁模型的清醒判断。

## 第二步：上下行不对称的传输设计

协议的第一原则是**双向不对称**：

```
上行（浏览器 → 进程）：一律 HTTP POST
    POST /api/session.prompt        ← 方法名即路径
    信封 { type:'client-request', rpcId:<前端铸造的uuid>,
           method:'session.prompt', payload:{…} }
    响应必须原样回显 rpcId 且再过一层 schema 校验

下行（进程 → 浏览器）：downlink-only WebSocket（进程内载体为 SSE）
    GET /api/events.mux   ← 会话事件与控制帧混编
    GET /api/events.host  ← 主机级状态（列表增删、运行状态）
    浏览器若敢发消息？服务器立刻以 1008 "downlink only" 关连接
```

为什么这么偏执？因为 uplink 走 HTTP 意味着每个请求天然有 identity、超时、日志三件套；而 downlink 纯推送没有回包负担。两边各用最擅长的形态。两路流量之上还有一道**严格代际握手**（`ConnectionController.loop`）：同一代内同时开 mux/host 两流 + `host.describe` 一跳成功才算 connected；任一断开整代作废，抖动退避重来。状态机只有 `'connected'|'reconnecting'` 两态——简单到不会有第四种坏状态。

## 第三步：seq 对账窗口与整集快照

断线重连如何不错不错乱？答案一句话：**seq 是唯一对账键**。

每个会话事件有全局单调 seq（第 04 篇的信封字段）。客户端 `Session` 维护一个连续窗口（`sessions/session.ts:688-713`）：

```
收到 seq ≤ 窗口尾     → 丢弃（重放重叠，正常现象）
收到 正好 = 窗口尾+1  → appendLive 追加并喂给折叠引擎
收到 出现空洞          → 缓冲 + 触发尾页历史重拉，缝合成连续区间后再统一喂入
```

另有一类**整集控制帧**（queue/jobs/projection 快照）：不推 diff 只推全集，last-write-wins 落地。看起来浪费带宽，实际上让编辑排队消息、取消任务、第二个标签页接入这些场景天然收敛——一致性逻辑薄到不需要分布式思维。

最后是一处体现深度的设计：需要人类回答的事项（提问 ask-user、审批 approval）以 **answerable server-request 帧**出现在下行情上，且携带稳定 rpcId；mux 重连时基线重放会把仍挂起的请求帧**原 id 重发**（`api-proxy.ts:3334-3345`）。于是用户刷新页面照样能回答三分钟前的提问；应答走固定端点 `POST /api/respond` 按 rpcId 路由回去，首个响应者赢。挂起的 promise 泄漏也不存在——provider 卸载时所有悬案统一按 cancelled 结算。

## 第四步：事件溯源式渲染管线

现在把 `assistant/chunk` 变成屏幕文字。总路线是"服务器给事实，前端做折叠"，全程不存在第二状态源：

```
session/event 帧 ──▶ seq 对账窗口 ──▶ ConversationNodeAssembler
                                          │ 按 Definition 匹配折叠
                                          ▼
                                   Node 树（keyed）
                                          │ 发布节奏（none/animation-frame/immediate）
                                          ▼
                              <ChatNodeSeat> keyed 座位 × N
                                          │ 按注册的分发到具体 renderer
                                          ▼
                          IncrementalMarkdownParser 增量解析
```

### 折叠引擎：ConversationNodeDefinition

每种对话节点是一份八件套声明的纯函数状态机（`runtime/src/client/contract/conversation.ts:171-228`）：`kind/target/match/start/update/publication/buildLocationData/buildViewNode`。以最重要的 `assistant-step` 为例（`ui-conversation/src/client/conversation-nodes/assistant.ts:245-310`）：

```ts
match(type, ev) {
  if (type === 'step/start') return 'start'
  if (type === 'assistant/chunk' || type === 'llm/retry') return 'update'
  ...
}                                    // 同一 step 的所有 chunk 归并进 ${turn}:${step} 一个节点

updateChunk(node, chunk) {
  switch (chunk.type) {
    case 'text-delta': 追加文本
    case 'tool-call-delta': argumentsRaw 累加      // ← 还记得第05篇的铁律吗？
    case 'block-end': 固化该块
    ...
  }
}

publication(ev) {
  return ev.type === 'step/start' ? 'none'            // 开壳不发版
       : ev 是 chunk ? 'animation-frame'               // RAF 合帧节流
       : 'immediate'                                   // 定稿立即发布
}
```

UI 层插件可以注册新的 Definition 处理新事件类型——又是一切皆插件。未注册类型的兜底节点保证未知事件永不白屏。

### 渲染性能的两根支柱

**keyed 座位**：ChatView 为每个 node 发一个 `<ChatNodeSeat>`，每个座位只订阅自己的 nodeKey——兄弟节点的更新绝不会触发自己的重渲（`chat/ChatNodeSeat.tsx:23`）。座位内部经 `renderSlot('conversation.chat.node', …)` 按 node.kind 分发给注册好的 renderer，插件可注入自定义渲染。

**增量 markdown 解析**：`IncrementalMarkdownParser` 冻结除最后 2 个顶层块外的全部内容（`UNSTABLE_TAIL_BLOCKS = 2`），每次只重新解析尾部（`ui-primitives/src/markdown/incremental.ts:75-120`）；块的 React key 取其源文本绝对 offset，跨 chunk 稳定杜绝 remount 闪烁。流式渲染成本从全文 O(n²) 降到每块 O(1)。还有个小细节见功夫：流式中用不含数学扩展的 GFM 方言（半截 TeX 不至于闪报错），定稿后切换含 math 扩展重解一遍。

### 前端本身也是一棵插件树

浏览器启动时读取 `window.__DSH_BOOT__` bundle 图清单（服务端扫描各包 package.json 的 `dsh.client` 声明生成），逐个从 `/plugins/<id>/client.js?rev=<内容哈希>` 加载进冻结的模块表，然后同样跑一个 cordis Context。React 组件树永远不直接碰 ctx——五件套 hooks（useSession/useSessions/useWorkspaces/useStore/renderSlot）是唯一入口。

## 第三方前端的“最小接口面”

如果你想做自己的 DSH 前端（桌面壳、IDE 插件、终端 TUI），面非常小：

1. **三个 primitive**：`POST /api/<method>` JSON 信封、两条下行 WS 流、`POST /api/respond`；
2. **一个可选捷径**：直接复用 `dsh-client-runtime` 包的 SessionManager/Session/折叠引擎（零 React），只换皮；
3. **整体换底座**：页面全局 `__DSH_TRANSPORT__` 可替换物理传输——Electron 用 IPC 桥发 fetch、worker 里用 postMessage 隧道，业务代码零改动；
4. 类型契约以浏览器安全的独立包发布（rpc-map.ts 与 events.ts 的 schema）。

这个最小面就是 DSH 前端生态能存在的原因——按第 09 篇的说法，这也是发行数据的一部分而非代码特权。

## 三原则对账

- **原则一**：线上传的是裸事实事件+现算的渲染提示；前端唯一的账本是日志的镜像窗口；
- **原则二**：折叠定义、座位渲染器、settings 卡片全是插件位；传输底座本身可整体替换；
- **原则三**：rpcId 回显校验、双级 schema 校验畸形帧丢弃不杀流、answerable 帧原 id 重放恢复。

## 复刻清单

- [ ] 上行 POST 信封（rpcId 铸造/回显校验）+ 下行 downlink-only 流 + 1008 关闭滥用；
- [ ] 代际握手状态机：describe 成功 ∧ 双流开 → connected，缺一抖动退避；
- [ ] seq 窗口：落后丢弃 / 连续追加 / 空洞缓冲拉页缝合；
- [ ] answerable 帧：稳定 rpcId、基线重放恢复、首答者赢、provider 卸载统一 cancelled；
- [ ] ConversationNodeDefinition 八件套契约 + assistant-step 参考实现（RAF 合帧）；
- [ ] keyed 座位订阅模型 + 增量 markdown（尾巴 N 块重解 + offset 作 key）。

验收标准：
1. 人为打乱事件投递顺序再补发缺口——最终画面与顺序投递是否完全一致？
2. 断网 30 秒内刷新两次，第三分钟那个待审批弹窗是否依然可答？
3. 往会话塞一条前端不认识的新事件类型——是否显示兜底卡片而非崩溃？

## 超越思考

- **投影已经开得很宽了，多客户端协同还没起步**：两个标签页同时编辑队列时是各自 last-wins 竞速；presence/协作光标这类能力其实可以在整集快照机制上自然生长；
- **离线队列**：弱网环境的本地草稿乐观上屏目前靠 blank 位翻转近似处理，CRDT 式的消息状态机值得探索；
- （换个角度想：当你能给同一棵插件树接任意多形态的前端时，"前端"这个词本身就到了被重新发明的时候。）

---

单个 agent 的故事讲完了。但真正的复杂任务从来不是一个 agent 干完的。下一站：[11 · 分身：Subagent 与多智能体](./11-分身：Subagent%20与多智能体.md)。
