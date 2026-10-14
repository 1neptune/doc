# 如何开发规范化 MCP

> 一份面向实战的技术规范文档：从作用定义 → 组成部分 → 各部分作用 → 完整案例 → 检查清单
>
> 规范依据：Model Context Protocol 官方 Specification（以 `2025-06-18` 版本为基准，协议保持向后兼容）

---

## 目录

- [一、作用与定义](#一作用与定义)
- [二、组成部分](#二组成部分)
- [三、各组成部分的作用](#三各组成部分的作用)
- [四、开发流程（七步法）](#四开发流程七步法)
- [五、协议层硬性红线](#五协议层硬性红线)
- [六、案例](#六案例)
- [七、常见错误清单](#七常见错误清单)
- [八、发布前检查清单](#八发布前检查清单)

---

## 一、作用与定义

### 1.1 什么是 MCP

**MCP（Model Context Protocol，模型上下文协议）是一套让 AI 应用与外部系统建立标准化连接的开源协议。**

它的本质是一份 **JSON-RPC 2.0 规范**：约定客户端（AI 应用）和服务端（你的系统）之间如何握手、如何列出能力、如何调用函数、如何返回结果。

一句话概括：

> MCP 把「AI 怎么连上你的系统」这件事，从每个团队各写一套私有插件，变成一个全行业通用的标准接口。

在没有 MCP 的时代，AI 要访问一个数据库或一个内部 API，需要为每个平台（Claude / Cursor / 某个 IDE）单独写一个插件插件，N 个平台 × M 个系统 = N×M 份代码。MCP 把这个矩阵变成了一次开发、多处使用。

### 1.2 架构角色：Host / Client / Server

理解这三个角色是理解整个规范的前提：

| 角色 | 是什么 | 谁来实现 |
|---|---|---|
| **Host（宿主）** | 用户实际面对的应用，如 IDE、桌面客户端 | MCP 客户端产品方 |
| **Client（客户端）** | Host 内部与某个 Server 建立的那**一条**连接 | Host 内部实现，一个 Host 可有多个 Client |
| **Server（服务端）** | 独立进程，封装你的业务系统能力 | **你** |

关键澄清（最容易混淆的一点）：

> **一个 Server 可以同时服务多个 Client。** Host 里可以装 5 个 MCP Server（数据库、GitHub、内部 OA…），它们互相隔离、生命周期独立。

```
        ┌─────────────────────────────────────┐
        │            Host（IDE / 客户端）       │
        │                                     │
        │   Client A ──── 连接 ────┐          │
        │   Client B ──── 连接 ────┤          │
        │   Client C ──── 连接 ────┤          │
        └───────────────────────────┼──────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
        ┌─────▼─────┐        ┌──────▼──────┐       ┌──────▼──────┐
        │  Server 1 │        │  Server 2   │       │  Server 3   │
        │  数据库    │        │   Git 仓库   │       │   内部 OA   │
        └───────────┘        └─────────────┘       └─────────────┘
```

### 1.3 MCP 解决的四类问题

| 能力类型 | 解决的问题 |
|---|---|
| **打通系统壁垒** | AI 无法访问非公开系统，MCP 提供受控通路 |
| **统一接入协议** | 一次开发，所有兼容 MCP 的客户端都能用 |
| **实时数据通路** | 相比「把数据塞进上下文」，MCP 是按需实时查询 |
| **行为可控** | 通过能力声明 + 注解 + 审批，让危险操作可识别、可拦截 |

### 1.4 MCP vs Skill vs 其他方案：什么时候该写 MCP

这是最需要先想清楚的问题。**MCP 和 Skill 不是竞争关系，而是互补关系。**

| 你的需求 | 推荐方案 | 理由 |
|---|---|---|
| 一次性指令、纯知识问答 | 直接对话 / **Skill** | 不需要系统连接 |
| 固定的多步流程、模板、规范 | **Skill** | Skill 是说明书，不需要数据通路 |
| **让 AI 读写你的真实系统** | **MCP Server** | MCP 提供实时、带鉴权的实际连接 |
| 已有 HTTP API，想让 AI 用 | **MCP Server** | 典型的 API → MCP 包装场景 |
| 需要第三方 SaaS 能力 | **官方 / 社区 MCP Server** | 不要重复造轮子，先搜有没有现成的 |
| 多轮对话的意图决策 | **MCP Server** | 工具调用是模型驱动的动态决策 |

**一句话判断法**：

> **Skill 教 AI「怎么做」，MCP 让 AI「做得到」。**
>
> 两者经常配套：MCP 提供 `search_docs` 工具，Skill 告诉 AI「遇到接口报错时先调 `search_docs` 再重试」。

### 1.5 收益与成本

| 维度 | 收益 | 成本 |
|---|---|---|
| 开发 | 一次开发，全客户端复用 | 需实现协议生命周期、能力声明 |
| 维护 | 协议版本向后兼容，老客户端不掉线 | 需跟进协议版本演进 |
| 安全 | 能力/注解/审批三层管控 | 自己要负责鉴权、输入校验、限流 |
| 性能 | 按需查询，不占上下文 | 需设计超时、取消、分页 |

---

## 二、组成部分

### 2.1 完整组成部分清单

一个规范的 MCP Server 由以下部分构成：

```
my-mcp-server/                    # 工程目录（非协议要求，但推荐如此组织）
├── src/                          # 服务端实现
│   ├── index.ts                  # 入口：协议注册 + 传输层启动
│   ├── server.ts                 # Server 实例：能力声明 + 元信息
│   ├── tools/                    # 【核心】工具实现
│   │   ├── search.ts
│   │   └── write.ts
│   ├── resources/                # 资源实现
│   │   └── schema.ts
│   └── prompts/                  # 提示模板实现
│       └── review.ts
├── package.json                  # 依赖与启动命令（stdio 型必需）
├── tsconfig.json
├── .env.example                  # 环境变量示例（严禁提交 .env）
└── README.md                     # 安装说明 + 工具清单
```

**协议层面必须实现的部分**：

| 层 | 内容 | 是否必需 |
|---|---|---|
| **基础协议** | JSON-RPC 2.0 消息（request / response / notification） | ✅ 必需 |
| **生命周期** | initialize → initialized → 运行 → 关闭 | ✅ 必需 |
| **传输层** | stdio 或 Streamable HTTP（二选一或都支持） | ✅ 必需 |
| **工具能力** | `tools`（list / call） | ⭕ 按需 |
| **资源能力** | `resources`（list / read） | ⭕ 按需 |
| **提示能力** | `prompts`（list / get） | ⭕ 按需 |
| **鉴权** | HTTP 型需要 OAuth 2.1；stdio 型走环境变量 | ⭕ 按传输层而定 |

**官方原话**：所有实现 **MUST** 支持基础协议和生命周期管理；其他组件 **MAY** 按需实现。这正是「渐进式实现」的依据——先跑通握手和一个工具，再逐步扩展。

### 2.2 最小可用 MCP Server（长这样就够了）

一个只有 **1 个工具**、走 **stdio** 的 MCP Server 就是合法的最小实现：

```
weather-mcp/
├── package.json
└── src/index.ts        # initialize + tools/list + tools/call 三件事
```

### 2.3 客户端配置（Server 之外必备的一环）

MCP Server 写完必须让客户端知道怎么启动它。

**WorkBuddy / CodeBuddy**：写入 `~/.workbuddy/mcp.json`

```json
{
  "mcpServers": {
    "weather-mcp": {
      "command": "node",
      "args": ["D:/tools/weather-mcp/dist/index.js"],
      "env": {
        "WEATHER_API_KEY": "${WEATHER_API_KEY}"
      }
    }
  }
}
```

**Claude Desktop / Cursor 类客户端**：`claude_desktop_config.json`

```json
{
  "mcpServers": {
    "weather-mcp": {
      "command": "npx",
      "args": ["-y", "@yourorg/weather-mcp"],
      "env": {
        "WEATHER_API_KEY": "sk-xxx"
      }
    }
  }
}
```

**远程 HTTP 型**：

```json
{
  "mcpServers": {
    "oa-mcp": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${OA_TOKEN}"
      }
    }
  }
}
```

### 2.4 传输层的选择

| 传输方式 | 适用场景 | 客户端配置关键字段 | 鉴权方式 |
|---|---|---|---|
| **stdio** | 本地进程、桌面客户端、单用户 | `command` + `args` + `env` | 环境变量 |
| **Streamable HTTP** | 远程服务、多客户端并发、多用户 | `url`（+ `headers`） | OAuth 2.1 / Bearer Token |

**选择原则**：客户端和服务端在同一台机器 → stdio；跨机器、多用户、需要长期运行 → Streamable HTTP。

> 注意：老的 `HTTP+SSE` 传输已在 `2025-06-18` 版本被 Streamable HTTP 取代，新项目不要基于它设计。

---

## 三、各组成部分的作用

### 3.1 基础协议：JSON-RPC 2.0 消息 —— 一切的地基

所有通信都是 UTF-8 编码的 JSON-RPC 2.0 消息，共三种形态：

| 消息类型 | 有 `id` | 有 `result`/`error` | 用途 |
|---|---|---|---|
| **Request（请求）** | ✅ 必须（string 或 number，**不能是 null**） | — | 发起一个操作，等待响应 |
| **Response（响应）** | ✅ 必须（与对应请求**相同**的 id） | ✅ 二选一 | 回复请求 |
| **Notification（通知）** | ❌ 禁止带 id | ❌ 不回复 | 单向通知 |

**Request 结构**：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": { "cursor": "optional-cursor-value" }
}
```

**Response 结构（成功 / 失败二选一，不可同时设置）**：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { "tools": [] }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Unknown tool: invalid_tool_name",
    "data": { "detail": "..." }
  }
}
```

**Notification 结构**：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

**协议层硬约束**：

- `id` 在**同一会话内不得重复使用**
- `error.code` 必须是整数
- `result` 与 `error` **不能同时存在，也不能同时缺失**
- `_meta` 字段为协议保留，用于挂载额外元数据；其中以 `modelcontextprotocol` 或 `mcp` 开头的 key 前缀为**协议保留**，实现不得假定其含义

### 3.2 生命周期 —— 连接的第一段代码

规范定义了严格的三阶段流程。**这是新手最常踩坑的地方。**

#### 阶段一：初始化（能力协商）

**必须是客户端与服务端的第一次交互**。客户端发送 `initialize`：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "roots": { "listChanged": true },
      "sampling": {},
      "elicitation": {}
    },
    "clientInfo": {
      "name": "ExampleClient",
      "title": "Example Client Display Name",
      "version": "1.0.0"
    }
  }
}
```

服务端必须响应自己的能力和信息：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "logging": {},
      "prompts": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "tools": { "listChanged": true }
    },
    "serverInfo": {
      "name": "ExampleServer",
      "title": "Example Server Display Name",
      "version": "1.0.0"
    },
    "instructions": "可选：给客户端的使用建议"
  }
}
```

#### 阶段二：初始化完成

初始化成功后，**客户端必须发送 `initialized` 通知**才能开始正常操作：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

**顺序铁律**：

- 客户端在收到 `initialize` 响应前，**不应发送除 ping 以外的任何请求**
- 服务端在收到 `initialized` 通知前，**不应发送除 ping 和 logging 以外的任何请求**

#### 阶段三：版本协商规则

| 情况 | 服务端应做 |
|---|---|
| 支持客户端请求的版本 | **必须**返回相同版本 |
| 不支持 | **必须**返回自己支持的某个版本（建议为其支持的最新版本） |
| 客户端不支持服务端返回的版本 | 客户端**应当断开连接** |

#### 阶段四：关闭

规范未定义专门的关闭消息，靠传输层机制表达：

- **stdio**：客户端先关闭子进程 stdin → 等待退出 → 超时发 `SIGTERM` → 再超时发 `SIGKILL`
- **HTTP**：关闭对应的 HTTP 连接

#### 阶段五：超时（必做）

规范要求：**所有发出的请求都必须设置超时**，超时后发送取消通知并停止等待。

- 即使收到进度通知（progress notification）可以重置计时器，也**必须同时设一个最大超时上限**
- 中间件应支持**按请求粒度**配置超时

> 常见事故：AI 调了一个必然超时的工具，整个客户端卡死。这通常是 Server 端没设默认超时导致的。

### 3.3 能力声明（Capabilities）—— 诚实是硬要求

能力协商决定本次会话启用哪些可选特性。**声明了就要实现，没实现就别声明。**

| 类别 | 能力 | 含义 |
|---|---|---|
| Client | `roots` | 能提供文件系统根目录 |
| Client | `sampling` | 支持让模型采样生成内容 |
| Client | `elicitation` | 支持服务端向用户索取额外信息 |
| Server | `prompts` | 提供提示词模板 |
| Server | `resources` | 提供可读资源 |
| Server | `tools` | 暴露可调用工具 |
| Server | `logging` | 发送结构化日志 |
| Server | `completions` | 支持参数自动补全 |
| 双方 | `experimental` | 非标准实验特性 |

子能力：

- `listChanged`：列表变化时发送通知（prompts / resources / tools）
- `subscribe`：支持订阅单个资源变更（**仅 resources**）

声明为「都不支持」也要显式写出来：

```json
{ "capabilities": { "resources": {} } }
```

**版本不兼容时的初始化失败示例**：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Unsupported protocol version",
    "data": {
      "supported": ["2024-11-05"],
      "requested": "1.0.0"
    }
  }
}
```

### 3.4 传输层（Transport）—— 决定 Server 怎么跑起来

#### 3.4.1 stdio

| 规则 | 说明 |
|---|---|
| 启动方式 | 客户端把 Server 作为**子进程**拉起 |
| 输入 | 从 `stdin` 读 JSON-RPC 消息 |
| 输出 | 向 `stdout` 写 JSON-RPC 消息 |
| 分帧 | **按换行分隔，且消息内不得含换行** |
| 日志 | **只能写 `stderr`**，客户端可捕获/转发/忽略 |
| **致命红线** | Server **绝不允许**向 `stdout` 输出任何非 MCP 消息 |

**最后一条是 stdio 型最经典的翻车点**：一句 `console.log("server started")` 就会让客户端解析失败、连接直接断掉。

规范原文：Server **MUST NOT** write anything to its `stdout` that is not a valid MCP message。客户端也**不得**向 Server 的 stdin 写入非 MCP 内容。

**适用范围**：`stdio` 型**不应**走 OAuth 授权框架，凭证应从环境变量获取。

#### 3.4.2 Streamable HTTP

Server 作为独立进程运行，可同时处理多个客户端连接。**必须**提供**单一**的 MCP 端点路径，同时支持 POST 和 GET，例如 `https://example.com/mcp`。

**发送消息给 Server 的规则**：

| # | 规则 |
|---|---|
| 1 | 每条 JSON-RPC 消息必须是**一次新的 HTTP POST** |
| 2 | POST 必须带 `Accept` 头，同时列出 `application/json` 和 `text/event-stream` |
| 3 | POST body 必须是**单个** request / notification / response |
| 4 | 输入若是 response 或 notification：接受则返回 **202 Accepted 无 body**；不接受则返回 HTTP 错误码 |
| 5 | 输入若是 request：服务端返回 `Content-Type: text/event-stream`（开 SSE 流）**或** `application/json`（单个对象）—— 客户端必须两种都支持 |
| 6 | 开 SSE 流后，流中**最终必须包含**对应请求的 response，发完应关闭流 |

**关键理解**：断开连接**不应**被解读为客户端取消请求。要取消，客户端必须显式发送 `CancelledNotification`。

**监听服务端消息（GET）**：

- 客户端**可以**发 GET 打开 SSE 流，让服务端无需 POST 即可主动推送
- GET 必须带 `Accept: text/event-stream`
- 服务端必须返回 `text/event-stream`，否则返回 **405 Method Not Allowed**
- 服务端**不得**在 GET 流上发 response，除非是在**恢复**之前的流

**多连接规则**：客户端可以同时保持多个 SSE 流，但**服务端必须把每条消息只发到其中一条流上，不得广播**。

**会话管理**：

| 环节 | 规则 |
|---|---|
| 分配 | 初始化时可通过响应头 `Mcp-Session-Id` 分配会话 ID |
| 要求 | ID 应全局唯一且密码学安全（UUID / JWT / 哈希）；只能包含可见 ASCII（0x21–0x7E） |
| 回传 | 客户端拿到后必须在**后续所有**请求带上 `Mcp-Session-Id` |
| 缺失处理 | 需要会话 ID 的服务端，对无该头（初始化除外）的请求应返回 **400 Bad Request** |
| 会话失效 | 服务端可随时终止会话，之后带该 ID 的请求返回 **404 Not Found** |
| 客户端应对 | 收到 404 后必须发起**新的** `initialize` 请求重建会话 |
| 主动关闭 | 客户端不再需要会话时，应发 **HTTP DELETE** 带 `Mcp-Session-Id`；服务端可回 405 表示不允许 |

**协议版本头**：

> 使用 HTTP 时，客户端**必须**在后续所有请求带上 `MCP-Protocol-Version: <version>` 头。
>
> 兼容性兜底：若服务端收不到该头且无其他方式确定版本（如初始化时已协商），**应当假设版本为 `2025-03-26`**。收到不支持的版本值时，**必须**返回 **400 Bad Request**。

**可恢复性（Resumability）**：

- 服务端**可以**给 SSE 事件附加 `id` 字段；id 在同一会话内（或同一客户端的所有流内）必须全局唯一
- 客户端断线后可用 GET + `Last-Event-ID` 头请求恢复
- 服务端**不得**重放本应投递到**其他**流的消息

#### 3.4.3 安全警告（规范强制要求）

实现 Streamable HTTP 时，以下三条是 **MUST / SHOULD** 级别：

| # | 要求 | 后果 |
|---|---|---|
| 1 | **必须**校验所有入站连接的 `Origin` 头，防止 DNS rebinding 攻击 | 否则远程网站可与本地 MCP Server 交互 |
| 2 | 本地运行时**应当**只绑定 `127.0.0.1`，而非 `0.0.0.0` | 避免暴露到所有网卡 |
| 3 | **应当**对所有连接实现恰当的认证 | 防止未授权访问 |

**没有这三条保护，攻击者可以通过 DNS rebinding 从远程网页与你的本地 MCP Server 对话。**

### 3.5 Tools（工具）—— MCP 的核心，也是最该花心思的部分

Tools 让模型与外部系统交互：查数据库、调 API、做计算。

#### 3.5.1 交互模型：模型驱动

Tools 设计为 **model-controlled**：由语言模型根据上下文自动发现并调用。同时规范强调：

> 为了信任与安全，**应当始终保持人在回路**，用户必须能够拒绝工具调用。

对客户端的要求：

- 提供 UI 让用户清楚**哪些工具被暴露给了模型**
- 工具被调用时给出**明确的视觉指示**
- 对敏感操作**弹出确认提示**

#### 3.5.2 能力声明

```json
{ "capabilities": { "tools": { "listChanged": true } } }
```

支持 tools 的服务端**必须**声明此能力。`listChanged` 表示列表变化时会发通知。

#### 3.5.3 核心消息

**列出工具 `tools/list`**（支持分页）：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": { "cursor": "optional-cursor-value" }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "title": "Weather Information Provider",
        "description": "Get current weather information for a location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": { "type": "string", "description": "City name or zip code" }
          },
          "required": ["location"]
        }
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

