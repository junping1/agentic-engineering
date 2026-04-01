# Chapter 10: Streaming & User Experience

Users don't trust what they can't see.

Picture this: you ask an agent to fix a bug. The cursor blinks. Nothing happens. Five seconds. Ten seconds. Thirty seconds. Is it thinking? Stuck? Crashed? You hover over Ctrl+C. At forty-five seconds, you give up and kill it.

The agent was fine. It had already found the bug, written a fix, and was halfway through running tests. All that work—gone.

This happens constantly. Not because agents are broken, but because they *feel* broken. An agent that works but provides no feedback is, for practical purposes, indistinguishable from one that's frozen. Users can't tell the difference. So they assume the worst.

This chapter is about building interfaces that earn trust. Streaming architecture. Progress reporting. The patterns that help users understand what's happening—and maintain control when they need to intervene.

## The Waiting Problem

Traditional software operates on human timescales. Click a button, something happens in milliseconds, see the result.

Agents shatter this model.

When an agent searches a codebase, reads fifteen files, runs a build, analyzes the output, and proposes a fix—that's not milliseconds. That's seconds to minutes. Sometimes much longer.

During those silent stretches, users experience a cascade of uncertainty. *Is it working? Did it crash? How much longer? Should I do something? Can I stop it?* Each unanswered question chips away at trust. After enough bad experiences, people stop using the agent entirely—not because it doesn't work, but because they can't tell whether it's working.

The solution is streaming: showing work as it happens rather than batching everything until the end.

Think about downloading a large file. You don't stare at a blank screen until bytes magically appear. You see a progress bar, bytes transferred, estimated time remaining. That feedback transforms frustration into comprehension.

Streaming can make agents feel faster even when they're actually slower. Research on perceived performance shows that users judge tasks with progress feedback as shorter than identical tasks without it. When you see an agent actively searching files, finding matches, analyzing results—you're engaged with the process. Time feels productive. Silence makes time drag.

This isn't deception. It's accurate representation. The agent *is* doing work during those thirty seconds. Streaming simply makes that work visible.

## The AsyncGenerator Pattern

Streaming architecture relies on a programming pattern that yields results incrementally rather than returning them all at once.

```
async function* runAgent(task):
    yield { type: 'status', message: 'Understanding task...' }
    
    plan = await analyzeTask(task)
    yield { type: 'plan', steps: plan.steps }
    
    for step in plan.steps:
        yield { type: 'step_start', step: step }
        
        async for progress in executeStep(step):
            yield { type: 'progress', ...progress }
        
        result = await step.getResult()
        yield { type: 'step_complete', step: step, result: result }
    
    yield { type: 'complete', summary: generateSummary() }
```

The principle is simple: *yield as you go*. Every time the agent has something meaningful to report—a decision made, progress on a tool, a result received—it yields immediately. The consumer processes these events incrementally, updating the UI in real-time.

This pattern has practical benefits.

**Natural backpressure.** If the consumer is slow (complex rendering, perhaps), the generator automatically slows down. No explicit flow control needed—it's built into the language construct.

**Composability.** Agent loops can yield from sub-generators. A tool running a long bash command yields output line by line, and those yields bubble up through the entire system.

**Testability.** Collect all yielded events in an array and assert on the sequence. Tests become declarative: given this input, the agent should yield these events in roughly this order.

**Cancellation.** Breaking out of a generator loop stops the iteration. A natural hook for user interrupts.

**Memory efficiency.** Yield and forget—you don't accumulate all results. An agent processing thousands of files doesn't hold them all in memory. It yields each one and moves on.

On the receiving end, consumption looks like this:

```
async function renderAgentExecution(task):
    ui = createAgentUI()
    
    try:
        async for event in runAgent(task):
            switch event.type:
                case 'status':
                    ui.setStatus(event.message)
                case 'progress':
                    ui.updateToolProgress(event.toolId, event.data)
                case 'step_complete':
                    ui.renderResult(event.result)
                case 'complete':
                    ui.showSummary(event.summary)
    except CancelledError:
        ui.showCancelled()
```

