# Chapter 5: Error Handling & Recovery

Your agent will fail. The question is whether it fails gracefully or catastrophically.

At 3am on a Tuesday, I got paged because one of our agents had been hammering the Claude API for six hours straight. The original request had timed out—a simple network blip—and our naive retry logic did exactly what we told it to: try again immediately. And again. And again. A thousand times per minute, for six hours. We burned through our rate limit, triggered abuse detection, and accomplished exactly nothing.

The fix took fifteen minutes. The lesson took longer to sink in.

## What Actually Goes Wrong

Failures in agentic systems cluster into distinct categories, and each demands a different response.

**Infrastructure failures**—network timeouts, connection resets, DNS hiccups—are transient. The request was fine; the pipes failed. Retry usually works.

**Rate limiting** isn't an error in the traditional sense. The API is telling you to slow down. Treating it as a failure and retrying immediately is exactly wrong.

**Invalid LLM responses** happen because language models are probabilistic. You'll get JSON with trailing commas, tool calls to nonexistent tools, truncated outputs. The model isn't broken — nondeterministic outputs are inherent to how language models work.

**Tool failures** occur when the agent's actions meet reality. Disk full. Permission denied. Unexpected API response. The agent asked for something reasonable; the world said no.

**Context overflow** is unique to LLM systems. Unlike traditional software, your agent has a hard ceiling on working memory. Exceed it and there's no graceful fallback—the request simply cannot proceed as-is.

The critical insight: a retry strategy that works for network errors will make rate limiting worse and do nothing for context overflow. You need to know *why* something failed before you can fix it.

## Retries Done Right

The simplest recovery is trying again. But simple doesn't mean naive.

### The Backoff Algorithm

Picture a server struggling under load. Now picture a hundred clients, all failing simultaneously, all retrying as fast as possible. You've just DDoS'd an already-struggling system.

```
function retryWithBackoff(operation, maxAttempts):
    baseDelay = 1 second
    maxDelay = 60 seconds
    
    for attempt in 1 to maxAttempts:
        result = try operation()
        if result.success:
            return result
        
        if not isRetryable(result.error):
            throw result.error
        
        delay = min(baseDelay * (2 ^ attempt), maxDelay)
        sleep(delay)
    
    throw MaxRetriesExceeded
```

First retry: 2 seconds. Second: 4 seconds. Third: 8. You're creating breathing room for the failing system while still making progress.

### The Thundering Herd

But there's a problem. Imagine a thousand clients hit the same network partition at the same moment. They all back off for 2 seconds. Then they all retry simultaneously. Another spike. They all back off for 4 seconds. Another synchronized retry.

You need jitter:

```
delay = min(baseDelay * (2 ^ attempt), maxDelay)
jitter = random(0, delay * 0.25)
sleep(delay + jitter)
```

That random offset spreads retries across time instead of concentrating them at predictable intervals.

### Knowing When to Retry

Not everything deserves a retry:

| Status | Meaning | Action |
|--------|---------|--------|
| 400 Bad Request | Your request is malformed | Fix it, don't retry |
| 401 Unauthorized | Auth failed | Refresh token, then retry |
| 404 Not Found | Resource doesn't exist | Don't retry |
| 429 Too Many Requests | Slow down | Retry after delay |
| 500 Server Error | Their problem | Retry with backoff |
| 503 Unavailable | Service down | Retry with backoff |
| 529 Overloaded (Anthropic-specific) | At capacity | Retry for user-facing work; give up for background tasks |

Connection resets are special—often caused by stale keep-alive connections. Disable connection reuse and retry once before escalating.

Context overflow can't be retried as-is. You need to transform the context first.

## The Escalation Ladder

When retries don't work, you need increasingly dramatic interventions.

**Level 1: Retry the same request.** Wait, try again, hope the transient issue resolved.

**Level 2: Modify the request.** Too many tokens? Truncate input. Model overloaded? Fall back to a different model. Request too complex? Break it apart.

**Level 3: Transform the context.** When accumulated state is the problem—not the immediate request—reshape that state. Compact conversation history. Summarize old tool results. Drop low-priority context.

**Level 4: Escalate to the user.** Some problems require human judgment: authentication that can't be auto-resolved, ambiguous situations needing clarification, destructive operations needing approval. This isn't failure — it's correct delegation.

**Level 5: Graceful failure with state preservation.** When all else fails, fail well. Save state so work isn't lost. Explain clearly what happened. Leave the system recoverable.

```
function executeWithRecovery(request, context):
    // Level 1: Simple retry
    for attempt in 1 to 3:
        result = try execute(request)
        if result.success:
            return result
        if not isRetryable(result.error):
            break
        sleep(exponentialBackoffWithJitter(attempt))
    
    // Level 2: Request modification
    if result.error.type == 'ContextOverflow':
        reducedRequest = truncateInput(request, 0.75)
        result = try execute(reducedRequest)
        if result.success:
            return result
    
    if result.error.type == 'ModelOverloaded':
        result = try execute(request, model: fallbackModel)
        if result.success:
            return result
    
    // Level 3: Context transformation
    if result.error.type == 'ContextOverflow':
        compactedContext = compactHistory(context)
        result = try execute(request, context: compactedContext)
        if result.success:
            return result
    
    // Level 4: User escalation
    if isUserPresent():
        userDecision = promptUser(
            "I encountered an error I can't automatically resolve: " +
            formatUserFriendlyError(result.error) +
            "\nWould you like me to try a different approach?"
        )
        return handleUserDecision(userDecision)
    
    // Level 5: Graceful failure
    saveState(context, request, result.error)
    throw GracefulFailure(
        message: "Unable to complete after multiple recovery attempts",
        state: savedStateId,
        suggestion: "Resume with: /resume " + savedStateId
    )
```