**调用工具 `tools/call`**：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "location": "New York" }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      { "type": "text", "text": "Current weather in New York:\nTemperature: 72°F" }
    ],
    "isError": false
  }
}
```

**列表变化通知**：

```json
{ "jsonrpc": "2.0", "method": "notifications/tools/list_changed" }
```

#### 3.5.4 Tool 定义字段

| 字段 | 必需 | 作用 | 写法要点 |
|---|---|---|---|
| `name` | ✅ | 唯一标识，模型据此调用 | 动宾结构、语义明确：`search_docs` 而非 `query` |
| `title` | ○ | 人类可读显示名 | UI 展示用，可含 emoji |
| `description` | ✅ | 功能说明 | **决定模型选不选它**，必须写清「做什么 + 什么时候用」 |
| `inputSchema` | ✅ | 参数定义（JSON Schema） | 每个属性都要有 `description`；必填项放 `required` |
| `outputSchema` | ○ | 输出结构（JSON Schema） | 提供后**必须**返回符合该 schema 的结构化结果 |
| `annotations` | ○ | 行为提示（见 3.5.6） | 危险操作必须标注 |

**关于 `outputSchema`**：若提供，服务端**必须**返回符合该 schema 的结构化结果，客户端**应当**校验。它能带来：严格 schema 校验、类型信息、更好的文档与开发体验。

⚠️ `structuredContent` 是**服务端产出的结果数据**，与 LLM 的「结构化输出」（受 schema 约束的模型生成）**是两回事**，不要混淆。

⚠️ 兼容性要求：提供了 `outputSchema` 的工具，**应当**同时把序列化后的 JSON 放进 TextContent 块，以兼容老客户端。

#### 3.5.5 结果内容类型（content）

`content` 数组可包含多种类型的块，每种都支持可选 `annotations`：

| type | 必需字段 | 用途 |
|---|---|---|
| `text` | `text` | 纯文本，最常用 |
| `image` | `data`（base64）、`mimeType` | 图片 |
| `audio` | `data`（base64）、`mimeType` | 音频 |
| `resource_link` | `uri`、`name` | 指向资源的链接，供订阅或读取 |
| `resource` | `resource` 对象 | **内嵌**资源内容 |

**关于 `resource_link`**：给模型提供额外上下文数据的入口。客户端可订阅或拉取。注意：**工具返回的 resource_link 不保证出现在 `resources/list` 结果中**。

**关于内嵌资源**：使用内嵌资源的服务端**应当**同时实现 `resources` 能力。

**`annotations` 结构**（text/image/audio/resource_link/resource 通用）：

```json
{
  "annotations": {
    "audience": ["user"],
    "priority": 0.9,
    "lastModified": "2025-05-03T14:30:00Z"
  }
}
```

| 字段 | 取值 | 作用 |
|---|---|---|
| `audience` | `"user"` / `"assistant"` 数组 | 预期受众，客户端据此过滤 |
| `priority` | `0.0` ~ `1.0` | 重要度，1 = 几乎必需，0 = 完全可选 |
| `lastModified` | ISO 8601 时间戳 | 最后修改时间，客户端可按新旧排序 |

#### 3.5.6 Tool 注解（annotations）—— 安全防护的关键

除了通用 annotations，Tool 定义还有一个专门的 `annotations` 对象，字段均为**提示性质（Hints）**：

| 字段 | 含义 | 默认值 |
|---|---|---|
| `title` | 展示用标题 | — |
| `readOnlyHint` | 是否只读，不修改环境 | 默认 `false` |
| `destructiveHint` | 是否可能执行破坏性操作 | 默认 `true` |
| `idempotentHint` | 重复调用是否无额外副作用 | 默认 `false` |
| `openWorldHint` | 是否与开放外部系统交互 | 默认 `true` |

> 🔒 **安全铁律**：客户端**必须**将工具注解视为**不可信**，除非来自可信服务端。
>
> 这意味着：服务端说自己「只读」不能作为安全依据，真正的拦截必须由客户端/宿主侧完成。**但正因如此，服务端必须诚实标注** —— 标错等于给客户端错误的决策依据。

**标注建议**：

| 工具类型 | annotations |
|---|---|
| 查询类（list/read/get/search） | `readOnlyHint: true, destructiveHint: false, idempotentHint: true` |
| 写入类（create/update） | `destructiveHint: false, openWorldHint: false` |
| 删除类（delete/drop/truncate） | `destructiveHint: true`，描述中显式警告 |
| 幂等写入（upsert/set） | `idempotentHint: true` |

#### 3.5.7 错误处理：两套机制，别搞混

| 机制 | 用法 | 场景 |
|---|---|---|
| **协议错误**（JSON-RPC error） | 返回 `error` 对象，`code` 为标准错误码 | 工具不存在、参数非法、服务端内部错误 |
| **工具执行错误** | 正常返回 `result`，但 `isError: true` | API 调用失败、输入数据不合法、业务逻辑错误 |

**协议错误示例**：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": { "code": -32602, "message": "Unknown tool: invalid_tool_name" }
}
```

