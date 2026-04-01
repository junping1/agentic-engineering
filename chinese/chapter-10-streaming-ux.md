# 第 10 章：流式传输与用户体验

用户不信任他们看不见的东西。

想象一下：你让 AI Agent 修复一个 Bug。光标在闪烁，但什么也没发生。5 秒，10 秒，30 秒过去了。它是在思考？卡住了？还是崩溃了？你的手指停在 `Ctrl+C` 上蠢蠢欲动。到了 45 秒，你终于失去耐心，强行终止了进程。

实际上，Agent 运行得很好。它已经找到了 Bug，写好了修复方案，甚至测试都跑了一半。但因为没有反馈，这一切努力都付诸东流。

这种情况经常发生。不是因为 Agent 坏了，而是因为它“看起来”坏了。一个只埋头干活却不提供反馈的 Agent，在用户眼里是不稳定的、令人不安的。用户无法判断它的状态，所以往往会做最坏的打算。

本章将讨论如何构建能够赢得用户信任的界面：包括流式架构、进度报告，以及帮助用户理解 Agent 行为并在必要时介入的交互模式。

## “等待”带来的挑战

传统软件在”人类的时间尺度”上运行。点击按钮，毫秒级响应，看到结果。

但 Agent 打破了这种模式。

当一个 Agent 检索代码库、阅读 15 个文件、执行构建命令、分析输出并提出修复方案时，耗时不是以毫秒计，而是以秒甚至分钟计。

在这些沉默的间隙里，用户会陷入极大的不确定性：*它还在运行吗？是不是死机了？还要多久？我该做点什么吗？能停下来吗？* 每一个未被回答的问题都在消磨信任。经历过几次糟糕的体验后，人们会彻底放弃使用——不是因为它没用，而是因为它的工作过程像个“黑盒”。

解决方案就是**流式传输 (Streaming)**：边做边展示，而不是等全部完成后再打包输出。

这就像下载大文件。你不会盯着空白屏幕直到最后 100% 完成，而是会看到进度条、已传输字节和预估剩余时间。这种反馈将挫败感转化为了掌控感。

**即使 Agent 的实际运行速度变慢了，流式反馈也能让它显得更快。** 关于感知性能的研究表明，用户认为有进度反馈的任务比没有反馈的同类任务耗时更短。当你看到 Agent 正在积极搜索文件、匹配代码、分析结果时，你参与到了这个过程中，时间过得更充实；而沉默则会让每一秒都显得漫长。

这并非欺骗，而是真实的呈现。Agent 在那 30 秒里确实在工作，流式传输只是让这些工作变得可见。

## AsyncGenerator（异步生成器）模式

流式架构依赖于一种能够增量产出结果，而非一次性返回的编程模式。

```javascript
async function* runAgent(task) {
    yield { type: 'status', message: '正在解析任务...' };
    
    const plan = await analyzeTask(task);
    yield { type: 'plan', steps: plan.steps };
    
    for (const step of plan.steps) {
        yield { type: 'step_start', step: step };
        
        // 嵌套流式输出工具执行进度
        for await (const progress of executeStep(step)) {
            yield { type: 'progress', ...progress };
        }
        
        const result = await step.getResult();
        yield { type: 'step_complete', step: step, result: result };
    }
    
    yield { type: 'complete', summary: generateSummary() };
}
```

其核心原则很简单：**边走边产出 (Yield as you go)**。每当 Agent 产生有意义的进展时——做出了一个决策、工具执行有了新进度、收到了一段中间结果——就立即 `yield`。消费端则增量处理这些事件，实时更新 UI。

这种模式具有明显的工程优势：

1.  **天然的背压控制 (Backpressure)**：如果消费端处理较慢（如复杂的 UI 渲染），生成器会自动减速，无需显式的流量控制。
2.  **可组合性 (Composability)**：Agent 循环可以嵌套其他生成器。即使是一个执行 Bash 命令的工具，也可以逐行产出输出，并透传到整个系统中。
3.  **可测试性**：你可以将产出的所有事件收集到一个数组中，对顺序进行断言。测试变成了声明式的。
4.  **取消机制**：退出生成器循环会自然停止迭代，为用户中断提供了一个优雅的钩子。
5.  **内存效率**：随产随弃，无需缓存所有中间结果。即使 Agent 处理成千上万个文件，也不必将它们全部保存在内存中。

在接收端，处理逻辑通常如下：