## Tool Error Containment

When a bash command fails, you have a choice: crash the agent loop or feed the failure back as information.

```
Tool: bash
Input: apt-get install nodejs
Result: {
    "success": false,
    "exit_code": 1,
    "stderr": "E: Could not open lock file - permission denied"
}
```

A fragile system throws an exception. A resilient system returns this as a tool result and lets the LLM reason about it: "Permission denied—I should try sudo or ask about alternative installation methods."

The implementation is minimal:

```
function executeTool(tool, input):
    try:
        result = tool.execute(input)
        return { role: 'tool_result', content: result, is_error: false }
    catch error:
        return { role: 'tool_result', content: formatToolError(error), is_error: true }
```

Errors don't crash the loop. They become part of the conversation.

## Mid-Task Recovery

Agents get interrupted. Process killed. Connection dropped. User closed laptop. When they resume: what was I doing?

If you persist the conversation to disk after each turn, recovery becomes possible. The key checks:

**Message chain integrity.** A transcript ending with assistant tool calls but no tool results indicates an interrupted turn.

**File state validation.** Were files being edited? Do they still exist? Have they been modified externally?

**Plan state restoration.** Multi-step plan in progress? Which steps completed? Which need retry?

```
function resumeSession(transcriptPath):
    transcript = loadTranscript(transcriptPath)
    
    lastMessage = transcript.messages.last()
    if lastMessage.role == 'assistant' and hasToolCalls(lastMessage):
        pendingTools = lastMessage.toolCalls
        
        if not hasMatchingToolResults(transcript, pendingTools):
            for tool in pendingTools:
                result = checkToolStateAndRecover(tool)
                transcript.append(result)
    
    for fileRef in extractFileReferences(transcript):
        if not fileExists(fileRef.path):
            transcript.append(systemMessage(
                "Note: " + fileRef.path + " no longer exists"
            ))
        else if fileModified(fileRef.path, since: fileRef.timestamp):
            transcript.append(systemMessage(
                "Note: " + fileRef.path + " modified externally"
            ))
    
    return transcript
```

The resumed agent won't pick up exactly where it left off, but it'll understand what happened and make informed decisions about how to proceed.

## Graceful Degradation

Sometimes the choice isn't success versus failure—it's partial success versus total failure.

An agent refactors 47 of 50 files before three fail with edge cases. Should it roll back everything? Or complete what it can and report what remains?

**Model fallback.** Primary model unavailable? Try a simpler one. Lower quality beats nothing.

```
function queryWithFallback(request):
    models = ['claude-sonnet-4-20250514', 'claude-3-5-haiku-20241022']
    
    for model in models:
        try:
            return query(request, model: model)
        catch error if error.type == 'ModelUnavailable':
            continue
    
    throw AllModelsUnavailable
```

**Feature degradation.** Caching broken? Disable it. Parallel execution causing issues? Fall back to sequential.

**Partial results.** Complete what you can. Report successes and failures separately. Let the user decide whether to retry failures or accept what was accomplished.

## Two Audiences for Errors

Every error has two audiences with different needs.

**Users** need clarity and actionability:

```
// Bad
"Error: ECONNRESET at TCP.onStreamRead (node:internal/stream:333:27)"

// Good
"I lost connection to Claude's servers. This usually resolves quickly.
Try again, or check your internet connection if the problem persists."
```

**Developers** need debugging context: stack traces, request IDs, timing, system state.

```
function handleError(error, context):
    log.error({
        message: error.message,
        stack: error.stack,
        requestId: context.requestId,
        timestamp: now(),
        systemState: captureSystemState()
    })
    
    return {
        userMessage: translateToUserFriendly(error),
        suggestion: getSuggestion(error),
        canRetry: isRetryable(error)
    }
```

Map technical errors to human explanations:

```
function translateToUserFriendly(error):
    if error.type == 'RateLimitExceeded':
        return "Too many requests. Please wait a moment."
    
    if error.type == 'ContextOverflow':
        return "Our conversation has grown too long. I'll summarize earlier parts."
    
    if error.type == 'AuthenticationFailed':
        return "Session expired. Run /login to reconnect."
    
    return "Something unexpected went wrong. Details logged for investigation."
```

## The Resilience Mindset

When building an agent, keep asking: "What happens when this fails?"

- API returns garbage?
- Tool takes 10x longer than expected?
- Context space exhausted?
- User closes tab mid-operation?
- File state assumptions wrong?

Each question needs an answer. Retry, escalate, save state and exit gracefully—all valid. "Crash and lose everything" never is.

The agents that earn trust handle adversity well. They say "I hit a problem, here's what I accomplished, here's how we proceed." They preserve work rather than losing it. They explain problems rather than hiding them. They degrade smoothly rather than failing catastrophically.

This isn't glamorous work. But it's the work that separates demos from products. But it's the work that separates demos from products—and reliability is what turns clever prototypes into tools people depend on.
