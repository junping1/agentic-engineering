# 第 6 章：上下文管理 (Context Management)

你正在进行一项复杂的代码重构任务，已经投入了十五分钟。Agent 的表现非常出色：它阅读了代码库，理解了架构，提出了清晰的迁移策略，并开始付诸实施。它已经编辑了 8 个文件，顺藤摸瓜解决了两个隐蔽的 bug，目前正处于更新测试套件的关键阶段。

突然，它说：“我不确定我们刚才在做什么，你能提醒我到目前为止我们做了哪些更改吗？”

上下文窗口已满，智能体学到的一切都丢失了。

---

上下文管理是智能体（Agentic Systems）系统中最棘手的问题。它是一个制约因素，决定了系统的方方面面——从工具设计到对话结构，再到复杂任务的处理方式。如果处理不当，智能体会恰恰在你最需要它的时候变得不可靠：尤其是在深入复杂工作、重新建立上下文的成本极高甚至无法实现时。

这个问题最令人痛苦的地方在于，你可以预见故障的到来。与悄然发生的内存泄漏不同，上下文限制是确定性的。你*知道*什么时候会撞墙。关键问题在于选择“忘记”什么——而这个决定直接关系到智能体能否完成任务。

## 堆积问题 (The Accumulation Problem)

每个大语言模型（LLM）都有固定的上下文限制。Claude 提供 20 万（200K）Token，GPT-4 提供 12.8 万，较小的模型可能只有 8000。在观看 Agent 实际工作之前，这些数字听起来很慷慨。

以下是一个典型的编码会话：

```
第 1 轮： 用户描述功能需求
第 2 轮： Agent 阅读 3 个源文件                    ~6,000 Token
第 3 轮： Agent 提出方案，开始编码
第 4 轮： 语法错误 —— Agent 读取错误输出            ~1,500 Token
第 5 轮： Agent 读取另外 2 个文件获取上下文         ~4,000 Token
第 6 轮： 修复成功，Agent 运行测试
第 7 轮： 测试结果返回                           ~3,000 Token
第 8 轮： Agent 阅读测试文件以调试失败项           ~2,000 Token
...
```

每一轮都在累积。文件内容不断堆叠，工具结果成倍增加，对话历史随每次交换而增长。经过 25 轮卓有成效的工作后，你可能已经消耗了 9 万个 Token——而 Agent 甚至还没触及最棘手的部分。

```
┌─────────────────────────────────────────────────────────────┐
│                    上下文窗口 (200K)                         │
├─────────────────────────────────────────────────────────────┤
│ 系统提示词 (System Prompt) │████████                    │  8K   │
│ 对话历史 (History)         │████████████████████████████│ 45K   │
│ 工具结果 (文件)             │████████████████████████████│ 52K   │
│ 最近消息 (Recent)          │████████████                │ 12K   │
│ ─────────────────────────────────────────────────────────── │
│ 已使用 (USED)                                       │ 117K  │
│ 剩余 (REMAINING)           │░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ 83K   │
└─────────────────────────────────────────────────────────────┘
```

图表看起来还不错——剩余 83K，空间充裕。但这里有一个隐藏的问题：剩余空间需要容纳 Agent 的响应、它读取的任何新文件，以及用于抵消估算误差的缓冲。*可用*空间远比看起来要小。

而真正的难题在于：与传统软件可以通过增加 RAM 来缓解内存压力不同，LLM 有一个硬上限。一旦触及，你就无法继续。你必须移除一些内容——而无论你移除什么，智能体就会忘记什么。

## Token 账目统计 (Token Accounting)

在管理上下文之前，你需要准确了解自己到底有多少空间。

### 真实预算 (The Real Budget)

```
可用输入 Token = 模型上限 - 输出预留 - 安全缓冲

示例：
  模型上限 (Model Limit):   200,000 Token
  输出预留 (Output Reserve): 20,000 Token  (为响应留出空间)
  安全缓冲 (Safety Buffer):  15,000 Token  (估算误差 + 开销)
  ─────────────────────────────────────────────────────
  可用输入 (Usable Input):  165,000 Token
```

**输出预留**是雷打不动的。如果你将上下文填满到 19.9 万，模型只能生成 1000 个 Token 的响应——这不足以写出有意义的代码。根据你的 Agent 编写长代码块还是短响应，预留 10-20%。

**安全缓冲**是为了应对 Token 计数不精确的问题。不同的分词器（Tokenizer）产生的结果不同。工具调用有额外开销，图像有复杂的计算。5-10% 的缓冲可以防止在“精确”计算下仍触及限制的失败情况。

