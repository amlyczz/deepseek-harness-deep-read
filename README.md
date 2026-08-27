# 从零看懂 DeepSeek Harness

> 把 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（DSH）彻底拆开：一份**写给普通人也能读懂、但工程师拿去就能复刻**的中文深度解读。

DeepSeek Harness（`dsh`）是 DeepSeek 官方开源的 agent harness（智能体运行框架）——给大模型装上"手脚、记忆、感官和安全带"的那层软件。本系列的写法是**构造式**的：每一章都从"如果你自己动手造，会遇到什么问题"出发，先给出天真的做法和它的坑，再看 DSH 是怎么解决的、为什么这样解决，最后给你一张**可执行的复刻清单**和一组**超越思考**。

读完你将获得：

1. 一条完整的心智模型——从敲下回车到屏幕渲染出 token，中间每一步发生了什么；
2. 每个论断都有 `文件:行号` 源码证据，可直接到仓库验证；
3. 一条从 500 行最小原型到全功能 harness 的复刻路线图；
4. 一份 DSH 的弱点清单与超越它的创新提案。

## 阅读路径

按顺序读即可，不需要来回翻。全部文章在 [`guide/`](./guide/) 目录：

| 篇 | 标题 | 你会学到 |
|---|---|---|
| [导读](./guide/README.md) | 这个系列怎么读 | 全书的逻辑主线与三条贯穿原则 |
| [01](./guide/01-开篇：Agent%20Harness%20到底是什么.md) | 开篇：Agent Harness 到底是什么 | 大脑与手脚的比喻；DSH 的定位；跑起来它 |
| [02](./guide/02-全景：一条消息的一生.md) | 全景：一条消息的一生 | 宏观架构地图 + 一次完整往返的全链路走读 |
| [03](./guide/03-心脏：Agent%20循环.md) | 心脏：Agent 循环 | turn/step 状态机、收件箱、拦截点、取消传播 |
| [04](./guide/04-真相：会话日志.md) | 真相：会话日志 | append-only 事件流；三个视图从一个源头投影 |
| [05](./guide/05-出口：LLM%20适配层.md) | 出口：LLM 适配层 | 消息词汇表、适配器 seam、流聚合、重试 |
| [06](./guide/06-双手：工具系统.md) | 双手：工具系统 | 工具定义、九站执行流水线、Code Mode 压轴创新 |
| [07](./guide/07-缰绳：沙箱与审批.md) | 缰绳：沙箱与审批 | OS 级兜底 × 策略级准入的双保险设计 |
| [08](./guide/08-骨架：Cordis%20插件底座.md) | 骨架：Cordis 插件底座 | 让"一切皆插件"成立的框架机制 |
| [09](./guide/09-组装：启动与发行.md) | 组装：启动与发行 | profile/bundle/patch 三概念；一行命令如何长成插件树 |
| [10](./guide/10-脸面：Web%20前端与协议.md) | 脸面：Web 前端与协议 | 事件溯源式 UI；seq 对账；ask-user 全栈穿越 |
| [11](./guide/11-分身：Subagent%20与多智能体.md) | 分身：Subagent 与多智能体 | fork/spawn、可继续子代理、事件上行、团队实验 |
| [12](./guide/12-自律：模型的自我记账工具族.md) | 自律：自我记账工具族 | todo/goal/plan/jobs/schedule 五兄弟的共同答案 |
| [13](./guide/13-大管家：上下文寿命管理.md) | 大管家：上下文寿命管理 | spill/compaction/token 计量/skill 目录 |
| [14](./guide/14-复刻路线：从零造一个%20harness.md) | 复刻路线图 | M0→M5 六个里程碑，每个都有验收标准 |
| [15](./guide/15-超越：DSH%20的弱点与你的机会.md) | 超越 DSH | 弱点总清单 + 十个创新提案 |

## 怎么自己验证

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
```

文中所有形如 `packages/core/session/src/index.ts:604` 的锚点都相对这个仓库根目录。用编辑器打开对应文件的对应行即可核对。

## 说明

- 本解读基于 2026-08-27 的上游 main 分支（0.1.1-rc.2）；DSH 处于开发者预览期，后续版本可能有破坏性变更。
- 旧版解读已移入 [`archive/`](./archive/)，不再维护。
- 上游源码克隆在本机 `research/` 目录（不入库），讲解仓库只收录自产内容。
