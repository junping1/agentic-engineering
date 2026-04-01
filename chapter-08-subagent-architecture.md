# Chapter 8: Sub-Agent Architecture

*When a single agent isn't enough*

---

Your agent starts a refactoring task. It needs to search the codebase, create a plan, edit thirty files, and run the test suite. It tries to do all four at once. The context window fills up. By the time it reaches file 23, it's forgotten the plan. It starts making changes that contradict what it did ten files ago. The tests fail. The agent tries to fix them, but it can't remember what it was trying to accomplish in the first place.

This is what happens when you treat an agent like an infinite resource. It isn't. Every agent operates within constraints—context limits, attention degradation, the difficulty of juggling many concerns.

The solution is the same one humans discovered millennia ago: divide and conquer. A parent agent spawns *sub-agents* to handle portions of the work.

Managing agents is a lot like managing a team. Sometimes you delegate a quick question to a colleague. Sometimes you spin up a whole project with its own lead and timeline. The skill lies in knowing which approach fits which problem—and in giving each delegate exactly what they need to succeed.

## Why Delegate at All?

Before you reach for sub-agents, understand why they exist. Premature decomposition adds complexity without benefit. But when you hit certain walls, sub-agents become essential.

The first wall is **scale**. A task like "migrate this codebase from REST to GraphQL" isn't one task—it's dozens. Understanding the current architecture. Identifying every file that touches the API. Planning the migration order. Updating each endpoint. Writing integration tests. Verifying nothing broke. A single agent attempting this as one task will exhaust itself long before finishing.

The second wall is **specialization**. A general-purpose agent carrying every tool and permission is like a Swiss Army knife—versatile but optimal for nothing. A focused "code review agent" with read-only access and a prompt tuned for finding bugs will outperform a general agent that's also ready to edit files, run commands, and browse the web. Smaller tool sets mean fewer chances of selecting the wrong tool. Focused prompts keep attention on the task.

The third wall is **parallelism**. If you need to analyze fifty files for security vulnerabilities, a single agent must process them sequentially. Fifty specialized agents can analyze all files simultaneously, reducing wall-clock time dramatically.

The fourth wall is **containment**. Some operations are dangerous. Running arbitrary shell commands, modifying production databases, making API calls with side effects—these carry risk. By isolating dangerous operations in sub-agents with restricted permissions and separate environments, you create blast radius containment. If a sub-agent makes a mistake, the damage stays in its sandbox.

## Isolation Modes

When you spawn a sub-agent, the first question is: how isolated should it be from the parent?

This isn't purely technical. It reflects the trust level and coordination needs of the task. Too little isolation, and a rogue sub-agent can corrupt the parent's state. Too much, and the coordination overhead swamps any benefit.

Think of it as a spectrum:

```
Shared ◄────────────────────────────────────────────► Isolated

Default         Fork           Worktree          Remote
(in-process)    (subprocess)   (git worktree)    (different machine)
```

### Default: Same Process, Same State

The child runs in the same process, sees the same files, and shares the parent's environment. Fast and low-overhead—like calling a function. But no isolation whatsoever.

Use default mode when the task is quick, low-risk, and the child needs to see recent changes the parent made. An "explain" agent describing what a function does. A "search" agent finding files that match a pattern. These don't need protection from each other.

### Fork: Independent Conversation

The child gets a copy of relevant context but runs in a separate process with fresh message history. It's like forking a process — the child starts with a snapshot of the parent's state but runs independently from that point.

Use fork mode when the task might take a while, when you want the child's conversation kept separate, or when the child might fail in ways that shouldn't pollute the parent's state. A refactoring agent that will make many changes falls into this category—if it goes off track, you can kill it cleanly.

### Worktree: File System Isolation

Git worktrees allow multiple working directories for the same repository. A worktree-isolated sub-agent gets its own copy of the codebase. It can make changes, run tests, even break things—all without touching the parent's working directory.

Use worktree mode for potentially destructive file modifications, when you want to evaluate results before merging, or when multiple agents need to work on the same codebase simultaneously. An "experimental approach" agent trying a risky refactoring strategy is perfect for this—if it works, cherry-pick the changes; if not, delete the worktree.

### Remote: Different Machine

The most isolated mode. The sub-agent runs on a different machine entirely, connected via network. Complete isolation, but latency and complexity increase accordingly.