The consumer doesn't know or care how long the agent will run. It just processes events as they arrive.

## Progress as First-Class Messages

Early in my agent-building adventures, I treated progress as an afterthought—a debug log that happened to be visible. Mistake. Progress deserves to be a first-class message type with well-defined structure.

```
ProgressMessage = {
    type: 'progress',
    tool_id: string,        // Which tool is reporting
    timestamp: number,      // When this progress occurred
    data: ToolProgressData  // Tool-specific information
}
```

The `data` field varies by tool. Different operations have different natural progress indicators:

| Tool | Progress Data |
|------|---------------|
| **Bash commands** | Output lines, elapsed time, running/complete status |
| **Search operations** | Files searched, matches found, current file |
| **File operations** | Bytes read, total bytes, percentage |
| **Network requests** | Status code, response size, latency |

The critical principle: *stream progress immediately*. If a bash command outputs a line, yield it now. If a search finds a match, report it now. Users should see work happening in real-time, not in delayed batches.

This has a subtle but powerful effect on trust. When users see continuous activity—lines appearing, counters incrementing, files being processed—they understand the agent is working. Silence punctuated by sudden bursts of output feels erratic and unsettling.

## Concurrent Tool Execution

Sophisticated agents don't run tools sequentially. When the model requests multiple independent operations—read three files, search two directories—a well-designed agent runs them in parallel.

This creates a UX challenge: how do you display multiple in-progress tools simultaneously?

```
async function* executeToolsConcurrently(toolCalls):
    // Start all tools
    executions = [startTool(call) for call in toolCalls]
    
    // Yield progress from all tools as it arrives
    async for progress in mergeStreams(executions):
        yield progress
    
    // Collect results (may complete out of order)
    results = await Promise.all([e.result for e in executions])
    
    // Yield results in request order for determinism
    for i, result in enumerate(results):
        yield {
            type: 'tool_result',
            tool_id: toolCalls[i].id,
            result: result
        }
```

Three details matter here. First, the display must handle multiple concurrent progress indicators—stacked progress bars, a list of active operations, or a dashboard-style layout. Second, results may complete out of order, and that's fine; each tool reports completion independently. Third, yield results in *request order*, not completion order. This makes logs reproducible and testing deterministic.

## Tool-Owned Rendering

Here's a design principle worth highlighting: each tool should control how it displays.

The agent framework doesn't know how to render bash output versus search results versus file diffs. It shouldn't try. Instead, tools provide their own rendering logic:

```
BashTool = {
    name: 'bash',
    
    renderProgress(progress, options):
        if options.verbose:
            return <OutputStream lines={progress.output_lines} />
        else:
            return <Spinner text={`Running: ${progress.command}`} />
    
    renderResult(result, options):
        if result.exitCode != 0:
            return <ErrorPanel 
                title="Command failed"
                output={result.stderr}
                code={result.exitCode}
            />
        else if options.verbose:
            return <CodeBlock content={result.stdout} />
        else:
            return <SuccessBadge text="Command completed" />
}
```

Tools know what information matters—a file read tool knows that raw bytes are useless but line count and encoding are helpful. Users learn to recognize how different tools present themselves, building familiarity. Verbose and compact modes can adapt without central coordination. Tools define components; the framework handles layout.

## Interrupt Handling

Users need to stop agents. Maybe they asked the wrong question. Maybe the agent is heading in the wrong direction. Maybe they just need the terminal back for something urgent.

Interrupt handling seems simple but involves several considerations:

- **Which tools can be cancelled?** Streaming reads are naturally interruptible. Database commits mid-flight are atomic. Tools should declare their cancellation semantics.
- **Graceful vs. immediate.** Graceful says "stop when convenient"—finish the current file, complete the current test. Immediate says "stop now"—kill the process, close the connection. Users typically want graceful first, with immediate as an escalation (pressing Ctrl+C twice).
- **Preserving partial results.** If the agent read ten files before being cancelled, those reads should still be available. The agent yielded them already; the UI should keep them.