**工具执行错误示例**：

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      { "type": "text", "text": "Failed to fetch weather data: API rate limit exceeded" }
    ],
    "isError": true
  }
}
```

**判断标准**：**调用方的问题**（工具不存在、参数结构错）→ 协议错误；**执行过程的问题**（目标系统挂了、超限、业务规则不满足）→ `isError: true`。

为什么业务错误要用 `isError`？因为它能让模型**看到错误内容并自行纠正**（换个参数重试），而协议错误往往直接中断调用链。

#### 3.5.8 Tools 安全要求

**服务端 MUST**：

- 校验所有工具输入
- 实现恰当的访问控制
- 对工具调用做限流
- 对工具输出做净化

**客户端 SHOULD**：

- 敏感操作弹用户确认
- 调用前向用户展示工具输入，防止恶意的或意外的数据外泄
- 把工具结果交给 LLM 前先校验
- 为工具调用实现超时
- 记录工具使用日志以便审计

### 3.6 Resources（资源）—— 给模型提供上下文数据

Resources 允许服务端向客户端共享为模型提供上下文的数据：文件、数据库 schema、应用特定信息。每个资源由一个 **URI**（RFC 3986）唯一标识。

#### 3.6.1 交互模型：应用驱动

与 Tools 的「模型驱动」不同，Resources 设计为 **application-driven**：宿主应用决定如何把资源纳入上下文。常见做法：

- 在 UI 中以树形/列表暴露，供用户显式选择
- 允许用户搜索、过滤可用资源
- 基于启发式规则或模型选择自动注入上下文

#### 3.6.2 能力声明

```json
{ "capabilities": { "resources": { "subscribe": true, "listChanged": true } } }
```

两个子能力**都是可选的**，可以都不支持、只支持其一、或都支持：

```json
{ "capabilities": { "resources": {} } }
{ "capabilities": { "resources": { "subscribe": true } } }
{ "capabilities": { "resources": { "listChanged": true } } }
```

#### 3.6.3 核心消息

| 方法 | 作用 | 关键参数 |
|---|---|---|
| `resources/list` | 列出资源（支持分页） | `cursor` |
| `resources/read` | 读取资源内容 | `uri`（必填） |
| `resources/templates/list` | 列出资源模板（支持分页） | `cursor` |
| `resources/subscribe` | 订阅资源变更 | `uri`（必填） |
| `notifications/resources/updated` | 变更通知 | `uri` |

**读取结果结构**：返回 `contents` 数组，每项含 `uri`、`mimeType`，以及 `text` 或 `blob`（二选一）：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "contents": [
      {
        "uri": "file:///project/src/main.rs",
        "mimeType": "text/x-rust",
        "text": "fn main() { }"
      }
    ]
  }
}
```