### 准确计数

初级方法（字符数除以 4）适用于粗略估算：

```python
def estimate_tokens(text: str) -> int:
    return len(text) // 4  # 大约每 4 个字符 1 个 token
```

但真正的智能体系统包含的内容远不止纯文本。

**工具调用 (Tool calls)** 包括函数名、JSON 参数和 Schema 信息。一个简单的 `read_file` 调用，除了文件内容本身，可能产生 50-100 个 Token 的额外开销。

**图像 (Images)** 非常昂贵。一张高分辨率截图可能耗费 1500+ Token。如果你的 Agent 在执行视觉任务，图像通常是首选的移除对象。

**结构化内容 (Structured content)** 的分词效率较低。1000 字符的 JSON 块可能耗费 400 个 Token，而 1000 字符的散文则只需 250 个。代码的代价介于两者之间。

API 响应会告诉你实际的使用量。利用这个数据进行校准：

```python
response = client.messages.create(...)
actual = response.usage.input_tokens
drift = actual - estimated

# 跟踪估算误差，如果由于系统性偏差，请调整缓冲值
```

## 渐进式压缩 (Progressive Compaction)

当上下文变得过大时，你需要对其进行压缩。但压缩是有代价的——移除的每个 Token 都可能是丢失的上下文。关键在于**逐步升级 (escalate gradually)**。在采取激进方案之前，先尝试成本低、影响小的选项。

### 第 1 级：微压缩 (Microcompact)

通过精准编辑减少 Token，而不损失语义内容：

- 截断冗长的工具输出
- 去除 ANSI 转义码和多余空格
- 折叠重复模式
- 移除文件内容中的冗余部分

```python
def microcompact_tool_result(result: str, budget: int) -> str:
    """在保留有用性的前提下修剪工具输出。"""
    
    # 清理噪音
    result = re.sub(r'\x1b\[[0-9;]*m', '', result)  # ANSI 转义码
    result = re.sub(r'\n{3,}', '\n\n', result)      # 多余换行符
    
    # 如果仍超出预算，则进行截断
    if count_tokens(result) > budget:
        truncated = result[:budget * 4]
        return f"{truncated}\n\n[已截断，共 {len(result)} 字符]"
    
    return result
```

微压缩执行快（无需 LLM 调用）、成本低，且能保留大部分信息。它可能为你收回 5-15% 的上下文空间。请始终首先尝试这一步。

### 第 2 级：总结摘要 (Summarization)

当精准编辑不足以解决问题时，利用 LLM 对较早的对话轮次进行总结：

```python
async def summarize_old_turns(
    messages: list[Message], 
    keep_recent: int = 5
) -> list[Message]:
    """用 LLM 生成的摘要替换旧消息。"""
    
    old_messages = messages[:-keep_recent]
    recent_messages = messages[-keep_recent:]
    
    summary_prompt = """请简要总结这段对话。
    保留：关键决策、当前任务状态、提到的约束条件。
    
    对话内容：
    {format_messages(old_messages)}"""
    
    summary = await llm.generate(summary_prompt)
    
    return [
        Message(role="system", content=f"[摘要]\n{summary}"),
        *recent_messages
    ]
```

这是一种显式的权衡：你失去了逐字记录的历史，但保留了要点。这种方式对于已完成的工作效果很好（例如：“之前我们实现了 JWT 认证，所有测试均已通过”），但对于细节仍然至关重要的进行中工作效果不佳。

预计可减少 40-60% 的空间，伴随中等程度的信息丢失。

### 第 3 级：激进修剪 (Aggressive Prune)

这是终极手段。仅保留最低限度的内容，并根据显式状态重建上下文：

```python
def aggressive_prune(
    messages: list[Message],
    current_task: str,
    critical_files: dict[str, str]
) -> list[Message]:
    """从零开始重建上下文。"""
    
    recent = messages[-3:]  # 仅保留最后几次交换
    
    state_injection = f"""
## 会话状态 (Session State)

你正处于任务中。以下是你需要了解的信息：

### 当前任务
{current_task}

### 关键文件
{format_files(critical_files)}

### 注意
较旧的上下文已被压缩。最近的对话保留如下。
"""
    
    return [
        Message(role="system", content=state_injection),
        *recent
    ]
```

这相当于一次重启。Agent 会失去对话记忆，但*前提*是你准确捕捉了核心状态，它仍保留继续执行的能力。预计减少 70-90% 的空间，伴随显著的信息丢失。

## 保留什么，舍弃什么

