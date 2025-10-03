# [Microsoft Semantic Kernel + Microsoft.Extensions.AI：完整的.NET AI 集成指南](https://medium.com/@shahriddhi717/microsoft-semantic-kernel-extensions-ai-the-complete-net-ai-integration-guide-6a97d18fa538)

## 介绍

大多数.NET 开发者都面临同样的 AI 集成困境：如何构建生产就绪的 AI 功能，同时避免供应商锁定或重新设计编排模式？微软的两项技术可以完美解决这一问题。

Semantic Kernel (SK) 解决了编排问题——这是一个开源 SDK，帮助你在.NET（以及 Python/Java）中构建提示、调用函数/插件并采用代理模式。它被设计为协调 AI 步骤与应用程序逻辑的粘合剂。

Microsoft.Extensions.AI 涵盖了提供程序抽象——它为 .NET 提供了一个统一的接口（ IChatClient ），用于聊天/多模态交换、流式处理和嵌入，所有这些都遵循熟悉的 Microsoft.Extensions.* 依赖注入和日志记录模式。当你切换提供程序（如 Azure OpenAI、OpenAI 或本地模型）时，你的调用站点保持稳定。

思维模型：SK = 编排（提示、插件、规划/代理）；Microsoft.Extensions.AI = 传输/原语（聊天、嵌入、选项、遥测）。

## 快速决策指南

- 仅使用 Extensions.AI：简单的聊天/嵌入场景，单步 AI 调用
- SK + Extensions.AI: 多步骤工作流、函数调用、代理模式、企业编排
- SK 与本地模型: 同上，但具有完全的数据控制能力(Ollama/本地托管)

## 目录

- Microsoft Semantic Kernel (SK) — .NET 概述
- Microsoft.Extensions.AI — 概述与软件包
- SK + Microsoft.Extensions.AI 如何协同工作（为什么 SK 很重要）
- 代码演示：自动化 C# 文档生成器
- 企业部署与数据治理

## Microsoft Semantic Kernel (SK) — .NET 概述

Semantic Kernel 是微软的开源 SDK，用于将 AI 与常规 .NET 代码进行编排。它帮助你连接提示、函数（"插件"）、与模型提供商（OpenAI、Azure OpenAI 等）的连接，以及可选的规划/内存功能 — 这样你的应用程序就能可预测地协调 AI 步骤。

Semantic Kernel 简化了集成，减少了不同 AI 服务之间的复杂性，并通过构建提示和步骤的运行方式使行为更可控 — 从课堂演示到企业应用都很有用。

### 安装

```bash
dotnet add package Microsoft.SemanticKernel
```

兼容性：SK 1.0+ 适用于 .NET 6/8/9。Extensions.AI 需要 .NET 8+。

### 创建内核

```csharp
using Microsoft.SemanticKernel;

var builder = Kernel.CreateBuilder();
// 在此处添加服务/插件/连接
var kernel = builder.Build();
```

### ASP.NET Core 注册

```csharp
var builder = WebApplication.CreateBuilder();
builder.Services.AddKernel(); // 注册 Semantic Kernel

// 在此处添加服务/插件/连接

var app = builder.Build();
```

这些遵循官方概述：内核是 DI 容器，它包含 SK 为你协调的服务和插件。

### 连接到模型（例如：Azure OpenAI 聊天）

```csharp
builder.Services.AddAzureOpenAIChatCompletion(
    "your-resource-name",
    "your-endpoint",
    "your-resource-key",
    "deployment-model");
```

### 核心构建块概览

- 连接 — AI 服务和数据的适配器。
- 插件 — 模型可以调用的函数（语义提示或原生 C#）。
- 规划器 — 构建和执行计划（可选）。
- 记忆 — 用于检索的嵌入/向量存储抽象。

## Microsoft.Extensions.AI — 统一的 .NET AI 抽象

Microsoft.Extensions.AI 为 .NET 提供了一种与 AI 服务通信的与提供程序无关的方式，使用熟悉的 Microsoft.Extensions.* 模式（DI、中间件）。它围绕两个关键抽象： IChatClient （聊天/多模态）和 IEmbeddingGenerator （嵌入）。在此基础上，它为工具调用、缓存、遥测等提供了现成的中间件。

需要引用哪些软件包取决于你使用的场景：

- Microsoft.Extensions.AI.Abstractions — 核心类型/接口（例如 IChatClient ）。
- Microsoft.Extensions.AI — 添加更高级的助手和管道/中间件。

大多数应用程序使用 Microsoft.Extensions.AI 加上一个提供程序库（OpenAI、Azure、Ollama 等）。

