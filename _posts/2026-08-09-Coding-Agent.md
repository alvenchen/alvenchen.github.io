---
layout: post
title: "Coding-Agent"
date: 2026-08-08
categories: tech
---

# Coding-Agent

## 整体结构

主流的Coding Agent(`codex, opencode, reasonix, zed`等)都是
- 基于ReAct范式的变体
- 以Agent loop为框架
- 以工具调用为核心的

基本流程都可以简化为:

```mermaid
flowchart LR
    A[用户输入] --> B[Turn]
    B --> C[输出]
```

而一次Turn是包含了核心为 `LLM请求、工具调用` 的loop，一次Turn也可以理解为一个`AgentTask`。

下面是一个简化的完整的Turn：

```mermaid
flowchart TD
    Start([用户输入]) --> AgentRun[Agent.Run]

    AgentRun --> Loop{runToolLoop<br/>step &lt; maxSteps}
    Loop --> Stream[采样一轮]
    Stream --> RetryOK{干净终态?}
    RetryOK -->|否 重试/失败| End([结束])
    RetryOK -->|是| Commit[session.Add<br/>assistant msg]
    Commit --> HasCalls{有 tool call?}

    HasCalls -->|否| Done[返回最终回答] --> End
    HasCalls -->|是| ExecTools[executeOne × N<br/>parse → policy → exec]
    ExecTools --> AddResult[session.Add<br/>tool 结果] --> Loop
```
- 其中`采样一轮`是为了拼接上下文、恢复异常中断等。

真实的流程会稍微复杂点。比如会在每次loop时，判断
- 要不要压缩
- 为llm添加MCP、skill的描述
- 防注入安全校验
- 丰富的内置工具、沙箱机制等。

## agent差异

不同的agent有的是为自家模型量身定制的武器，有的则是为了探索新模式。

### reasonix

前面说了主流的agent都是ReAct的变体，reasonix的变在于添加了Plan-and-Execute、添加了工程性的鲁棒机制等。

- 在处理input前，进行路由决策，判断哪些turn需要plan; 不像其他agent，plan是用户手动触发的。

- Loop guards，在工程上对一些异常情况做处理，防止流程风暴。

- 新增了PrefixShape的逻辑，每一次loop前先判断本次会话的shape或者说指纹(`prompt + tools + memory`)，让用户能观察到一次请求是否真的命中缓存，揭示了服务端的cache逻辑。

- YOLO功能，直接自动允许所有操作，不用再每次授权：不用考虑安全、绝对效率优先。

### codex

通过分析codex，放个与之前简化Turn流程对比稍微详细点的时序图：

```mermaid
sequenceDiagram
    participant User as 用户/TUI
    participant Turn as run_turn
    participant Sampling as run_sampling_request
    participant Model as LLM 模型
    participant Tools as ToolCallRuntime

    User->>Turn: submit(user_input)

    rect rgb(240, 248, 255)
        Note over Turn: === 前置阶段 ===
        Turn->>Turn: 1. run_pre_sampling_compact() 若 token 超限则压缩
        Turn->>Turn: 2. 解析 MCP 服务器依赖
        Turn->>Turn: 3. build_skills_and_plugins() 注入提示词
    end

    rect rgb(255, 250, 240)
        Note over Turn,Model: === Agent 主循环 ===
        loop needs_follow_up == true
            Turn->>Turn: 4. 拉取 pending_input（用户追加消息）
            Turn->>Turn: 5. clone_history().for_prompt() 组装 prompt

            Turn->>Sampling: run_sampling_request(prompt)

            rect rgb(245, 255, 245)
                Note over Sampling,Model: === 采样请求 ===
                Sampling->>Sampling: 6. build_prompt() 附加 instructions + tools
                Sampling->>Model: 7. client_session.stream(prompt)
                Model-->>Sampling: 流式 SSE 事件

                loop 每个 ResponseEvent
                    alt OutputItemAdded
                        Sampling-->>User: turn_item_started（AgentMessage/工具调用）
                    else ContentDelta
                        Sampling-->>User: 流式推送文本增量
                    else OutputItemDone (是 ToolCall)
                        Sampling->>Sampling: record_conversation_items() 持久化
                        Sampling->>Tools: tool_runtime.handle_tool_call() 异步入队
                        Note over Sampling: needs_follow_up = true
                    else OutputItemDone (是 Message)
                        Sampling->>Sampling: finalize_non_tool_response_item()
                        Sampling-->>User: turn_item_completed
                    end
                end

                Sampling->>Sampling: 8. drain_in_flight() 等待工具执行完成
                Sampling->>Sampling: 9. 结果写入对话历史
            end

            Sampling-->>Turn: SamplingRequestResult { needs_follow_up, last_agent_message }

            rect rgb(255, 240, 240)
                Note over Turn: === 后采样决策 ===
                alt needs_follow_up && token_limit_reached
                    Turn->>Turn: 10a. run_auto_compact() 压缩上下文 → continue
                else needs_follow_up
                    Note over Turn: 10b. 工具结果已写入历史 → continue<br/>模型在下一轮看到工具输出继续推理
                else needs_follow_up == false
                    Turn->>Turn: 10c. run_turn_stop_hooks()
                    opt stop hook 要求继续
                        Turn->>Turn: 注入 hook prompt → continue
                    end
                    Note over Turn: 否则 → break（turn 结束）
                end
            end
        end
    end

    Turn-->>User: TurnComplete（last_agent_message）
```


### Zed

Zed的创新在于用ACP把Agent这一层抽象出来，可以让用户快速切换agent，它自己则专注于快速处理和用户的交互。
实际测下来会有些小bug，比如某些工具在处理任务时一直会失败，实际上切换agent作用也不大，因为前面说了，自家模型配自家agent，好马配好鞍才是最优解。

### tools 对比

如果大脑（模型）一样、方法论（agent loop）一样，指导方针（prompt）一样，那还有什么能拉开agent间的差距呢？答案是tools，tools是agent的武器，更是利器。

而差距的提现不只是在于某种工具有没有，而是在于如何使用，比如工具编排、验证和自动重试机制、甚至是失败后重新规划整个流程。

reasonix和codex的工具体系完全不一样，在解决实际coding任务时，效果很接近。

reasonix定义了一个个具体职责的工具：读写编辑等文件操作工具、ls blob grep web_fetch等检索工具、bash confine等shell工具、沙箱工具等。

而codex则是更抽象，没有按职责分那么细，只是定义出shell工具、cmd工具、mcp工具等，这和自家模型擅长处理的事件类型也有关系。例如，如果需要ls工具，模型会输出
```shell
{"name": "exec_command","cmd": "ls"}
```

## 总结
