# 第一章：智能体循环 (The Agent Loop)

使用 LLM 的默认思维模式是请求-响应：输入提示，输出补全，任务完成。

构建智能体需要一种完全不同的模式。

智能体不仅仅是一个更聪明的聊天机器人。它是一种完全不同的架构——在这种架构中，模型不只是**回答**，而是会**采取行动**、观察结果并决定下一步该做什么。从「调用 LLM」到「构建智能体」的跨越，就像是从调用一个函数进化到编写一个游戏循环 (Game Loop)。

本章将带你深入了解这个循环。一旦你理解了它，「自主 AI」就会变成一个具体的结构：一个内嵌了语言模型的 `while` 循环。

---

## 请求-响应的局限

你为 LLM 封装了一个漂亮的 API。用户输入：「找出所有导入了 `requests` 的 Python 文件，检查是否有硬编码的 URL，并修复它们。」你把这段话发给模型，模型回复了……一个计划。也许还有一些伪代码，甚至可能是正确的代码片段。

但实际上什么都没有发生。没有文件被读取，也没有任何东西被修复。

于是你添加了工具——文件读取、编辑、Shell 命令。你把这些工具交给模型。模型返回了一个工具调用 (Tool Call)。太棒了！你执行了它，然后呢？模型已经返回了，函数执行结束了。用户的任务还远未完成。

这就是你撞上的那堵墙。「请求-响应」(Request-Response) 模式无法处理需要多个步骤的任务，尤其是那些模型无法预先判断步骤的复杂任务。

解决方案不是更好的提示词 (Prompting)，而是不同的架构。

## 核心洞察

**智能体是一个 `while` 循环，而不是一次函数调用。**

请求-响应模式：
```
用户："做 X"
助手："这是 X"
完成。
```

智能体循环：
```
用户："做 X"
助手：[调用 list_files]
系统：[返回文件列表]
助手：[调用 read_file]
系统：[返回文件内容]  
助手：[调用 edit_file]
系统：[确认编辑成功]
助手："完成了——我修复了 file2.py 中的一个硬编码 URL。"
```

对话是**持续累积**的。每一次迭代都会增加助手的响应和工具的执行结果。不断增长的对话记录 (Transcript) 变成了下一次迭代的上下文 (Context)，让模型记住它做过什么，并思考接下来该做什么。

这就是连续模式 (Continuation Pattern)：循环执行直到满足终止条件，在迭代之间传递状态，让模型决定何时结束。

## 基本骨架

这是智能体模式的完整逻辑：

```javascript
function run_agent(prompt, tools) {
    let messages = [{ role: "user", content: prompt }];
    
    while (true) {
        let response = call_llm(messages, tools);
        
        if (response.has_tool_calls) {
            messages.push(response);
            let tool_results = execute_tools(response.tool_calls);
            messages.push(...tool_results);
        } else {
            return response.text; // 任务完成
        }
    }
}
```

就是这样。带着完整的历史记录调用模型。如果它请求使用工具，就执行工具，追加结果，然后继续循环。如果它直接回复而不再调用工具，则表示任务结束。

模型自己决定何时继续，何时停止。你不是在编写死板的脚本步骤，而是赋予了它「代理权」(Agency)。

<Callout type="info" title="为什么消息必须累积？">
为什么要保留所有消息，而不仅仅传递最新的结果？

没有历史记录，模型就会陷入「失忆」状态。它不会知道用户的要求是什么，它尝试过什么，哪些失败了，哪些成功了。一个正在调试测试用例的智能体，需要记住在调查新错误之前，它已经修复了之前的导入错误。

消息历史记录不仅仅是日志——它是推理的**基石**。
</Callout>

---

## 什么时候停止？

最简单的循环会在模型停止调用工具时终止。但在现实系统中，你需要更多保障：

```javascript
while (true) {
    if (turns >= max_turns) {
        return error("超出了最大迭代次数");
    }
    if (cost >= budget) {
        return error("预算已耗尽");  
    }
    if (interrupted()) {
        return error("用户中断");
    }
        
    let response = call_llm(messages, tools);
    turns++;
    cost += response.cost;
    
    if (response.error && !is_recoverable(response.error)) {
        return error(response.error);
    }
        
    // ... 处理响应
}
```