API 文档指出，所有 IChatClient 成员都是线程安全的，可并发使用——非常适合 Web 应用程序和后台服务。

典型用法（单次调用和流式处理）。

```csharp
using Microsoft.Extensions.AI;

// 单次调用
ChatResponse r = await chatClient.GetResponseAsync("What is AI?");
Console.WriteLine(r.Text);

// 流式处理
await foreach (var u in chatClient.GetStreamingResponseAsync("Stream a haiku."))
    Console.Write(u);
Console.WriteLine();
```

这些模式——请求/流式处理，加上工具调用、缓存、遥测、DI 管道。

为什么它与 SK 配对。将编排（提示/插件/流程）保留在 SK 中，并将模型 I/O 保留在 IChatClient 之后。这个清晰的边界让你无需重构应用程序逻辑即可更换提供商，并在一个地方处理横切关注点（日志记录/Otel、缓存、速率限制）。

## SK + Microsoft.Extensions.AI 如何协同工作（为什么 SK 很重要）

每个应用层的作用。

- Microsoft.Extensions.AI 为你提供了一个统一的、与提供商无关的客户端（ IChatClient ），具有流式传输、工具调用、缓存、遥测、DI 等管道。它标准化了"如何与模型对话"。
- Semantic Kernel (SK) 为你提供编排功能：一个内核（DI 容器）、插件（你的应用向模型公开的函数）、可选的内存和规划指导——即"AI 步骤如何围绕你的.NET 代码组合在一起"。

### 为什么基于 SK 的 Ext.AI 应用更优秀

1. 有组织的函数公开
    - 与手动编写 JSON 工具规范或分散辅助方法不同，SK 允许你注册模型可以调用的插件（原生 C#或导入的）。你获得了一种一致的方式来公开功能、重用它们并保持其可发现性。
2. 真正的编排中心
    - Extensions.AI 标准化调用；Semantic Kernel 标准化流程。SK 的内核承载你的服务和插件，因此多步骤交互（提示 → 工具 → 提示）集中在一个地方，而不是临时的代码路径中。
3. 无需传统规划器的现代"规划"
    - 目前的指导是优先使用函数调用（由 SK 驱动循环）而非传统规划器；微软记录了从分步规划器到自动函数调用的迁移，因为它更可靠且更节省令牌。SK 将这种编排模式内置其中，因此你无需重新实现。
4. 扩展后的内存与检索
    - 当你的提示超出单个文件范围时，SK 的内存抽象允许你插入向量存储（如 Azure AI Search、Redis 等），而无需将你的应用程序硬编码到特定的后端方案。
5. 与 Ext.AI 的管道协同工作
    - 保留 Ext.AI 的优势——流式传输、遥测（OpenTelemetry）、缓存和中间件——用于传输层，同时 SK 专注于协调和插件生命周期。这样你就能获得清晰的关注点分离和可观测性。

### 如果你跳过 SK（仅使用 Ext.AI），会有什么变化？

你可以单独使用 Ext.AI 发布——但最终你可能会重新构建编排粘合代码：

1. 临时工具接口：你需要自己描述和管理可调用函数（命名、参数、版本控制），并手动编写调用/返回循环。
2. 分散的提示/流程逻辑：多步骤链（提示→工具→后续操作）将位于控制器/服务中，没有一个统一的编排模型。
3. 难以扩展：添加检索（"记忆"）、导入外部工具（OpenAPI/MCP）或发展为多步骤类似代理的行为都需要定制工作。
4. 规划迁移需要你自己完成：你需要自己模拟较新的自动函数调用方法及其重试/保护机制。

> **经验法则**
>
> 如果你的应用是单步骤（提问→回答）并带有一些中间件，仅使用 Ext.AI 就足够了。
> 如果你需要可重复的多步骤流程、标准化插件、可选内存或代理模式的实现路径，请将 SK 作为编排层添加。

### 额外优势：结构化输出适用于两个层面

对于机器可读的结果（如代码文档 JSON），你可以在客户端层（Ext.AI）请求 JSON/JSON 模式，然后让 SK 将结果作为另一个步骤进行消费/路由。Azure 的结构化输出解释了模式保证与旧版"JSON 模式"的区别。

## 代码演示：使用 SK + Extensions.AI 自动生成 C# 文档

演示功能：此演示展示了一个实用的"代码转文档"工作流程，它以任何 C# 文件作为输入，并生成干净、结构化的 Markdown 文档。它结合了 Semantic Kernel 的编排能力与 Extensions.AI 的提供者无关聊天界面，创建了一个自动记录类、方法、参数和返回类型的工具。

工作流程：

