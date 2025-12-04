---
title: Realtime 交接机制详解
description: 从底层事件到架构实践全面解析 Realtime 会话中的 handoff。
---

## TL;DR

- Realtime handoff 是通过 function tool 完成的“人格/职责切换”。模型调用 `transfer_to_xxx` 后，`RealtimeSession` 会把新的 `RealtimeAgent` 安装为当前 agent，并更新会话配置。
- `handoff.onInvokeHandoff` 会被 `await`，因此你可以在其中执行异步准备（如检索、编排、写入 context），但返回前模型会一直等待；请保持高效以免打断语音流畅度。
- 所有 `RealtimeAgent` 共享同一个 `RealtimeSession`、`RunContext` 与会话历史；需要手动使用 `session.updateHistory()` 等方式来裁剪或压缩长对话。
- 交接不会自动“返回”原 agent。要回退或在多 agent 间轮换，必须把相应的 handoff 链接配置好，或在应用层根据事件决定下一步。

## Realtime 的 handoff 是什么？和 “subagent” 有什么关系？

`RealtimeAgent` 是 `Agent` 的语音特化版本，支持 `voice`、实时工具和音频流。给 `RealtimeAgent` 的 `handoffs` 选项传入其他 `RealtimeAgent` 或 `handoff()` 包装结构后，SDK 会在发送给 Realtime 模型的工具清单里生成一个 `transfer_to_<agent_name>` 的函数工具。

模型调用该工具时，`RealtimeSession` 会：

1. 调用 handoff 对象的 `onInvokeHandoff`（默认只是返回目标 agent，也可由你重写）。
2. 将返回的 `RealtimeAgent` 设为新的 `currentAgent`。
3. 重新计算并推送会话配置（instructions、prompt、voice、工具清单等）。
4. 把 `{"assistant": "<agent_name>"}` 作为工具输出回写给模型，以便它知道交接已完成。

从语义上说，它和“子智能体”很像——你确实把部分职责转交给另外一个 agent——但在实现上更像是“当前会话 persona 的热切换”：底层只有一个 Realtime 会话，模型换成了另一段 instructions 来继续同一场对话。

## 它是同步还是异步？如何与 RAG 配合？

`onInvokeHandoff(context, args)` 的返回值会被 `await`。因此：

- 你可以在其中执行异步操作，例如查询检索向量、生成摘要或动态构造新 agent 的 instructions。
- 但 Realtime 模型会等到你调用 `sendFunctionCallOutput()`（SDK 自动完成）后才继续发声，因此时延会直接体现在用户体验中。建议将重操作移到后台工具或提前缓存。

针对 RAG：

- 不要把未经处理的原始文档直接塞进 Realtime 的上下文窗口。将检索结果在本地压缩成摘要或关键信息列表，再写回 `RunContext.context` 或通过 `session.updateHistory()` 注入。
- 如果需要一个“中间小 agent”来专门做检索整合，可以把它作为 handoff 目标：router agent 负责识别意图，检索 agent 在 `onInvokeHandoff` 中执行 RAG 并把整理后的结论写入共享 context，再将控权交还。
- 对实时语音尤为重要：保持回答字数可控、避免长篇无结构的上下文，才能稳定命中模型的记忆。

## handoff 之后会发生什么？还能返回吗？历史记录如何处理？

交接触发后的流程如下：

1. 当前 agent 的工具返回一个 `TransportToolCallEvent`。
2. `RealtimeSession.#handleHandoff()` `await` 目标 agent，并更新会话配置。
3. `agent_handoff` 事件会同时在 session 和旧 agent 上触发，便于你更新 UI 或日志。
4. 会话继续进行，此时模型使用新 agent 的 instructions 和工具。

不会自动回退到原 agent。如果你需要“回来”，可以：

- 在新 agent 的 `handoffs` 中显式包含原 agent 或一个调度 agent。
- 在应用层监听 `agent_handoff` / `agent_end` 事件，根据业务逻辑再次触发 `session.updateAgent(...)` 或等待模型调用另一个 handoff。
- 也可以直接调用 `await session.updateAgent(targetAgent)` 主动切换，无需等模型发起工具调用。

