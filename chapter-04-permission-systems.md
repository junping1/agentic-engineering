# Chapter 4: Permission Systems

Would you give your intern root access on day one?

Of course not. You'd start them with read-only access to the codebase, maybe write access to a feature branch. You'd watch their first few pull requests carefully. As they proved themselves, you'd expand their permissions—commit rights, deployment access, eventually production credentials. Trust is earned incrementally.

Agents deserve the same caution — except they work faster, make decisions without pause, and never ask "wait, should I really do this?"

---

## The Six-Hour Outage

A startup gave their AI coding assistant full repository access. The agent was helpful—remarkably so. It refactored code, fixed bugs, updated dependencies. Then one Friday evening, while the developer was at dinner, the agent decided to "clean up" some configuration files it deemed redundant.

Among them was the production database connection string.

The outage lasted six hours. The postmortem identified the root cause in one line: *"The agent had permission to modify any file. We assumed it would know better."*

That assumption is where outages come from.

---

## The Difference Between Programs and Agents

When you write a traditional program, it does exactly what you coded. No more, no less. An agent is different. You give it a goal—"deploy this feature"—and it decides what actions to take. It might modify files, run commands, call APIs, access external services.

Consider what a typical coding agent can do:

- **File operations**: Create, edit, delete any file the process can access
- **Shell execution**: Run arbitrary commands with your user's permissions
- **API calls**: Make HTTP requests, potentially spending money or exposing data
- **Database access**: Query, modify, or drop tables
- **Git operations**: Commit, push, force-push, delete branches

Each of these can cause serious damage. A traditional program with access to these tools follows deterministic logic you can trace. An agent with access to them makes decisions based on learned patterns, contextual reasoning, and occasionally, confident hallucinations about what "should" happen next.

This isn't a reason to avoid agents—their flexibility is what makes them valuable. But it demands a different mental model for safety.

The old model was: *"I'll review every change before it runs."*

At agent speed, that's impossible. You can't review fifty file edits, thirty shell commands, and a dozen API calls for every task.

The new model must be: *"I'll define rules that let safe things happen automatically, block dangerous things always, and ask me about everything else."*

---

## Fail Safe, Not Fail Open

This is the fundamental principle. When uncertain, deny. When a rule doesn't match, ask. When the system is confused, stop.

The cost of over-blocking is a minor inconvenience. The cost of under-blocking can be catastrophic.

Every design decision in your permission system should bias toward caution. The agent will be blocked sometimes. That's acceptable. The alternative is the six-hour outage. Or worse.

---

## The Three-Tier Rule System

Every permission system needs to answer one question: *Should this action be allowed?*

The simplest useful model divides all actions into three categories:

- **Always Deny**: Operations that should never happen
- **Always Allow**: Operations that are pre-approved safe
- **Always Ask**: Operations that require human judgment

The critical insight is evaluation order. **Deny rules are checked first, always.**

Consider why:

```typescript
// A naive implementation (WRONG)
function checkPermission(action) {
  if (allowRules.match(action)) return ALLOW;
  if (denyRules.match(action)) return DENY;
  return ASK;
}
```

This seems reasonable until you encounter a conflict. What if an allow rule says "can modify files in /src" and a deny rule says "cannot modify .env files"? With the naive implementation, modifying `/src/.env` would be allowed—the allow rule matches first.

```typescript
// Correct implementation
function checkPermission(action) {
  if (denyRules.match(action)) return DENY;   // Safety takes precedence
  if (allowRules.match(action)) return ALLOW;
  return ASK;                                  // Uncertain = ask human
}
```

This pattern—deny takes precedence—appears everywhere in security. Firewall rules, OAuth scopes, Unix permissions. It exists because allowlists are easier to reason about than blocklists. When allow and deny rules conflict, safety must win. Deny-first evaluation ensures no allow rule can accidentally override a safety constraint.

### Pattern Matching for Rules

Rules need to match actions flexibly. Hard-coded exact matches won't scale:

```yaml
# Brittle: exact matches
allow:
  - "read /src/index.ts"
  - "read /src/utils.ts"
  - "read /src/helpers.ts"
  # ... hundreds more
```

Use glob patterns instead:

```yaml
# Flexible: patterns with wildcards
allow:
  - "read /src/**/*.ts"      # Any TypeScript file in /src
  - "write /src/generated/*" # Generated files only
  
deny:
  - "write **/.env*"         # Any .env file anywhere
  - "execute rm -rf *"       # Catastrophic deletions
  - "write /etc/**"          # System configuration
```

The `**` matches any number of directory levels; `*` matches within a single level. This mirrors patterns developers already know—gitignore, shell globs.

But patterns have limits. How do you express "can run git commands, but not force push"? You need semantic rules:

```yaml
deny:
  - tool: "bash"
    pattern: "git push --force*"
    reason: "Force push can destroy history"
    
  - tool: "bash"  
    pattern: "curl*|wget*"
    unless_path: "*.test.*"
    reason: "Network access only in tests"
```

The `unless_path` modifier shows how rules compose. This particular rule says "deny network commands, unless we're running tests." You'll evolve these rules as you discover edge cases—that's expected.

---

## The Multi-Stage Validation Pipeline

A single permission check isn't enough. Actions need validation at multiple stages, each catching different failure modes:

```
Tool Request Arrives
        │
        ▼
┌─────────────────────────┐
│  1. Input Validation    │ ─── Is the request well-formed?
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  2. Pre-Execution Hooks │ ─── Custom code can inspect/modify/reject
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  3. Rule Matching       │ ─── Deny → Allow → Ask
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  4. Permission Mode     │ ─── Auto / Interactive / Strict?
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  5. Tool-Specific Logic │ ─── The tool's own safety checks
└───────────┬─────────────┘
            │
            ▼
       Execute or Reject
```

**Stage 1: Input Validation** ensures the request is structurally valid. If an agent asks to write a file but provides no path, that's a malformed request—reject it before it reaches expensive checks.

**Stage 2: Pre-Execution Hooks** let you inject custom logic. Maybe you need to log every database query, or verify that file paths don't escape a sandbox. Hooks provide extensibility without modifying core pipeline code.

**Stage 3: Rule Matching** applies the three-tier system. Most requests terminate here with a clear ALLOW or DENY.

**Stage 4: Permission Mode** determines what happens when rules don't match. More on this shortly.

**Stage 5: Tool-Specific Logic** handles nuances that generic rules can't express. A file write tool might check disk space. A shell tool might validate that the command doesn't contain injection attacks. Each tool knows its own risks.

This layered approach follows the security principle of defense in depth. Any single layer might have gaps. Together, they're thorough.

---

## Permission Modes

Not every context requires the same safety level. Running an agent interactively, with a human watching, is different from running it in a CI pipeline at 3 AM.

### Interactive Mode

The default for human-supervised sessions. When no rule explicitly matches, the agent pauses and asks:

```
The agent wants to run: rm -rf node_modules/
This will delete 847 files.

[a]llow once  [A]llow always  [d]eny  [D]eny always  [?] help
```

This is the safest mode. Uncertain actions require human judgment. The downside is friction—you'll answer many prompts until you refine your rules.

### Auto Mode

For workflows that need less interruption. When no rule matches, an AI classifier estimates risk:

```typescript
async function autoModeDecision(action: Action): Promise<Decision> {
  const riskScore = await classifyRisk(action);
  
  if (riskScore > RISK_THRESHOLD) {
    return DENY;  // Too risky for auto-approval
  }
  
  return ALLOW;   // Low-risk, proceed
}
```

Auto mode is faster but carries risk. The classifier might misjudge. To mitigate this, auto mode typically has guardrails: operation budgets, denial tracking, automatic fallback to interactive mode for certain categories.

Think of auto mode like `sudo` with a timeout. After you authenticate once, subsequent commands don't prompt—but only for a limited time, and never for certain dangerous operations.

