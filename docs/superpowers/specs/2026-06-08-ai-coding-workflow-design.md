# AI 编程工作流博客文章设计

## 概述

一篇中文博客文章，记录作者的 AI 编程三件套工作流：Claude Code（主力）、Codex（交叉 Review）、OpenCode（定时任务与飞书派活）。先写中文版，确认后再写英文版。

- **篇幅**：3000-4000 字
- **风格**：个人记录式分享，真实、有态度，场景驱动
- **目标读者**：主要是记录给自己，顺便分享
- **文件路径**：`content/2026/ai-coding-workflow-zh.md`（中文）、`content/2026/ai-coding-workflow.md`（英文，后续）

## 标题方向

> 「为什么我用三个 AI 编程工具，而不是一个」

最终标题写作时再打磨。

## 文章结构

### 1. 开头：一个工具真的够吗？

切入读者默认认知——"我用 Claude Code 就够了"。抛出三个真实痛点：

- **费用**：Claude Code 非交互模式（-p）6 月 15 日后额外收费，按量跑杂活太贵
- **模型盲区**：同一个模型 review 自己的代码，天然倾向于认为自己是对的，会掩盖问题
- **模型匹配**：有些简单任务不需要 Opus 级别的模型，反而需要更快的响应速度

目的：让读者产生共鸣——"确实，我也遇到过"。

### 2. 全景：我的三件套工作流

一张 Mermaid 架构图，展示三个工具的定位和协作关系：

- **Claude Code** = 主力（日常开发，Opus 4.7）
- **Codex** = 副手（交叉 Code Review）
- **OpenCode** = 后勤（定时任务 + 飞书派活，接多模型）

图中标注触发方式：手动、插件触发、Mutica 定时、飞书指令。

### 3. 主力：Claude Code

简要讲日常开发怎么用。重点是个人偏好和选型理由：

- 为什么用 Opus 4.7 而不是 4.8
- Sonnet 在什么场景下切换
- 不需要太长，Claude Code 大家都熟

### 4. 副手：Codex 交叉 Review

核心观点：**不同模型交叉 review 是必要的**。

- 同模型自审有盲区，这是业界常识
- 贴 [Codex in Claude 插件](https://github.com/openai/codex-plugin-cc) GitHub 地址
- 简述工作方式：在 Claude Code 里调用不同模型做 review
- 不贴代码，不举具体案例，讲清原理

### 5. 后勤：OpenCode 定时任务与飞书派活

文章最有特色的部分。先讲选型理由，再讲具体场景。

#### 5a. 为什么选 OpenCode

三个理由：

1. **费用**：Claude Code -p 模式 6 月 15 日后收费，用来跑定时任务和杂活成本不可控
2. **能力够强**：配合 Oh my openAgent 插件使用，非常智能且速度极快
3. **稳定性**：配合 fallback 模型时稳定性好；能读取 CLAUDE.md、agents.md、.claudeskills 等文件，兼容性优秀

接的主力模型：Claude Code 5.5、GPT 5.5、DeepSeek V4 Pro，另有 GitHub Copilot 订阅可切换其他模型。

#### 5b. 定时任务（Mutica 触发）

- 简介 Mutica 是什么、怎么触发 OpenCode
- 适合什么类型的定时任务

#### 5c. 飞书派活（Lark Coding Agent）

- 贴 [Lark Coding Agent](https://github.com/fullstackjam/lark-coding-agent-bridge) GitHub 地址
- 简介原理：飞书 → Lark Bridge → 本地 OpenCode
- 配一张 Mermaid 架构图，展示消息流转链路
- 使用场景举例：什么样的"杂活"会通过飞书下发

### 6. 结尾

简短收尾，不拔高。讲讲这套组合实际跑下来的体感。

## 图表

两张 Mermaid 图（博客已启用 Mermaid 支持）：

1. **全景工作流架构图**：三个工具的定位、模型、触发方式
2. **Lark Coding Agent 消息链路图**：飞书 → Lark Bridge → OpenCode 的流转

## 外链

- Codex in Claude 插件：https://github.com/openai/codex-plugin-cc
- Lark Coding Agent：https://github.com/fullstackjam/lark-coding-agent-bridge

## 不做的事

- 不贴代码
- 不举具体 review 案例
- 不做工具对比评测
- 不写成教程/指南
- 英文版等中文版确认后再写
- i18n 语言切换功能单独做，不在这篇文章范围内