```
class AgentExecution:
    def __init__(self):
        self.cancel_requested = False
        self.abort_requested = False
    
    def requestCancel(self):
        self.cancel_requested = True
    
    def requestAbort(self):
        self.abort_requested = True
        for tool in self.running_tools:
            tool.abort()
    
    async def* run(self):
        while has_more_work and not self.cancel_requested:
            if self.abort_requested:
                yield { type: 'aborted' }
                return
            
            async for event in self.executeNextStep():
                yield event
                if self.abort_requested:
                    break
        
        if self.cancel_requested:
            yield { type: 'cancelled', partial_results: self.results }
        else:
            yield { type: 'complete', results: self.results }
```

The UI connects interrupt signals:

```
process.on('SIGINT', () => {
    if (execution.cancel_requested) {
        execution.requestAbort()   // Second Ctrl+C = abort
    } else {
        execution.requestCancel()  // First Ctrl+C = graceful cancel
    }
})
```

## Terminal UI Patterns

If you're building a CLI agent, the terminal is your canvas. Modern terminals support more than you might expect—colors, cursor movement, live updates.

### Spinners

The humble spinner is surprisingly effective. Continuous visual feedback that something is happening:

```
⠋ Searching codebase...
⠙ Searching codebase... (142 files)
⠹ Analyzing matches...
⠸ Generating fix...
```

Cycle through Unicode braille characters on a timer, updating the text as status changes.

### Progress Bars

For operations with known bounds:

```
Reading config files  [████████░░░░░░░░] 52%  7/15
```

Unicode blocks (█ ░) work reliably across terminals. Include both percentage and absolute numbers—different users find different representations intuitive.

### Streaming Output

For tools producing live output (builds, tests), display as it arrives:

```
$ npm test
 PASS  src/auth.test.js
 PASS  src/api.test.js
 FAIL  src/user.test.js
  ● User.create should validate email
    Expected: valid@email.com
    Received: undefined
```

This feels natural because it matches how terminals normally work. Users can read along and interrupt if something looks wrong.

### Scrolling Regions

When output exceeds terminal height, you have choices: natural scrolling (simple but loses context), fixed regions (keeps UI elements visible while content scrolls), or truncation with expansion.

```
┌─ Running tests ──────────────────────────────┐
│  PASS  src/auth.test.js                      │
│  PASS  src/api.test.js                       │
│  PASS  src/db.test.js                        │
│  ↓ 12 more lines (press Enter to expand)     │
└──────────────────────────────────────────────┘
[Ctrl+C to cancel]  Elapsed: 23s  Tests: 47/120
```

The fixed footer provides persistent status and controls.

### Layered Information Density

Different users need different levels of detail. Design your UI in layers:

1. **Status line** — What's happening right now? "Searching files..." or "Running tests..."
2. **Summary metrics** — Key numbers. "47/120 tests, 2 failures" or "142 files searched, 7 matches"
3. **Recent activity** — The last few events, scrolling. Enough to follow along without overwhelming.
4. **Full detail** — Expandable on demand. Complete logs, raw responses, all intermediate states.

Users naturally operate at the layer they need. Beginners watch the status line. Experts expand to full detail when debugging. The key is making each layer accessible without forcing users to wade through layers they don't need.

## Trust Through Transparency

All these patterns serve a deeper goal: building trust.

Trust matters because agents have significant capabilities. They read files, execute commands, modify code, interact with services. Users need to believe the agent is doing what they expect.

Consider the difference:

**Opaque:**
```
> Fix the authentication bug

[Working...]

Done! I've fixed the authentication bug.
```