```javascript
async function renderAgentExecution(task) {
    const ui = createAgentUI();
    
    try {
        for await (const event of runAgent(task)) {
            switch (event.type) {
                case 'status':
                    ui.setStatus(event.message);
                    break;
                case 'progress':
                    ui.updateToolProgress(event.toolId, event.data);
                    break;
                case 'step_complete':
                    ui.renderResult(event.result);
                    break;
                case 'complete':
                    ui.showSummary(event.summary);
                    break;
            }
        }
    } catch (err) {
        if (err.name === 'AbortError') ui.showCancelled();
    }
}
```

消费端不需要关心 Agent 会运行多久，它只需要根据到来的事件进行响应。

## 进度报告：一等公民的消息类型

在开发 Agent 的早期，我常把进度反馈当作可有可无的调试日志。这是个错误。**进度 (Progress)** 应当是系统中核心的消息类型。

```typescript
type ProgressMessage = {
    type: 'progress',
    tool_id: string,        // 报送进度的工具 ID
    timestamp: number,      // 时间戳
    data: ToolProgressData  // 工具特有的详细信息
}
```

`data` 字段的结构会根据工具类型的不同而变化。通常，每种操作都有其最直观的进度指标：

| 工具类型      | 进度数据内容                                 |
| :------------ | :------------------------------------------- |
| **Bash 命令** | 输出行、运行耗时、执行状态（运行中/已完成）  |
| **搜索操作**  | 已检索文件数、已发现匹配项、当前处理的文件名 |
| **文件操作**  | 已处理字节数、文件总大小、完成百分比         |
| **网络请求**  | HTTP 状态码、响应体大小、请求延迟            |

核心原则是：**即时传输进度**。Bash 命令输出一行，就立即 `yield`；搜索每发现一个匹配，就立即上报。用户应当看到实时的“心跳”，而不是感受到明显的延迟。

持续的交互活动（如实时滚动的日志、不断增长的计数器）能够向用户传递明确的信号：Agent 正在按计划积极推进。相反，长时间的静默往往会导致用户产生焦虑。

## 并发工具执行

成熟的 Agent 不会僵化地顺序执行工具。当模型请求多个独立操作时（如并行读取多个文件或搜索多个目录），Agent 理应提供并发执行的能力。

这给 UI 带来了新的挑战：如何同时展示多个正在进行的工具进度？

```javascript
async function* executeToolsConcurrently(toolCalls) {
    // 启动所有工具执行
    const executions = toolCalls.map(call => startTool(call));
    
    // 合并并分发所有工具的流式进度
    for await (const progress of mergeStreams(executions)) {
        yield progress;
    }
    
    // 等待所有结果（可能乱序完成）
    const results = await Promise.all(executions.map(e => e.result));
    
    // 按并行的请求顺序产出结果，保证执行的确定性
    for (let i = 0; i < results.length; i++) {
        yield {
            type: 'tool_result',
            tool_id: toolCalls[i].id,
            result: results[i]
        };
    }
}
```

这里涉及三个关键的技术细节：首先，UI 层必须支持并发的进度指示器（如堆叠的进度条、活动的任务面板）；其次，工具应具备独立报告状态的能力；最后，为了方便日志回放和测试，最终结果的产出应遵循*请求顺序*而非实际完成顺序。

## 工具自有的渲染逻辑 (Tool-Owned Rendering)

一个核心设计原则是：**每个工具应控制自己的展示方式。**

Agent 框架本身不需要预知如何渲染 Bash 输出、搜索结果或代码差异（Diff）。相反，这些逻辑应当解耦到工具内部：

```javascript
const BashTool = {
    name: 'bash',
    
    // 渲染进度状态
    renderProgress(progress, options) {
        if (options.verbose) {
            return <OutputStream lines={progress.output_lines} />;
        }
        return <Spinner text={`正在运行: ${progress.command}`} />;
    },
    
    // 渲染最终结果
    renderResult(result, options) {
        if (result.exitCode !== 0) {
            return <ErrorPanel title="命令执行失败" output={result.stderr} />;
        }
        return options.verbose ? 
            <CodeBlock content={result.stdout} /> : 
            <SuccessBadge text="执行成功" />;
    }
}
```

工具本身最清楚哪些信息对用户最具参考价值。例如，文件读取工具知道原始二进制字节可能对用户无用，但行数和编码格式却非常有价值。通过这种方式，框架仅负责整体布局调度，而具体表现则交由工具定义。

## 中断与取消处理

用户必须具备随时终止 Agent 的能力。这可能是因为发现任务方向偏差、需要立刻回收终端控制权，或是用户进行了误操作。