Use remote mode when you need a specific environment (different OS, specific hardware), when security requirements demand air-gapped execution, or when building distributed agent systems across multiple machines.

<Callout type="principle">
**Default to more isolation.** When uncertain, err toward fork or worktree over default. Shared state breeds subtle bugs. You can always loosen isolation later; tightening it after problems emerge is painful.
</Callout>

## Synchronous vs Asynchronous

Beyond *where* a sub-agent runs, you must decide *how* the parent waits for results.

**Synchronous execution** is the simple case. The parent spawns the child and blocks until it completes:

```
Parent                     Child
  │                          │
  │──── spawn ──────────────►│
  │                          │
  │         (working...)     │
  │                          │
  │◄─── result ─────────────│
  │                          │
  ▼ (continues)              
```

Like a function call. Use this when the task is quick, when the parent can't proceed without the result, or when you want predictable control flow.

**Asynchronous execution** is more powerful but more complex. The parent spawns the child and immediately continues other work:

```
Parent                     Child
  │                          │
  │──── spawn ──────────────►│
  │                          │
  │ (continues working)      │ (working independently)
  │                          │
  │     ◄── progress ────────│ 
  │                          │
  │ (still working)          │
  │                          │
  │     ◄── complete ────────│
  │                          │
  │──── read result ────────►│
  │◄─── result ─────────────│
```

Use this when the task takes a long time, when the parent has other work to do in parallel, or when spawning multiple children simultaneously. "Analyze these twenty files for security issues" becomes twenty parallel agents—the parent doesn't block, just orchestrates and collects results as each finishes.

The async pattern requires more coordination. You need unique IDs for each child. You poll for progress or wait for completion notifications. You handle the cases where children finish out of order, or some fail while others succeed.

## What Does the Child Know?

Context passing is a design decision with real tradeoffs.

The naive approach is **full context**—pass the entire parent conversation to the child. No information loss, but expensive in tokens. The child may be confused by irrelevant history. And for long conversations, you'll exceed context limits entirely.

The efficient approach is **summary context**—pass a compressed version of relevant background plus the specific task. Scales to longer conversations, but summarization loses details. The child might ask for information the parent already has.

The targeted approach is **task-specific context**—pass only what the child needs for its specific task:

```
Task: "Review the authentication changes in auth.py"

Context passed:
  - The contents of auth.py (current version and diff)
  - The overall goal ("migrating to JWT authentication")
  - Specific review instructions

Context NOT passed:
  - Earlier discussion about database schema
  - The parent's failed attempts at other approaches
  - Unrelated files the parent examined
```

Task-specific context is usually optimal. The child gets what it needs without noise. But it requires the parent to think carefully about what the child actually needs.

<Callout type="tip">
**Structure context for cache hits.** If many children receive the same base prompt and system instructions, those tokens can be cached and reused. Put the common parts first (system prompt, project description, conventions), then the task-specific portion last. Only the unique part incurs fresh processing.
</Callout>

## Agent Specialization

Different tasks call for different capabilities. Building specialized agent types—rather than one general-purpose agent that does everything—reduces errors through focus.

**Explore agents** are read-only analysts. They search, analyze, and answer questions. Their tools are limited to file reading, grep, code navigation. They can't break anything, so spawn them liberally.

- "What does this function do?"
- "Find all usages of the deprecated API"
- "Summarize the architecture of this module"

**Execute agents** are full-capability workers. They make changes, run commands, complete tasks. Their permissions match task requirements. They're more dangerous and should receive specific, bounded assignments.

- "Refactor this function to use the new API"
- "Write tests for this module"  
- "Fix this bug and verify the fix"

**Plan agents** are strategists. They create plans without executing them—mostly reading and reasoning. This separation prevents "ready, fire, aim" mistakes where an agent starts implementing before fully understanding the problem.

- "Plan the migration from MySQL to PostgreSQL"
- "Design the test strategy for this feature"
- "Create a step-by-step approach for this refactor"

**Verify agents** are quality checkers. They review work, find issues, suggest improvements. Read-only, maybe with permission to run tests. They provide a second opinion on what execute agents produced.

- "Review these changes for bugs"
- "Check if this implementation matches the specification"
- "Verify all tests pass after these changes"

The orchestration pattern that emerges:

