# 08 · 骨架：Cordis 插件底座

> 本篇目标：终于要打开那个"魔法盒子"了。前七章里，我们无数次看到"插件注册工具""监听事件""提供服务"——但没有一章解释过**这套魔法本身是怎么实现的**。本篇拆解 Cordis：一个 2700 行的框架，如何让"一切皆插件"从口号变成机械事实。

## 从一个问题开始

读完前四章你可能已经隐约不安了：DSH 里到处是"注册"，可这些组件互相不知道对方存在——

- 工具流水线怎么找到审批服务？（审批是可选的！没装时它还能优雅降级）
- 压缩插件怎么把手伸进循环的 pre-step？
- 理论上我能不能写个插件换掉 agent 循环本身？

用传统的 `import` 是不可能的：import 是编译期硬连接，A import B 意味着 A 没了 B 就起不来、B 升级 A 必须重发、想把 B 换掉就得改 A 的代码。随着功能膨胀，你会收获一个谁都离不开的"上帝核心模块"，上面列的所有灵活性全部死绝。

而 DSH 的做法是把整个产品架在一个通用框架上。README 的原话："不存在需要打补丁的特权内核：扩展 dsh 的方式是把插件挂载到其他插件旁边，而各项注册都是副作用，会在其插件卸载时撤销。"（`docs/architecture.zh.md:13`）

这个框架叫 **Cordis**（希腊语弦乐合奏团的"乐队席"）。它是独立开源项目，DSH 以 vendor 方式内置其源码——`vendor/cordis/src/` 总共 **9 个文件约 2700 行**。2700 行撑起了 55 个子包的产品，性价比惊人。

## 四个概念：Context、插件、服务、事件

### Context：一个"读属性=查依赖"的容器

每个插件开工时会拿到一个 `ctx` 对象。你写 `ctx.tools`、`ctx.llm`、`ctx.sessions` 时发生什么？真相藏在 Proxy 的 get 陷阱里：

```
读 ctx.foo
  → 自有属性？直接返回
  → 否则沿 Fiber 链逐层找名为 foo 的服务实现   (vendor/cordis/src/reflect.ts:153-166)
  → 找不到？抛 "cannot get property without inject"（错误信息会告诉你缺谁）
```

**ctx 属性读取本身就是依赖解析**。这带来两个后果：服务名就是 API 面（`ctx.fs`、`ctx.sandbox` 都是稳定 key）；以及一条纪律——插件该用的依赖要写进静态 `inject` 列表声明（挂载前框架保证就绪），没声明就读是运行时错误的来源。

三个造子作用域的方法:`extend(meta)`（原型链继承）、`isolate(name)`（给某个服务名换隔离标签）、`intercept(name, config)`（给子树的某服务追加配置）。

### 插件：带元数据的可执行入口

不是类也不是对象，三种形态皆可（`registry.ts:92-133`）：函数 `(ctx, config)=>any`、类 `new (ctx, config)`、对象 `{apply(ctx, config)}`。统一元数据四件套：

```ts
export const name = 'tool-presentation'        // 身份
export const inject = ['tools']                 // 静态依赖列表
export function Config = Schema({...})          // 配置 schema（自动校验）
export function apply(ctx, config) { ... }      // 干活
```

加载入口 `ctx.plugin(P, config)` 会为每次挂载创建一个 **Fiber**（同一份插件代码可以在不同上下文以不同配置起多个实例）。

### 服务：挂在 ctx 键上的受管能力

底层原语 `ctx.provide(name, value)`：以当前 Fiber 为属主写入全局存储，**返回反注册 disposer**；提供者活跃时唤醒所有等待此名的依赖方。上层封装 `Service` 基类在构造时自动 provide、随所属 Fiber 卸载自动注销。可用性谓词还支持"已提供但尚不可用"的状态（比如凭证没配好）。

### 事件：类型化总线 + internal 族

通过 TS 声明合并往 `interface Events` 加条目（`ctx.on('foo', handler)`），运行时只有一个 `EventsService`，内部就是 `_hooks: Record<name, Hook[]>`（`events.ts:132`）。第 02 篇讲的三个事件域全是它的用户。

## 五种分发模式与十行瀑布

