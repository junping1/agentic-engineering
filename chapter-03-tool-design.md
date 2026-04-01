# Chapter 3: Tool Design

Chapter 2 introduced tool descriptions as part of the prompt. Now we turn to tool design itself — the schemas, safety annotations, and lifecycle patterns that make tools reliable.

The gap is easy to miss. If you're used to writing functions, it's natural to think a tool is just a function with a different caller. And then their agent starts hallucinating parameters, misusing tools in loops, and burning through API costs because it doesn't understand what it's actually working with.

Here's the difference. A function is code that runs. A tool is code wrapped in everything the AI needs to use it correctly—validation, permissions, documentation, error handling. The function is for your runtime. The contract is for the model.

## The Anatomy of a Contract

```
Tool = {
  name           // what the LLM calls it
  description    // what the LLM reads to decide whether to use it
  schema         // what inputs are valid
  execute        // what actually runs
  permissions    // what's allowed
  render         // how results appear to users
}
```

Most of this gets sent to the model. The execute function doesn't. That asymmetry matters: the model decides what to call based on everything *except* the implementation. If your name is vague, your description unclear, or your schema ambiguous, the model guesses. And models guess wrong.

**Names** should be self-documenting. `searchCodeByPattern` tells the model what to do. `helper` tells it nothing. You'd never name a function `helper` in production code. Don't name a tool that way either.

**Descriptions** are documentation for inference time, not for humans reading source. Be precise. If the tool only works after authentication, say so. If it fails on files over 10MB, mention that. The model can't read your mind or your source code.

**Schemas** we'll cover next—they're too important for a paragraph.

**Permissions** determine whether execution is even allowed. Write access? User confirmation? Network access? These aren't details. They're the boundary between "the agent tried to do this" and "this actually happened."

**Render** separates "what happened" from "how it looks." The same result might render differently in a CLI versus a web UI. Keep this concern isolated.

## Schema-First Design

The schema *is* the documentation. When an LLM decides to use a tool, it doesn't see your code. It sees the name, description, and schema. The schema must be precise enough that the model can construct valid inputs without guessing.

Bad schema:

```
schema: {
  path: { type: string }
}
```

Good schema:

```
schema: {
  path: {
    type: string
    description: "Absolute path to the file. Relative paths will fail."
    required: true
  },
  encoding: {
    type: string
    description: "Text encoding (utf-8, ascii, latin-1)"
    default: "utf-8"
  },
  maxLines: {
    type: integer
    description: "Maximum lines to return. Use for large files to avoid context overflow."
    minimum: 1
    maximum: 10000
  }
}
```

Every field has a type. Every field explains when you'd use it. Constraints like `minimum` and `maximum` prevent the model from passing nonsensical values.

This is schema-first design: define the contract before you write the implementation. When the model passes `maxLines: "fifty"` instead of `maxLines: 50`, schema validation rejects it *before* your code runs. Your execute function can assume valid input.

<Callout type="info">
Schemas are self-enforcing documentation. Code comments can be ignored. Schemas cannot.
</Callout>

### Schema Evolution

Systems change. How do you evolve schemas without breaking callers?

**Adding optional fields is safe.** A new `startLine` parameter with a default value doesn't break existing calls.

**Removing fields breaks things.** If the model learned to use a parameter that no longer exists, calls fail. Deprecate first—mark the field deprecated in its description, keep accepting it, then remove it later.

**Changing semantics is treacherous.** If `path` used to accept relative paths and now requires absolute paths, every existing call breaks. The error won't be obvious from the schema.

When breaking changes are unavoidable, version your tools (`readFileV2`) or use aliases that adapt parameters internally.

## The Lifecycle

When an LLM requests a tool, here's what happens:

```
LLM requests tool
  → Validate input (does it match the schema?)
  → Check permissions (is this allowed?)
  → Execute (do the work)
  → Format result (prepare for return)
  → Return to LLM
```

