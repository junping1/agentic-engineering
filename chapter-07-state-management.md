# Chapter 7: State Management

State is not conversation history. Confusing the two is one of the most common mistakes in agentic systems—and one of the most costly to fix later.

Here's the scenario that exposes the problem: User closes terminal. Reopens an hour later. The conversation history is intact—every message, every tool call, every response. But the agent doesn't know which files it modified. Doesn't remember which permissions were granted. Has no idea about the three background tasks that were running. The user's preference for verbose output? Gone. The cached analysis of the codebase? Evaporated.

Messages capture *what was said*. State captures *what is true right now*. These are fundamentally different things.

## Why the Distinction Matters

Think of it like reading the transcript of a business meeting. The transcript tells you everything that was discussed: proposals made, arguments heard, decisions reached. But it doesn't tell you the *current state* of the company. For that, you need separate records—the org chart, the budget, the project status dashboard.

Messages are the transcript. Append-only. Never edited, only added to. They capture the conversation's flow and reasoning.

State is the dashboard. Mutable. Changes as the situation changes. It captures operational truth: what permissions exist, what processes are running, what the user has configured.

Both need to be tracked. But they require different management strategies. Messages go to the language model to maintain conversational context. State keeps your application functioning correctly. The model doesn't care that the user is running macOS with dark mode enabled. Your application absolutely does.

A simple litmus test: if you wiped the message history and started fresh, what information would you still need to preserve? That's state.

## What State Must Track

Agentic systems typically maintain several categories of state:

- **Tasks** — Background processes: test suites running, servers started, builds in progress. Their status, output, process IDs.
- **Permissions** — What the user has allowed or denied. "Yes, edit files in /src" needs to persist. So does "never run npm commands."
- **Settings** — User preferences that shape behavior. Model selection, verbosity, autonomy level.
- **Session** — Connection health. Authentication status, token expiration, online/offline state.
- **UI** — Interface state. Collapsed sections, scroll positions, which tool outputs are expanded.
- **Context** — Cached information about the environment. File contents read earlier, plan progress, validated assumptions.

The permission case makes this concrete. Without persisted permissions, your agent asks the same question every single time. "Can I edit files in /src?" Yes. *Edits file.* "Can I edit files in /src?" Yes. *Edits another file.* "Can I edit—" The user loses patience.

<Callout type="note">
Good permission state tracks denials too. If the user said no to something, asking again thirty seconds later isn't polite—it's harassment.
</Callout>

## The External Store Pattern

The naive implementation scatters state across modules. Permission tracking here. Task management there. Settings somewhere else. Each module maintains its own little state bucket.

This works until you need to persist state across restarts. Or synchronize between components. Or debug why the application thinks it has permission to do something the user never approved.

The better approach centralizes state in a dedicated store with subscriptions — similar to patterns in Redux or event-driven architectures. A database for your application's runtime state.

The principles:

**Single source of truth.** All state lives in one place. When you need to know the current truth, there's exactly one place to look—no hunting through module internals.

**Subscription-based updates.** Components register interest in specific slices of state. The task display subscribes to `state.tasks`. The permission checker subscribes to `state.permissions`. Changing settings doesn't cause the task display to re-render.

**Immutable updates.** Instead of `state.tasks[0].status = 'done'`, you create a new tasks array with the updated task. This makes changes trackable and, if you retain previous versions, rollback possible.

**Selector-based access.** Components access state through functions that extract exactly the slice they need. Clean interface. Easy optimization.

A minimal implementation:

```javascript
function createStore(initialState) {
  let state = initialState
  const listeners = new Map()
  
  function getState() {
    return state
  }
  
  function setState(updater) {
    const prev = state
    state = typeof updater === 'function' 
      ? updater(prev) 
      : { ...prev, ...updater }
    
    // Notify only affected listeners
    for (const [selector, callbacks] of listeners) {
      if (selector(prev) !== selector(state)) {
        callbacks.forEach(cb => cb(selector(state)))
      }
    }
  }
  
  function subscribe(selector, callback) {
    if (!listeners.has(selector)) {
      listeners.set(selector, new Set())
    }
    listeners.get(selector).add(callback)
    return () => listeners.get(selector).delete(callback)
  }
  
  return { getState, setState, subscribe }
}
```

The store doesn't know what's subscribing to it. Subscribers don't know who else is subscribed. State updates are explicit and traceable. When something goes wrong, you inspect the store and see exactly what's true.

## Persistence: Fire-and-Forget with Ordered Writes

Memory-only state disappears when the application closes. For an agentic system, that's usually unacceptable. Users expect to close their terminal, reopen it tomorrow, and pick up where they left off.

But naive persistence creates a performance problem. If every state change blocks on a disk write, the system becomes sluggish. The UI freezes while waiting for I/O.

The solution trades immediacy for throughput: **fire-and-forget with ordered writes**.

Fire-and-forget means you initiate the write and immediately continue. The write happens asynchronously. From the application's perspective, the state change is already done; persistence is catching up in the background.

