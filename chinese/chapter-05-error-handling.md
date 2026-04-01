# 第 5 章：错误处理与恢复策略

Agent 总会失败。关键在于它是优雅地应对，还是毁灭性地崩溃。

在一个周二的凌晨三点，我被报警电话叫醒了：我们的一个 Agent 已经连续六个小时疯狂请求 Claude API。起因只是一个简单的网络波动导致请求超时，而我们那幼稚的重试逻辑忠实地执行了指令：立即重试。一遍又一遍。每分钟上千次，持续了整整六个小时。结果是我们耗尽了速率限制（rate limit），触发了滥用检测，却一事无成。

修复代码只花了 15 分钟，但这个教训我记了很久。

## 究竟哪里会出错？

Agent 系统的故障通常可以归为几大类，每一类都需要不同的应对策略。

**基础设施故障**（如网络超时、连接重置、DNS 抖动）通常是瞬时的。请求本身没问题，只是“管道”断了。这种情况下，简单的重试通常奏效。

**速率限制（Rate Limiting）** 严格来说并不是传统意义上的错误。它是 API 在告诉你：“请慢一点。”如果把它当作普通故障并立即重试，只会适得其反。

**无效的 LLM 响应** 源于语言模型的概率本质。你可能会遇到带有多余逗号的 JSON、调用了不存在的工具、或者输出被截断。非确定性输出是语言模型的固有特性。

**工具调用失败** 发生在 Agent 的动作碰撞到现实世界时。磁盘空间不足、权限被拒绝、API 返回了非预期的响应。Agent 的请求很合理，但现实说“不”。

**上下文溢出（Context Overflow）** 是 LLM 系统特有的。与传统软件不同，Agent 的工作内存有硬上限。一旦超过这个上限，就没有优雅的退路——请求根本无法照原样继续。

核心洞见在于：适用于网络错误的重试策略会加剧速率限制问题，且对上下文溢出毫无帮助。在尝试修复之前，你必须先搞清楚*为什么*失败。

## 正确的重试姿势

最简单的恢复手段是重试，但“简单”不代表“幼稚”。

### 退避算法（Backoff Algorithm）

想象一台在负载下挣扎的服务器。现在再想象一百个客户端同时失败，并以最高频率重试。你刚刚对一个本就脆弱的系统发起了一场 DDoS 攻击。

```javascript
function retryWithBackoff(operation, maxAttempts) {
    let baseDelay = 1000; // 1秒
    const maxDelay = 60000; // 60秒
    
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
            return operation();
        } catch (error) {
            if (!isRetryable(error)) throw error;
            
            let delay = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);
            sleep(delay);
        }
    }
    throw new Error('超过最大重试次数');
}
```

第一次重试：2 秒后；第二次：4 秒后；第三次：8 秒后。你在为故障系统留出喘息空间的同时，依然在推动进度。

### 惊群效应（The Thundering Herd）

但还有一个问题。如果一千个客户端在同一时刻遇到了网络分区，它们都会在 2 秒后同时重试。又是一个峰值。然后它们又会在 4 秒后同步重试。

你需要引入**抖动（Jitter）**：

```javascript
let delay = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);
let jitter = Math.random() * (delay * 0.25);
sleep(delay + jitter);
```

这种随机偏移将重试请求在时间轴上打散，避免了在预测的时间点形成流量洪峰。

### 识别可重试的场景

并非所有错误都值得重试：

| 状态码                | 含义         | 建议方案                         |
| --------------------- | ------------ | -------------------------------- |
| 400 Bad Request       | 请求格式错误 | 修复代码，不要重试               |
| 401 Unauthorized      | 认证失败     | 刷新 Token 后重试                |
| 404 Not Found         | 资源不存在   | 不要重试                         |
| 429 Too Many Requests | 频率受限     | 等待后重试                       |
| 500 Server Error      | 服务端故障   | 使用退避策略重试                 |
| 503 Unavailable       | 服务不可用   | 使用退避策略重试                 |
| 529 Overloaded (Anthropic 特有) | 负载过高     | 用户操作可重试；后台任务建议放弃 |