历史记录：

- `RealtimeSession` 默认把全量 `RealtimeItem` 留在 `history` 中，新 agent 在 `RunContext.context.history` 里可以直接看到。
- 目前 Realtime 模式不会自动应用 `handoff.inputFilter`。如需裁剪，请在 `agent_handoff` 事件或 `onInvokeHandoff` 中调用 `session.updateHistory()` 手动删减/替换（例如仅保留最近若干轮，或压缩成摘要消息）。
- 也可以在构造 session 时把 `historyStoreAudio` 设为 `true/false`，决定是否在本地历史中保留原始音频，避免内存暴涨。

```ts title="使用 updateHistory 主动裁剪历史"
import { RealtimeSession, RealtimeAgent } from '@openai/agents/realtime';

const agent = new RealtimeAgent({
  name: 'Assistant',
});

const session = new RealtimeSession(agent, {
  model: 'gpt-realtime',
});

await session.connect({ apiKey: '<client-api-key>' });

// listening to the history_updated event
session.on('history_updated', (history) => {
  // returns the full history of the session
  console.log(history);
});

// Option 1: explicit setting
session.updateHistory([
  /* specific history */
]);

// Option 2: override based on current state like removing all agent messages
session.updateHistory((currentHistory) => {
  return currentHistory.filter(
    (item) => !(item.type === 'message' && item.role === 'assistant'),
  );
});
```

> **注意：** Realtime 会话与文本 Runner 不同，`handoff.inputFilter` 暂时不会自动生效。需要自行在本地历史上做裁剪，然后交给模型。

## 我究竟在 handoff 什么对象？生命周期怎么安排？

- handoff 返回的必须是 `RealtimeAgent` 实例（或继承自它的自定义类）。SDK 会直接把它当成新的 `currentAgent`。
- 一般会在应用初始化时先创建好所有参与者，并互相放入 `handoffs`。也可以在 `onInvokeHandoff` 内即时 new 一个 agent——比如为了按需拼装 instructions 或 voice——只要确保返回的是 `RealtimeAgent`。
- `RealtimeSession` 自身只维护当前 agent、共享 `RunContext` 和历史。session 关闭 (`session.close()`) 后连接释放，agents 只是普通对象，如无引用会被 GC。
- 如果你的 agent 携带外部资源（例如 MCP server、数据库连接），请在 session 结束或确定不用时手动清理。

## 一个 session 可以有多少个 agent？它们如何结束？

一个 `RealtimeSession` 可以暴露任意数量的 handoff 目标：

- 初始 agent 通过 `handoffs` 列出可切换的角色，新的 agent 又可以继续列出它自己的 handoffs，从而形成链路。
- session 生命周期与底层 Realtime 连接一致。调用 `session.close()` 或终止 WebRTC/WebSocket 时，所有 agent 的对话都会自然结束。
- agent 实例本身可以复用到新的 session；如果希望 session 结束后自动清理上下文，可以在监听 `transport_event` 或 `history_updated` 时重置你自己的本地状态。

## handoff 之后，是切换 agent 还是共存？能做多智能体编排吗？

交接生效后只有一个“当前” agent：

- `RealtimeSession` 只会把当前 agent 的 instructions、工具和 voice 推送给模型。其他 agent 处于“待命”状态，直到模型再次调用它们的 handoff。
- 新 agent 的工具表会覆盖旧 agent 的工具表（包括函数工具、MCP 工具集合和继续可用的 handoff 工具）。
- 当前 agent 完成一轮回答后，如果模型没有再次调用其它 handoff，它会继续作为对话一方；只有模型或你主动切换时才会换人发声。
- 语音相关注意事项：一旦某个 agent 开始发声，再切换到配置了不同 `voice` 的 agent 会被 OpenAI Realtime 拒绝，因此多 agent 场景要么共用 voice，要么在第一段语音播出前完成交接。

