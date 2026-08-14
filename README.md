# DeepSeek Harness 深度解读

> 把 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）读透：一份**从上往下一遍读完、不用来回翻**的中文深度解读。

DeepSeek Harness 是 DeepSeek 开源的 agent harness，采用"一切皆插件"架构。这份解读面向**对该领域零了解**的工程师读者：前置知识垫层在前、每个概念首次出现即完整定义、每个源码论断带 `file:line` 证据锚点、承重名词锚点案例贯穿全文——按顺序读完一遍，即建立对整个系统的完整心智模型。

## 怎么读

按顺序读，不需要跳读回翻：

```
00-总览.md            ← 从这里开始（自包含，单读总览也成立）
└── parts/
    ├── 01-cordis-插件底座.md
    ├── 02-核心spine与一轮turn.md
    ├── 03-llm能力层与流式管道.md
    ├── 04-工具seams与执行管道.md
    ├── 05-会话数据面与持久化.md
    └── 06-组合发行与多前端.md
```

| 篇 | 定位 | 一句话 |
|---|---|---|
| **00 总览**（8.6k 字） | 全景 | 垫层 + C4 全景 + 三个关键场景主线 + 六篇导览 |
| **01 Cordis 插件底座** | 垫层 | 一行 cordis.yml 如何变成活插件又被干净卸载——Cordis 把卸载做成一等公民，dsh vendor 它并打了 18 条修改补丁 |
| **02 核心 spine 与一轮 turn** | 主战场 S1 | 一句话走完一轮 turn 的状态机与事件序列，model-visible ⟺ logged 由 invariant 逐字节断言，并验证了 architecture.md turn flow 假设成立 |
| **03 LLM 能力层与流式管道** | S1 模型段 | 一次调用拆成词汇→seam→adapter 三段，双 waterfall 分工、twin adapters 证明替换性，重试与计量都遵守"可见皆可重建" |
| **04 工具 seams 与执行管道** | 主战场 S2 | 从 tool_use 到进程的 approval→sandbox→spawn 七站，三角色 seam 范式在十几个能力上机械重复，subprocess 是唯一进程 substrate |
| **05 会话数据面与持久化** | S1 durable 侧 | 200ms 批窗口 + 三个语义检查点，崩溃不截断工作，resume/fork 走版本闸与合成关闭——日志是唯一被当事实对待的字节 |
| **06 组合发行与多前端** | 主战场 S3 | 组合层是纯函数 + 一摞 YAML（78 行 insert 在空根上长树），preset standing mount 补完 Web 浏览器/宿主分半，五种发行形态共享同一组合语义 |

`assets/` 内附目录树快照、repo map（PageRank 枢纽文件排序）、组合树全表等物化资产。

## 证据与版本约定

- **版本锁定**：本解读基于 **2026-08-14 下载的 master tarball 快照**（无 git 元数据，故无 commit hash）。对照锚点：根 `package.json` 版本 `0.1.0-rc.5`；其 SHA-256 前 40 位 `3fec1dd1d3faf02a591c8cf792a1eeedf810491c`（`shasum -a 256 package.json` 可复算）。
- **`file:line` 锚点**：均为仓库根相对路径（如 `packages/core/agent/src/agent.ts:255`），按上述快照逐条验证过存在性与行内容。建议 clone [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 后用编辑器 goto-line 对照阅读。
- **行为论断三层标注**：`已验证（跑过）` / `静态推断` / `未验证`。解读中标注"已验证"的测试论断（vitest 共 4100+ 用例：config-reload 12/12、user-patches 16/16、core 五包 815、core/tools 386、llm 675）全部实际运行复现过。
- 源码里的 TODO/FIXME 标记在解读中以「待办标记（代号 xxx）」转述，避免与文档占位符混淆。

## 这份解读怎么做的

由 deep-read 方法论驱动（对仓库/书籍/长文章的"深读"流水线）：

```
G0 材料门（规模估算/判型/产出计划）
 → P1 检视扫描（X-ray：主旨+骨架+问题清单）
 → P2 粗读成 gist（结构边界切分，追踪器演化）
 → P3 分析精读（问题驱动 + file:line 取证 + 运行验证）
 → P4 综合写稿（单线叙事，总览+分篇）
 → G3 lint（0 error 硬门：章节/图链/前向引用/锚点存在性）
 → G4 全面终审（线性走查 + 证据抽样 + 深度 7 问）
 → G5 修订循环（≤3 轮，逐条消化）
```

质量证据：7 篇全部通过 lint（0 error；仅 1 条已确认的启发式误报）；终审抽查 12+ 条 `file:line` 与源码逐字一致；版本锚点哈希复算一致；发现的 1 处事实错误与多处计数口径问题已在终审修订中修正。

## 致谢与许可

- 解读对象：[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（MIT License, Copyright (c) 2026 DeepSeek）。本解读引用的代码片段与结构事实版权归 DeepSeek 所有。
- 本仓库的解读正文以 [CC BY 4.0](./LICENSE) 发布。
