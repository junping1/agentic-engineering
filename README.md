# Agentic Engineering

**How to Build AI Agents Like Claude Code**

This book extracts production-tested architectural patterns from [Claude Code](https://claude.ai/code) and generalizes them for building any agentic AI system.

Whether you're creating a coding assistant, a research agent, or an automation system, you'll face the same fundamental challenges: How do I give my agent tools safely? How do I keep context manageable? How do I handle failures gracefully? This book answers those questions through patterns that have been battle-tested at scale.

> **[中文版点击这里 (Chinese version)](chinese/README.md)**

## Who This Book Is For

Developers who understand APIs and control flow but are new to building AI agents. No prior agent-building experience required — just basic programming knowledge and some familiarity with LLMs.

## How to Read

You can read sequentially to build a complete mental model, or jump to any chapter when you hit a specific design challenge. Each chapter introduces a concept, shows how Claude Code implements it, then distills the generalizable principle.

---

## Table of Contents

### Front Matter

- [Introduction](00-introduction.md) — Why this book exists and what you'll learn

### Part I: Foundations

| # | Chapter | Description |
|---|---------|-------------|
| 1 | [The Agent Loop](chapter-01-agent-loop.md) | The core continuation pattern: message accumulation, tool dispatch, and termination. Why agents are while-loops, not functions. |
| 2 | [Prompt Architecture](chapter-02-prompt-architecture.md) | Layered prompt construction, cache-aware design, and dynamic context injection. Prompts are programs, not strings. |
| 3 | [Tool Design](chapter-03-tool-design.md) | Schema-first contracts, the tool lifecycle, safety annotations, and result design. Tools are more than functions. |

### Part II: Safety & Control

| # | Chapter | Description |
|---|---------|-------------|
| 4 | [Permission Systems](chapter-04-permission-systems.md) | Defense in depth, multi-stage validation, hook-based extensibility, and dangerous operation detection. |
| 5 | [Error Handling & Recovery](chapter-05-error-handling.md) | Retry strategies, multi-level recovery, graceful degradation, and mid-task recovery. Design for failure. |

### Part III: Scale & Efficiency

| # | Chapter | Description |
|---|---------|-------------|
| 6 | [Context Management](chapter-06-context-management.md) | Token accounting, progressive compaction, information preservation, and prompt caching. The hardest problem in agentic systems. |
| 7 | [State Management](chapter-07-state-management.md) | External store patterns, persistence strategies, session restore, and configuration hierarchy. State is not conversation history. |

### Part IV: Composition

| # | Chapter | Description |
|---|---------|-------------|
| 8 | [Sub-Agent Architecture](chapter-08-subagent-architecture.md) | Isolation modes, delegation patterns, context passing, and agent specialization. How agents coordinate. |
| 9 | [Extensibility via Protocols](chapter-09-extensibility.md) | Protocol-based extension, transport abstraction, tool discovery, and plugin architecture. |

### Part V: User Experience

| # | Chapter | Description |
|---|---------|-------------|
| 10 | [Streaming & Progress](chapter-10-streaming-ux.md) | AsyncGenerator patterns, progress as first-class messages, concurrent execution, and trust through transparency. |

### Back Matter

- [Conclusion](11-conclusion.md) — Synthesis and next steps

---

## About

~30,000 words. Concepts over code. Patterns over implementation. Principles that outlast any specific framework.

## License

Content is for educational purposes. Claude Code is a trademark of Anthropic.
