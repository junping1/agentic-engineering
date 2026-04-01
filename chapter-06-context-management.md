# Chapter 6: Context Management

You're fifteen minutes into a complex refactoring task. The agent has been brilliant—it read the codebase, understood the architecture, proposed a clean migration strategy, and started implementing. It's edited eight files, tracked down two subtle bugs, and is halfway through updating the test suite.

Then it says: "I'm not sure what we were working on. Could you remind me what changes we've made so far?"

The context window filled up. The context window filled up, and everything the agent learned was lost.

---

Context management is the hardest problem in agentic systems. It's the constraint that shapes everything else—how you design tools, how you structure conversations, how you approach complex tasks. Get it wrong, and your agent becomes unreliable at exactly the moment you need it most: deep into difficult work where re-establishing context is expensive or impossible.

What makes this problem especially painful is that you can see the crash coming. Unlike memory leaks that sneak up on you, context limits are deterministic. You *know* when you'll hit the wall. The question is what to forget—and that decision directly determines whether your agent can finish its job.

## The Accumulation Problem

Every LLM has a fixed context limit. Claude offers 200,000 tokens, GPT-4 offers 128,000, smaller models might give you 8,000. These numbers sound generous until you watch an agent actually work.

Here's a typical coding session:

```
Turn 1:   User describes feature request
Turn 2:   Agent reads 3 source files                    ~6,000 tokens
Turn 3:   Agent proposes approach, starts coding
Turn 4:   Syntax error—agent reads error output         ~1,500 tokens
Turn 5:   Agent reads 2 more files for context          ~4,000 tokens
Turn 6:   Fix works, agent runs tests
Turn 7:   Test output comes back                        ~3,000 tokens
Turn 8:   Agent reads test file to debug failure        ~2,000 tokens
...
```

Each turn accumulates. File contents pile up. Tool results multiply. Conversation history grows with every exchange. After 25 turns of productive work, you might be at 90,000 tokens—and the agent hasn't even reached the tricky part yet.

```
┌─────────────────────────────────────────────────────────────┐
│                    Context Window (200K)                    │
├─────────────────────────────────────────────────────────────┤
│ System Prompt          │████████                    │  8K   │
│ Conversation History   │████████████████████████████│ 45K   │
│ Tool Results (files)   │████████████████████████████│ 52K   │
│ Recent Messages        │████████████                │ 12K   │
│ ─────────────────────────────────────────────────────────── │
│ USED                                                │ 117K  │
│ REMAINING              │░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ 83K   │
└─────────────────────────────────────────────────────────────┘
```

The diagram looks fine—83K remaining, plenty of room. But here's what it doesn't show: that remaining space needs to hold the agent's response, any new files it reads, and a buffer for estimation errors. The *usable* space is much smaller than it appears.

And here's the real problem: unlike traditional software where you can throw more RAM at memory pressure, LLMs have a hard ceiling. When you hit it, you can't proceed. You have to remove something—and whatever you remove, the agent forgets.

## Token Accounting

Before you can manage context, you need to understand exactly how much space you actually have.

### The Real Budget

```
Usable Input Tokens = Model Limit - Output Reserve - Safety Buffer

Example:
  Model Limit:    200,000 tokens
  Output Reserve:  20,000 tokens  (room for the response)
  Safety Buffer:   15,000 tokens  (estimation errors + overhead)
  ─────────────────────────────────────────────────────
  Usable Input:   165,000 tokens
```

The **output reserve** is non-negotiable. If you fill the context to 199,000 tokens, the model can only generate 1,000 tokens of response—not enough for meaningful code. Reserve 10-20% depending on whether your agent writes long code blocks or short responses.

The **safety buffer** accounts for the fact that token counting is imprecise. Different tokenizers produce different counts. Tool calls have overhead. Images have complex calculations. A 5-10% buffer prevents failures where your "careful" accounting still hits the limit.

### Counting Accurately