Each stage exists for a reason. Validation exists so execute can assume valid input. Permission checks exist so security is centralized, not scattered. Formatting exists so you control what the model sees without complicating execution.

Respect these boundaries. Don't validate in execute. Don't check permissions in format. When you mix concerns, debugging becomes archaeology.

## Safety Annotations

In production, an agent that calls an unguarded API tool in a tight loop can easily burn hundreds of dollars in minutes — particularly with tools that trigger expensive downstream services. Tools carry metadata about their behavior precisely to prevent it.

**isReadOnly**: Does this tool only observe, never modify?

Read-only tools are inherently safer. The system can allow them without user confirmation. They can be cached—if nothing changed, why re-read?

**isConcurrencySafe**: Can multiple instances run simultaneously?

When an agent wants to perform multiple independent operations, the system can parallelize concurrency-safe tools. Mark a tool as safe when it isn't, and you get race conditions. Race conditions in agentic systems are hard to debug because the agent's reasoning is non-deterministic to begin with.

**isDestructive**: Is this operation irreversible?

Deleting a file is destructive. The system might require explicit confirmation, add extra logging, or prohibit destructive operations entirely in certain contexts.

<Callout type="warning">
When in doubt, mark it unsafe. False positives (marking safe tools as unsafe) cost performance. False negatives (marking unsafe tools as safe) cause bugs.
</Callout>

## Tool Composition

As your system grows, tools build on other tools.

### Tools That Call Tools

A "find and replace" naturally decomposes: read the file, transform content, write it back. Should this be one tool or three tools composed?

Both work. A monolithic tool is simpler for the model to use. Composed tools are more modular. If you allow composition, decide: when `findAndReplace` calls `readFile`, should permissions be checked again? Should the nested call be logged? Should it count against rate limits?

### Agent Tools

A useful pattern: tools that spawn sub-agents.

```
researchTool(query):
  subAgent = createAgent(tools=[webSearch, readPage])
  result = subAgent.run("Research: " + query)
  return summarize(result)
```

The sub-agent has its own conversation, tools, and lifecycle. It might fail. It might run for a long time. It might need permissions the parent doesn't have.

When designing agent tools, decide on isolation. Does the sub-agent share the parent's permissions? Its conversation history? Its working directory? These questions must be answered before problems surface in production.

### Wrapper Tools

Decorators for tools:

```
withRetry(tool, maxAttempts=3):
  return Tool({
    ...tool,
    execute: (input) => {
      for attempt in range(maxAttempts):
        try:
          return tool.execute(input)
        except TransientError:
          if attempt == maxAttempts - 1:
            raise
          sleep(exponentialBackoff(attempt))
    }
  })
```

Wrappers add cross-cutting concerns—logging, retry, caching—without modifying individual tools. But too many layers create debugging nightmares where actual behavior is buried under decoration.

## Result Design

What a tool returns matters as much as what it accepts.

### Structured vs. Raw

Two ways to return search results:

**Raw text:**
```
Found 3 matches in src/auth.js:
Line 42: function authenticate(user)
Line 87: if (!authenticate(token))
Line 103: return authenticate(admin)
```

**Structured:**
```json
{
  "matches": [
    { "file": "src/auth.js", "line": 42, "content": "function authenticate(user)" },
    { "file": "src/auth.js", "line": 87, "content": "if (!authenticate(token))" },
    { "file": "src/auth.js", "line": 103, "content": "return authenticate(admin)" }
  ],
  "totalMatches": 3
}
```

Structured wins. The model can work with it programmatically—"go to line 87" becomes unambiguous. Structured results validate cleanly and compose naturally.

Use raw text only when content is truly unstructured (prose, logs) or when human readability is the primary concern.

### Truncation

Tools often produce more output than is useful. A file read returns megabytes. A command produces endless logs. Flooding the context wastes tokens and confuses the agent.

Design for truncation:

```
readFile(path, maxBytes=100000):
  content = read(path)
  if length(content) > maxBytes:
    return {
      content: content[:maxBytes],
      truncated: true,
      totalSize: length(content),
      message: "Output truncated. Use offset to read more."
    }
  return { content: content, truncated: false }
```

Always tell the agent when you truncate. Include the full size. Provide guidance on how to get more.

### Progress Reporting

Some tools take minutes. A build runs. A download continues. Silence frustrates users and masks failures.

```
execute(input, reportProgress):
  for i, file in enumerate(files):
    reportProgress({
      current: i,
      total: len(files),
      message: f"Processing {file}..."
    })
    process(file)
```

Progress isn't just for users—it helps the system detect stuck operations and make timeout decisions.

### Error Results

Compare:

```
// Bad
Error: Operation failed

// Good
{
  "error": true,
  "code": "FILE_NOT_FOUND",
  "message": "File 'config.json' not found",
  "path": "/app/config.json",
  "suggestion": "Check '/app/settings/config.json'"
}
```

Good errors help agents recover. The code enables programmatic handling. The message explains what went wrong. The suggestion points toward fixes.

Never swallow errors. Never return vague failures. A clear error is infinitely more useful than a silent one.

## Deferred Tool Loading

Ten tools work fine. A hundred tools consume thousands of context tokens just describing what's available, leaving less room for actual conversation.

### The Tool Search Meta-Tool

Instead of loading everything upfront:

```
searchTools(query):
  description: "Find tools matching a description"
  execute: 
    return search(allTools, query)
```

The agent's initial toolset is small—basic operations plus `searchTools`. When it needs something specific, it searches, discovers the tools, and those get loaded.

This trades immediate capability for efficiency. The agent can't use every tool instantly but can discover what it needs.

### Contextual Tool Sets

Load tools based on context:

```
loadToolsForFile(path):
  if path.endsWith(".py"):
    return [pythonLinter, pytestRunner, pipInstall]
  if path.endsWith(".js"):
    return [eslint, jestRunner, npmInstall]
```

When the agent works on a Python file, Python tools appear. This works smoothly while keeping context lean.

### When to Defer

For critical tools—those the agent almost always needs—keep them always available. For specialized tools—database operations, deployment commands—deferred loading makes sense.

Start simple. Load all tools. When you hit context limits, implement deferral. Premature optimization wastes effort on a problem you might not have.

## Common Pitfalls

### Overly Broad Tools

A tool called `doEverything(action, params)` that dispatches internally is an anti-pattern. The model can't understand it—the description would need to document every action, making it uselessly long.

Design tools that do one thing well. `readFile`, `writeFile`, `deleteFile`. Not `fileOperation` with an `operation` parameter. The Unix principle applies: each tool should do one thing well, with structured inputs and outputs that compose cleanly.

### Missing Input Validation

"I'll validate in execute" means validation fails halfway through, with cryptic errors.

Validate exhaustively in the schema. Types, constraints, formats. Execute should assume valid input.

### Swallowing Errors

```
execute(input):
  try:
    return actualWork(input)
  except Exception:
    return { success: false }
```

This is useless to an agent. What failed? Why? What should it try instead?

Surface errors with detail. Error codes, messages, context, recovery suggestions.

### Tools That Can't Be Interrupted

A tool that runs for ten minutes with no way to stop is a UX disaster. A tool that modifies thousands of files with no abort mechanism is dangerous.

Design for interruption. Long-running tools should check for cancellation. Batch operations should be stoppable between items.

### Vague Names and Descriptions

The model reads the name and description to decide how to use the tool. `proc` or "Does processing" tells it nothing.

Write names and descriptions as if explaining to a colleague who's never seen the codebase. When in doubt, be verbose—better to spend tokens on clear descriptions than on failed tool calls.

---

Tools are where your agentic system touches reality. They're how the AI reads files, runs commands, queries databases, and modifies the world.

As you build, keep returning to one question: if a model reads this tool's contract, will it understand exactly what the tool does, how to use it correctly, and what to expect in response?

If yes, ship it. If no, iterate.