**Transparent:**
```
> Fix the authentication bug

● Reading src/auth.ts (4.2 KB)
● Reading src/middleware/session.ts (2.1 KB)
● Searching for "authenticate" in src/ (found 12 matches)
● Running npm test -- --testPathPattern=auth
  ✓ login with valid credentials (142ms)
  ✓ login with invalid credentials (98ms)
  ✗ session refresh after expiry
    Expected: new token
    Received: 401 Unauthorized
● Found issue: session.refresh() not checking token validity
● Editing src/middleware/session.ts
  + Added token validation before refresh attempt
● Running npm test -- --testPathPattern=auth
  ✓ login with valid credentials (145ms)
  ✓ login with invalid credentials (96ms)
  ✓ session refresh after expiry (201ms)

Done! Fixed the authentication bug by adding token validation 
in session.refresh(). All auth tests now pass.
```

Same result. Vastly different trust level.

The transparent version shows its reasoning, lets you verify each step, and gives you opportunities to intervene if something looks wrong. You could have stopped it after seeing the test failure if you realized it was investigating the wrong code path. With the opaque version, you'd never know until it was too late.

Transparency means showing what the agent is doing (not just "working..." but "Reading src/auth.js"), explaining why it made decisions ("Found 7 matches. Analyzing auth.ts because it has the most references"), surfacing errors clearly (visually distinct, not buried in output), and letting users see under the hood through verbose modes and debug views.

## Common Pitfalls

<Callout type="warning">
**Silent hangs** — The cardinal sin. Nothing happening on screen while the agent works. Users assume it's broken.

*Fix:* Always show activity. If you have nothing to report, show elapsed time or a spinner at minimum.
</Callout>

<Callout type="warning">
**Information overload** — Every log line, every intermediate result floods the screen. Users can't find what matters.

*Fix:* Default to summaries. "47 files searched" not 47 file paths. Let users expand for detail.
</Callout>

<Callout type="warning">
**Buried errors** — An error occurred three screens ago. The agent continued. The user has no idea until the final result is mysteriously wrong.

*Fix:* Make errors visually distinct. Consider stopping on errors. Summarize them at the end.
</Callout>

<Callout type="warning">
**No interrupt mechanism** — The agent is doing something unwanted, but the user can't stop it.

*Fix:* Always handle Ctrl+C. Show that interrupts are available. Respond promptly.
</Callout>

<Callout type="warning">
**Lost context** — Long operations push earlier context off screen. When finished, users don't remember the starting point.

*Fix:* Keep key context visible: original task, overall progress, elapsed time.
</Callout>

<Callout type="warning">
**Unrecoverable states** — The agent hits an error and just... stops. No explanation, no way forward.

*Fix:* Every error should explain what happened and offer a path forward. "Test failed. Would you like me to investigate?" or "API rate limited. Retrying in 30 seconds."
</Callout>

## Bringing It Together

Good agent UX isn't any single technique. It's the combination of streaming architecture, well-structured messages, concurrent execution, tool-specific rendering, interrupt handling, and transparency. Each piece reinforces the others.

Think of your agent like an IDE.

When you run a build in your IDE, you don't see "Building..." then "Done." You see a build panel with streaming output, a progress indicator, the ability to cancel, errors highlighted and linked to source, and a summary when complete. The IDE doesn't hide the complexity of building software—it presents it in an organized, scannable, actionable way.

Your agent interface should aspire to the same standard. File searches should show matches as they're found. Test runs should stream results and highlight failures. Code edits should preview changes before committing.

When these elements work together, the agent stops feeling like a black box you invoke and wait for. It starts feeling like a collaborator you work alongside—one that shows its work, explains its thinking, and responds to your guidance.

That shift in perception separates agents people tolerate from agents people trust. And trust isn't optional — without it, even a perfectly functional agent gets killed at the forty-five second mark.

---

*That concludes the core patterns. In the final chapter, we'll step back and synthesize everything into a cohesive picture of agentic system design.*