#### 3.6.4 资源模板

用 RFC 6570 URI 模板暴露**参数化**资源，参数可通过补全 API（completion）自动补全：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/templates/list",
  "params": { "cursor": "optional-cursor-value" }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resourceTemplates": [
      {
        "uriTemplate": "file:///{path}",
        "name": "Project Files",
        "title": "📁 Project Files",
        "description": "Access files in the project directory",
        "mimeType": "application/octet-stream"
      }
    ]
  }
}
```

#### 3.6.5 常见 URI 方案

| 方案 | 用途 | 注意 |
|---|---|---|
| `https://` | 网页资源 | **仅当**客户端能自行从网络加载时才用；否则应定义自定义方案 |
| `file://` | 类文件系统资源 | 不必映射真实文件系统；可用 XDG MIME 类型表示目录等非普通文件 |
| `git://` | 版本控制集成 | — |
| 自定义 | 领域内自定义 | **必须**符合 RFC 3986 |

#### 3.6.6 错误码

| 场景 | code |
|---|---|
| 资源未找到 | `-32002` |
| 内部错误 | `-32603` |

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "error": {
    "code": -32002,
    "message": "Resource not found",
    "data": { "uri": "file:///nonexistent.txt" }
  }
}
```

#### 3.6.7 Resources 安全要求

- 服务端 **MUST** 校验所有资源 URI
- 敏感资源 **SHOULD** 实现访问控制
- 二进制数据 **MUST** 正确编码
- 操作前 **SHOULD** 检查资源权限

### 3.7 Prompts（提示模板）—— 暴露给用户的预设指令

Prompts 允许服务端向客户端暴露提示模板，提供结构化的消息和指令。客户端可发现可用 prompts、拉取其内容、传参定制。

#### 3.7.1 交互模型：用户驱动

Prompts 设计为 **user-controlled** —— 意图是让用户能显式选择使用。典型形态是作为**斜杠命令（slash command）**出现在 UI 中，便于用户自然发现和调用。

#### 3.7.2 核心消息

`prompts/list`：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "prompts": [
      {
        "name": "code_review",
        "title": "Request Code Review",
        "description": "Asks the LLM to analyze code quality and suggest improvements",
        "arguments": [
          { "name": "code", "description": "The code to review", "required": true }
        ]
      }
    ]
  }
}
```

