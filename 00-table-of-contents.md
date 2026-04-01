# Table of Contents
## Agentic Engineering: How to Build AI Agents Like Claude Code

## Front Matter
- [Introduction](00-introduction.md)

---

## Part I: Foundations

### Chapter 1: The Agent Loop
The core continuation pattern: message accumulation, tool dispatch, and termination conditions. Why agents are while-loops, not functions.

### Chapter 2: Prompt Architecture
Layered prompt construction, cache-aware design, tool self-description, and dynamic context injection. Prompts are programs, not strings.

### Chapter 3: Tool Design
Schema-first contracts, the tool lifecycle, safety annotations, and result design. Tools are more than functions.

---

## Part II: Safety & Control

### Chapter 4: Permission Systems
Defense in depth, multi-stage validation, hook-based extensibility, and dangerous operation detection. How to build safe autonomous agents.

### Chapter 5: Error Handling & Recovery
Retry strategies, multi-level recovery, graceful degradation, and mid-task recovery. Design for failure.

---

## Part III: Scale & Efficiency

### Chapter 6: Context Management
Token accounting, progressive compaction, information preservation priorities, and prompt caching. Managing the hardest problem in agentic systems.

### Chapter 7: State Management
External store patterns, persistence strategies, session restore, and configuration hierarchy. State is not conversation history.

---

## Part IV: Composition

### Chapter 8: Sub-Agent Architecture
Isolation modes, delegation patterns, context passing, and agent specialization. How agents coordinate with other agents.

### Chapter 9: Extensibility via Protocols
Protocol-based extension, transport abstraction, tool discovery, and plugin architecture. Making agents extensible without code changes.

---

## Part V: User Experience

### Chapter 10: Streaming & Progress
AsyncGenerator patterns, progress as first-class messages, concurrent execution, and trust through transparency. Building responsive agent interfaces.

---

## Back Matter
- [Conclusion](11-conclusion.md)