```javascript
function updateState(changes) {
  const newState = { ...state, ...changes }
  state = newState
  persistState(newState)  // Non-blocking
  notifySubscribers(newState)
}
```

Ordered writes ensure that even though writes are asynchronous, they happen in sequence. Two rapid state changes queue two writes, and the second doesn't start until the first finishes. Without ordering, you risk race conditions where stale state overwrites fresh state.

```javascript
class WriteQueue {
  queue = []
  processing = false
  
  enqueue(writeFn) {
    this.queue.push(writeFn)
    this.processNext()
  }
  
  async processNext() {
    if (this.processing || this.queue.length === 0) return
    
    this.processing = true
    const writeFn = this.queue.shift()
    
    try {
      await writeFn()
    } catch (error) {
      // Log but don't crash—persistence failure
      // shouldn't break the application
      console.error('Persistence failed:', error)
    }
    
    this.processing = false
    this.processNext()
  }
}
```

<Callout type="warning">
The failure mode matters. If persistence fails, you log the error and continue. The application keeps working. The user might lose state if they close before a successful write, but that's better than crashing.
</Callout>

## The Transcript Pattern

For conversation messages specifically, there's a persistence approach that deserves separate attention: the append-only transcript.

Instead of storing current state, you store every message as it happens, in order, as a log file. Current state is derived by replaying the log. This is event sourcing for conversations.

```jsonl
{"type":"user","content":"Edit the README","ts":"..."}
{"type":"assistant","content":"I'll update the README...","ts":"..."}
{"type":"tool_use","tool":"file_edit","input":{...},"ts":"..."}
{"type":"tool_result","output":"File updated","ts":"..."}
{"type":"assistant","content":"Done!","ts":"..."}
```

**Recovery:** if the application crashes mid-conversation, replay the transcript and you're exactly where you were. **Debugging:** when something goes wrong, replay step-by-step to understand the failure. **Audit:** for sensitive applications, an immutable record of every action.

The downside: replay can be slow for long conversations. Production systems often combine transcripts with periodic snapshots—full state dumps that let you start replay from a recent point rather than the beginning.

## Session Restore

Let's walk through what happens when the user closes the terminal and reopens it later. This is the scenario that separates hobby projects from production systems.

The user is halfway through a complex task. Several permissions granted. Background processes running. The agent has cached context about the codebase. Then—intentionally or not—the terminal closes.

An hour later, they reopen it.

**Detect the previous session.** Check for session files. If they exist, you have something to potentially restore.

**Validate staleness.** How old is this session? Is it from the same project directory? A session from six months ago probably isn't worth restoring. One from two hours ago probably is.

**Handle interrupted turns.** This is subtle. If the user closed mid-response, you have an incomplete turn. The transcript might end with a tool call that was never completed. You need to detect this—and either discard the incomplete turn or mark it as interrupted and let the user decide what to do.

**Restore selectively.** Not all state deserves unconditional restoration:

| State Type | Restore Strategy |
|------------|------------------|
| Messages | Usually yes, but offer fresh start if stale |
| Permissions | Restore grants; maybe re-verify sensitive ones |
| Tasks | Check if processes still running (probably not) |
| File cache | Validate files haven't changed on disk |
| UI state | Restore positions and preferences |

**Communicate clearly.** Tell the user what happened. "Continuing session from 2 hours ago. 3 messages, 5 permissions restored." Give them the option to start fresh.

```javascript
async function restoreSession(sessionPath) {
  const session = await loadSession(sessionPath)
  if (!session) return { restored: false, reason: 'no_session' }
  
  const age = Date.now() - session.timestamp
  if (age > MAX_SESSION_AGE) {
    return { restored: false, reason: 'expired', session }
  }
  
  // Detect interrupted turn
  const last = session.messages.at(-1)
  if (last?.type === 'tool_use' && !last.completed) {
    session.interrupted = true
    session.messages.pop()
  }
  
  // Invalidate stale file cache
  for (const [path, cached] of session.fileCache) {
    const current = await getFileHash(path)
    if (current !== cached.hash) {
      session.fileCache.delete(path)
    }
  }
  
  return { restored: true, session }
}
```

## Configuration Hierarchy

Settings are a special kind of state because they come from multiple sources. Understanding the precedence rules is essential for predictable behavior.

```
Precedence (highest → lowest):
  CLI arguments          --model gpt-4
  Environment variables  AGENT_MODEL=gpt-4
  Local config           .agent/config.json (project)
  User config            ~/.config/agent/settings.json
  Managed config         Organization-pushed defaults
  Defaults               Hardcoded fallbacks
```

Higher wins. If the user passes `--model gpt-4` on the command line, that's what they want—override everything else. If they don't, check environment. If not set, check project config. And so on down the chain.

The rationale for this specific ordering:

**CLI arguments** are explicit and immediate. The user typed them right now, for this invocation.

**Environment variables** are explicit but more persistent. Often set in shell profiles or by deployment systems.

**Local config** captures project-specific needs. Different projects might require different models or permission profiles.

**User config** captures personal preferences that apply everywhere—default model, preferred verbosity, etc.