处理中断需要考虑以下几个层面：

-   **工具的可取消性**：文件读取通常可以随时中止，但数据库提交或某些原子操作则不可轻易拆分。工具应明确声明其取消策略。
-   **优雅取消 vs. 强行终止**：建议将第一次 `Ctrl+C` 处理为“在安全点停止”（如处理完当前文件、运行完当前测试），而第二次 `Ctrl+C` 为强行杀掉进程。
-   **状态持久化**：即使任务被中途取消，已完成的部分工作（如已读取的文件）也不应被丢弃，而应保留在上下文中以供后续参考。

```python
class AgentExecution:
    def __init__(self):
        self.cancel_requested = False
        self.abort_requested = False
    
    def requestCancel(self):
        self.cancel_requested = True # 优雅取消标志
    
    def requestAbort(self):
        self.abort_requested = True  # 强行终止标志
        for tool in self.running_tools:
            tool.abort()
```

## 终端 UI 模式 (TUI Patterns)

在 CLI Agent 中，终端不仅是输出框，更是一个可构建富交互的画布。

### 状态指示器 (Spinners)
简单的状态旋转动画能提供极佳的持续反馈：
```
⠋ 正在检索代码库...
⠙ 正在检索代码库... (已处理 142 个文件)
⠹ 正在分析匹配项...
```

### 进度条
适用于有明确操作边界的场景：
```
读取配置文件  [████████░░░░░░░░] 52%  7/15
```

### 实时流式日志
对于构建、测试等产生大量输出的工具，应实时展示其日志流。

### 固定布局与页脚
当内容较多时，使用固定页脚（Fixed Footer）来保持关键状态始终可见：
```
┌─ 正在运行测试 ──────────────────────────────┐
│  PASS  src/auth.test.js                      │
│  FAIL  src/user.test.js                      │
│  ↓ 还有 12 行 (按回车键展开)                  │
└──────────────────────────────────────────────┘
[Ctrl+C 取消]  总耗时: 23s  进度: 47/120
```

## 提升透明度，建立核心信任

所有的交互设计最终都指向一个深层目标：**建立透明度。**

Agent 被赋予了阅读、修改、执行等强大权限，用户必须确信它的每一步都在预期的轨道上。对比以下两种反馈：

**不透明模式：**
> 用户：修复鉴权 Bug
>
> Agent：[工作中...]
>
> Agent：任务完成！我已修复了鉴权 Bug。

**透明模式：**
> 用户：修复鉴权 Bug
>
> ● 正在读取 src/auth.ts (4.2 KB)
> ● 正在全局搜索 "authenticate" (发现 12 处匹配)
> ● 正在执行测试: npm test auth
>   ✗ session 过期刷新失败: 预期 200, 实际返回 401
> ● 问题定位: session.refresh() 未进行有效性预校
> ● 正在修改 src/middleware/session.ts
>   + 在刷新逻辑前增加了校验步骤
> ● 重新运行测试... 全部通过
>
> Agent：修复完成！在 session.refresh() 中增加了预校验逻辑，目前所有相关测试均已通过。

透明模式不仅展示了最终结果，还呈现了逻辑推导和验证路径。如果 Agent 的思路出现偏差，用户可以在大规模修改发生前及时矫正。

**透明度意味着：** 展示具体的动作、解释执行动机、突出显示错误，并提供可选的详细视图（Verbose 模式）。

## 交互设计的避坑指南

- **无反馈的静默**：后台忙碌但前端死锁。这是最糟糕的体验。
- **信息淹没**：毫无筛选地倾倒所有日志，导致关键状态不可见。应默认保持摘要视图。
- **由于滚动被掩埋的错误**：错误信息消失在屏幕上方。应通过视觉高亮或固定面板让错误立现。
- **缺失干预机制**：当 Agent 执行错误操作时，用户无法点击“红按钮”制止。
- **丢失上下文导航**：过长的输出导致用户忘记了最初的任务输入。

---

通过流式架构、结构化的进度上报以及透明的交互设计，AI Agent 能够从一个令人焦虑的“黑盒”，真正转变为一个可协作、可信赖的开发伙伴。

还记得开头那个 45 秒后按下 `Ctrl+C` 的场景吗？流式传输确保这种情况永远不会发生——用户始终知道 Agent 在做什么，因此有理由让它继续完成工作。

---
*核心模式到此结束。在最后一章中，我们将退后一步，将所有内容整合为智能体系统设计的完整图景。*
