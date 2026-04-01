# 第 3 章：工具设计

第二章介绍了工具描述作为提示词的组成部分。现在我们转向工具设计本身——Schema、安全标注和生命周期模式，这些让工具变得可靠。

这个差距容易被忽略。如果你习惯了编写函数，很自然会认为工具不过是换了一个调用者的函数。然而实际上，智能体可能会编造（Hallucinate）参数、在循环中滥用工具，或因不理解工具的运作方式而造成高昂的 API 成本。

这就是本质区别：函数是运行的代码，而工具则是封装了校验、权限、文档和错误处理等要素的完整契约，确保 AI 能正确调用。函数面向运行时（Runtime），而契约则面向模型。

## 契约的构成

```typescript
工具 = {
  name: string,         // LLM 调用时的名称
  description: string,  // LLM 决定是否使用该工具的参考说明
  schema: ZodSchema,    // 输入参数的校验规范
  execute: Function,    // 实际运行的代码
  permissions: Rule[],  // 允许执行的操作范围
  render: Component,    // 结果的呈现方式
}
```

在上述属性中，除了执行函数（execute）外，大部分信息都会发送给模型。这种不对称性至关重要：模型是基于实现细节之外的元数据来决定调用行为的。如果名称模糊、描述不清或模式（schema）存在歧义，模型就会产生误判或猜测，而预测往往会导致错误。

**名称（Name）** 应当具有自解释性。例如，`searchCodeByPattern` 能直接告知模型其功能，而 `helper` 则无法提供有效信息。正如你不会在生产代码中滥用无意义的函数名，在设计工具时也应保持严谨。

**描述（Description）** 是供模型在推理时参考的文档。务必保持精确：如果该工具依赖特定的身份验证，或对处理的文件大小有上限（如 10MB），请明确注明。模型无法预知你的设计意图，也无法直接阅读你的源码。

**模式（Schema）** 是工具设计的核心，我们将在下一节详细讨论。

**权限（Permissions）** 决定了操作的合法性。无论是写权限、用户确认还是网络访问，权限控制是划定智能体行为边界的红线。

**渲染（Render）** 实现了逻辑与表现的分离。同样的工具输出，在 CLI 和 Web UI 中可能有截然不同的呈现方式。解耦渲染逻辑能显著提升工具的适配性。

## 模式优先设计 (Schema-First Design)

模式即文档。由于 LLM 在决策时无法查看代码实现，它完全依赖名称、描述和模式。因此，模式必须足够精确，确保模型能够构建有效的输入。

糟糕的模式示例：

```javascript
schema: {
  path: { type: string }
}
```

优秀的模式示例：

```javascript
schema: {
  path: {
    type: string,
    description: "文件的绝对路径。不支持相对路径。",
    required: true
  },
  encoding: {
    type: string,
    description: "文本编码格式 (utf-8, ascii, latin-1)",
    default: "utf-8"
  },
  maxLines: {
    type: integer,
    description: "返回的最大行数。建议大文件设置此参数以避免上下文溢出。",
    minimum: 1,
    maximum: 10000
  }
}
```

通过详细定义字段类型、使用场景以及 `minimum` / `maximum` 等约束，可以有效防止模型传入无效值。

这就是“模式优先设计”：在编写业务逻辑之前先定义契约。如果模型尝试传递 `maxLines: "fifty"` 而非数字 `50`，模式校验会在代码运行**之前**拦截请求。这样，执行函数即可安全地假设输入数据始终合法。

<Callout type="info">
模式是具有约束力的文档。代码注释可能会被忽略，但模式校验是强制性的。
</Callout>

### 模式的演进

随着系统的迭代，如何在保证兼容性的背景下演进模式？

*   **添加可选字段是安全的**：增加带有默认值的参数（如 `startLine`）不会破坏现有的调用逻辑。
*   **删除字段往往会导致破坏**：如果模型已习惯使用某个参数，删除它将导致调用失败。建议先进行弃用（Deprecate）处理——在描述中标记字段已废弃，维持接收逻辑，稍后再彻底移除。
*   **更改语义极具风险**：例如将 `path` 从支持相对路径改为仅支持绝对路径。这种错误在模式层面难以察觉，却会直接导致运行异常。

当必须进行破坏性变更时，建议对工具进行版本化（如 `readFileV2`），或在内部提供参数映射转换。

## 生命周期

当 LLM 请求调用工具时，标准流程如下：

1.  **校验输入**：输入是否符合模式规范？
2.  **检查权限**：该操作是否获得授权？
3.  **执行逻辑**：实际运行业务代码。
4.  **格式化结果**：处理输出，为模型返回做准备。
5.  **反馈模型**：将结果传递回 LLM。

每个阶段职责明确：校验确保了执行逻辑的健壮性；权限检查实现了安全逻辑的集中管理；格式化则允许你在不增加业务复杂度的前提下，精准控制模型获取的信息量。