上下文的价值并不均等。建立一套清晰的层级结构：

**始终保留 (Preserve always):**
- 当前任务描述（没有它，Agent 就失去了方向）
- 最近的决策及其依据（防止反复无常）
- 正在被积极编辑的文件
- 尝试失败产生的错误消息（防止重蹈覆辙）

**按需总结 (Summarize when needed):**
- 已完成的子任务（“评估了认证方案，因为 X 选择了 JWT”）
- 旧的对话轮次（早期的需求、澄清性的提问）
- 文件的历史版本（如果编辑了多次，仅保留当前版本）

**优先丢弃 (Discard first):**
- 冗长的测试通过输出（“47 个测试全部通过”即可）
- 早先读取的已失效文件内容
- 图像（成本高，提取信息后通常是冗余的）
- 探索性的死胡同（保留有效的，丢弃无效的）

```
┌─────────────────────────────────────────────────────────────┐
│                    压缩优先级 (Compaction Priority)          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐                                        │
│  │     优先丢弃     │  图像、冗长输出、死胡同                 |
│  │   (DISCARD)     │  失效的文件版本                        │
│  └────────┬────────┘                                        │
│           ↓                                                 │
│  ┌─────────────────┐                                        │
│  │     进行总结     │  已完成的任务、旧对话                  |
│  │  (SUMMARIZE)    │  之前的迭代过程                        │
│  └────────┬────────┘                                        │
│           ↓                                                 │
│  ┌─────────────────┐                                        │
│  │     始终保留     │  当前任务、最近决策                    |
│  │  (PRESERVE)     │  活跃文件、最近的错误提示               │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 重新注入：反直觉的关键步骤

这里是许多人容易掉坑的地方：移除上下文后，你通常需要*回传*一部分内容。压缩是有损的。重新注入（Re-injection）可以找回那些原本会丢失的关键细节。

可以将其理解为"恢复与压缩"：压缩阶段移除冗余，恢复阶段将关键信息重新注入，确保智能体保持完成任务的能力。

**需要重新注入的内容：**

当前计划状态：
```
## 进度
步骤 1: ✓ 搭建项目结构
步骤 2: ✓ 实现数据模型  
步骤 3: → 实现 API 接口 (进行中)
步骤 4: □ 编写测试
步骤 5: □ 添加文档
```

活跃文件内容（按优先级阶梯分配 Token 预算——给正在编辑的文件分配最多的预算，仅作参考的文件分配较少预算）。

关键决策日志：
```
## 已定决策
- 使用 PostgreSQL (用户要求)
- 使用 TypeScript 而非 Python (为了类型安全)
- REST API (符合用户偏好)
- JWT 认证 (已在 src/auth/ 中实现)
```

跳过重新注入是压缩过程中最常见的错误。Agent 最终会有足够的上下文来“继续”，但不足以“正确地继续”。

## Prompt 缓存的考量 (Prompt Caching)

现代 LLM API 提供了 Prompt 前缀缓存功能——如果你的请求前半部分与之前的请求相同，缓存部分的处理速度更快且成本更低。这与压缩之间存在冲突。

在构思上下文结构时应考虑缓存友好性：

```
┌─────────────────────────────────────────────────────────────┐
│  静态前缀 (可缓存)                                           │
│  ├─ 系统提示词 (System prompt)                              │
│  ├─ 工具定义 (Tool definitions)                             │
│  └─ 项目背景 (README、架构文档)                              │
├─────────────────────────────────────────────────────────────┤
│                    ↑ 缓存边界 ↑                              │
├─────────────────────────────────────────────────────────────┤
│  动态后缀 (随请求变化)                                       │
│  ├─ 对话历史                                                │
│  ├─ 当前文件内容                                            │
│  └─ 最近的工具结果                                          │
└─────────────────────────────────────────────────────────────┘
```

权衡在于：激进的压缩可能会改变静态前缀的内容，导致缓存失效。如果你的系统提示词包含动态状态，每次压缩都会导致缓存未命中。

设计压缩策略时应尽量尊重缓存边界：

```python
def compact_preserving_cache(
    messages: list[Message], 
    cache_boundary: int
) -> list[Message]:
    """仅压缩动态部分。"""
    
    static_prefix = messages[:cache_boundary]
    dynamic_suffix = messages[cache_boundary:]
    
    # 仅压缩缓存边界之后的内容
    compacted = compact(dynamic_suffix)
    
    return static_prefix + compacted