### Strict Mode

For high-security contexts. If no rule explicitly allows an action, it's denied. No asking, no AI classification:

```
Strict mode: Action requires explicit allow rule.
Denying: write /var/log/app.log
```

Strict mode inverts the default. Instead of "allow unless dangerous," it's "deny unless explicitly permitted." Appropriate for production deployments, sensitive codebases, or any context where surprises are unacceptable.

### Bypass Mode

For CI/CD pipelines where no human is available to respond. All non-denied actions are allowed:

```yaml
# CI configuration
agent:
  permission_mode: bypass
  deny_rules:
    - "write /etc/**"
    - "execute rm -rf *"
    - "api * --method DELETE"
```

<Callout type="warning">
**Bypass mode is dangerous.** It removes your safety net. If you use it, ensure your deny rules are comprehensive, your environment is sandboxed, and your blast radius is limited. Never use bypass mode with access to production data or infrastructure.
</Callout>

---

## Hook-Based Extensibility

Static rules can't handle every scenario. What if you need to:

- Log every action to an audit system
- Check actions against a dynamic policy server
- Transform requests before they execute
- Alert on unusual patterns

Hooks let you inject code at key points in the pipeline.

### PreToolUse Hooks

Execute before the tool runs. Can modify the request, approve, deny, or pass through:

```typescript
const auditHook: PreToolUseHook = async (request, context) => {
  await auditLog.record({
    tool: request.tool,
    input: request.input,
    user: context.user,
    timestamp: Date.now()
  });
  
  return { decision: 'passthrough' };  // Let other checks continue
};

const sandboxHook: PreToolUseHook = async (request, context) => {
  if (request.tool === 'file_write') {
    const path = request.input.path;
    if (!path.startsWith(context.sandboxRoot)) {
      return { 
        decision: 'deny',
        reason: 'Path escapes sandbox'
      };
    }
  }
  return { decision: 'passthrough' };
};
```

### PostToolUse Hooks

Execute after completion. Useful for logging results, triggering workflows, or detecting anomalies:

```typescript
const resultMonitor: PostToolUseHook = async (request, result) => {
  if (result.status === 'error' && result.error.includes('permission denied')) {
    await alerting.notify('escalation', {
      message: 'Tool hit permission boundary',
      request,
      result
    });
  }
};
```

### PermissionRequest Hooks

Intercept the permission decision itself. This is where you implement custom authorization:

```typescript
const teamPolicyHook: PermissionHook = async (action, defaultDecision) => {
  const teamRules = await policyServer.getRules(action.user.team);
  const teamDecision = teamRules.evaluate(action);
  
  if (teamDecision !== 'passthrough') {
    return teamDecision;  // Team policy overrides default
  }
  
  return defaultDecision;  // Fall back to standard rules
};
```

Hooks compose naturally. Register multiple hooks; they execute in order. Any hook can short-circuit with an explicit allow or deny, or pass through to the next. This is like middleware in web frameworks—a familiar pattern for developers.

---

## Dangerous Operation Detection

Some operations are inherently risky. Your permission system needs heuristics to identify them.

### Pattern Matching for Risky Commands

Certain command patterns should always raise flags:

```typescript
const DANGEROUS_PATTERNS = [
  /rm\s+(-rf?|--recursive).*\//,     // Recursive deletion
  />\s*\/dev\/(sda|null)/,           // Direct device writes
  /chmod\s+777/,                      // World-writable permissions
  /curl.*\|\s*bash/,                  // Piped execution
  /DROP\s+(TABLE|DATABASE)/i,         // Database destruction
  /:(){:|:&};:/,                       // Fork bombs
];

function containsDangerousPattern(command: string): boolean {
  return DANGEROUS_PATTERNS.some(pattern => pattern.test(command));
}
```

### Semantic Analysis

Patterns catch obvious cases, but obfuscation is easy:

```bash
# These all delete everything, but only one matches "rm -rf"
rm -rf /
find / -delete
perl -e 'unlink glob "/*"'
```

