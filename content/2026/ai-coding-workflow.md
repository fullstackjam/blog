+++
title = "Claude Code + Codex + OpenCode：我的 AI 编程三件套工作流"
date = 2026-06-08
description = "一个开发者的 AI 编程工作流实录：Claude Code 做主力开发，Codex 做交叉 Review，OpenCode 跑定时任务和飞书派活。为什么一个工具搞不定所有事。"
tags = ["AI", "编程工具", "工作流", "Claude Code", "Codex", "OpenCode", "效率"]

[extra.comments]
issue_id = 8
+++

"我就用 Claude Code，一个工具管所有，为什么还需要别的？"

如果你现在是这个想法，我理解。一个月前我也这么想。Claude Code 强，日常开发一把梭没问题。

但用了一段时间之后，我发现有三个问题是一个工具解决不了的。

<!--more-->

---

## 三个绕不开的问题

**第一，费用。**

Claude Code 的非交互模式（`-p` 参数）在 6 月 15 日之后转入独立的 credit pool，按 API 费率计费——本质上就是额外收费。偶尔用还好，但跑定时任务、自动化杂活这类场景，成本不可控。

**第二，模型盲区。**

同一个模型 review 自己的代码，大概率说"没问题"，甚至主动帮你找理由。所有大模型都这样——同模型自审有盲区。

**第三，模型匹配。**

格式化文件、跑常规检查这类活，用 Opus 是杀鸡用牛刀。而且 Opus 响应不快，有些场景你要的是又快又便宜。

想清楚这三件事之后，我的工具组合就自然成型了。

---

## 我的三件套

<img src="/images/ai-workflow-overview.svg" alt="AI 编程三件套工作流：Claude Code 70% 主力开发 + OpenCode 30% 打杂" style="width:100%;max-width:900px;" />

三个工具，三个角色，各管一摊。

---

## 主力：Claude Code

用 Claude Code 做日常开发的人多，这里只聊我的个人偏好。

**重活用 Opus 4.8，日常用 Sonnet 4.6。**

很多人觉得 Claude Code 必须全程 Opus，不是。我日常大量任务用 Sonnet 4.6——批量改命名、补 boilerplate、写测试、简单 bug fix，又快又稳，够用。Claude Code 支持通过 `/model` 随时切换，我的默认档位就是 Sonnet。

Opus 4.8 留给需要深度思考的场景：复杂的架构设计、疑难 bug 调试、跨多文件的重构。

另外提一嘴 `CLAUDE.md`——它定义项目约定和持久化指令，每次会话自动加载。我在里面写了代码风格、提交规范，Opus 4.8 和 Sonnet 4.6 都能保持一致输出。后面会提到，OpenCode 也能读这个文件。

Claude Code 负责重活，但它产出的代码还需要一双独立的眼睛——这就是第二个工具的用处。

---

## 副手：Codex 交叉 Review

同模型 review 有盲区，解决方案简单：**换一个同级别的模型来审。**

关键是"同级别"。Opus 4.8 和 GPT 5.5 都是 SOTA，能力弱的模型审不出 Opus 的问题。