`prompts/get`：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": { "code": "def hello():\n    print('world')" }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "description": "Code review prompt",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Please review this Python code:\ndef hello():\n    print('world')"
        }
      }
    ]
  }
}
```

#### 3.7.3 字段与错误

**Prompt 定义**：`name`（必需）、`title`（可选，展示名）、`description`（可选）、`arguments`（可选）

**PromptArgument 字段**：`name`（必需）、`description`、`required`

**PromptMessage**：`role`（`"user"` 或 `"assistant"`）+ `content`（text / image / audio / 内嵌 resource）

| 场景 | code |
|---|---|
| prompt 名称无效 | `-32602`（Invalid params） |
| 缺少必填参数 | `-32602`（Invalid params） |
| 内部错误 | `-32603` |

**安全要求**：实现**必须**仔细校验所有 prompt 的输入与输出，防止注入攻击或未授权访问资源。

### 3.8 三种能力对比：选错是最常见的设计失误

| 对比项 | Tools | Resources | Prompts |
|---|---|---|---|
| 交互模型 | **模型驱动** model-controlled | **应用驱动** application-driven | **用户驱动** user-controlled |
| 是否执行副作用 | **会**（可能改数据） | **不会**（只读） | **不会**（只产出消息） |
| 标识方式 | `name` | `uri` | `name` |
| 核心方法 | `tools/call` | `resources/read` | `prompts/get` |
| 典型用途 | 查库、调 API、写数据、跑命令 | 提供 schema、文档、配置文件、日志 | 固定分析流程、评审模板、报告框架 |
| 能力声明 | `tools` | `resources` | `prompts` |

**选择口诀**：

> **要"做"事 → Tool；要"读"资料 → Resource；要"照着做"的模板 → Prompt。**

一个常见错误是**用 Tool 返回大段静态文档**。静态资料应当走 Resource，因为它可以被缓存、被用户预览、被标注 audience，而 Tool 的每次调用都会真实执行一次。

### 3.9 分页（Pagination）

`tools/list`、`resources/list`、`resources/templates/list`、`prompts/list` **都支持分页**：

| 机制 | 说明 |
|---|---|
| 请求 | `params.cursor` 传入游标，首次请求不传或传空 |
| 响应 | 返回 `nextCursor`，无更多页时该字段**省略或为 null** |
| 约定 | 游标是**不透明**的，客户端**不得**解析其内容 |

客户端**必须**处理 `nextCursor` 循环拉取；服务端**必须**在游标失效时给出明确的错误，而非静默返回空列表。

### 3.10 鉴权（Authorization）

| 传输层 | 鉴权要求 |
|---|---|
| **HTTP 型** | **应当**遵循规范的 Authorization 框架（OAuth 2.1）；也**可以**协商自定义鉴权策略 |
| **stdio 型** | **不应**遵循该框架，**应当**从环境变量获取凭证 |

客户端配置里的 `env` / `headers` 是凭证传递的落点，**绝不要把密钥硬编码进代码或提交进仓库**。

### 3.11 辅助工具（Utilities）

| 能力 | 方法 | 作用 |
|---|---|---|
| **Ping** | `ping` | 连通性检测，初始化前唯一允许的请求 |
| **Logging** | `notifications/message` | 服务端结构化日志 |
| **Completion** | `completion/complete` | 参数自动补全（依赖 `completions` 能力） |
| **Cancellation** | `notifications/cancelled` | 显式取消一个在途请求 |
| **Progress** | `notifications/progress` | 长任务进度，可重置超时计时器 |

---

## 四、开发流程（七步法）

### 第 1 步：明确能力边界

先回答三个问题：

- **要接什么系统？**（数据库 / 内部 API / 文件系统 / SaaS）
- **模型需要"做"什么、"读"什么？** → 决定 Tools 和 Resources 的划分
- **有没有现成的？** → 先搜官方/社区 Registry，别重复造轮子

**这一步的产出是一张能力清单**，而不是代码。

### 第 2 步：选择 SDK 与传输层

| 场景 | 推荐 |
|---|---|
| Node.js / TypeScript 生态 | 官方 TypeScript SDK |
| Python 生态 | 官方 Python SDK（FastMCP） |
| 其他语言 | 手写 JSON-RPC，或找社区 SDK |
| 本地单用户 | stdio |
| 远程多用户 | Streamable HTTP + OAuth 2.1 |

**原则**：除非有明确的跨语言互操作需求，**优先用官方 SDK** —— 它已经处理了握手、版本协商、分页、超时这些细节。

### 第 3 步：定义工具契约（先于实现）

为每个工具写清楚四栏表，这是最值得花时间的一步：

| 工具名 | description | inputSchema | outputSchema |
|---|---|---|---|
| `search_docs` | 在内部文档库中按关键词检索。用于回答「XX 流程怎么走」「这个字段什么意思」类问题；不要用于查实时业务数据。 | `query`(string, 必填)、`limit`(integer, 默认 5) | `results[]` 含 uri/title/snippet |

**关键**：`description` 里要写清**不该用它做什么**。模型选错工具，八成是因为描述里没说清边界。

### 第 4 步：实现服务端与传输层

实现顺序建议：

1. `initialize` / `initialized` 生命周期
2. `tools/list` 返回静态工具列表
3. `tools/call` 实现一个工具，跑通端到端
4. 加第二个工具、第三个工具
5. 再考虑 Resources / Prompts
6. 最后处理分页、订阅、补全等进阶特性

**不要一开始就上完整功能集。** 先跑通最小闭环，再扩。

### 第 5 步：安全加固

按 3.4.3 和 3.5.8 逐条落实：

- [ ] 输入校验（schema 校验 + 业务白名单）
- [ ] 访问控制（谁能调什么）
- [ ] 限流
- [ ] 输出净化
- [ ] `Origin` 校验（HTTP 型）
- [ ] 绑定 `127.0.0.1`（本地 HTTP 型）
- [ ] 危险操作标注 `destructiveHint: true`
- [ ] 所有请求设置超时

### 第 6 步：本地联调

不要依赖手工点。推荐用 **MCP Inspector** 做端到端测试：

```bash
# 启动 Inspector
npx @modelcontextprotocol/inspector

