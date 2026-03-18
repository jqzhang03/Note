# Model Context Protocol(MCP)

MCP是一个将AI应用程序与连接到外部系统的开源协议。
为了赋予大模型调用工具的能力，需要一种协议，使得大模型能够调用工具暴露的API，得到期望的结果。但由于当前市面上存在各种各样的大模型、各种各样的工具以及各种各样的操作系统，AI应用接入新工具需要复制完整的代码和功能描述，但很多企业不提供源代码，且跨语言的代码不一定能够执行，因此需要一种通用的协议来规范AI应用与工具之间的通信流程。
针对上述问题，模型调用工具分为**本地服务式调用**和**远程服务式调用**
* 本地服务式调用：直接将工具包拉到本地，利用进程间的通信机制将服务运行起来，自动获取工具的描述信息和自动完成调用
* 远程服务式调用：访问工具的远程服务，自动获取工具的描述信息和自动完成调用

**技术思路**：
* 工具与AI应用解耦合
* 工具与AI应用利用统一的通信协议
* 工具与AI应用利用统一的接口
* 工具与AI应用利用统一的数据交换格式
* AI应用接入工具时使用标准化配置内容
* AI应用内部实现标准化的工具加载调用逻辑

## 架构

MCP关键参与者包括：
* MCP主机：用于协调和管理一个或多个MCP客户端
* MCP客户端：维护与MCP服务端之间的连接，并从服务端获取上下文供MCP主机使用
* MCP服务端：一个为MCP客户端提供上下文的程序

```mermaid  theme={null}
graph TB
    subgraph "MCP Host (AI Application)"
        Client1["MCP Client 1"]
        Client2["MCP Client 2"]
        Client3["MCP Client 3"]
        Client4["MCP Client 4"]
    end

    ServerA["MCP Server A - Local<br/>(e.g. Filesystem)"]
    ServerB["MCP Server B - Local<br/>(e.g. Database)"]
    ServerC["MCP Server C - Remote<br/>(e.g. Sentry)"]

    Client1 ---|"Dedicated<br/>connection"| ServerA
    Client2 ---|"Dedicated<br/>connection"| ServerB
    Client3 ---|"Dedicated<br/>connection"| ServerC
    Client4 ---|"Dedicated<br/>connection"| ServerC
```

MCP包含两层，内层为数据层，外层为传输层：
* 数据层：定义了客户端和服务端之间基于JSON-RPC的对话协议，包括生命周期管理、核心原语，比如工具、资源、提示词和通知。
* 传输层：定义了客户端和服务器之间实现数据交换的通信机制和通道，包括特定于传输的连接建立、消息帧结构和授权。

**数据层**

数据层实现了基于JSON-RPC2.0的交换协议，定义了数据的结构和语义，包括：
* 生命周期管理：处理客户端和服务端之间的连接初始化、能力协商以及连接终止。
* 服务端功能：使服务端能够提供核心功能，包括AI工具、上下文数据资源、来源和来自客户端的交互提示词模板。
* 客户端功能：使服务端能够请求客户端从主机LLM中进行采样，获取用户输入，并向客户端发送日志消息。
* 实用功能：支持添加功能，如实时更新通知和长时间执行的程序跟踪。

**传输层**

传输层负责管理客户端和服务端之间的交流通道和鉴权，处理MCP参与者之间的连接建立、消息格式以及通信安全。MCP支持两种
传输机制：
* STDIO Transport(标准传输)：使用标准输入输出流在同一台主机的本地进程之间进行直接通信，在没有网络的前提下提供最佳性能。
* Streamable HTTP Transport(流式HTTP传输)：使用HTTP POST协议进行客户端和服务端之间的消息通信，并可选择服务器发送事件来得到流式传输能力。这个传输协议支持远程服务端通信，还支持包括持有者令牌、API密钥和自定义标头在内的标准HTTP身份验证方法。MCP建议使用OAuth来获取身份验证令牌。

传输层将通信细节与协议层隔离，从而确保所有通信协议都能够使用相同的JSON-RPC2.0消息格式。

MCP是有状态的协议(流式HTTP传输协议存在无状态的子集)，生命周期管理的目的是协商客户端和服务端所提供的能力。

MCP原语定义了客户端和服务端可以为对方提供什么，原语指定了可以共享给AI应用程序的上下文信息类型以及可执行的操作范围。

MCP定义了三种服务端可以暴露的原语：
* Tools：AI应用程序可以调用执行的函数。(文件操作、API调用、数据库查询)
* Resources：为AI应用程序提供上下文信息的数据源。(文件内容、数据库记录、API响应)
* Prompts：有助于与大模型结构化交互的可复用模板。(系统提示词、少量示例)

MCP客户端可以使用`*/list`方法得到可用的原语，`tools/list`可以查看可用工具，利用`*/get`进行检索。