我用 [Codex in Claude](https://github.com/openai/codex-plugin-cc)，OpenAI 官方插件，在 Claude Code 里直接调 Codex 做 review。安装：

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

装好之后主要用两个命令：`/codex:review` 做常规审查，`/codex:adversarial-review` 做对抗性审查——后者会主动挑战你的设计决策，比普通 review 更尖锐。如果有比较复杂的任务想丢给 Codex 在后台跑，还可以用 `/codex:rescue`。

不同架构、不同训练数据的模型，关注点不一样。Claude 觉得没问题的地方，GPT 可能会质疑——和人类 code review 一个道理，写的人自己检查，不如换人来看。

---

## 打杂：OpenCode

开发和 review 都有了着落，剩下的问题是：谁来跑日常杂活——定时任务、小修小补、不值得启动 Claude Code 的活？这就是 OpenCode 的位置。

### 为什么选 OpenCode

[OpenCode](https://opencode.ai/) 是基于 Go 的开源 AI 编程工具，GitHub 170k+ star。TUI 和 CLI 两种模式，支持 75+ 个 LLM 提供商。

我选它来干杂活，有几个很实际的理由：

- **多模型对接**。我配合 [Oh my openAgent](https://omo.dev)（简称 OMO）作为插件使用。OMO 是 agent 编排框架，内置 11 个专业 agent，根据任务类型自动匹配模型，速度极快。
- **fallback 稳定性好**。主模型挂了能自动切备用的，跑无人值守的定时任务时这点很重要。
- **兼容性强**。OpenCode 原生支持 `AGENTS.md`，没有时自动 fallback 读 `CLAUDE.md`；`.claude/skills/` 下的文件也能识别。Claude Code 的项目配置直接复用，不需要额外适配。

用 OpenCode 配合便宜模型跑这些杂活，正好绕开了前面提到的 Claude Code 非交互模式收费问题。

我在 OpenCode 上接的主力模型是 GPT 5.5 和 DeepSeek V4 Pro。另外我有一个 GitHub Copilot 订阅，如果想尝试其他模型的时候会用它来切换。

### 定时任务：Multica 调度

定时任务的触发我交给了 [Multica](https://multica.ai/)。

Multica 是一个开源的 coding agent 管理平台（GitHub 35k+ star），把 agent 当团队成员管理。agent 有自己的 profile，能出现在任务看板上，能创建 issue、发评论、主动上报阻塞。

它目前支持 12 种 coding agent 工具，包括 Claude Code、Codex、OpenCode、Gemini 等等。我主要用它的 **Autopilots** 功能来设置定时触发：通过 cron 表达式或 webhook 来定期创建任务，然后自动路由给 OpenCode 执行。

任务有完整生命周期管理：排队、认领、执行、完成，每一步有状态追踪。它还有 **Squads** 功能，可以多 agent 编组、由 leader 分配任务。

相比自己写 cron job，Multica 两个好处：一是 agent 遇到问题主动上报，不会默默失败；二是**所有执行过程都有完整记录**，跑完了也能回溯。

### 飞书派活：Lark Coding Agent Bridge

这个是基于开源项目 [lark-coding-agent-bridge](https://github.com/zarazhangrui/lark-coding-agent-bridge) 二次开发的工具。上游项目已经支持了 Claude Code 和 Codex，我 fork 之后主要加了 OpenCode 适配，让它也能通过飞书来调用。

**项目地址**：[lark-coding-agent-bridge](https://github.com/fullstackjam/lark-coding-agent-bridge)

原理不复杂：

<img src="/images/lark-bridge-flow.svg" alt="Lark Coding Agent Bridge 消息链路：飞书 → WebSocket → Bridge 守护进程 → OpenCode" style="width:100%;max-width:780px;" />

飞书发消息给 Bot，飞书服务器通过 WebSocket 推送给本地 Bridge 守护进程，Bridge 转发给 OpenCode 执行。执行过程通过流式卡片实时更新回飞书。

fork 版本加的功能：每个飞书窗口对应独立 OpenCode 会话，上下文不串；OpenCode 需要权限确认时会在飞书里弹出交互卡片，可以直接点"允许"或"拒绝"；OMO 插件的后台任务跑完了还能主动往飞书推一张卡片通知你。

什么样的活会通过飞书来派？一般是那些不值得我打开终端、切到项目目录、启动 Claude Code 来做的小事：

- 快速查个问题："帮我看下这个接口的返回格式是什么"
- 简单的文件操作："把 config 里的 timeout 从 30 改成 60"
- 状态检查："跑下 lint，看看有没有新的 warning"
- 临时的小需求："给这个函数加个参数校验"

单独看都很小，累积起来打断心流。飞书下发指令，走路时、开会时顺手处理。

---

## 跑下来的感受

这套组合用了一个月，最大的感受是：**工具之间的分工比工具本身的强弱更重要。**

举个实际的例子。有一次 Claude 写了一段并发逻辑，我自己看了觉得没问题，Claude 自己 review 也说没问题。丢给 Codex 做 adversarial review，它指出了一个竞态条件——两个 goroutine 在没有锁保护的情况下写同一个 map。Claude 回头一看，确认是 bug。如果没有交叉 review 这一步，这个问题大概率会带到线上。

分工之后，效率更高、成本更低。Claude Code 写代码，Codex 挑毛病，OpenCode 处理杂活。

这是我的工作流，只做交互式开发的话一个工具够用。但一旦加上定时任务、交叉 review、移动端派活，单一工具很快到瓶颈，可以试试类似的组合。

多花点时间找适合自己的组合，比死磕一个工具划算。