# 指定启动命令
npx @modelcontextprotocol/inspector -- node dist/index.js
```

Inspector 提供交互式界面，可直接查看 `tools/list` 输出、手工发起 `tools/call`、观察原始 JSON-RPC 报文。

**必须验证的四件事**：

| # | 验证项 | 通过标准 |
|---|---|---|
| 1 | 握手 | `initialize` 返回能力声明，`initialized` 后无报错 |
| 2 | 列表 | `tools/list` 返回的工具元信息完整，schema 合法 |
| 3 | 调用 | 每个工具都能返回正确结果 |
| 4 | 边界 | 错误参数、超时、外部服务挂掉时行为正确 |

### 第 7 步：交付与分发

| 分发方式 | 做法 |
|---|---|
| 本地使用 | 在客户端配置 `command` + `args` + `env` |
| npm 包 | 发包，客户端用 `npx -y <pkg>` |
| 远程服务 | 部署 HTTP 服务，客户端配 `url` + `headers` |
| 团队共享 | 把配置模板写进项目 README |

**配置注意事项**：

- 路径用**绝对路径**，避免客户端工作目录不同导致找不到
- 用 `${ENV_VAR}` 引用环境变量，不要把密钥写死在配置里
- 写入位置：`~/.workbuddy/mcp.json`（**不是** `~/.workbuddy/.mcp.json`）
- 配置后**不会自动生效**，需在连接器管理页右上角的自定义连接器入口点击「信任」启用

---

## 五、协议层硬性红线

以下违反会直接导致协议不合规、连接失败或安全漏洞。

### 5.1 消息层

| # | 红线 |
|---|---|
| 1 | 所有消息必须符合 **JSON-RPC 2.0**，且 UTF-8 编码 |
| 2 | Request 的 `id` 必须为 string 或 number，**不得为 null** |
| 3 | Request 的 `id` 在**同一会话内不得重复** |
| 4 | Response 的 `id` 必须与对应请求**完全相同** |
| 5 | `result` 与 `error` **必须恰好设置一个**，不能都设或都不设 |
| 6 | `error.code` 必须是整数 |
| 7 | Notification **不得**带 `id`，接收方**不得**回复 |
| 8 | `_meta` 中以 `modelcontextprotocol` / `mcp` 为前缀的 key 为协议保留，实现不得假定其含义 |

### 5.2 生命周期

| # | 红线 |
|---|---|
| 9 | `initialize` **必须是**首次交互 |
| 10 | 收到 `initialize` 响应前，客户端**不得**发除 `ping` 外的请求 |
| 11 | 收到 `initialized` 通知前，服务端**不得**发除 `ping`/`logging` 外的请求 |
| 12 | 服务端支持请求的版本时，**必须**返回相同版本 |
| 13 | **所有**请求必须设超时，且必须有最大超时上限 |
| 14 | 断连**不应**被当作取消；取消必须显式发 `CancelledNotification` |

### 5.3 传输层

| # | 红线 |
|---|---|
| 15 | **stdio**：Server 绝不可向 `stdout` 输出非 MCP 消息（日志只能走 `stderr`） |
| 16 | **stdio**：消息按换行分隔，消息内不得含换行 |
| 17 | **HTTP**：必须提供**单一**端点路径，同时支持 POST 和 GET |
| 18 | **HTTP**：POST 必须带 `Accept: application/json, text/event-stream` |
| 19 | **HTTP**：必须校验 `Origin` 头 |
| 20 | **HTTP**：本地运行应只绑 `127.0.0.1` |
| 21 | **HTTP**：客户端后续请求必须带 `MCP-Protocol-Version` 头 |
| 22 | **HTTP**：收到无效/不支持的 `MCP-Protocol-Version` 必须返回 400 |
| 23 | **HTTP**：客户端收到带会话 ID 请求的 404 后必须重新 `initialize` |
| 24 | **HTTP**：不得向多个流广播同一条消息 |

### 5.4 能力声明

| # | 红线 |
|---|---|
| 25 | 支持某能力**必须**声明对应 capabilities |
| 26 | **不得**声明未实现的能力（客户端会调用然后失败） |
| 27 | 工具注解**必须**被视为不可信（除非来自可信服务端） |
| 28 | 提供 `outputSchema` 时**必须**返回符合该 schema 的结构化结果 |
| 29 | 提供 `outputSchema` 时**应当**同时返回序列化的 TextContent（兼容老客户端） |

### 5.5 安全

| # | 红线 |
|---|---|
| 30 | 服务端 **MUST** 校验所有工具输入 |
| 31 | 服务端 **MUST** 实现访问控制 |
| 32 | 服务端 **MUST** 限流 |
| 33 | 服务端 **MUST** 净化工具输出 |
| 34 | 服务端 **MUST** 校验所有资源 URI |
| 35 | Prompt 的输入输出 **MUST** 严格校验，防注入 |
| 36 | HTTP 型**应当**实现恰当认证；stdio 型**应当**从环境变量取凭证 |

---

## 六、案例

### 案例 A：重量级完整 Server —— `internal-infra-mcp`

**场景**：运维团队希望 AI 能查询 CMDB 中的服务器信息、检索内部文档、执行只读诊断命令。

**能力规划**：

| 需求 | 归属 | 理由 |
|---|---|---|
| 按条件查服务器 | `tools/query_hosts` | 需要查实时数据 |
| 检索内部文档 | `tools/search_docs` | 需要实时检索 |
| 执行只读诊断 | `tools/run_diagnostic` | 有副作用（执行命令），只读但需标注 |
| 服务器表结构 | `resources` | 静态 schema，供模型理解 |
| CMDB 快照 | `resources` | 静态数据 |
| 故障排查流程 | `prompts` | 固定流程模板，用户显式调用 |

#### 目录结构

```
internal-infra-mcp/
├── src/
│   ├── index.ts
│   ├── server.ts
│   ├── tools/
│   │   ├── queryHosts.ts
│   │   ├── searchDocs.ts
│   │   └── runDiagnostic.ts
│   ├── resources/
│   │   └── cmdb.ts
│   └── prompts/
│       └── troubleshoot.ts
├── package.json
├── .env.example
└── README.md
```

#### `src/server.ts` —— 能力声明

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

export function createServer(): McpServer {
  // ⚠️ 绝不传入 version 以外的额外参数给构造器，日志只能走 stderr
  const server = new McpServer(
    {
      name: "internal-infra-mcp",
      version: "1.0.0",
    },
    {
      capabilities: {
        tools: { listChanged: false },     // 工具列表运行期不变
        resources: { subscribe: false, listChanged: false },
        prompts: { listChanged: false },
        logging: {},
      },
      instructions:
        "内部基础设施查询服务。查询服务器信息用 query_hosts；查文档用 search_docs；" +
        "执行诊断命令用 run_diagnostic（只读，但会真实执行，请确认后再用）。",
    }
  );

  // ===== Tools =====

  server.registerTool(
    "query_hosts",
    {
      title: "查询服务器",
      description:
        "按条件查询 CMDB 中的服务器清单（主机名、IP、机房、状态、负责人）。" +
        "适用于回答「A 机房有哪些服务器」「谁负责 xxx 这台机器」类问题。" +
        "不要用于查询实时监控指标（那是 Prometheus 的事），也不用于查询业务数据。",
      inputSchema: {
        env: z.string().optional().describe("环境过滤：prod / staging / dev"),
        region: z.string().optional().describe("机房/区域过滤，如 A、B、C"),
        status: z.enum(["running", "stopped", "maintenance"])
          .optional()
          .describe("运行状态过滤"),
        keyword: z.string().optional().describe("主机名或 IP 模糊匹配关键词"),
        limit: z.number().int().min(1).max(200).default(20)
          .describe("返回条数上限，默认 20，最大 200"),
      },
      annotations: {
        readOnlyHint: true,
        destructiveHint: false,
        idempotentHint: true,
        openWorldHint: false,
      },
    },
    async ({ env, region, status, keyword, limit }) => {
      // 参数已在 schema 层校验，这里做业务校验
      const rows = await cmdb.queryHosts({ env, region, status, keyword, limit });

      return {
        content: [
          { type: "text" as const, text: `找到 ${rows.length} 台服务器：` },
          ...rows.map((r) => ({
            type: "text" as const,
            text: `- ${r.hostname} (${r.ip}) | ${r.region} | ${r.status} | 负责人 ${r.owner}`,
          })),
        ],
      };
    }
  );

  server.registerTool(
    "run_diagnostic",
    {
      title: "执行只读诊断命令",
      description:
        "在指定服务器上执行只读诊断命令（ping / df / free / uptime）。" +
        "适用于回答「这台机器磁盘满了没」「服务是否还活着」类问题。" +
        "命令会真实执行，因此目标主机必须已通过 query_hosts 确认存在。" +
        "严禁用于任何写操作，命令白名单外的请求会被拒绝。",
      inputSchema: {
        hostname: z.string().describe("目标服务器主机名，必须存在于 CMDB"),
        command: z
          .enum(["ping", "df", "free", "uptime"])
          .describe("诊断命令，仅支持白名单内的四个"),
      },
      // ⚠️ 真实执行命令 → 不能标 readOnlyHint，标了客户端会免确认
      annotations: {
        readOnlyHint: false,
        destructiveHint: false,
        idempotentHint: false,
        openWorldHint: false,
      },
    },
    async ({ hostname, command, extra }) => {
      // 业务层二次校验
      const host = await cmdb.getHost(hostname);
      if (!host) {
        // 调用方的问题 → 用协议错误（-32602 由 SDK 自动映射）
        throw new McpError(
          ErrorCode.InvalidParams,
          `主机 ${hostname} 不存在于 CMDB，请先用 query_hosts 确认`
        );
      }

      try {
        const stdout = await executor.run(hostname, command);
        return { content: [{ type: "text" as const, text: stdout }] };
      } catch (e) {
        // 执行过程的问题 → 用 isError，让模型能自行纠正
        return {
          isError: true,
          content: [
            {
              type: "text" as const,
              text: `诊断执行失败：${(e as Error).message}。该主机可能离线或 SSH 不可达。`,
            },
          ],
        };
      }
    }
  );

  // ===== Resources =====

  server.registerResource(
    "cmdb-schema",
    "infra://cmdb/schema",
    {
      title: "CMDB 表结构",
      description: "CMDB 数据库的表结构定义，查询前可读取以了解可用字段",
      mimeType: "application/json",
    },
    async (uri) => ({
      contents: [
        {
          uri: uri.href,
          mimeType: "application/json",
          text: JSON.stringify(cmdbSchema, null, 2),
        },
      ],
    })
  );

  // ===== Prompts =====

  server.registerPrompt(
    "troubleshoot_host",
    {
      title: "主机故障排查",
      description: "按标准流程排查一台服务器的故障",
      argsSchema: { hostname: z.string().describe("待排查的主机名") },
    },
    ({ hostname }) => ({
      description: `排查 ${hostname} 的故障`,
      messages: [
        {
          role: "user",
          content: {
            type: "text",
            text:
              `请排查主机 ${hostname} 的问题。按以下顺序执行，每一步都要说明观察到什么：\n` +
              `1. 用 query_hosts 确认该主机在 CMDB 中的状态与负责人\n` +
              `2. 用 run_diagnostic 执行 uptime，判断是否存活\n` +
              `3. 用 run_diagnostic 执行 df，检查磁盘水位\n` +
              `4. 用 run_diagnostic 执行 free，检查内存\n` +
              `5. 用 search_docs 检索该主机相关的历史故障记录\n` +
              `6. 汇总判断根因，给出处置建议；若信息不足，明确说明还缺什么，不要编造`,
          },
        },
      ],
    })
  );

  return server;
}

// ⚠️ 入口只做启动，绝不能有 console.log —— 会污染 stdout
async function main() {
  const server = createServer();
  const transport = new StdioServerTransport();
  await server.connect(transport);
  process.stderr.write("internal-infra-mcp started\n");
}

main().catch((err) => {
  process.stderr.write(`Fatal: ${err?.stack ?? err}\n`);
  process.exit(1);
});
```