```
                    ┌─────────────────┐
                    │  Parent Agent   │
                    │  (Orchestrator) │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Plan Agent    │───►│ Execute Agent │───►│ Verify Agent  │
│ (strategize)  │    │ (implement)   │    │ (check)       │
└───────────────┘    └───────────────┘    └───────────────┘
        │                    │                    │
        │                    ▼                    │
        │            ┌───────────────┐            │
        └───────────►│ Explore Agent │◄───────────┘
                     │ (investigate) │
                     └───────────────┘
```

Plan, execute, verify—with exploration supporting all phases.

## Collecting Results

Sub-agents produce results. Parents must collect and integrate them intelligently.

**Require structure.** Children return structured data, not just prose:

```json
{
  "status": "success",
  "task": "Review auth.py for security issues",
  "findings": [
    {
      "severity": "high",
      "line": 42,
      "issue": "SQL injection vulnerability",
      "suggestion": "Use parameterized queries"
    }
  ],
  "summary": "Found 1 high-severity issue, 3 low-severity issues"
}
```

Structured results let the parent programmatically decide next steps rather than parsing natural language.

**Expect failures.** Children fail. They hit errors, exceed limits, produce incorrect results. Good parents handle this gracefully:

- **Success** → use the result
- **Partial success** → use what's usable, note gaps
- **Failure with error** → log it, try an alternative approach
- **Timeout** → decide whether to wait longer or abort
- **Invalid result** → retry with clarified instructions

Never let a child failure crash the parent. Contain and handle.

**Aggregate deliberately.** When spawning multiple children in parallel, aggregate results systematically. If you spawn ten security scanners and one times out, you need to decide: retry that one, or proceed with partial coverage? The parent needs a policy, not improvisation.

## Task Lifecycle

Sub-agents have a lifecycle. Parents must manage it.

```
┌────────┐    ┌─────────┐    ┌───────────┐    ┌────────────┐
│ Spawn  │───►│ Running │───►│ Progress  │───►│ Complete   │
└────────┘    └─────────┘    │ (optional)│    │ or Failed  │
                   │         └───────────┘    └────────────┘
                   │                                
                   ▼                                
             ┌──────────┐                           
             │ Cancel   │ (parent-initiated abort)  
             └──────────┘                           
```

The parent's capabilities during this lifecycle:

- **Check status**: Is the child still running? How far along?
- **Read partial results**: Get intermediate output without waiting for completion
- **Cancel/abort**: Kill a child that's taking too long or going wrong
- **Wait for completion**: Block until finished, with optional timeout

For long-running children, the pattern looks like:

```python
agent_id = spawn_agent(
    task="Comprehensive security audit",
    mode="async"
)

while True:
    status = check_agent(agent_id)
    
    if status.state == "completed":
        return status.result
    elif status.state == "failed":
        return handle_failure(status.error)
    elif status.elapsed_time > MAX_ALLOWED_TIME:
        cancel_agent(agent_id)
        return handle_timeout()
    
    # Optionally process intermediate results
    if status.partial_results:
        process_intermediate(status.partial_results)
    
    wait(CHECK_INTERVAL)
```

## Common Patterns

Several coordination patterns emerge repeatedly. Understanding them gives you templates to start from.

### Map-Reduce

Spawn N children to process N items in parallel, then aggregate results.

```
           ┌────────────────────────────────────────────┐
           │               Parent Agent                 │
           │                                            │
           │  1. Split work into N chunks               │
           │  2. Spawn N children (map)                 │
           │  3. Collect N results (reduce)             │
           └─────────────────┬──────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
    ┌─────────┐         ┌─────────┐         ┌─────────┐
    │ Child 1 │         │ Child 2 │   ...   │ Child N │
    └────┬────┘         └────┬────┘         └────┬────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │   Aggregate    │
                    └────────────────┘
```

Classic example: analyzing many files for security vulnerabilities. Each child scans one file; the parent collects and prioritizes all findings.

### Pipeline

Child A's output becomes Child B's input. Work flows through stages.

```
    ┌─────────┐      ┌─────────┐      ┌─────────┐
    │ Stage A │─────►│ Stage B │─────►│ Stage C │
    │ (parse) │      │(analyze)│      │(report) │
    └─────────┘      └─────────┘      └─────────┘
         │                │                │
    Raw input       Structured       Final report
                       data
```

Use when work has natural stages that transform data. A code review pipeline might parse changes, analyze them for issues, then generate a formatted report.

### Supervisor

