# 评估 Harness（Evaluation Harness）：给 Agent 打分的台架

## 一、什么是评估 Harness？

**评估 Harness = 围绕 agent 搭建的自动化测试与打分框架。**

它本身不是 agent 的一部分，而是一套**外部裁判系统**：给 agent 出题、观察它怎么做、检查结果对不对、最后给出分数或排行榜。类似实验室里的测功机之于汽车——汽车能不能跑不取决于测功机，但我们需要它来判断汽车跑得多好。

> **术语消歧义**：本笔记讨论的是 **evaluation harness / test harness**。与之平行的另一个概念是 **runtime harness / agent harness**（把 LLM 包装成能干活的 agent 的运行时脚手架），见 [[harness_engine_architecture]]。两者同名但不是一层东西——前者是"裁判"，后者是"选手的身体"。

---

## 二、核心用途

### 1. 评估（Evaluation）
给 agent 设定标准化任务，自动检查它的输出是否正确、行为是否符合预期。输出是**可量化指标**：通过率、accuracy、F1、BLEU、resolve rate 等。

### 2. 基准测试（Benchmarking）
在同一套题目上对比不同 agent、不同模型、不同 prompt 策略。只有固定了"考场"，比较才有意义。典型产物是排行榜（leaderboard）。

### 3. 沙盒执行
提供隔离的运行环境（通常是 Docker 容器或虚拟机），让 agent 安全地执行代码、操作文件、调用工具。**隔离是必要的**——你不希望 agent 在评测过程中把宿主机的文件删掉。

### 4. 可观测性 & 回放
记录 agent 的每一步决策、工具调用、中间状态。评测失败时可以回放 trace 做根因分析，也可以用来做可解释性研究。

---

## 三、典型架构

```
┌─────────────────────────────────────────────────────────────┐
│                   Evaluation Harness                        │
│                                                             │
│  ┌────────────┐   ┌──────────────┐   ┌──────────────────┐   │
│  │  任务数据集  │──▶│  任务分发器   │──▶│  Agent Runner   │   │
│  │  (题库)    │   │  (取一道题)   │   │ (执行被测 agent) │   │
│  └────────────┘   └──────────────┘   └────────┬─────────┘   │
│                                                │            │
│                                                ▼            │
│  ┌────────────┐   ┌──────────────┐   ┌──────────────────┐   │
│  │  打分器     │◀──│  结果收集器  │◀──│     沙盒环境     │   │
│  │  (Scorer)  │   │  (Trace)     │   │  (Docker / VM)  │   │
│  └──────┬─────┘   └──────────────┘   └──────────────────┘   │
│         │                                                   │
│         ▼                                                   │
│  ┌────────────┐                                             │
│  │  报告 / 榜  │                                             │
│  │  (Report)  │                                             │
│  └────────────┘                                             │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼ 被测对象
              Agent 运行时（可能是 Claude Code、Cursor、
              或其他 runtime harness + LLM 的组合）
```

---

## 四、典型例子

### SWE-bench —— agent 评估 harness 的代表作
流程：
1. 从数据集取一个真实的 GitHub issue + 对应仓库的 commit 快照
2. 把仓库克隆到 Docker 容器里
3. 让 agent 读 issue 和代码，生成补丁（patch）
4. harness 自动应用 patch
5. 运行仓库自带的测试用例
6. 记录 PASS / FAIL，汇总成 **resolve rate**（问题解决率）

**关键观察**：SWE-bench 必须在内部运行一个 agent 来产出补丁，所以它**嵌套了一个简化的 runtime harness**（或直接调用 Claude Code 这类现成的 runtime harness）再加外层评估逻辑。这就是两个 harness 概念最容易混淆的根源——评估 harness 把运行时 harness 当作"被测物"包在里面。

### lm-evaluation-harness（EleutherAI）
偏向纯语言模型的评估框架，侧重 zero-shot / few-shot 能力测试。覆盖 MMLU、HellaSwag、ARC 等 200+ 任务，但基本不涉及工具调用，所以严格说是 "LLM eval harness" 而非 "agent eval harness"。

### OpenAI Evals
开源的评估模板库。特点：支持用 YAML 快速定义新评测、支持 **model-graded eval**（用另一个 LLM 当裁判，适合无法机器判分的开放题）。

### HumanEval / MBPP runner
代码生成评测的经典 harness：给函数签名和 docstring，让模型生成实现，然后跑单元测试判断 **pass@k**（生成 k 个解里至少一个通过的概率）。

---

## 五、关键设计要素

| 要素 | 说明 | 为什么重要 |
|------|------|-----------|
| **确定性** | 同一 agent 对同一道题，多次评测分数应接近 | 否则排行榜没有可信度 |
| **隔离性** | 每道题在独立沙盒里跑，互不污染 | 防止状态泄漏、防止 agent 搞破坏 |
| **可复现** | 固定数据集版本、容器镜像、随机种子 | 别人才能复现你的分数 |
| **公平性** | 所有参赛 agent 拿到相同输入和相同资源上限 | 避免"偷看答案"或"用 100 倍算力换分数" |
| **自动打分** | 机器判分优先于人工判分 | 人工评分无法 scale |
| **可观测** | 保留完整 trace 便于事后分析 | 调试、归因、可解释性 |

---

## 六、评估 harness vs 运行时 harness 对照

| 维度 | 运行时 harness | 评估 harness |
|---|---|---|
| **作用** | 让 agent 能干活 | 判断 agent 干得好不好 |
| **跑的时机** | 用户每次使用都在跑（永远在线） | 离线 / CI 跑一次拿数据 |
| **谁在用** | 终端用户 | 研究者、基准作者、模型团队 |
| **输出** | 真实副作用（改文件、调 API） | 分数、通过率、排行榜 |
| **典型代表** | Claude Code、Cursor、Aider、OpenHands | SWE-bench、lm-eval-harness、OpenAI Evals |
| **类比** | 汽车的底盘、方向盘、刹车 | 汽车测试场的测功机、碰撞台 |
| **嵌套关系** | 被评估 harness 包在里面作为"被测物" | 从外部包住运行时 harness |

---

## 七、一句话总结

> **运行时 harness 是"马的挽具"（让马能拉车），评估 harness 是"赛马场"（看哪匹马跑得快）。**
>
> 两者都借用了英文 harness（挽具/套具）这个词，但一个强调"套住让它能工作"，一个强调"套住让它被观察和打分"。

---

## 八、延伸阅读

- [[harness_engine_architecture]] — 运行时 harness 的深度分析（以 Claude Code 为例）
- SWE-bench — 最具代表性的 agent eval harness
- EleutherAI `lm-evaluation-harness` — LLM 评测的事实标准
- OpenAI Evals — 模板化的通用评估框架
