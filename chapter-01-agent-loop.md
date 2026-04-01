# Chapter 1: The Agent Loop

The default mental model for LLM usage is request-response: prompt in, completion out, done.

Building an agent requires a different model entirely.

An agent isn't a smarter chatbot. It's a different architecture—one where the model doesn't just *answer* but *acts*, observing results and deciding what to do next. The gap between "call an LLM" and "build an agent" is the gap between calling a function and writing a game loop.

This chapter teaches you that loop. Once you see it, "autonomous AI" reduces to something concrete: a while-loop with a language model inside.

---

## The Limits of Request-Response

You've wrapped an LLM in a nice API. A user types "Find all Python files that import `requests`, check for hardcoded URLs, and fix them." You send it to the model. The model responds with... a plan. Maybe some pseudocode. Perhaps even correct code snippets.

But nothing actually happened. No files were read. Nothing was fixed.

So you add tools—file reading, editing, shell commands. You pass them to the model. It responds with a tool call. Great! You execute it, and... now what? The model returned. The function ended. The user's task isn't done.

This is the wall. Request-response doesn't work for tasks with multiple steps, steps the model can't even predict in advance.

The fix isn't better prompting. It's a different architecture.

## The Core Insight

**An agent is a while-loop, not a function call.**

Request-response:
```
User: "Do X"
Assistant: "Here's X"
Done.
```

Agent loop:
```
User: "Do X"
Assistant: [calls list_files]
System: [returns files]
Assistant: [calls read_file]
System: [returns contents]  
Assistant: [calls edit_file]
System: [confirms edit]
Assistant: "Done—I fixed one hardcoded URL in file2.py."
```

The conversation **accumulates**. Each iteration adds the assistant's response plus tool results. The growing transcript becomes context for the next iteration, letting the model remember what it's done and reason about what comes next.

This is the continuation pattern: loop until a termination condition, carry state between iterations, let the model decide when it's finished.

## The Skeleton

Here's the entire pattern:

```
function run_agent(prompt, tools):
    messages = [{ role: "user", content: prompt }]
    
    while true:
        response = call_llm(messages, tools)
        
        if response.has_tool_calls:
            messages.append(response)
            messages.append(execute_tools(response.tool_calls))
        else:
            return response.text  // Done
```

That's it. Call the model with the full history. If it requests tools, execute them, append results, loop. If it responds without tool calls, it's finished.

The model itself decides when to continue and when to stop. You're not scripting steps—you're giving it agency.

<Callout type="info" title="Why Messages Must Accumulate">
Why keep all messages instead of just passing the latest results?

Without history, the model is amnesiac. It won't know what the user asked for, what it already tried, what failed, or what succeeded. An agent debugging a test needs to remember that it already fixed the import error before investigating the new failure.

The message history isn't a log—it's the substrate of reasoning.
</Callout>

---

## When Does It Stop?

The naive loop terminates when the model stops calling tools. Real systems need more:

```
while true:
    if turns >= max_turns:
        return error("Max turns exceeded")
    if cost >= budget:
        return error("Budget exhausted")  
    if interrupted():
        return error("User interrupted")
        
    response = call_llm(messages, tools)
    turns++
    cost += response.cost
    
    if response.error && !is_recoverable(response.error):
        return error(response.error)
        
    // ... handle response
```

**Five termination conditions:**

1. **Natural completion**—the model responds without tool calls. Usually means the task is done, though "I couldn't complete that" is also natural completion.

2. **User interrupt**—someone watching decides to stop. Agents should always be interruptible.

3. **Max turns**—prevents runaway loops. Simple tasks need 5-10 turns; complex projects might need hundreds.

4. **Budget exhaustion**—API calls cost money. Large codebases burn through tokens fast. Set caps to avoid surprises.

5. **Unrecoverable errors**—authentication expired, API down, fundamental failures. (Recoverable errors—like "file not found"—get converted to messages and fed back to the model.)

---

## Think Like a State Machine

Think of your agent as a **state machine**.

State at any moment:
- Message history
- Metadata (turn count, cost, files modified)
- External world (disk, databases, APIs)

Each iteration is a state transition. Current state + model response = next state.

```
function agent_step(state):
    response = call_llm(state.messages, state.tools)
    
    if response.has_tool_calls:
        results = execute_tools(response.tool_calls)
        return {
            messages: state.messages + [response] + results,
            turns: state.turns + 1,
            cost: state.cost + response.cost,
            done: false
        }
    else:
        return { ...state, result: response.text, done: true }
```

Treat these transitions as **immutable**. Don't mutate state in place. Why?

- **Debugging**: Inspect exactly what state existed at any step
- **Pause/resume**: Serialize state and pick up later
- **Branching**: Fork from any point to explore alternatives
- **Reproducibility**: Same state → same next state (given same model output)

<Callout type="warning" title="State ≠ Messages">
State is more than messages. You'll need:

```
state = {
    messages: [...],
    session_id: "abc123",
    turns: 7,
    tokens_used: 45000,
    files_modified: ["foo.py", "bar.py"],
    checkpoints: [...]
}
```

Theoretically you could reconstruct metadata from messages. In practice, that's fragile and expensive. Track it explicitly.
</Callout>

---

## Streaming and Eager Execution

Modern agents stream responses token-by-token for responsiveness. This creates an opportunity: why wait for the full response before executing tools?

```
stream = call_llm_streaming(messages, tools)
pending = []

for chunk in stream:
    if chunk.is_text:
        display(chunk.text)
    if chunk.is_tool_call_complete:
        // Start immediately
        pending.append(execute_async(chunk.tool_call))

results = await_all(pending)
```

If the model requests three file reads, you can start all three in parallel as each tool call is parsed. In practice, parallel reads can cut turn latency by half or more.

But concurrency is tricky:

- **Ordering**: "Write A, then read A" can't parallelize
- **Partial calls**: Don't execute until parameters are fully streamed
- **Cancellation**: User interrupts mid-stream—what happens to in-flight executions?

The safe approach: run read-only tools in parallel, serialize mutating tools.

---

## Failure Modes

These will bite you. Plan for them.

### Infinite Loops

Infinite loops are the most common failure. The model gets confused, keeps retrying a failing tool, or enters a cycle: check status → not ready → wait → check status → not ready → wait...

**Fix**: Always enforce max turns. *Always*.

### Recursive Explosion

Recursive explosion is subtler:

```
tools = {
    "ask_assistant": (question) => run_agent(question, tools)  // Recursion
}
```

Agent spawns agent spawns agent. You need limits at each level *and* a global limit.

### Context Overflow Mid-Task

The agent reads a large file (50K tokens). Reads another (50K). On turn three—context length exceeded. Task half-done, can't continue.

Strategies:
- **Summarization**: Compress old messages periodically
- **Truncation**: Drop oldest messages (lose history, free space)
- **Output limits**: Cap tool output before appending to context
- **Larger models**: Use extended-context models for complex tasks

### Lost State on Crash

Twenty minutes of work, then the process dies. Without persistence, you start over.

**Fix**: Checkpoint after each transition.

```
while true:
    save_checkpoint(state)
    response = call_llm(state.messages)
    state = apply_transition(state, response)
    save_checkpoint(state)
```

### Partial Execution

Three file writes requested. First succeeds. Second crashes mid-write. Third never runs. System is now inconsistent—and on restart, the agent doesn't know what happened.

Options:
- **Idempotent tools**: Running twice = running once
- **Transaction logging**: Record what executed before proceeding
- **Atomic batches**: All succeed or all roll back (hard for file operations)

---

## Mental Models

### The REPL

```
// Traditional REPL
while true:
    input = read()
    result = eval(input)
    print(result)

// Agent loop  
while true:
    response = call_llm(messages)   // "Read" what model wants
    results = execute(response)     // "Eval" by running tools
    messages.append(results)        // "Print" to context
```

In a REPL, humans decide the next action. In an agent loop, the model decides. Same structure.

### The Game Loop

```
while game_running:
    process_input()
    update_game_state()
    render_frame()
```

Games don't wait for the player's complete plan before updating. Each frame: process input, update world, render. Agent loops match this—process context, update world via tools, render results to context.

The game loop shows something else: **each iteration should be fast**. Slow frames cause lag. Slow turns kill user experience.

---

## Debugging

When an agent misbehaves, the state machine model is your friend.

1. **Dump messages**: The full history often reveals exactly where things went wrong. What did it see that caused the bad decision?

2. **Log transitions**: When turn 47 fails, inspect states 45, 46, 47 to trace how you got there.

3. **Replay from checkpoint**: Restore to an earlier state. Modify something. See if the outcome changes.

4. **Trace tool effects**: "Called write_file" tells you little. "Wrote 347 bytes to src/utils.py, replacing lines 12-15" tells you everything.

Debugging agents is harder than normal code—behavior is probabilistic. But state machine thinking gives you concrete snapshots to examine, regardless of *why* the model chose an action.

---

## Key Takeaways

The agent loop is deceptively simple:

1. **It's a while-loop.** Not request-response. Loop until termination.

2. **Messages accumulate.** The growing transcript is how agents "remember."

3. **The model decides.** When to use tools, when to stop—agency emerges from loop structure.

4. **Termination needs guardrails.** Max turns, budgets, interrupt handling, error recovery.

5. **Think in state transitions.** Immutable state enables debugging, persistence, branching.

6. **Streaming adds complexity.** Eager tool dispatch helps performance but demands careful concurrency handling.

7. **Failure is inevitable.** Infinite loops, context overflow, crashes, partial execution—design for all of them.

Nail this loop and the foundation is solid. Everything else—tools, prompts, permissions, multi-agent coordination—builds on this pattern.