# Introduction
## Agentic Engineering: How to Build AI Agents Like Claude Code

The most useful AI systems don't just answer questions — they take action. This book teaches you how to build them.

**What Are Agentic Systems?**

An agentic system is software that autonomously pursues goals through action. Unlike a chatbot that generates text responses, an agent executes commands, edits files, calls APIs, and makes decisions across multiple steps. It perceives its environment, reasons about what to do next, acts, observes the results, and continues until the task is complete.

Building these systems is fundamentally different from building traditional LLM applications. The challenges aren't about prompt engineering or model selection—they're about control flow, error handling, permission boundaries, context limits, and user trust. How do you let an AI edit code without destroying a codebase? How do you handle tools that fail? When do you compress conversation history without losing critical context? These are engineering problems, and they need engineering solutions.

**Why Claude Code?**

Most agentic AI tutorials show toy examples: a weather bot, a SQL query generator, a script that orders pizza. This book takes a different approach. We study Claude Code, a production coding agent used daily by working developers. It handles 40+ tools, manages gigabyte-scale file operations, enforces permission systems, recovers from failures, and operates within strict token budgets—all while maintaining a responsive terminal UI.

Claude Code isn't academic research. It's a deployed system that solves real problems under real constraints. The patterns in this book were forged under real production constraints.

**What You'll Learn**

This book distills Claude Code's architecture into 10 chapters covering the essential patterns of agentic systems:

1. **The Agent Loop** — The core control flow that drives autonomous behavior
2. **Prompt Architecture** — Structuring system prompts for reliability and control
3. **Tool Design** — Building callable functions that agents can use safely
4. **Permission Systems** — Letting agents act without giving them free rein
5. **Error Handling** — Recovering gracefully when tools fail or contexts break
6. **Context Management** — Staying within token limits across long conversations
7. **State Management** — Tracking conversation, tool results, and user preferences
8. **Sub-Agent Architecture** — Decomposing complex tasks through agent hierarchies
9. **Extensibility** — Plugin systems and third-party integrations
10. **Streaming UX** — Building responsive interfaces for long-running agents

**What This Book Is Not**

This is not a Claude Code manual — we extract *patterns*, not specifics. You won't learn how to configure Claude Code; you'll learn how to architect your own agentic systems using principles proven in production.

The emphasis is on concepts over code, and the concepts are model-agnostic: they apply whether you're building with Claude, GPT-4, or open-source models. The audience is builders, not researchers — we favor pragmatic solutions over theoretical purity.

**Structure**

The book is organized into five parts:

- **Part I: Foundations** (Chapters 1-3) — Core loop, prompts, and tools
- **Part II: Control** (Chapters 4-5) — Permissions and error handling
- **Part III: Scale** (Chapters 6-7) — Context and state management
- **Part IV: Composition** (Chapters 8-9) — Sub-agents and extensibility
- **Part V: Experience** (Chapter 10) — Streaming and UI

Each chapter stands alone, but they build on each other. Read sequentially for the full picture, or jump to specific chapters as needed.

Chapter 1 starts with the core structure every agent needs: the loop.
