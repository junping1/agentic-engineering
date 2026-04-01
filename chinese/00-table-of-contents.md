# 目录：代理工程
## 如何构建像 Claude Code 一样的 AI Agent

## 前言
- [引言 (Introduction)](00-introduction.md)

---

## 第一部分：核心基础 (Foundations)

### 第 1 章：Agent 循环 (The Agent Loop)
核心运行机制：消息累积、工具调度及终止条件。深入探讨为什么 Agent 的本质是持续运行的 while 循环，而非简单的单次函数调用。

### 第 2 章：提示词架构 (Prompt Architecture)
分层提示词构建、缓存感知设计、工具自描述与动态上下文注入。提示词应作为结构化程序进行设计，而非简单的文本字符串。

### 第 3 章：工具设计 (Tool Design)
Schema 优先的契约、工具生命周期、安全标记及结果设计。探讨工具在函数功能之外的系统定位。

---

## 第二部分：安全与控制 (Safety & Control)

### 第 4 章：权限系统 (Permission Systems)
纵深防御、多阶段验证、基于钩子（Hook）的扩展机制以及危险操作检测。如何构建安全且自主的代理。

### 第 5 章：错误处理与恢复 (Error Handling & Recovery)
重试策略、多级恢复机制、优雅降级及任务断点恢复。面向失败的设计艺术。

---

## 第三部分：规模化与效率 (Scale & Efficiency)

### 第 6 章：上下文管理 (Context Management)
Token 计数、渐进式压缩、信息保留优先级及提示词缓存。解决 Agent 系统中最具挑战性的资源平衡问题。

### 第 7 章：状态管理 (State Management)
外部存储模式、持久化策略、会话恢复及配置分层。理解为什么“状态”不等于单纯的对话历史。

---

## 第四部分：协作与扩展 (Composition)

### 第 8 章：子代理架构 (Sub-Agent Architecture)
隔离模式、委派模式、上下文传递及 Agent 特化。Agent 之间的协同作业与任务拆解。

### 第 9 章：基于协议的扩展性 (Extensibility via Protocols)
协议驱动的扩展、传输层抽象、工具探测及插件架构。实现无需修改核心代码的功能增强。

---

## 第五部分：用户体验 (User Experience)

### 第 10 章：流式输出与进度管理 (Streaming & Progress)
AsyncGenerator 模式、进度信息的原生化处理、并发执行以及通过透明度建立用户信任。构建响应式的 Agent 交互界面。

---

## 结尾
- [结语 (Conclusion)](11-conclusion.md)