要实现“多 agent 协同/意图识别”，可以让一个 router agent 作为入口，其他角色 agent 放在它的 `handoffs` 列表里。例如：

```ts title="为 Realtime 会话声明多个手交 agent"
import { RealtimeAgent } from '@openai/agents/realtime';

const mathTutorAgent = new RealtimeAgent({
  name: 'Math Tutor',
  handoffDescription: 'Specialist agent for math questions',
  instructions:
    'You provide help with math problems. Explain your reasoning at each step and include examples',
});

const agent = new RealtimeAgent({
  name: 'Greeter',
  instructions: 'Greet the user with cheer and answer questions.',
  handoffs: [mathTutorAgent],
});
```

在该结构中：

1. Router 的 instructions 负责听取用户需求，并在需要时调用 `transfer_to_<role>`。
2. 被交接到的 agent 可以完成任务后再把 router 放在自己的 `handoffs` 中，从而交还控制权。
3. 每次交接都会触发 `agent_handoff` 事件，你可以把 UI 上的“当前角色”切换过去，让用户知道是谁在说话。

## Agents 之间共享哪些状态？哪些是隔离的？如何协作？

共享的：

- **会话历史**：`session.history` 与 `RunContext.context.history`。所有 agent 看到的是同一份 RealtimeItem 列表。
- **RunContext**：`RunContext<RealtimeContextData<T>>` 会传给工具、handoff 回调与输出 guardrail。可在自定义字段上存放业务状态（如检索缓存、用户档案）。
- **使用量计数与工具审批**：`RunContext.usage`、`approveTool`/`rejectTool` 状态在整个 session 范围内共用。
- **输出护栏**：在构造 `RealtimeSession` 时传入的 guardrails 对所有 agent 生效。

独立的：

- **instructions/prompt**：每个 agent 持有自己的角色设定，可在 `onInvokeHandoff` 中动态生成。
- **工具与 MCP 服务器**：工具清单在交接时会重算；只有当前 agent 的工具会暴露给模型。
- **handoffs 列表**：每个 agent 可以决定下一步能交给谁。
- **voice**：设定在 agent 级别，但模型不允许在已经开口后切换声线。

协作建议：

- 在 `RunContext.context` 中存放共享的结构化状态（例如 `context.shared = { userProfile, cachedSummary }`），所有 agent 和工具都能访问。
- 借助 `session.on('agent_handoff', ...)`、`session.on('history_updated', ...)` 维护你自己的可视化或日志。
- 如需差异化对话记忆，可在交接前通过 `session.updateHistory()` 注入摘要或删除不相关内容。

## 有没有简洁的 demo？适用场景是什么？

最小化示例流程：

1. 定义各个 `RealtimeAgent`（router、领域专家、检索器等），配置 `instructions`、`handoffs` 以及可能的工具。
2. `new RealtimeSession(initialAgent, options)`，传入 `model`、`historyStoreAudio`、`outputGuardrails` 等配置。
3. `await session.connect({ apiKey })` 建立 WebRTC 或 WebSocket 连接。
4. 监听 `agent_handoff`、`history_updated`、`audio_start` 等事件来同步 UI。
5. 使用 `session.updateHistory()` 和 `RunContext.context` 维护你的业务状态。

常见场景：

- 语音客服或 IVR：入口 agent 负责身份识别与问题分类，domain agent 处理具体事务，必要时 handoff 到人工坐席工具。
- 实时 RAG：检索 agent 负责查找资料并整理成摘要，再 handoff 给回答 agent 播报。
- 多语言体验：一个 agent 做语言检测，按需 handoff 到不同语种/语气的 agent（注意声线限制）。
- 守护/监控流程：当输出护栏触发时，切换到安全 agent 提示用户或请求确认。

结合上面的示例与 `updateHistory` 片段，就可以搭建一个可维护的 Realtime 多智能体架构。

---

若需要进一步了解 Realtime 声音管道，可参阅《语音智能体》系列文档；要掌握 handoff API 的更多细节，可同时阅读英文站点中的《Handoffs》指南。