```ts
// events.ts:32
export type DispatchMode = 'emit' | 'parallel' | 'serial' | 'bail' | 'waterfall'
```

| 模式 | 行为 | 典型用途 |
|---|---|---|
| `emit` | 同步逐个调，无视返回值 | 广播通知（`agent/status`） |
| `parallel` | 并行全跑完，错误聚合 AggregateError | 无序的多方观测 |
| `serial` | 顺序 await，非空返回即短路并作为结果 | 可否决的表决（`agent/turn-stopping`） |
| `bail` | serial 的同步版 | 快速否决 |
| `waterfall` | 环绕中间件链（洋葱模型） | 策略栈（pre-step/request/tools 流水线） |

waterfall 的完整实现只有十行（`events.ts:234-243`），值得逐行看：

```ts
waterfall(...args) {
  const cbs = this.dispatch('waterfall', args)
  const inner = args.pop()           // 最后一个参数是最内层默认行为
  const next = () => {
    const cb = cbs.shift() ?? inner  // 监听器耗尽后落到内建行为
    return cb(...args)               // args 尾部带着 next 本身
  }
  args.push(next)
  return next()
}
```

语义三条记住即可：**调用 `next()` 放行并把下游结果带回加工；不调直接返回即短路/否决**；默认内建行为由事件声明方注入（例如 system-prompt 组装传入 `() => Promise.resolve(assembly)`）。Koa 洋葱模型×一个类型化事件总线=整个 harness 的策略栈骨架。

配套还有第二个维度——**接收范围过滤**：派发时按接收者上的 `[Context.filter]` 谓词筛监听者（`events.ts:165-175`）。这一机制让"子代理的事件只有它的谱系长辈能看到"成为可能（`scopeTarget` 路由载体，`packages/core/scope/src/index.ts:170-185`，"事件只向上流，永不向下"），也让我们后面讲 subagent 时不需在 payload 里手工过滤。监听器侧可用 `{global:true}` 免疫过滤、`{prepend:true}` 抢先。

## Fiber 与 effect：可逆性是一等公民

这是 Cordis 最值得一学的设计。每个插件实例是一个 Fiber，生命周期五态机：`PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED`（+FAILED）。

**启动由数据驱动**：Fiber 的 `_refresh()` 计算依赖快照 epoch——任一必需服务缺失则 INACTIVE，否则把各提供者 fiber uid 拼成字符串；`_setEpoch()` 只在状态翻转时触发 `_reload`/`_unload`（`fiber.ts:611-639`）。于是"等依赖就绪才启动、依赖消失自动卸载"不是一段逻辑，而是**同一个比较操作的两种走向**。配置热更新同理：`fiber.update(config)` 先过一道 `internal/update` waterfall（任何监听者可改写或否决更新），然后重启自己（736-753 行）。

**一切注册皆是 effect**：核心 API `fiber.effect(execute, label)` 把 execute 产出的清理函数存入 `_disposables` 列表，卸载时**逆序**清空（`utils.ts:28-31` + `fiber.ts:675-686`）。然后框架让自己每个 API 都变成 effect：

| API | 注册的东西 |
|---|---|
| `ctx.on(...)` | 事件监听器（254-260 行包 effect）|
| `ctx.provide(...)` | 服务提供（disposer 注销+唤醒依赖方）|
| `ctx.mixin/accessor` | ctx 方法混入 |
| `ctx.plugin(...)` | 子插件整体作为父 fiber 的一条 effect |
| logger exporter | 日志导出器 |

收益一句话：**任何卸载路径——HMR、父级卸载、依赖消失、手动 dispose——走的是同一条清理代码路径**。写插件的人不需要实现"关闭逻辑"，因为他没写过"打开逻辑"，只写过一堆会被自动回收的副作用。这就是第 09 篇热更新能做对的根源。

**isolate（隔离域）**澄清一个概念：这份 vendored 版没有 `fork()` API。"一次代码多实例"由 Fiber 天然表达；而 isolate 解决另一个问题——同名服务的不同实现互不可见。服务存取都以其 isolate 标签 symbol 为键，相同 label 即同域（`context.ts:121-125` + `reflect.ts:238`），Loader 把它产品化为 realm（entry 私有域 / 同名共享域）。之后第 11 篇 subagent 讲"每个 agent 一套自己的工具"时，地基就是这个。