- 解析 C#源代码 — 一个自定义 SK 插件使用正则表达式模式从任何 .cs 文件中提取符号（命名空间、类、公共方法）
- 结构化为 JSON — 解析器将代码符号转换为结构化的 JSON 数据
- AI 分析 — SK 编排一个提示模板，该模板接收 JSON 并生成专业文档
- Markdown 输出 — 结果是干净、一致的文档，可用于 wiki、README 文件或入职材料

架构亮点：

- SK 插件： CodeParserPlugin 暴露了一个 GetSymbols 函数，内核可以调用该函数
- 提示编排：SK 管理将原始代码符号转换为文档的模板
- Extensions.AI：提供底层聊天客户端抽象，使代码保持提供者无关性

示例输入（RcaAgent.cs）：

```csharp
public sealed class RcaAgent
{
    public RcaResult AnalyzeIncident(IncidentData incident, IEnumerable<LogEntry> logs) { ... }
    public LogAnalysis AnalyzeLogs(IEnumerable<LogEntry> logs) { ... }
    // ... 其他方法
}
```

测试输出：

```bash
dotnet run -- "/path/to/RcaAgent.cs"
```

生成输出：

```markdown
# Summary for RcaAgent

## Classes
- **RcaAgent** — Root Cause Analysis Agent that analyzes logs and incidents to identify potential causes

## Methods

### AnalyzeIncident
`public RcaResult AnalyzeIncident(IncidentData incident, IEnumerable<LogEntry> logs)`
- Analyzes the provided incident together with a sequence of log entries and returns an RcaResult.
- Parameters:
  - incident (IncidentData) — the incident instance to be analyzed
  - logs (IEnumerable<LogEntry>) — the sequence of log entries to use in the analysis
- Returns: RcaResult — the result of the root-cause analysis

### AnalyzeLogs
`public LogAnalysis AnalyzeLogs(IEnumerable<LogEntry> logs)`
- Analyzes the provided sequence of log entries and returns a LogAnalysis.
- Parameters:
  - logs (IEnumerable<LogEntry>) — the sequence of log entries to analyze  
- Returns: LogAnalysis — the outcome of the log analysis
```

为什么这展示了 SK + Extensions.AI 的价值：

- 清晰的分离：Extensions.AI 处理模型通信，SK 处理编排
- 可重用的插件： CodeParserPlugin 可以在不同的文档工作流中重用
- 提供商灵活性：从 OpenAI 切换到 Azure OpenAI 再到本地模型，无需更改业务逻辑
- 结构化提示：SK 的模板引擎使文档格式保持一致且易于维护

实际作用：这种自动化文档生成可以显著改善开发者入职、代码审查和知识共享——特别是对于手动文档跟不上开发进度的大型代码库而言。

## 企业部署与数据治理

### 企业现实

许多公司由于数据治理担忧以及敏感代码被发送到外部提供商的风险，已经阻止了 AI 的使用。虽然这个演示为了简单起见使用了 OpenAI API 密钥，但整个工作流程可以完全离线运行，使用 Ollama 和 Llama、CodeLlama 或 Mistral 等开源模型。

### 完全数据控制

使用 Ollama，你的代码永远不会离开你的环境——你在保持对数据的完全控制的同时，仍然能获得 AI 辅助文档的生产力优势。Extensions.AI 抽象使得这种提供商切换变得简单：只需更改客户端配置，你的编排逻辑保持不变。

### 企业优势

- AI 集成代码减少 40-60%
- 提供程序切换只需不到 5 行代码更改
- 内置遥测技术减少调试时间
- 标准化模式提高团队速度
- 为敏感代码库提供完整的离线功能

### 常见陷阱

- SK 提示模板使用 {{$variable}} 而不是 {{variable}}
- Extensions.AI 需要 List\<ChatMessage\> 而不是原始字符串
- 插件方法需要 [KernelFunction] 属性

### 准备好自己尝试了吗？

准备好自己尝试了吗？完整的工作示例可点击下方阅读原文链接下载后，插入任何 .cs 文件，即可获得即时的方法和类摘要，这将使新开发人员的入职变得显著更容易。无论你是使用 OpenAI 进行快速实验，还是使用 Ollama 保护生产环境的隐私，选择权在你手中。

## 结论

Semantic Kernel + Extensions.AI 为 .NET 开发人员提供了一条生产就绪的 AI 集成路径，无需供应商锁定或架构复杂性。SK 负责处理编排和工作流模式，而 Extensions.AI 则提供简洁的提供程序抽象和企业级中间件。

对于构建超越简单聊天场景的团队而言，这种组合提供了结构化、可维护的基础，使 AI 功能能够在企业应用程序中扩展 — 同时完全控制你的数据去向。

今天就到这里。期待与你再次相见。