连接重置（Connection Resets）比较特殊——通常是由过期的长连接（keep-alive）引起的。建议禁用连接复用并重试一次，若仍失败再向上抛出。

上下文溢出不能原样重试。你必须先对上下文进行转换。

## 升级处理阶梯

当简单重试无果时，你需要采取更激进的干预措施。

**第一级：原样重试。** 等待一会再试，寄希望于瞬时问题已解决。

**第二级：修改请求。** Token 太多？截断输入。模型过载？切换到另一个备选模型。请求太复杂？拆分成更小的步骤。

**第三级：转换上下文。** 当累积的状态（而非单次请求）成为瓶颈时，重塑状态。压缩对话历史，总结旧的工具执行结果，丢弃低优先级的背景信息。

**第四级：升级给用户。** 有些问题需要人类的判断：无法自动解决的身份认证、需要澄清的歧义、需要确认的高风险操作。这不叫失败，这叫正确的委托。

**第五级：带状态保存的优雅退出。** 当所有手段都失效，也要优雅地倒下。保存当前状态，确保工作不丢失，清晰说明原因，并保留系统可恢复的线索。

```javascript
async function executeWithRecovery(request, context) {
    // 第一级：简单重试
    for (let attempt = 1; attempt <= 3; attempt++) {
        try {
            return await execute(request);
        } catch (error) {
            if (!isRetryable(error)) break;
            await sleep(exponentialBackoffWithJitter(attempt));
        }
    }
    
    // 第二级：修改请求
    if (error.type === 'ContextOverflow') {
        const reducedRequest = truncateInput(request, 0.75);
        try { return await execute(reducedRequest); } catch {}
    }
    
    if (error.type === 'ModelOverloaded') {
        try { return await execute(request, { model: fallbackModel }); } catch {}
    }
    
    // 第三级：转换上下文
    if (error.type === 'ContextOverflow') {
        const compactedContext = await compactHistory(context);
        try { return await execute(request, { context: compactedContext }); } catch {}
    }
    
    // 第四级：请用户定夺
    if (isUserPresent()) {
        const userDecision = await promptUser(
            "遇到无法自动解决的错误：" + formatUserFriendlyError(error) +
            "\n是否尝试其他方案？"
        );
        return handleUserDecision(userDecision);
    }
    
    // 第五级：优雅失败
    await saveState(context, request, error);
    throw new GracefulFailure({
        message: "多次尝试后仍无法完成任务",
        stateId: savedStateId,
        suggestion: `使用此命令恢复: /resume ${savedStateId}`
    });
}
```

## 工具调用的错误隔离

当 Bash 命令执行失败时，你有两个选择：让整个 Agent 循环崩溃，或者将错误作为“信息”反馈给 Agent。

```
工具: bash
输入: apt-get install nodejs
结果: {
    "success": false,
    "exit_code": 1,
    "stderr": "E: Could not open lock file - permission denied"
}
```

脆弱的系统会抛出异常。稳健的系统将其包装成工具执行结果，让 LLM 自己去推理：“权限被拒绝——我应该尝试使用 sudo，或者询问用户其他安装方式。”

实现起来非常简单：

```javascript
async function executeTool(tool, input) {
    try {
        const result = await tool.execute(input);
        return { role: 'tool_result', content: result, is_error: false };
    } catch (error) {
        return { role: 'tool_result', content: formatToolError(error), is_error: true };
    }
}
```

错误不会终结循环，它们只是对话的一部分。

## 任务执行中的断点恢复

Agent 会被中断。进程被杀掉、连接断开、用户关上了笔记本。当它们重启时，需要回答：“我刚才在干什么？”