## 一个真实插件的 52 行解剖

看一个教科书式的小插件（`packages/core/agent-tool-presentation/src/index.ts` 全文仅 52 行）：

```ts
export const name = 'tool-presentation'
export const inject = ['tools']            // 静态声明：我要 tools 服务

export function apply(ctx, config) {
  // 在"调用我的上下文"上登记展示模式；presentAs 本身是 effect，
  // 返回值就是那条精确 disposer —— 但我们什么都不用保存：
  // fiber 卸载时全部自动撤销。
  return ctx.tools.presentAs(mode)
}
```

注意三件事：静态 inject 让挂载器保证顺序；它消费别人的服务而不贡献新键；**没有任何清理代码，因为没有任何不可逆操作**。对比重型插件 `AgentLoop`（引擎本体也是普通插件！）的开场白：

```ts
// packages/core/agent-loop/src/index.ts:295-297
export class AgentLoop extends Service implements AgentFactory {
  static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt']
```

五个依赖、一纸 Config schema（zod 校验、含 `agents: []` 默认值）、构造期两条 effect（transaction 清理器、setFactory）……循环把自己登记成 `ctx.agentLoop` 服务的姿势，和 tool-presentation 登记展示模式的姿势**完全一样**。"没有特权内核"至此有了物证：连心脏都是树上普通的一个节点。

## 这套架构的代价（诚实的评估）

浅尝辄止的吹捧没有价值，说说账单的另一面：

1. **性能税**：每次 ctx 属性读取都过 Proxy 陷阱与服务解析。热路径上无所谓（每轮几千次），但在每 chunk 循环这种位置就要小心设计；
2. **调试的间接性**："这个工具是谁注册的？"答案要顺着 patch 栈、profile 层、runtime 条件逐层还原；好在官方提供了 `--dump-config`（第 09 篇）救场；
3. **学习曲线前置**：理解 dsh 之前必须先接受 Effect、Fiber、epoch 这些心智负担；
4. **启动次序的隐性契约**：epoch 机制解决了"何时跑"，但没解决"配置错了何时暴露"——好在有 `assertEntriesActivated` 启动审计兜底报错。

综合判断：对 DSH 这种以"可组合性"为生命线的系统，这笔交易划算到近乎抢劫；但一个 500 行的单体 harness 完全不需要它——**框架选择跟着产品野心走**。

## 复刻清单

来亲手写一个 mini-cordis（~300 行 TS，本周就能完成的那种）：

- [ ] `class MiniContext`：Proxy get 陷阱转发服务查找；找不到抛出带名字的错误；
- [ ] `provide(name, value)` 返回 disposer；等待方 Promise + 唤醒机制；
- [ ] `effect(fn)` 注册表 + `dispose()` 逆序清空——先写这条，其他一切都建立在它上面；
- [ ] 五种事件分发 + 十行 waterfall 原样复刻；
- [ ] Fiber 简化版：inject 依赖检查 + 就绪才执行 apply；
- [ ] 终极验收：用它重写你的第 06 篇工具注册表，加一个能在运行中卸载的重试插件，确认无泄漏（进程句柄数不涨）。

## 超越思考

- **类型级服务契约**：目前 ctx 键的正确性一半靠运行时报错。可以用 TS 项目引用或生成 .d.ts 校验，让"inject 写错名"变成编译失败而非线上日志；
- **可视化树检视器**：`--dump-config` 是文本形态，运行中的插件树（谁等谁、谁被谁挡住、epoch 卡在哪）值得一个实时的可视化面板——这对插件生态开发者是刚需级工具；
- **插件能力沙盒**：现在插件拿到的是全权 ctx。像 OS 权限那样把"插件可以碰哪些 seam"做成安装声明的 manifest，能把恶意/劣质插件的爆炸半径压下来。

---

骨架看清了。最后一个悬念：一棵树是怎么从一行命令长出来的？那些 patch 文件到底按什么规则叠罗汉？下一站 [09 · 组装：启动与发行](./09-组装：启动与发行.md)。