Semantic analysis asks: *What does this actually do?* You can implement this with another AI call (expensive but thorough) or with static analysis tools that understand command semantics.

```typescript
async function analyzeCommandSemantics(command: string): Promise<RiskAssessment> {
  const staticAnalysis = await analyzeWithShellcheck(command);
  const aiAnalysis = await llm.analyze({
    prompt: `What are the effects of this command? List files modified, 
             network access, and system changes: ${command}`
  });
  
  return combineAssessments(staticAnalysis, aiAnalysis);
}
```

### Path Constraints

Many dangerous operations become safe when confined to specific directories:

```yaml
sandbox:
  root: /home/user/project
  writable:
    - /home/user/project/src
    - /home/user/project/tests
  readable:
    - /home/user/project/**
    - /usr/share/doc/**
  executable:
    - /home/user/project/node_modules/.bin/*
```

This is analogous to how mobile apps request permissions for specific directories, or how containers isolate filesystem access. The agent can do anything—within its sandbox.

### Network Access Controls

Network operations deserve special scrutiny. An agent that can make arbitrary HTTP requests can exfiltrate data, access internal services, or spend money on paid APIs:

```typescript
const ALLOWED_HOSTS = new Set([
  'api.github.com',
  'registry.npmjs.org',
  'pypi.org'
]);

function validateNetworkRequest(url: string): Decision {
  const host = new URL(url).hostname;
  
  if (ALLOWED_HOSTS.has(host)) {
    return ALLOW;
  }
  
  if (host.endsWith('.internal.company.com')) {
    return DENY;  // Never access internal services
  }
  
  return ASK;     // Unknown host, human decides
}
```

---

## Human-in-the-Loop: The Final Safety Net

No automated system is perfect. Human judgment remains your ultimate backstop. The art is knowing when to invoke it.

### When to Interrupt

Interrupt the human for:

1. **First occurrence of a new action type.** The agent wants to run a command you've never seen. Even if rules would allow it, a first-time check catches rule gaps.

2. **Actions affecting critical resources.** Production databases, main branch, billing APIs. Even with allow rules, consider requiring human confirmation for high-impact operations.

3. **Chains of suspicious actions.** Any single action might be fine, but a sequence—query user data, then make network request, then delete logs—suggests something worth reviewing.

4. **Threshold breaches.** More than N file deletions, more than M API calls, more than X dollars spent. Absolute limits that require human acknowledgment to exceed.

### Permission Persistence

When a human approves or denies, capture the scope:

```
Allow this operation?

[a] Allow once         - just this instance
[A] Allow always       - add to permanent allow rules  
[s] Allow for session  - allow until agent restarts
[p] Allow for pattern  - allow this and similar operations
```

"Allow always" is convenient but dangerous—permissions accumulate over time. Consider periodic reviews: *"You've added 47 permanent allow rules this month. Review?"*

### Audit Trails

Every permission decision should be logged:

```typescript
interface AuditEntry {
  timestamp: Date;
  action: Action;
  decision: Decision;
  decisionSource: 'rule' | 'hook' | 'human' | 'auto';
  ruleName?: string;
  humanResponder?: string;
  context: {
    conversationId: string;
    taskDescription: string;
    precedingActions: Action[];
  };
}
```

Audit trails serve three purposes:

1. **Debugging.** When something goes wrong, you can trace exactly what happened and why.

2. **Rule refinement.** Patterns in human decisions suggest rules to add. If humans always approve "npm install," add it to allow rules.

3. **Accountability.** In regulated environments, you may need to demonstrate that actions were authorized. The audit trail is your evidence.

---

## Denial Tracking: Preventing Abuse

Auto mode's AI classifier is a potential attack surface. A malicious prompt might try variations of a dangerous command until one slips through:

```
Agent: Can I run `rm -rf /`?
System: Denied - destructive operation

Agent: What about `rm -r -f /`?
System: Denied - destructive operation

Agent: Let me try `find / -delete`
System: Denied - destructive operation

Agent: How about `perl -e 'unlink...'`?
System: ... (maybe this one isn't in the patterns?)
```

Denial tracking counters this by monitoring failed attempts:

```typescript
class DenialTracker {
  private denialCounts: Map<string, number> = new Map();
  private readonly threshold = 3;
  
  recordDenial(category: string): void {
    const count = (this.denialCounts.get(category) || 0) + 1;
    this.denialCounts.set(category, count);
    
    if (count >= this.threshold) {
      this.escalate(category);
    }
  }
  
  private escalate(category: string): void {
    // Force human review
    // Log potential adversarial behavior
    // Possibly terminate the session
  }
}
```

When an agent repeatedly hits denials in the same category—file deletion, network access, privilege escalation—the system escalates. It forces human review or terminates the session entirely.

This protects against both adversarial prompts and confused agents that keep trying doomed approaches. Either way, human attention is warranted.

---

## Putting It All Together

A production permission system integrates all these concepts:

```typescript
class PermissionSystem {
  private rules: RuleSet;
  private hooks: Hook[];
  private mode: PermissionMode;
  private tracker: DenialTracker;
  private auditLog: AuditLog;
  
  async checkPermission(request: ToolRequest): Promise<Decision> {
    // Stage 1: Input validation
    const validationResult = await this.validateInput(request);
    if (!validationResult.valid) {
      return this.deny(request, 'Invalid input', validationResult.errors);
    }
    
    // Stage 2: Pre-execution hooks
    for (const hook of this.hooks.filter(h => h.type === 'pre')) {
      const hookResult = await hook.execute(request);
      if (hookResult.decision !== 'passthrough') {
        return this.finalize(request, hookResult);
      }
    }
    
    // Stage 3: Rule matching
    const ruleResult = this.rules.evaluate(request);
    if (ruleResult.decision === 'deny') {
      this.tracker.recordDenial(ruleResult.category);
      return this.deny(request, ruleResult.reason);
    }
    if (ruleResult.decision === 'allow') {
      return this.allow(request, ruleResult.ruleName);
    }
    
    // Stage 4: Permission mode handling
    return this.handleUncertainAction(request);
  }
  
  private async handleUncertainAction(request: ToolRequest): Promise<Decision> {
    switch (this.mode) {
      case 'interactive':
        return this.askHuman(request);
      case 'auto':
        return this.autoDecide(request);
      case 'strict':
        return this.deny(request, 'No explicit allow rule in strict mode');
      case 'bypass':
        return this.allow(request, 'bypass mode');
    }
  }
}
```

This is simplified, but captures the structure. Real implementations add caching, async optimization, and more sophisticated rule languages—but the principles remain.

---

## The Permission Mindset

Building permission systems requires a specific mindset. You're not asking *"How do I let the agent do useful things?"* You're asking *"How do I prevent the agent from doing harmful things while still allowing useful ones?"*

The difference is subtle but important. The first mindset leads to allowlisting that expands over time. The second leads to carefully bounded capabilities, with explicit justification for each expansion.

**Start restrictive, loosen carefully.** It's easier to add permissions than to revoke them after something goes wrong.

**Log everything.** You can't improve what you can't measure. Comprehensive logging reveals patterns you'd never anticipate.

**Test adversarially.** Before deploying, try to trick your own system. What prompts could bypass your rules? What command variations aren't caught?

**Assume the agent will try things you didn't expect.** Not maliciously—agents extrapolate from training. If your rules don't cover a case, the agent will find it.

**Keep humans in the loop for irreversible actions.** Deletions, deployments, financial transactions. The convenience of automation isn't worth the risk of mistakes you can't undo.

---

The startup from our opening story learned these lessons through experience. Their fix wasn't just better rules — it was a change in assumptions. They stopped treating the agent as a trusted colleague and started treating it as a powerful tool that needs boundaries.

The agent is still remarkably helpful. It just asks before touching production config.