请严格遵守这些边界。避免在执行阶段处理校验，或在格式化阶段检查权限。一旦职责混淆，后续的调试与维护将变得异常困难。

## 安全标注 (Safety Annotations)

在生产环境中，智能体在紧密循环中调用未加防护的 API 工具，可以在几分钟内轻松烧掉数百美元——尤其是那些会触发昂贵下游服务的工具。工具携带关于其行为的元数据，正是为了防止此类问题的发生。

**isReadOnly (是否只读)**：该工具是否仅进行观察而不做任何修改？

只读工具天生更安全。系统可以在无需用户确认的情况下允许它们运行。它们也可以被缓存——如果内容没有变化，何必重新读取？

**isConcurrencySafe (是否并发安全)**：是否可以同时运行多个实例？

当智能体想要执行多个独立的并行操作时，系统可以并行化处理具有并发安全标识的工具。如果将不安全的工具标记为安全，会导致竞态条件（Race Conditions）。在智能体系统中，竞态条件极难调试，因为智能体的推理过程本身就具有非确定性。

**isDestructive (是否具有破坏性)**：该操作是否不可逆？

删除文件是具有破坏性的。系统可能需要显式确认、增加额外日志，或在某些上下文中完全禁止破坏性操作。

<Callout type="warning">
如有疑问，请将其标记为不安全。将安全工具误标为不安全（假阳性）只会损失性能，但将不安全工具误标为安全（假阴性）则会导致 Bug。
</Callout>

## 工具组合 (Tool Composition)

随着系统的增长，工具之间会产生依赖和嵌套。

### 工具调用工具

一个“查找并替换”工具很自然地可以拆解为：读取文件、转换内容、写回文件。这应该是一个工具，还是三个工具的组合？

两者皆可。单一的（Monolithic）工具对模型来说使用更简单；组合式工具则更具模块化。如果你允许组合，需要决定：当 `findAndReplace` 调用 `readFile` 时，是否需要再次检查权限？嵌套调用是否应该记录日志？是否应该计入速率限制（Rate Limits）？

### 智能体工具 (Agent Tools)

一种非常有用的模式是：由工具衍生出子智能体。

```javascript
researchTool(query):
  subAgent = createAgent(tools=[webSearch, readPage])
  result = subAgent.run("调研主题: " + query)
  return summarize(result)
```

子智能体拥有自己的对话上下文、工具库和生命周期。它可能会失败，可能会运行很长时间，也可能需要父智能体不具备的权限。

在设计智能体工具时，需要考虑隔离性：子智能体是否共享父智能体的权限？是否共享对话历史？是否共享工作目录？这些问题必须在生产环境暴露问题之前得到明确。

### 包装器工具 (Wrapper Tools)

工具的装饰器：

```javascript
withRetry(tool, maxAttempts=3):
  return Tool({
    ...tool,
    execute: (input) => {
      for (let attempt = 0; attempt < maxAttempts; attempt++) {
        try {
          return tool.execute(input)
        } catch (error) {
          if (isTransient(error) && attempt < maxAttempts - 1) {
            sleep(exponentialBackoff(attempt))
            continue
          }
          throw error
        }
      }
    }
  })
```

包装器可以在不修改单个工具的情况下增加横切关注点（Cross-cutting concerns），如日志、重试或缓存。但过多的渲染层级会导致调试噩梦，实际的行为会被层层装饰掩盖。

## 结果设计 (Result Design)

工具返回的内容与它接收的内容同样重要。

### 结构化 vs. 原始文本

返回搜索结果的两种方式：

**原始文本：**
```
在 src/auth.js 中找到 3 处匹配：
第 42 行: function authenticate(user)
第 87 行: if (!authenticate(token))
第 103 行: return authenticate(admin)
```

**结构化数据：**
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

**结构化数据完胜**。模型可以编程化地处理这些信息，例如“跳转到第 87 行”等指令将变得清晰无误。此外，结构化结果更易于校验，也方便进行链式组合。

只有当内容确实属于非结构化数据（如纯文本段落、原始日志），或首要目标是提供给人类阅读时，才考虑使用原始文本。

### 截断处理 (Truncation)

工具输出的内容往往会超出模型处理的上下文限制。例如，读取一个大文件可能返回数 MB 数据，或者一个命令会产生海量的日志。将这些信息全部塞入上下文不仅浪费 Token，还会严重干扰智能体的判断。

设计截断机制势在必行：

```javascript
readFile(path, maxBytes=100000):
  content = read(path)
  if (content.length > maxBytes) {
    return {
      content: content.slice(0, maxBytes),
      truncated: true,
      totalSize: content.length,
      message: "输出已截断。请使用偏移量 (offset) 参数读取后续内容。"
    }
  }
  return { content, truncated: false }
```