> 注：`run_diagnostic` 的实现里，`extra` 参数并未在 inputSchema 中声明，这正是**必须做业务层二次校验**的原因 —— 客户端只保证按 schema 传参，但恶意构造的请求可能绕过。

#### 客户端配置

```json
{
  "mcpServers": {
    "internal-infra-mcp": {
      "command": "node",
      "args": ["D:/tools/internal-infra-mcp/dist/index.js"],
      "env": {
        "CMDB_DSN": "${CMDB_DSN}",
        "SSH_JUMP_HOST": "${SSH_JUMP_HOST}"
      }
    }
  }
}
```

### 案例 B：轻量级 Server —— `note-taker-mcp`

**场景**：个人用户想让 AI 记录和管理 Markdown 笔记。**只有 Tools，没有 Resources 和 Prompts**。全文如下：

```typescript
#!/usr/bin/env node
/**
 * 极简 MCP Server 示例：Markdown 笔记管理。
 *
 * 演示内容：
 *   - 最小可用的能力声明
 *   - 一个只读工具 + 一个写入工具
 *   - 正确的 stdout 纪律（日志走 stderr）
 *
 * 依赖：npm i @modelcontextprotocol/sdk zod
 */

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { readFile, writeFile, readdir, mkdir } from "node:fs/promises";
import { join, resolve } from "node:path";

const NOTES_DIR = resolve(process.env.NOTES_DIR ?? "./notes");

const server = new McpServer(
  { name: "note-taker-mcp", version: "0.1.0" },
  {
    capabilities: { tools: {} },   // 只声明 tools，其余不声明就不实现
    instructions: "Markdown 笔记管理。list 列出笔记，read 读笔记，write 写笔记。",
  }
);

server.registerTool(
  "read_note",
  {
    title: "读取笔记",
    description: "按文件名读取一篇 Markdown 笔记的完整内容。",
    inputSchema: { filename: z.string().describe("笔记文件名，含 .md 后缀") },
    annotations: { readOnlyHint: true, destructiveHint: false, idempotentHint: true },
  },
  async ({ filename }) => {
    const path = safeJoin(filename);
    try {
      const text = await readFile(path, "utf-8");
      return { content: [{ type: "text" as const, text }] };
    } catch (e) {
      return {
        isError: true,
        content: [{ type: "text" as const, text: `读取失败：${(e as Error).message}` }],
      };
    }
  }
);

server.registerTool(
  "write_note",
  {
    title: "写入笔记",
    description: "写入或覆盖一篇 Markdown 笔记。同名文件会被覆盖。",
    inputSchema: {
      filename: z.string().describe("笔记文件名，含 .md 后缀"),
      content: z.string().describe("笔记的完整 Markdown 内容"),
    },
    // 写操作 → 客户端应提示用户确认
    annotations: { readOnlyHint: false, destructiveHint: true, idempotentHint: true },
  },
  async ({ filename, content }) => {
    const path = safeJoin(filename);
    await mkdir(NOTES_DIR, { recursive: true });
    await writeFile(path, content, "utf-8");
    return { content: [{ type: "text" as const, text: `已写入 ${filename}` }] };
  }
);

/** 防路径穿越：拒绝 .. 和绝对路径 */
function safeJoin(filename: string): string {
  const path = join(NOTES_DIR, filename);
  if (!path.startsWith(NOTES_DIR)) {
    throw new Error(`非法路径：${filename}`);
  }
  return path;
}

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  process.stderr.write("note-taker-mcp ready\n");
}

main().catch((e) => {
  process.stderr.write(`${e}\n`);
  process.exit(1);
});
```

**这个案例说明什么**：

- **能只实现 Tools 就只实现 Tools** —— 能力声明写 `{ tools: {} }` 即可，不要为了"显得完整"声明用不到的能力
- `safeJoin` 这类**路径校验**不是可选项 —— `filename: "../../.ssh/id_rsa"` 是最经典的攻击向量
- `write_note` 标了 `destructiveHint: true`，客户端就会在覆盖前提示用户

### 案例 C：工具描述写法对比

| 写法 | 示例 | 评价 |
|---|---|---|
| ❌ **只有动词** | `description: "执行诊断"` | 模型不知道什么时候该用，也不知道边界在哪 |
| ⚠️ **有场景但无边界** | `description: "用于查询服务器信息，执行运维诊断命令"` | 仍可能被误用（比如拿它查业务数据） |
| ✅ **做什么 + 什么时候用 + 边界** | `description: "在指定服务器上执行只读诊断命令（ping/df/free/uptime）。适用于回答「磁盘满了没」「服务是否活着」类问题。命令会真实执行，目标主机须先通过 query_hosts 确认存在。严禁用于任何写操作。"` | 覆盖动作、场景、前置条件、禁止事项，模型选型准确 |

### 案例 D：工具粒度对照

| 需求 | ❌ 反模式 | ✅ 正确粒度 |
|---|---|---|
| 查服务器 | `query(type, payload)` 万能工具 | `query_hosts(filters)` 一个工具一种语义 |
| 改配置 | `execute(cmd, args)` 通用执行器 | `update_ntp_config(host, server)` 专用工具 + 严格校验 |
| 导出数据 | `export(format, scope)` | `export_hosts_csv(filters)` / `export_hosts_json(filters)` |

**通用工具是 MCP 最常见的坏味道**：模型的参数填充准确率远低于人类，参数一多就容易填错。更细粒度的工具虽然数量多，但每次调用的成功率更高。

### 案例 E：两种错误处理写法对照

```typescript
// ❌ 错误：把业务失败当协议错误抛出 —— 模型看不到原因，无法自我纠正
if (!host) throw new McpError(ErrorCode.InvalidParams, "主机不存在");

// ✅ 正确：调用方问题用协议错误
if (!host) {
  throw new McpError(
    ErrorCode.InvalidParams,
    `主机 ${hostname} 不存在于 CMDB，请先用 query_hosts 确认`
  );
}

// ✅ 正确：执行过程问题用 isError —— 模型能读到原因并调整
try {
  const out = await executor.run(hostname, command);
  return { content: [{ type: "text", text: out }] };
} catch (e) {
  return {
    isError: true,
    content: [{
      type: "text",
      text: `诊断失败：${(e as Error).message}。该主机可能离线或 SSH 不可达。`,
    }],
  };
}
```