**Managed config** is for enterprise environments where organizations push baseline settings to all users.

**Defaults** apply when no other source provides a value.

```javascript
function resolveConfig(key) {
  if (cliArgs[key] !== undefined) return cliArgs[key]
  
  const envKey = `AGENT_${key.toUpperCase()}`
  if (process.env[envKey] !== undefined) return process.env[envKey]
  
  if (localConfig?.[key] !== undefined) return localConfig[key]
  if (userConfig?.[key] !== undefined) return userConfig[key]
  if (managedConfig?.[key] !== undefined) return managedConfig[key]
  
  return defaults[key]
}
```

<Callout type="note">
Some settings shouldn't be overridable. Security-critical configuration might be locked by managed config, ignoring higher-precedence sources. The hierarchy isn't always pure precedence—sometimes it's about who's *allowed* to set certain values.
</Callout>

## State Synchronization

So far we've discussed state in isolation. In practice, changes need to propagate in multiple directions.

**Notify dependents.** When state changes, interested components need to know. The subscription pattern handles this within a single process.

**Persist to disk.** Important changes should survive restarts. Fire-and-forget handles this without blocking.

**Sync with remote.** In multi-device scenarios, state might need to travel across machines. This is where things get complicated.

The hard part of remote sync is conflict resolution. The user changes a setting on their laptop, then changes it differently on their desktop, before either syncs. Now what?

Common strategies:

- **Last-write-wins** — Simple but lossy. Most recent change overwrites. Works for settings where there's no meaningful merge.
- **Merge** — For collections like permissions, combine changes. Device A grants X; device B grants Y; merged result grants both.
- **Conflict detection** — Flag conflicts for manual resolution. "You changed X on two devices. Which value?"
- **CRDTs** — Conflict-free replicated data types are mathematically guaranteed to merge correctly. Complex to implement but powerful.

For most agentic systems, remote sync is overkill. But if you're building for enterprise or multi-device use cases, plan for it early. Retrofitting sync onto a system designed for single-device use is painful.

## UI Integration

If your agent has a user interface (React, Ink for terminal, or similar), integrating state management requires care. Done poorly, every state change re-renders everything. Done well, only affected components update.

**Context providers** make state available throughout the component tree:

```jsx
const AgentStateContext = createContext(null)

function AgentStateProvider({ children }) {
  const store = useMemo(() => createStore(initialState), [])
  return (
    <AgentStateContext.Provider value={store}>
      {children}
    </AgentStateContext.Provider>
  )
}
```

**Custom hooks** provide efficient subscription:

```jsx
function useAgentState(selector) {
  const store = useContext(AgentStateContext)
  const [slice, setSlice] = useState(() => selector(store.getState()))
  
  useEffect(() => {
    return store.subscribe(selector, setSlice)
  }, [store, selector])
  
  return slice
}

// Only re-renders when tasks change
function TaskList() {
  const tasks = useAgentState(state => state.tasks)
  return tasks.map(task => <TaskItem key={task.id} task={task} />)
}
```

**Stable references** prevent unnecessary re-renders. Memoize callbacks passed to children:

```jsx
function TaskControls({ taskId }) {
  const store = useContext(AgentStateContext)
  
  const cancelTask = useCallback(() => {
    store.setState(state => ({
      tasks: state.tasks.filter(t => t.id !== taskId)
    }))
  }, [store, taskId])
  
  return <button onClick={cancelTask}>Cancel</button>
}
```

The goal is minimal re-rendering: update exactly the components whose data changed. Update exactly what changed, nothing more. In a complex agentic UI with real-time updates, sloppy state management creates visible lag. Good state management makes the interface feel instant.

## Putting It Together

Let's trace through a real scenario to see how these pieces interact.

User types: "Run the tests in the background and let me know when they finish."

The message gets added to conversation history. Before running tests, the agent creates a task entry in state:

```javascript
store.setState(state => ({
  tasks: [...state.tasks, {
    id: 'task-1',
    command: 'npm test',
    status: 'running',
    startedAt: Date.now()
  }]
}))
```

The store triggers fire-and-forget persistence—task state queued for disk. The TaskList component, subscribed to `state.tasks`, re-renders to show the running task.

Tests run asynchronously. When they complete:

```javascript
store.setState(state => ({
  tasks: state.tasks.map(t => 
    t.id === 'task-1' 
      ? { ...t, status: 'completed', output: testOutput }
      : t
  )
}))
```

Another persistence write. Another UI update showing results.

User closes terminal. Task state is already persisted.

User reopens later. System loads persisted state, sees the completed task, displays results—even though the actual process is long gone.

None of this lives in conversation messages. The messages know the user asked to run tests and the agent reported completion. But the operational reality—task lifecycle, timing, output—that's state.

---

When state management works, it's invisible. Permissions are remembered. Tasks are tracked. Settings persist. The application just *works*.

When it doesn't, users face constant friction. Re-granting permissions. Losing progress. Fighting with configuration that won't stick.

Messages are the conversation your users see. State is the operational truth that keeps the system functioning between those conversations.
