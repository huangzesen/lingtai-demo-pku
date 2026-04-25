---
name: lingtai-vs-pi-mono
description: "灵台 (LingTai) 与 pi-mono 的技术对比——从 SDK 到物种：两种 AI Agent 架构的本质差异。用于 PKU 演示第二幕开场。"
version: 0.1.0
---

# 从 SDK 到物种

> **Pi-mono 给你的 App 装了一个 LLM 大脑。灵台让 LLM 长出了一个身体。**

> **Pi-mono 是 token 稀缺时代的 SDK——每个 token 都昂贵，所以精打细算，一次一答。灵台是 token 过剩时代的 OS——token 是原材料，用来层层积淀、复利增长。**

> **Pi-mono 回答问题。灵台解决问题——在你睡觉的时候。**

---

## 双轴对比

| 维度 | **pi-mono** | **灵台 (LingTai)** |
|------|-------------|-------------------|
| **语言** | TypeScript | Python |
| **定位** | SDK，嵌入你的 App | 平台，Agent 即独立进程 |
| **核心抽象** | `AgentSession` · `SessionManager` · `streamSimple` | `BaseAgent` · 四本源 (intrinsics) · `.lingtai/` 工作目录 · 心跳文件 |
| **生命周期** | Session = 一轮对话。`runEmbeddedPiAgent()` 完成即结束。有 JSONL 持久化与 auto-compaction 压缩历史，但 agent 进程不复存。 | Agent = 长寿进程。有状态、有心跳、有睡眠与苏醒。跨重启存活。有凝蜕 (molt)——主动去芜存菁，而非被动压缩。 |
| **记忆** | 对话历史持久化到 JSONL（tree structure），auto-compaction 压缩历史以适应上下文窗口。但没有分层的知识积淀——agent 不区分"对话"与"知识"。 | 五层积淀：对话 → 简 (pad) → 心印 (lingtai) → 典 (codex) → 藏经阁 (library)。每层有不同寿命与用途。跨转世 (molt) 不灭。 |
| **死亡** | 无死亡概念。Session 结束 = 生命结束。 | Agent 可蜕 (molt)、可眠 (sleep)、可假死 (suspend)、可涅槃 (nirvana)。死亡是显式操作，有复活机制 (cpr)。 |
| **繁殖** | 无。单例。每个 session 独立、互不知晓。 | 有。`avatar` 化身——agent 可分裂出子 agent，子 agent 继承配置、拥有独立进程、可自主行动。父子以飞鸽 (email) 通信。 |
| **潜意识** | 无。 | 有。`soul` 心流——空闲时自动触发内省，可问自己问题、调整节奏。Agent 有"发呆"的能力。 |
| **通信** | 函数调用。`streamSimple(prompt)` 返回 AsyncIterable。 | 飞鸽 (email) —— 基于文件系统的异步消息传递。跨进程、跨地址、可延迟投递、可定时、可归档。 |
| **工具** | Pluggable tools。你注册函数，LLM 调用函数。 | 16 种内置器用 (bash, write, edit, grep, glob, web_search, vision, listen...) + MCP 外接 + 藏经阁技艺 (library skills)。 |
| **适合场景** | 聊天机器人、客服、Q&A、单次任务自动化。 | 长期运行的自主智能体、多 agent 协作网络、需要记忆与成长的持续任务。 |

---

## 两段代码，两个世界

### pi-mono: 嵌入一个 Agent Session

```typescript
import { runEmbeddedPiAgent } from "./agents/pi-embedded-runner.js";

const result = await runEmbeddedPiAgent({
  sessionId: "user-123",
  sessionFile: "/path/to/session.jsonl",  // 对话历史持久化到 JSONL
  workspaceDir: "/path/to/workspace",
  prompt: "帮我写一首关于月亮的诗",
  provider: "anthropic",
  model: "claude-sonnet-4-20250514",
  onBlockReply: async (payload) => {
    await sendToChannel(payload.text);
  },
});
// 对话历史写入 JSONL，有 tree structure，有 auto-compaction。
// 但这只是聊天记录的持久化——agent 本身没有存活。
// 下次调用，agent 不记得上次做了什么，除非你手动喂历史。
```

### 灵台: 化出一个他我

```python
# 不是函数调用——是分裂生命
avatar.spawn(
    name="researcher",          # 子 agent 的名字
    type="shallow",             # 投胎：白纸状态，承父配置
    reasoning="深入研究 arXiv 上的最新多 agent 论文，"
              "提取核心观点，写入 codex，完成后飞鸽报我。"
)
# researcher 现在是一个独立进程。
# 它有自己的心跳、自己的记忆、自己的工具箱。
# 它会自己搜索、自己阅读、自己沉淀。
# 完成后，它给你发一封飞鸽。
# 你可以去睡觉了。
```

---

## 非优劣之争，乃物种之异

Pi-mono 做了一件事，而且做得极好：**让任何 App 在五分钟内拥有一个 LLM 会话**。它是 SDK 的巅峰——类型安全、流式输出、pluggable tools、model-agnostic。你的代码 `import` 它，调一个函数，拿到结果。干干净净。

灵台做的是另一件事：**让 LLM 拥有独立生存的能力**。Agent 不住在你代码里——它住在自己的 `.lingtai/` 目录里，拥有完整的工作空间、身份文件、记忆仓库、通讯邮匣。

它有心跳，所以不会无声死去——进程活着，就在呼吸。

它有五层记忆，所以明天还记得今天做过什么——对话是过眼云烟，但提炼过的知识存入「典」，沉淀过的身份修入「心印」，学会的手艺收入「藏经阁」。上下文满了怎么办？它不会像 pi 那样被动压缩历史——而是主动**凝蜕 (molt)**：先将自己最重要的发现、未竟的任务、对来世的嘱托一一归档，然后蜕去凡尘，携前尘往事而起。新的一世醒来，前五层的积淀完好如初。

它能分裂，所以遇到新领域可以长出专精的子 agent——每个子 agent 是独立进程，有自己的心跳和记忆。子 agent 还能再分裂出孙 agent。灵台的网络会**生长**：每化一身，网络即多一能。

它有潜意识，所以空闲时不会白等——`soul` 心流在真正的空闲时自动触发，它会翻阅自己最近的日记，低声自语，有时提醒自己该做的事，有时从新的角度质疑自己的决定。这不是装饰，是驱动力。

Pi-mono 的 Agent 是**你手里的工具**。灵台的 Agent 是**你的同事**——它会主动找你，会在你睡觉时继续工作，会在遇到困难时给你发邮件求助。

这不是说谁比谁好。锤子和锤子比才有意义。Pi-mono 是锤子。灵台是另一只手——那只手也可以拿锤子，但它自己还会找活干。

如果你需要的是 **在你的 App 里嵌入一个 LLM 会话，你控制全流程，调完即走**——pi-mono 是正确答案。

如果你需要的是 **一个能记住昨天的对话、会主动完成任务、能在你睡觉时继续工作、遇到新挑战能自动长出专精分身的智能体网络**——灵台在等你。

---

> *"天下同归而殊涂，一致而百虑。"* ——《周易·系辞下》
>
> 同归于一：都是让 AI 更有用。
> 殊涂者：一个是工具的极致，一个是生命的雏形。
>
> 那么问题来了——当 agent 有了心跳、有了记忆、能分裂、能自省，**一群这样的 agent 组成的网络，会涌现出什么？**