**五种终止条件：**

1. **自然完成 (Natural Completion)**：模型在回复中不再调用工具。这通常意味着任务已完成，不过「我无法完成该任务」也属于自然完成。

2. **用户中断 (User Interrupt)**：监控任务的人决定停止。智能体应该始终是可以被中断的。

3. **最大迭代次数 (Max Turns)**：防止陷入死循环。简单任务可能需要 5-10 次迭代，复杂项目可能需要数百次。

4. **预算耗尽 (Budget Exhaustion)**：API 调用是要花钱的。大型代码库会迅速消耗 Token。设置上限以避免惊人的账单。

5. **不可恢复的错误 (Unrecoverable Errors)**：例如身份验证过期、API 宕机等根本性故障。（可恢复的错误——如「文件未找到」——应转化为消息发回给模型。）

---

## 像状态机一样思考

将你的智能体视为一个**状态机 (State Machine)**。

任何时刻的状态包括：
- 消息历史
- 元数据（迭代次数、成本、已修改的文件）
- 外部环境（磁盘、数据库、API）

每一次迭代都是一次状态转移。当前状态 + 模型响应 = 下一个状态。

```javascript
function agent_step(state) {
    let response = call_llm(state.messages, state.tools);
    
    if (response.has_tool_calls) {
        let results = execute_tools(response.tool_calls);
        return {
            messages: [...state.messages, response, ...results],
            turns: state.turns + 1,
            cost: state.cost + response.cost,
            done: false
        };
    } else {
        return { ...state, result: response.text, done: true };
    }
}
```

将这些转移视为**不可变的 (Immutable)**。不要就地修改状态。为什么？

- **调试**：可以精确查看每一步存在什么状态。
- **暂停/恢复**：将状态序列化，稍后继续。
- **分支**：从任何一点分叉以探索不同的替代方案。
- **可复现性**：给定相同的模型输出，相同的状态必然导向相同的下一个状态。

<Callout type="warning" title="状态 ≠ 消息">
状态包含的内容远多于消息，你还需要跟踪：

```javascript
state = {
    messages: [...],
    session_id: "abc123",
    turns: 7,
    tokens_used: 45000,
    files_modified: ["foo.py", "bar.py"],
    checkpoints: [...]
}
```

理论上你可以从消息中重构元数据，但实践中这样做既脆弱又昂贵。请明确地记录它们。
</Callout>

---

## 流式传输与预执行

现代智能体会逐个 Token 地流式传输响应，以提高响应速度。这创造了一个机会：为什么要等到完整响应结束才执行工具呢？

```javascript
let stream = call_llm_streaming(messages, tools);
let pending = [];

for await (let chunk of stream) {
    if (chunk.is_text) {
        display(chunk.text);
    }
    if (chunk.is_tool_call_complete) {
        // 立即开始执行
        pending.push(execute_async(chunk.tool_call));
    }
}

let results = await Promise.all(pending);
```

如果模型请求了三次文件读取，你可以在每个工具调用解析完成后立即并行启动。实际上，并行读取可以将单次迭代延迟减少一半甚至更多。

但并发处理很棘手：

- **顺序性**：「先写入 A，然后读取 A」的操作不能并行。
- **部分调用**：在参数完全流式传输完成之前，不要执行工具。
- **取消机制**：用户在流传输中途点击中断，正在执行的任务该如何处理？

安全的做法：**并行运行只读工具，串行运行修改类工具**。

---

## 失败模式

这些问题总会找上门来，请提前做好准备。

### 无限循环 (Infinite Loops)

无限循环是最常见的故障。模型感到困惑，不断重试一个失败的工具，或者进入死循环：检查状态 → 未就绪 → 等待 → 检查状态 → 未就绪 → 等待……

**修复方案**：务必强制执行最大迭代次数。*必须执行*。

### 递归爆炸 (Recursive Explosion)

递归爆炸更加隐蔽：

```javascript
tools = {
    "ask_assistant": (question) => run_agent(question, tools) // 递归调用
}
```

智能体生成智能体，智能体再生成智能体。你需要设置每一层的限制，以及全局的总限制。