**两者的差别是模型能不能自我恢复。** 错误信息要写成「发生了什么 + 下一步怎么办」，而不是只说「失败了」。

---

## 七、常见错误清单

| # | 错误 | 后果 | 修正 |
|---|---|---|---|
| 1 | stdio 型用 `console.log` 输出启动日志 | **stdout 被污染，客户端解析失败，连接直接断** | 所有日志走 `stderr` |
| 2 | 消息内容含换行符 | 换行分帧被破坏，解析错乱 | JSON 序列化保证单行 |
| 3 | 声明了未实现的能力 | 客户端调用后失败，体验极差 | 只声明实际实现的 |
| 4 | 未实现 `initialize` 阶段 | 客户端无法完成握手 | 先跑通生命周期 |
| 5 | 所有错误都抛 `McpError` | 模型无法自我纠正 | 业务失败用 `isError: true` |
| 6 | 错误信息只说「失败了」 | 模型不知道该怎么办 | 写清「原因 + 下一步」 |
| 7 | 工具描述只有动词，无边界 | 模型频繁选错工具 | 补齐场景、前置条件、禁止事项 |
| 8 | 万能工具 / 通用执行器 | 参数填充错误率高 | 一个工具一种语义 |
| 9 | `run_command(cmd: string)` 不做白名单 | **命令注入，任意命令执行** | 白名单枚举 + 路径/参数校验 |
| 10 | 不校验路径，信任 `filename` | 路径穿越读到任意文件 | `safeJoin` 类校验 + 符号链接防护 |
| 11 | HTTP 型不校验 `Origin` | DNS rebinding 攻击 | 强制校验 Origin |
| 12 | 本地 HTTP 服务绑 `0.0.0.0` | 暴露到局域网 | 绑 `127.0.0.1` |
| 13 | 所有请求都不设超时 | 客户端卡死 | 设默认超时 + 最大超时上限 |
| 14 | 危险操作标 `readOnlyHint: true` | 客户端免确认直接执行 | 诚实标注 `destructiveHint` |
| 15 | 密钥硬编码进代码或配置 | 安全泄露 | 环境变量 + `${VAR}` 引用 |
| 16 | 提供 `outputSchema` 但返回非结构化数据 | 客户端校验失败 | 遵守 schema + 兼容返回 TextContent |
| 17 | 忽略 `nextCursor` 无限循环 | 客户端卡死或数据不全 | 循环拉取直到无 `nextCursor` |
| 18 | 把游标当字符串解析 | 破坏不透明约定 | 游标只做透传 |
| 19 | 静态文档用 Tool 返回而非 Resource | 每次调用真实执行，无法缓存/预览 | 静态资料走 Resources |
| 20 | 断连当作取消处理 | 重复执行任务 | 显式 `CancelledNotification` |
| 21 | HTTP 后续请求不带 `MCP-Protocol-Version` | 老服务端行为不确定 | 每请求必带该头 |
| 22 | 会话失效返回空列表而非 404 | 客户端无法感知需重建会话 | 返回 404 让客户端重新 initialize |
| 23 | SSE 事件 `id` 跨流重复 | 恢复时消息错乱 | id 在会话内全局唯一 |
| 24 | 不看 `initiator`/`_meta` 保留前缀 | 未来版本语义冲突 | 保留前缀不假定含义 |

---

## 八、发布前检查清单

### 协议合规

- [ ] 所有消息符合 JSON-RPC 2.0，UTF-8 编码
- [ ] Request 的 `id` 非 null 且会话内唯一
- [ ] Response 的 `id` 与请求匹配，`result` / `error` 恰好一个
- [ ] Notification 不带 `id`
- [ ] `initialize` → `initialized` 顺序正确
- [ ] 声明的 capabilities 全部已实现
- [ ] 支持的协议版本范围内返回正确版本
- [ ] `MCP-Protocol-Version` 头正确处理（HTTP 型）
- [ ] 分页游标正确处理，未将游标当字符串解析

### 传输层

- [ ] **stdio**：无任何非 MCP 内容写入 `stdout`（已全局检查 `console.log`）
- [ ] **stdio**：日志全部走 `stderr`
- [ ] **stdio**：消息单行无嵌入换行
- [ ] **HTTP**：单一端点同时支持 POST 和 GET
- [ ] **HTTP**：POST 带 `Accept: application/json, text/event-stream`
- [ ] **HTTP**：接收 response/notification 返回 202 无 body
- [ ] **HTTP**：GET 不支持 SSE 时返回 405
- [ ] **HTTP**：不向多个流广播同一条消息

### 会话与超时

- [ ] 所有请求设置超时，且有最大超时上限
- [ ] 超时后正确处理（取消 + 停止等待）
- [ ] 正确处理取消通知
- [ ] 会话 ID 唯一、密码学安全、仅含可见 ASCII
- [ ] 会话失效返回 404，客户端能重新 initialize
- [ ] SSE 事件 `id` 在会话内全局唯一，支持 `Last-Event-ID` 恢复

### 工具设计

- [ ] 每个工具的 `description` 含「做什么 + 何时用 + 边界」
- [ ] 每个 `inputSchema` 属性都有 `description`
- [ ] 必填参数正确列入 `required`
- [ ] 提供 `outputSchema` 的工具返回了合规的结构化结果
- [ ] 提供 `outputSchema` 时同时返回序列化 TextContent
- [ ] 所有工具标注了 `annotations`
- [ ] 写操作/删除操作标注 `destructiveHint: true`
- [ ] 工具粒度合理，无万能工具

### 错误处理

- [ ] 调用方问题 → JSON-RPC 协议错误
- [ ] 执行过程问题 → `isError: true`
- [ ] 错误信息含「原因 + 下一步建议」
- [ ] 资源未找到用 `-32002`，内部错误用 `-32603`

### 安全

- [ ] 校验所有工具输入
- [ ] 实现访问控制
- [ ] 实现限流
- [ ] 净化工具输出
- [ ] 校验所有资源 URI
- [ ] **HTTP 型校验 `Origin` 头**
- [ ] **本地 HTTP 服务只绑 `127.0.0.1`**
- [ ] 危险命令走白名单，不接受任意命令字符串
- [ ] 文件路径做了穿越校验（`..`、绝对路径、符号链接）
- [ ] 凭证从环境变量读取，未硬编码、未提交 `.env`
- [ ] Prompt 输入输出做了注入防护

### 测试与交付

- [ ] 用 MCP Inspector 完成端到端验证
- [ ] 验证了：握手 / 列表 / 调用 / 边界 四项
- [ ] 客户端配置使用绝对路径
- [ ] 配置中密钥用 `${VAR}` 引用
- [ ] README 说明安装方式、工具清单、所需环境变量
- [ ] 权限请求策略已配置（`mcp-approvals`）

---

## 附：三条速记原则

> **1. 协议层：诚实比聪明重要。**
> 能力声明、注解、版本号——如实反映你的实现。谎报 `readOnlyHint` 只会让客户端基于错误依据做决策。
>
> **2. 设计层：`description` 是你对模型的唯一说明书。**
> 模型看不到你的代码，只看得到 `name` 和 `description`。写清边界比写清功能更有价值。
>
> **3. 安全层：Server 是最后一道防线。**
> 客户端的审批提示可能被绕过，注解也可能被伪造。所有校验、鉴权、限流、路径防护，必须在 Server 侧自己完成。
