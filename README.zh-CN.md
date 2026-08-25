# review-bounded — 三轮有界代码审查 skill

<p align="center">
  <a href="README.md">English</a> | <a href="README.zh-CN.md">简体中文</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="License"/>
  <img src="https://img.shields.io/badge/AGENTS%20Skills%20Standard-compatible-purple?style=flat-square" alt="AGENTS Skills Standard"/>
</p>

> **三轮 AI 代码审查 + 固定严重度标尺——把"要不要继续 review"从无限循环变成数据决策。**
>
> **review → fix → 判定。** 一个 skill，零构建，零依赖。

## 这是什么

review-bounded 是一个 [Agent Skills](https://agentskills.io/specification)（SKILL.md）包：在当前分支上编排**恰好三轮** review-and-fix——所有问题按同一把固定标尺打 P0-P3、每轮修复前征求你的授权、三轮后给出收敛判定。

**为什么叫 "bounded"（有界）？** 因为让无限循环停下来的唯一可靠方法，是事先承诺一个预算——三轮，不多不少。

## 为什么 AI review 永远没有终点

```
"review一下当前改动" → 发现问题 123 → fix
"再 review" → 又发现问题 456 → fix
"再 review" → 又发现问题 789 → fix
"再 review" → ……
```

AI code review **不存在"零问题"终点**。这不是某个模型的缺陷，而是结构性的：

- **什么都不报的审查者，等于没完成任务。** LLM 的训练目标就是"有帮助、够彻底"；让它 review，产出 findings 本身就是被奖励的行为，沉默没有任何收益。
- **提出批评对模型零成本。** 误报不会伤害模型——付出成本的是你（多一轮往返）；漏掉真实 bug 对模型不可见。产出 findings 永远廉价。
- **每一轮都是重新采样。** 每次新的 review 都会从新的角度重读代码；新一轮总能提出上一轮没看到的东西。
- **没有"完成"的基准。** 流程里不存在 ground truth 说"可以停了"。循环只能由人主动终止。

所以"再 review 一次"永远能找到更多问题。真正的问题不是 review 能不能找出问题——而是**剩下的问题还有多重**。

## 核心：问题更轻，而不是问题为零

AI review 永远到不了零，但它**收敛**：严重度逐轮下降。

```
Round 1:  死锁、数据丢失、安全漏洞            (P0/P1)
Round 2:  性能、健壮性、错误处理              (P2)
Round 3:  命名、死代码、风格                  (P3)
```

代码审查的目的**不是**"零发现"，而是把发现的问题**减轻到可接受的程度**——并且有证据地知道，这个点到了。

本 skill 让收敛可度量，让"停止"成为你能站得住的决定：

1. **恰好三轮**——固定预算，"再 review 一次"不可能变成无限循环；
2. **三轮共用同一把标尺（P0-P3）**——"问题是不是变轻了"变成数字，而不是感觉；
3. **每轮修复前征求授权**——AI 提议，你拍板；没有你的同意，代码不会被改动；
4. **CONVERGED / NOT CONVERGED 判定**——附检方陈述（反对合并的最强理由）、最后一轮问题的上线影响、未修复项清单。

## 快速开始

要求：任何支持 Agent Skills 标准的 coding agent（Claude Code、Codex、ZCode 等）。

**安装**——一条命令：

```sh
npx skills add ZM-BAD/review-bounded
```

**发起**——在 agent 会话里：

- ZCode / Claude Code：`/review-bounded`，或直接说「跑三轮 CR」「对这个分支做有界 review」；
- Codex：自然语言——"run three rounds of bounded code review on this branch"（没有斜杠命令）。

开始时由你选择审查范围——当前代码与基准分支的对比、仅未提交改动、当前分支最近若干 commit、或你自己指定的任意范围。然后 skill 跑恰好三轮，给你一个判定。

> 仅交互式使用：按轮授权和提交决策需要人参与；非交互模式（如 `-p`/CI）不支持。

## 判定长什么样

```text
判定：CONVERGED

元问题1（严重度趋势）：三轮最高级别 P1→P2→P3，P0+P1 数量 2→1→0
元问题2（fix 回归）：未检测到——三轮测试通过 + diff 追溯
元问题3（最后一轮问题）：R3-1（P2，边界）：仅在畸形输入时出问题

检方陈述：
1. 新增缓存模块没有淘汰路径（代码证据）
2. 端到端验证未运行（测试证据）
3. 两个辅助函数命名偏离约定（推测）

未修复项：
- [ ] R2-2：重试逻辑无退避（建议处理方式）
```

**CONVERGED** 意味着最后一轮的问题很轻、没有发现修复引入的回归、所有未决事项都在清单里。**NOT CONVERGED** 意味着你确切知道 reviewer 还在担心什么——继续 review 是正确的选择。

## 工作方式

1. **第一轮**：按初始确认的审查范围（当前代码与 main/master 的对比 / 仅未提交改动 / 最近若干 commit）审查，汇报问题（P0-P3），征求授权，在工作区修复（不提交），跑回归检查（有测试/lint 就跑）。
2. **第二轮**：重新读取最新代码（禁止凭记忆），优先看上一轮 fix 的增量和未覆盖的类别，与第一轮去重，然后同样的 汇报 → 授权 → 修复 → 验证 循环。
3. **第三轮**：同一循环，然后输出最终判定。全部修复均未提交，由你决定是否提交、如何提交（单 commit 或按 finding 分组）。

每一轮都通过 `git diff` 和重新读文件审查**实际最新代码**，绝不凭记忆重构。修复只改工作区、**过程中不提交**；最终判定后向你展示全部修复清单，由你决定是否提交及如何提交。轮次记录保存在对话内，**不产生额外文件**（用户要求留档时，agent 才把汇总写入 `REVIEW.md`）。

## FAQ

- **还需要人工 review 吗？** 需要——这正是它的意义。AI 汇报，你逐轮授权，最终提交决策也是你的。skill 按设计仅交互式使用。
- **AI 会不会自己把问题全改了？** 不会。没有你的授权，代码不会被改动；三轮过程中不产生任何 commit。
- **三轮够吗？** 判定会回答这个问题：CONVERGED 说明剩余问题很轻、没有隐藏项；NOT CONVERGED 会明确指出 reviewer 还在担心什么——认同就继续。
- **支持我的语言/框架吗？** 它是纯指令，不是 linter：agent 能读什么（代码、测试、文档）就能审什么。无运行时、无依赖。

## 目录结构

```
SKILL.md                  # 编排核心：轮次循环 + 记录方式 + 铁律
references/
  rubric.md               # P0–P3 严重度标尺 + 审查类别清单（第一轮前加载）
  round-prompt.md         # 单轮指令：最新代码 / 去重 / 证据 / 授权 / 回归检查
  decision-rules.md       # 三轮后收敛判定规则 + 检方陈述 + 输出格式
```

## 设计原则

- **编排层，非执行层**：执行用 agent 原生工具（git、文件、Bash）和项目已有测试/lint，按 `references/round-prompt.md` 内置指令直接执行，不委托其他审查 skill。
- **零运行时**：纯 Markdown，无脚本、无依赖、无 API，可移植到所有支持 SKILL.md 标准的 agent。
- **结构性约束写死**：第一轮前固定 rubric、每轮修复前征求授权、"不确定一律 NOT CONVERGED"、判定前先写检方陈述——这些不依赖模型自觉。
- **每轮都是新代码**：记忆永远不能替代读取仓库真实状态。

⭐ 觉得有用？Star 一下，分享给还在 review 循环里挣扎的团队。

## License

[MIT](LICENSE)