在触发截断时，务必明确告知智能体，并提供完整数据的大小信息以及获取剩余部分的指引。

### 进度报告 (Progress Reporting)

某些工具（如代码构建或文件下载）可能需要运行较长时间。长时间的“静默”会导致用户体验下降，甚至让人误以为程序已崩溃。

```javascript
execute(input, reportProgress):
  for (let i = 0; i < files.length; i++) {
    reportProgress({
      current: i,
      total: files.length,
      message: `正在处理 ${files[i]}...`
    })
    process(files[i])
  }
```

进度报告不仅面向用户，还能帮助系统监控任务状态，从而做出更合理的超时决策。

### 错误处理 (Error Results)

对比以下两种错误反馈方式：

**糟糕的示例：**
`Error: 操作失败`

**优秀的示例：**
```json
{
  "error": true,
  "code": "FILE_NOT_FOUND",
  "message": "找不到文件 'config.json'",
  "path": "/app/config.json",
  "suggestion": "请检查 '/app/settings/config.json' 是否存在"
}
```

详尽的错误信息有助于智能体进行自我修复。错误码支持编程化逻辑处理，明确的消息解释了失败原因，而修复建议则指明了下一步动作。

永远不要通过空返回来“吞掉”错误，更不要返回含糊不清的失败信息。清晰的错误反馈比无声的失败更有价值。

## 延迟加载工具 (Deferred Tool Loading)

当工具数量从十个增加到一百个时，仅仅在初次加载时描述所有功能就会消耗大量的上下文 Token，挤占有意义的对话空间。

### “元工具”搜索机制

与其一次性加载所有工具，不如引入搜索机制：

```javascript
searchTools(query):
  description: "根据功能描述搜索匹配的工具"
  execute: 
    return search(allTools, query)
```

智能体初始只携带核心工具集和 `searchTools`。当需要特定功能时，它会主动搜索并发现合适的工具，系统随后再动态加载该工具。这种方式以极低的即时成本换取了长期的功能扩展性。

### 上下文感知的动态加载

根据当前的操作环境自动匹配工具：

```javascript
loadToolsForFile(path):
  if (path.endsWith(".py")) {
    return [pythonLinter, pytestRunner, pipInstall]
  }
  if (path.endsWith(".js")) {
    return [eslint, jestRunner, npmInstall]
  }
```

当智能体处理 Python 文件时，相关的 Python 开发工具会自动激活。这种模式既保证了操作的连贯性，又维持了上下文的精简。

### 适用场景

核心工具（几乎随时需要的工具）应保持常驻；而专业化工具（如数据库迁移、部署指令等）则非常适合延迟加载。

建议从全量加载开始，只有当触及上下文上限或遇到明显的性能瓶颈时，再考虑引入延迟加载。避免过度设计。

## 常见陷阱 (Common Pitfalls)

### 过于宽泛的“全能”工具

设计一个类似 `doEverything(action, params)` 的工具并在内部进行逻辑分发是典型的反模式。这会导致描述文档异常臃肿，模型难以准确理解每个 action 的具体行为。

应遵循 Unix 哲学，设计职责单一且清晰的工具：如 `readFile`、`writeFile`、`deleteFile`，而非一个带有 `operation` 参数的通用文件工具。Unix 原则在此同样适用：每个工具应只做好一件事，通过结构化的输入和输出实现整洁的组合。

### 缺失输入校验

“在执行阶段再做校验”往往意味着程序运行到一半才会报错。

应在模式（Schema）定义中尽可能详尽地声明类型、约束和格式要求。执行阶段应建立在“输入必然合法”的假设之上。

### 吞掉错误信息

```javascript
execute(input):
  try {
    return actualWork(input)
  } catch (error) {
    return { success: false }
  }
```

此类代码对智能体毫无帮助。失败的原因是什么？它该尝试哪种替代方案？

务必提供包含错误码、详细描述、上下文背景以及修复建议的完整响应。

### 缺乏中断机制

运行十分钟却无法停止的工具会造成糟糕的用户体验；一次性修改数千个文件却无法撤回的操作则极具风险。

在设计长时任务或批处理操作时，务必考虑中断机制，允许检查取消信号或在处理间隙安全停止。

### 描述过于模糊

模型完全依赖名称和描述来决策。`proc` 或 "处理数据" 等描述无法提供任何实质信息。

编写描述时，请务必保证其准确性及完整性。宁可多消耗一些 Token 描述清楚，也远好过因模型误判导致的调用失败。

---

工具是智能体系统感知并改变现实的触角。智能体通过它们读取文件、执行指令、查询数据并产生实际影响。

在构建工具时，请始终自省：如果模型只阅读了这份契约，它是否能准确理解工具的功能、调用方式以及预期的反馈？

如果答案是肯定的，那么它就是合格的。