The parent monitors children, restarts failures, ensures completion.

```
    ┌─────────────────────────────────────────┐
    │            Supervisor Agent             │
    │                                         │
    │  - Monitor all children                 │
    │  - Detect failures                      │
    │  - Restart with backoff                 │
    │  - Redistribute work if needed          │
    └──────────────────┬──────────────────────┘
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │Worker A │     │Worker B │     │Worker C │
  │ (alive) │     │ (alive) │     │(restart)│ ◄── failed, restarted
  └─────────┘     └─────────┘     └─────────┘
```

Use when work must complete reliably despite individual failures. The supervisor's job is resilience: detecting problems and recovering from them.

### Swarm

Many agents work on shared state with loose coordination. Each agent observes the current state, identifies work to do, contributes changes, and avoids conflicts with others.

```
    ┌─────────────────────────────────────────┐
    │           Shared State / Goal           │
    └─────────────────────────────────────────┘
         ▲           ▲           ▲           ▲
         │           │           │           │
    ┌────┴───┐  ┌────┴───┐  ┌────┴───┐  ┌────┴───┐
    │Agent A │  │Agent B │  │Agent C │  │Agent D │
    └────────┘  └────────┘  └────────┘  └────────┘
```

Swarm coordination is tricky. It requires careful design of the shared state and conflict resolution. Use it for complex, unpredictable tasks where rigid coordination would be too constraining—but don't reach for it early.

## A Complete Example

Let's trace through a realistic scenario: "Migrate the codebase from REST to GraphQL."

**Phase 1: Planning.** The parent spawns a Plan Agent synchronously:

```
Task: "Create migration plan from REST to GraphQL"
```

The Plan Agent analyzes the codebase and returns structured phases:

```json
{
  "phases": [
    { "name": "Setup GraphQL server", "files": ["server.js"] },
    { "name": "Create schema", "files": ["schema.graphql"] },
    { "name": "Migrate endpoints", "files": ["api/*.js"], "parallel": true },
    { "name": "Update clients", "files": ["client/*.js"], "parallel": true },
    { "name": "Remove REST code", "files": ["legacy/*.js"] }
  ]
}
```

**Phase 2: Sequential setup.** The parent executes the first two phases with synchronous Execute Agents. These must happen in order—you can't create the schema before the server exists.

**Phase 3: Parallel migration.** The parent spawns Execute Agents asynchronously for each API file. Twenty files, twenty agents, running simultaneously. The parent collects results as each finishes.

**Phase 4: Verification.** The parent spawns a Verify Agent with the list of all changes made. The Verify Agent runs tests, checks schema validity, and reports issues.

**Phase 5: Iteration.** If verification finds issues, the parent spawns Execute Agents to fix them, then re-runs verification. If clean, proceed to the next phase.

This orchestration—planning, parallel execution, verification, iteration—emerges naturally from the sub-agent architecture. The parent stays oriented because it delegates details rather than trying to hold them all. Each sub-agent handles its piece, and the parent just coordinates.

## Principles

**Start simple.** Use a single agent until you hit its limits. Only decompose when you have a concrete reason.

**Define clear boundaries.** Each sub-agent should have a single, well-defined responsibility. If you can't explain what it does in one sentence, it's doing too much.

**Prefer isolation.** When in doubt, give sub-agents more isolation rather than less.

**Handle failures at every level.** Sub-agents fail. Networks fail. Context limits get exceeded. Build resilience into your design from the start.

**Make context passing explicit.** Don't assume sub-agents know what the parent knows. Pass exactly what they need—no more, no less.

**Log everything.** Multi-agent debugging is hard. Comprehensive logging of spawns, results, and failures is essential for understanding what went wrong.

---

## Summary

Sub-agent architecture is how single agents scale to complex problems. By spawning specialized children—isolated appropriately, running synchronously or asynchronously, with carefully scoped context—a parent agent orchestrates complex workflows that would overwhelm any single agent.

The key insights:

- **Isolation modes** trade coordination ease for safety—default through remote, pick what fits
- **Sync vs async** determines whether parents block or continue
- **Specialization** reduces errors by letting each agent focus on one kind of work
- **Patterns** like map-reduce, pipeline, supervisor, and swarm provide templates for common coordination needs

In Chapter 9, we'll explore extensibility—how protocol-based extension, transport abstraction, and plugin architecture let agents grow new capabilities without code changes.