MCP为客户端定义了三种可暴露的原语，这些原语可以让MCP服务端作者构建更丰富的交互：
* Sampling：利用`sampling/complete`，允许服务器向客户端的AI应用程序请求语言模型补全。
* Elicitation：利用`elicitation/request`，允许服务器向用户请求额外的信息。
* Logging：允许服务端向客户端发送日志信息，以用于debug和监测的目的。

除了服务端和客户端原语，MCP协议还提供了实用原语，用于增强请求的执行方式：
* Tasks(Experimental)：可以MCP请求的延迟结果检索和状态追踪的持久执行封装器。

MCP协议提供实时通知，支持服务端和客户端之间的动态更新，Notifications发送JSON-RPC2.0通知信息(无需响应)和支持MCP服务端向其连接的客户端提供实时更新。

简单总结来说，MCP协议如同一个拓展坞，Cursor、Claude Code、Codex等就是MCP主机，它们打开的代码文件就是一个客户端，利用拓展坞上的接口，服务端便可以提供本地和远程服务。

原语使用方法见[官方文档](https://modelcontextprotocol.io/docs/learn/architecture)

### 服务器

**Tools**
Tools是大模型可以调用的模式定义的接口，MCP使用JSON Schema进行验证，工具执行的每一个操作都有标准的输入和输出格式，Tools在执行操作前可能会需要得到用户的同意，以确保用户对模型的行为具有控制权。

协议指令：
|方法|目的|返回|
|-|-|-|
|`tools/list`|查找可用工具|包含模式的工具定义数组|
|`tools/call`|执行特定工具|工具执行结果|

工具定义的样例
```typescript
{
  name: "searchFlights",
  description: "Search for available flights",
  inputSchema: {
    type: "object",
    properties: {
      origin: { type: "string", description: "Departure city" },
      destination: { type: "string", description: "Arrival city" },
      date: { type: "string", format: "date", description: "Travel date" }
    },
    required: ["origin", "destination", "date"]
  }
}
```

为了确保安全，应用程序可以通过各种机制来实现用户的控制，如
* 在用户界面中显示可用工具，使用户能够定义在特定交互中是否应显示某个工具。
* 针对单个工具执行的审批对话框
* 用于预先批准某些安全操作的设置
* 展示所有工具执行以及结果的活动日志

**Resources**
Resources提供了结构化访问信息的方式，AI应用可以检索和提供给大模型信息作为上下文。

Resources从文件、API、数据库或其他任何来源暴露AI需要理解的上下文数据，应用可以直接访问这些数据并决定如何使用这些数据，无论是选择相关部分、使用嵌入搜索还是把它们全部交给模型，都可以实现。

Resources提供两种查找模式：
* Direct Resources：直接填写访问数据的具体URI，如`calendar://events/2024`
* Resource Templates：带参数的动态URI，如`travel://activities/{city}/{category}`，city和category都是可选的。

Resource Templates包含标题、描述和预期 MIME 类型等元数据，使其易于发现且具有自文档性。

协议指令：
|方法|目的|返回|
|-|-|-|
|`resources/list`|列出可用的直接资源|资源描述数组|
|`resources/templates/list`|查找资源模板|资源模板定义数组|
|`resources/read`|检索资源内容|带有元数据的资源数据|
|`resources/subscribe`|监控资源变化|订阅确认|

资源模板样例
```typescript
{
  "uriTemplate": "weather://forecast/{city}/{date}",
  "name": "weather-forecast",
  "title": "Weather Forecast",
  "description": "Get weather forecast for any city and date",
  "mimeType": "application/json"
}

{
  "uriTemplate": "travel://flights/{origin}/{destination}",
  "name": "flight-search",
  "title": "Flight Search",
  "description": "Search available flights between cities",
  "mimeType": "application/json"
}
```

**Prompts**

Prompts提供可复用的模板，它们允许MCP服务器作者为域提供参数化提示词，或展示如何更好地使用MCP服务器。

Prompts是定义了预期输入和交互模式的结构化模板，它们是用户控制的、需要显式调用的，而不是自动触发的。Prompts是可以感知上下文的，并参考可用资源和工具来创建综合的工作流。和Resources类似，Prompts提供参数补全，帮助用户发现有效参数值。

协议指令：
|方法|目的|返回|
|-|-|-|
|`prompts/list`|查找可用prompts|prompt描述数组|
|`prompts/get`|检索prompt细节|带参数的完整的prompt定义|

样例
```typescript
{
  "name": "plan-vacation",
  "title": "Plan a vacation",
  "description": "Guide through vacation planning process",
  "arguments": [
    { "name": "destination", "type": "string", "required": true },
    { "name": "duration", "type": "number", "description": "days" },
    { "name": "budget", "type": "number", "required": false },
    { "name": "interests", "type": "array", "items": { "type": "string" } }
  ]
}
```

### 客户端

概念：
- host是用户与之交互的应用程序
- clients是允许服务器连接的协议级组件