如果你在每一轮对话后都将内容持久化到磁盘，恢复就变得可能。关键检查点包括：

**消息链完整性。** 如果记录以 Assistant 发起的工具调用结束，但没有对应的工具结果，说明这一轮被中断了。

**文件状态验证。** 当时是否正在编辑文件？文件是否还存在？是否被外部修改过？

**计划状态恢复。** 多步骤计划进行到哪一环了？哪些已完成？哪些需要重试？

```javascript
function resumeSession(transcriptPath) {
    const transcript = loadTranscript(transcriptPath);
    const lastMessage = transcript.messages.at(-1);

    if (lastMessage.role === 'assistant' && hasToolCalls(lastMessage)) {
        const pendingTools = lastMessage.toolCalls;
        if (!hasMatchingToolResults(transcript, pendingTools)) {
            for (const tool of pendingTools) {
                const result = checkToolStateAndRecover(tool);
                transcript.append(result);
            }
        }
    }
    // ... 其他状态检查
    return transcript;
}
```

恢复后的 Agent 可能无法百分之百还原现场，但它会理解发生了什么，并能基于现状做出明智的后续决定。

## 优雅降级（Graceful Degradation）

有时候，选择不在于“成功”还是“失败”，而在于“部分成功”还是“全盘皆输”。

一个 Agent 在重构 50 个文件，完成了 47 个后遇到了 3 个边缘情况。是该回滚所有修改，还是完成能做到的并报告遗留问题？

**模型回退。** 主模型不可用？尝试 Haiku 等轻量模型。低质量的回复总比没有回复好。

**功能降级。** 缓存坏了？禁用它。并行执行有问题？回退到串行。

**部分结果。** 尽力而为。分别报告成功和失败的任务，让用户决定是针对失败的部分重试，还是见好就收。

## 错误消息的两个受众

每一条错误消息都有两个需求完全不同的受众。

**用户**需要清晰且可操作的信息：

❌ **糟糕的示范**
"Error: ECONNRESET at TCP.onStreamRead (node:internal/stream:333:27)"

✅ **更好的示范**
"我与 Claude 服务器失去了连接。这通常很快就能恢复。
请稍后重试，如果问题持续，请检查您的网络连接。"

**开发者**则需要调试上下文：堆栈信息、请求 ID、时间戳、系统状态。

```javascript
function handleError(error, context) {
    log.error({
        message: error.message,
        stack: error.stack,
        requestId: context.requestId,
        systemState: captureSystemState()
    });
    
    return {
        userMessage: translateToUserFriendly(error),
        suggestion: getSuggestion(error),
        canRetry: isRetryable(error)
    };
}
```

将技术错误映射为人类能听懂的话：

- `RateLimitExceeded` → "请求太频繁了，请稍等片刻。"
- `ContextOverflow` → "对话内容太长了，我会先总结一下之前的内容。"
- `AuthenticationFailed` → "登录已过期，请运行 /login 重新连接。"

## 韧性思维（Resilience Mindset）

在构建 Agent 时，要不断问自己：“如果这里挂了会发生什么？”

- API 返回了乱码怎么办？
- 工具执行时间比预期长了 10 倍怎么办？
- 上下文空间耗尽了怎么办？
- 用户在操作中途关掉了网页怎么办？
- 对文件状态的假设错误了怎么办？

每一个问题都需要一个答案。重试、升级、保存状态后优雅退出——这些都是有效的。唯独“崩溃并丢失所有进度”不是。

那些赢得信任的 Agent，往往是那些能妥善处理逆境的 Agent。它们会说：“我遇到了一个问题，这是我目前已经完成的工作，建议我们接下来这样处理。”它们保留成果而非丢失进度，解释问题而非掩盖矛盾，平滑降级而非毁灭性崩溃。

这并不是什么光鲜亮丽的工作。但正是这些工作将演示与产品区分开来。可靠性，才是将精巧的原型转化为生产力工具的基石。