The naive approach—divide characters by four—works for rough estimates:

```python
def estimate_tokens(text: str) -> int:
    return len(text) // 4  # ~4 characters per token
```

But real agent systems have more than plain text.

**Tool calls** include function names, JSON parameters, and schema information. A simple `read_file` call might cost 50-100 tokens in overhead beyond the file content itself.

**Images** are expensive. A single high-resolution screenshot can cost 1,500+ tokens. If your agent is doing visual work, images often become the first candidates for removal.

**Structured content** tokenizes inefficiently. A 1,000-character JSON blob might cost 400 tokens while 1,000 characters of prose costs 250. Code falls somewhere in between.

The API response tells you actual usage. Use this to calibrate:

```python
response = client.messages.create(...)
actual = response.usage.input_tokens
drift = actual - estimated

# Track estimation errors, adjust buffer if consistently off
```

## Progressive Compaction

When context grows too large, you need to compact it. But compaction has costs—every removed token is potentially lost context. The principle: **escalate gradually**. Try cheap, low-impact options before aggressive ones.

### Level 1: Microcompact

Surgical edits that reduce tokens without losing semantic content:

- Truncate verbose tool outputs
- Strip ANSI codes and excess whitespace
- Collapse repeated patterns
- Remove redundant sections from file contents

```python
def microcompact_tool_result(result: str, budget: int) -> str:
    """Trim tool output while preserving usefulness."""
    
    # Strip noise
    result = re.sub(r'\x1b\[[0-9;]*m', '', result)  # ANSI codes
    result = re.sub(r'\n{3,}', '\n\n', result)      # Excess newlines
    
    # Truncate if still over budget
    if count_tokens(result) > budget:
        truncated = result[:budget * 4]
        return f"{truncated}\n\n[Truncated, {len(result)} chars total]"
    
    return result
```

Microcompaction is fast (no LLM calls), cheap, and preserves most information. You might reclaim 5-15% of context. Always try it first.

### Level 2: Summarization

When surgical edits aren't enough, use the LLM to summarize older conversation turns:

```python
async def summarize_old_turns(
    messages: list[Message], 
    keep_recent: int = 5
) -> list[Message]:
    """Replace old messages with an LLM-generated summary."""
    
    old_messages = messages[:-keep_recent]
    recent_messages = messages[-keep_recent:]
    
    summary_prompt = """Summarize this conversation concisely.
    Preserve: key decisions, current task state, constraints mentioned.
    
    Conversation:
    {format_messages(old_messages)}"""
    
    summary = await llm.generate(summary_prompt)
    
    return [
        Message(role="system", content=f"[SUMMARY]\n{summary}"),
        *recent_messages
    ]
```

The trade-off is explicit: you lose verbatim history but retain the gist. This works well for completed work ("Earlier, we implemented JWT auth and all tests pass") but poorly for ongoing work where details still matter.

Expect 40-60% reduction with moderate information loss.

### Level 3: Aggressive Prune

The nuclear option. Keep only the bare minimum and rebuild context from explicit state:

```python
def aggressive_prune(
    messages: list[Message],
    current_task: str,
    critical_files: dict[str, str]
) -> list[Message]:
    """Rebuild context from scratch."""
    
    recent = messages[-3:]  # Keep only last few exchanges
    
    state_injection = f"""
## Session State

You are mid-task. Here's what you need to know:

### Current Task
{current_task}

### Critical Files
{format_files(critical_files)}

### Note
Older context has been compacted. Recent exchanges preserved below.
"""
    
    return [
        Message(role="system", content=state_injection),
        *recent
    ]
```

This is a reboot. The agent loses its conversational memory but retains the ability to continue—*if* you've properly captured the essential state. Expect 70-90% reduction with significant information loss.

## What to Keep, What to Cut

Not all context is equal. Develop a clear hierarchy:

**Preserve always:**
- Current task description (without this, the agent has no direction)
- Recent decisions and their rationale (prevents flip-flopping)
- Files being actively edited
- Error messages from failed attempts (prevents repeating mistakes)