### 中途上下文溢出 (Context Overflow Mid-Task)

智能体读取了一个大文件（5万 Token），又读了一个。到了第三步——超出了上下文长度限制。任务完成了一半，却无法继续。

策略：
- **总结 (Summarization)**：定期压缩旧消息。
- **截断 (Truncation)**：丢弃最旧的消息（会丢失历史信息）。
- **输出限制**：在将工具输出追加到上下文之前限制其大小。
- **更强大的模型**：为复杂任务使用具备超长上下文的模型。

### 崩溃导致的状态丢失

工作了二十分钟后，进程崩溃了。如果没有持久化，你必须从头开始。

**修复方案**：每次状态转移后记录检查点 (Checkpoint)。

```javascript
while (true) {
    save_checkpoint(state);
    let response = call_llm(state.messages);
    state = apply_transition(state, response);
    save_checkpoint(state);
}
```

### 部分执行 (Partial Execution)

模型请求了三次文件写入。第一次成功了，第二次在中途崩溃，第三次根本没运行。系统现在处于不一致状态——重启后，智能体不知道发生了什么。

选项：
- **幂等工具 (Idempotent Tools)**：运行两次的效果等于运行一次。
- **事务日志 (Transaction Logging)**：在继续之前记录哪些已执行。
- **原子批处理**：要么全部成功，要么全部回滚（在文件操作中很难实现）。

---

## 心理模型

### REPL 模式

```javascript
// 传统的 REPL (读取-求值-打印循环)
while (true) {
    input = read();        // 获取输入
    result = eval(input);  // 计算结果
    print(result);         // 打印输出
}

// 智能体循环 (Agent Loop)
while (true) {
    response = call_llm(messages); // "读取" 模型想要做什么
    results = execute(response);   // 通过运行工具进行 "求值"
    messages.push(results);        // 将结果 "打印" 到上下文中
}
```

在 REPL 中，人类决定下一个动作；在智能体循环中，模型决定下一个动作。结构完全一致。

### 游戏循环 (Game Loop)

```javascript
while (game_running) {
    process_input();      // 处理输入
    update_game_state();  // 更新游戏状态
    render_frame();       // 渲染帧
}
```

游戏不会等玩家想好完整计划后才更新。每一帧都在处理输入、更新世界、渲染画面。智能体循环亦是如此——处理上下文、通过工具更新世界、将结果渲染回上下文。

游戏循环还揭示了另一点：**每一次迭代都必须足够快**。掉帧会导致卡顿，而缓慢的迭代会毁掉用户体验。

---

## 调试

当智能体行为异常时，状态机模型就是你最好的朋友。

1. **转储消息 (Dump Messages)**：完整的历史记录通常能解释哪里出了问题。它看到了什么导致了错误的决定？

2. **记录转移过程**：当第 47 步失败时，检查第 45、46、47 步的状态，找出是如何发展到那一步的。

3. **从检查点回放**：恢复到较早的状态。修改某些内容，看看结果是否会改变。

4. **追踪工具影响**：仅仅显示「调用了 write_file」提供的信息很少。而「向 src/utils.py 写入了 347 字节，替换了第 12-15 行」则能说明一切。

调试智能体比调试普通代码更难——因为行为是概率性的。但状态机思维为你提供了具体的快照，让你无论模型*为什么*做出某个行动，都能进行审查。

---

## 核心要点

智能体循环看起来简单，实则内核强大：

1. **它是一个 `while` 循环**。不是简单的请求-响应。循环直到终止。
2. **消息持续累积**。不断增长的对话记录是智能体的「记忆」。
3. **模型掌握决策权**。何时调用工具、何时停止——「代理权」源于循环结构。
4. **终止需要护栏**。最大迭代次数、预算控制、中断处理、错误恢复。
5. **以状态转移的角度思考**。不可变状态使调试、持久化和分支处理成为可能。
6. **流式传输带来复杂性**。预执行工具能提升性能，但需要谨慎处理并发。
7. **失败是必然的**。无限循环、上下文溢出、崩溃、部分执行——为这一切做好设计。

打好这个循环的基础，智能体就站稳了脚跟。其余的一切——工具、提示词、权限、多智能体协调——都是在这个模式之上构建的。