```

有时为了进行激进压缩，你不得不接受缓存未命中。通过跟踪两个指标——缓存命中率和上下文利用率——来为你的用例寻找最佳平衡点。

## 压缩的时机

有两种派系：

**主动式 (Proactive)**：监控使用量，在触及限制前进行压缩。体验平滑，行为可预测。但可能会进行不必要的压缩。

```python
async def maybe_compact(messages: list[Message], threshold: float = 0.8):
    usage = sum(count_tokens(m) for m in messages)
    if usage / max_input > threshold:
        return await compact_level_1(messages)
    return messages
```

**响应式 (Reactive)**：等到 API 拒绝请求后再压缩。绝不进行多余压缩。但你会因为请求被拒而浪费一次 API 调用，并感受到明显的延迟峰值。

```python
async def send_with_reactive_compact(messages: list[Message]):
    level = 0
    while level <= 3:
        try:
            return await api.send(messages)
        except ContextLimitExceeded:
            level += 1
            messages = await compact(messages, level=level)
    raise ContextUnrecoverable("最大压缩后仍无法容纳")
```

**推荐方案：混合模式。** 在 85% 处进行主动监控，并配合响应式回退：

```python
async def send_message(messages: list[Message]):
    messages = await maybe_compact(messages, threshold=0.85)
    try:
        return await api.send(messages)
    except ContextLimitExceeded:
        messages = await emergency_compact(messages)
        return await api.send(messages)
```

既能提供平滑体验，又有应对极端情况的安全网。

## 常见陷阱

**过度激进的压缩。** Agent 忘记决策、重复工作、自我矛盾。在移除上下文前问问自己：“如果这段消失了，会发生什么坏事？”如果答案是“Agent 可能会重做这部分工作”，请保留它或明确地总结它。

**压缩过晚。** 频繁的 API 拒绝、浪费的 Token、延迟峰值。应在 80-85% 时主动压缩，而不是 99%。

**丢失任务核心背景。** Agent 失去了目标，偏离任务，要求用户重复内容。通过为消息打上优先级标签来解决：

```python
@dataclass
class Message:
    role: str
    content: str
    compactable: bool = True  # 核心上下文设为 False
```

**无限压缩循环。** 系统进行压缩、重试、被拒、再压缩，无止境。这通常发生在最大化压缩后仍无法容纳请求时——往往是因为当前消息本身或必需的 Schema 过大。检测循环并优雅地报错：

```python
async def send_with_compact(messages: list[Message], max_attempts: int = 3):
    for attempt in range(max_attempts):
        try:
            return await api.send(messages)
        except ContextLimitExceeded:
            if attempt == max_attempts - 1:
                raise ContextUnrecoverable(
                    "最大化压缩后仍无法容纳。请考虑将任务拆分为更小的部分。"
                )
            messages = await escalate_compaction(messages, level=attempt + 1)
```

**忘记重新注入。** 压缩后，即使有“足够”的上下文，Agent 似乎也处于混乱状态。压缩移除了细节，却没有恢复核心状态。请将压缩视为两个阶段：移除，然后恢复。

## 实施清单

<Callout type="note">
正在构建上下文管理系统？请按顺序检查以下各项。
</Callout>

1. **先测量。** 在做任何事之前，先实现准确的 Token 计数。准确的 Token 计数是其他一切的基础。

2. **设定明确预算。** 记录你的模型上限、输出预留和安全缓冲。

3. **构建渐进层级。** 微压缩 → 总结 → 激进修剪。优先尝试代价最小的选项。

4. **标注消息优先级。** 在创建消息时（而非压缩时）就将其标记为“核心”或“可压缩”。

5. **实施重新注入。** 任何压缩之后，务必恢复当前任务状态、活跃文件和关键决策。

6. **尊重缓存边界。** 如果使用 Prompt 缓存，尽可能仅压缩动态部分。

7. **增加监控指标。** 跟踪利用率随时间的变化。接近限制时发出预警。

8. **测试极端情况。** 在 100% 利用率下会发生什么？达到最大压缩后呢？遇到巨大的单条消息时呢？

9. **优雅失败。** 当压缩也无法挽救请求时，提供清晰的错误信息和恢复路径。

---

上下文管理本质上是**受控的遗忘**。与人类不由自主的遗忘不同，你的 Agent 可以选择记住什么、丢弃什么。这是一种超能力——前提是你能谨慎运用。

核心原则不在于任何单一技术。而是在于理解：上下文是有限的，压缩是有损的，你关于“忘记什么”的决定将直接塑造你的智能体所能达到的高度。

如果这一点做错了，你将不得不花费大量时间，去重新解释那些本不该被遗忘的上下文。