**Summarize when needed:**
- Completed sub-tasks ("Explored auth options, chose JWT because X")
- Old conversation turns (early requirements, clarifying questions)
- Previous file versions (if edited multiple times, keep only current)

**Discard first:**
- Verbose passing test output ("All 47 tests passed" suffices)
- Superseded file contents from earlier reads
- Images (high cost, often redundant once information extracted)
- Exploratory dead ends (keep what worked, drop what didn't)

```
┌─────────────────────────────────────────────────────────────┐
│                   Compaction Priority                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐                                        │
│  │   DISCARD       │  Images, verbose output, dead ends     │
│  │   FIRST         │  Superseded file versions              │
│  └────────┬────────┘                                        │
│           ↓                                                 │
│  ┌─────────────────┐                                        │
│  │   SUMMARIZE     │  Completed tasks, old conversation     │
│  │                 │  Previous iterations                   │
│  └────────┬────────┘                                        │
│           ↓                                                 │
│  ┌─────────────────┐                                        │
│  │   PRESERVE      │  Current task, recent decisions        │
│  │   ALWAYS        │  Active files, recent errors           │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Re-injection: The Counterintuitive Step

Here's what trips people up: after removing context, you often need to *add* content back. Compaction is lossy. Re-injection restores critical details that would otherwise be lost.

You're restoring essential details that the compression phase stripped away — trading a small amount of space for the agent's ability to continue working correctly. You're duplicating information to optimize for the access pattern—in this case, the agent's ability to do its job.

**What to re-inject:**

Current plan state:
```
## Progress
Step 1: ✓ Set up project structure
Step 2: ✓ Implement data models  
Step 3: → Implement API endpoints (IN PROGRESS)
Step 4: □ Write tests
Step 5: □ Add documentation
```

Active file contents (with token budgets per priority tier—most budget to files being edited, less to files merely referenced).

Critical decisions log:
```
## Decisions Made
- Using PostgreSQL (user requirement)
- TypeScript over Python (for type safety)
- REST API (per user preference)
- JWT auth (implemented in src/auth/)
```

Skipping re-injection is the most common compaction mistake. The agent ends up with enough context to *continue* but not enough to continue *correctly*.

## Prompt Caching Considerations

Modern LLM APIs cache prompt prefixes—if your request starts with the same content as a previous request, the cached portion processes faster and cheaper. This creates tension with compaction.

Structure your context with caching in mind:

```
┌─────────────────────────────────────────────────────────────┐
│  STATIC PREFIX (cacheable)                                  │
│  ├─ System prompt                                           │
│  ├─ Tool definitions                                        │
│  └─ Project context (README, architecture docs)             │
├─────────────────────────────────────────────────────────────┤
│                    ↑ Cache boundary ↑                       │
├─────────────────────────────────────────────────────────────┤
│  DYNAMIC SUFFIX (changes per request)                       │
│  ├─ Conversation history                                    │
│  ├─ Current file contents                                   │
│  └─ Recent tool results                                     │
└─────────────────────────────────────────────────────────────┘
```

The trade-off: aggressive compaction might change content in the static prefix, invalidating the cache. If your system prompt includes dynamic state, every compaction creates a cache miss.

Design compaction to respect cache boundaries when possible:

```python
def compact_preserving_cache(
    messages: list[Message], 
    cache_boundary: int
) -> list[Message]:
    """Compact only the dynamic portion."""
    
    static_prefix = messages[:cache_boundary]
    dynamic_suffix = messages[cache_boundary:]
    
    # Only compact what comes after the cache boundary
    compacted = compact(dynamic_suffix)
    
    return static_prefix + compacted
```

Sometimes you'll need to accept a cache miss for aggressive compaction. Track both metrics—cache hit rate and context utilization—to find the right balance for your use case.

## When to Compact

Two schools of thought:

**Proactive**: Monitor usage, compact before hitting the limit. Smooth experience, predictable behavior. But you might compact unnecessarily.

```python
async def maybe_compact(messages: list[Message], threshold: float = 0.8):
    usage = sum(count_tokens(m) for m in messages)
    if usage / max_input > threshold:
        return await compact_level_1(messages)
    return messages
```

**Reactive**: Wait for the API to reject your request, then compact. Never compacts unnecessarily. But you waste an API call on rejection and experience a latency spike.

```python
async def send_with_reactive_compact(messages: list[Message]):
    level = 0
    while level <= 3:
        try:
            return await api.send(messages)
        except ContextLimitExceeded:
            level += 1
            messages = await compact(messages, level=level)
    raise ContextUnrecoverable("Cannot fit even after max compaction")
```

**Recommended: hybrid approach.** Proactive monitoring at 85% with reactive fallback:

```python
async def send_message(messages: list[Message]):
    messages = await maybe_compact(messages, threshold=0.85)
    try:
        return await api.send(messages)
    except ContextLimitExceeded:
        messages = await emergency_compact(messages)
        return await api.send(messages)
```

Smooth experience with safety net for edge cases.

## Pitfalls

**Compacting too aggressively.** The agent forgets decisions, repeats work, contradicts itself. Before removing context, ask: "What would break if this disappeared?" If the answer is "the agent might redo this work," preserve it or summarize explicitly.

**Compacting too late.** Frequent API rejections, wasted tokens, latency spikes. Compact proactively at 80-85%, not 99%.

**Losing task-critical context.** The agent loses its goal, goes off-task, asks the user to repeat things. Fix this by tagging messages with priority:

```python
@dataclass
class Message:
    role: str
    content: str
    compactable: bool = True  # False for critical context
```

**Infinite compaction loops.** System compacts, retries, gets rejected, compacts more, forever. This happens when even maximum compaction can't fit the request—usually because the current message or required schemas are themselves too large. Detect the loop and fail gracefully:

```python
async def send_with_compact(messages: list[Message], max_attempts: int = 3):
    for attempt in range(max_attempts):
        try:
            return await api.send(messages)
        except ContextLimitExceeded:
            if attempt == max_attempts - 1:
                raise ContextUnrecoverable(
                    "Cannot fit after max compaction. "
                    "Consider breaking the task into smaller pieces."
                )
            messages = await escalate_compaction(messages, level=attempt + 1)
```

**Forgetting to re-inject.** After compaction, the agent seems confused despite having "enough" context. Compaction removed detail without restoring essential state. Treat compaction as two phases: remove, then restore.

## Implementation Checklist

<Callout type="note">
Building context management? Work through these in order.
</Callout>

1. **Measure first.** Implement accurate token counting before anything else. Accurate token counting is the foundation for everything else.

2. **Set clear budgets.** Document your model limit, output reserve, and safety buffer.

3. **Build progressive levels.** Microcompact → summarize → aggressive prune. Always try the gentlest option first.

4. **Tag message priority.** Mark messages as critical or compactable at creation time, not compaction time.

5. **Implement re-injection.** After any compaction, restore current task state, active files, and key decisions.

6. **Respect cache boundaries.** If using prompt caching, compact only the dynamic portion when possible.

7. **Add monitoring.** Track utilization over time. Alert when approaching limits.

8. **Test edge cases.** What happens at 100% utilization? After maximum compaction? With huge single messages?

9. **Fail gracefully.** When compaction can't save the request, provide a clear error and recovery path.

---

Context management is fundamentally about *controlled forgetting*. Unlike humans who forget involuntarily, your agent can choose what to remember and what to discard. This is a superpower—if you wield it carefully.

The key insight isn't any single technique. It's understanding that context is finite, that compaction is lossy, and that the decisions you make about what to forget directly shape what your agent can accomplish.

Get it wrong, and you'll spend your time re-explaining context that should never have been lost.

