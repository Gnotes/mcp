# MCP 基础：写给第一次实现协议的人

配套文档 · 2026-11-11 · 依据 [MCP 规范 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28)

本文记录实现 `mcp` 前必须先搞清的概念，是补课材料：协议怎么演化来的、两种传输差在哪、方法名和错误码有没有规矩、模型怎么知道你有什么工具。

---

## 1. MCP 解决的是什么问题

一个 AI 应用想读文件、查数据库、调接口，如果每种能力都写死在自己的代码里，就会变成 M 个应用乘 N 个工具的组合爆炸。**MCP(Model Context Protocol, 模型上下文协议)** 做的事是把中间那一层标准化：应用只实现一次客户端，工具只实现一次服务端，两边用同一套报文对话。

两个成熟的类比能帮上忙：

- **USB-C**：以前每个设备一种接口，现在设备和服务共用一种插头。
- **LSP(Language Server Protocol, 语言服务器协议)**：VS Code 不需要为每门语言写补全逻辑，语言的作者提供一个统一协议的服务端即可。MCP 是同一个思路，只是把"语言"换成了"工具与数据"。

一句话记法：**MCP 是 AI 应用和外部能力之间的插头标准。**

---

## 2. 一条演进线：五个版本，四次纠错

理解版本史不是为了背日期，而是为了知道**为什么**现在的写法长这样。很多教程还停在旧版，你按旧版写会连不上新客户端。

| 版本                 | 关键动作                                                   | 当时的痛点                                                                                      |
| ------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| 2024-11-05         | 首版：`stdio` + `HTTP + SSE` 双端点                          | 建立最小可用协议                                                                                   |
| 2025-03-26         | 废双端点，换 **Streamable HTTP**；引入 OAuth 2.1                | `SSE` 要求一条连接常驻：无服务器函数留不住，`CDN(Content Delivery Network, 内容分发网络)` 和反向代理会缓冲或掐断，每会话一条连接导致扩容昂贵 |
| 2025-06-18         | **撤掉** JSON-RPC 批处理；加结构化工具输出、征询机制、`Resource Indicator` | 批处理省下的往返，换来部分失败、超时粒度、取消传播的全套复杂度；工具只能返回纯文本，模型得从字符串里猜结构                                      |
| 2025-11-25         | OIDC 发现、增量授权、实验性任务、URL 模式征询                            | 企业已有 `IdP(Identity Provider, 身份提供商)` 不想重造；一次性索要全部权限没人敢点同意；长任务不能占着连接干等                      |
| **2026-07-28（当前）** | **删会话、删握手**；MRTR；头路由；列表可缓存；扩展框架；授权加固                   | 有状态等于必须粘性路由或共享存储，滚动发布与水平扩容都受限；握手在无服务器环境是纯负担                                                |

三条能直接指导你写代码的结论：

1. **每一版都在让 MCP 更像普通 HTTP 服务**，而不是像一个长期保持连接的会话。2026-07-28 是这条路的终点：协议本身不再记得你是谁。
2. **"撤掉批处理"是罕见的回滚**。它说明协议设计里"省往返"常常不如"省复杂度"。
3. **废弃有 12 个月窗口**。规范给了正式弃用策略，所以看到旧写法不必惊讶，但新实现不该再用。

---

## 3. 两层结构：数据层与传输层

MCP 把问题劈成两半，这是理解一切的前提。

```text
┌──────────────────────────────────────────┐
│ 数据层（协议语义，所有传输上完全一样）        │
│   JSON-RPC 2.0 报文                       │
│   request / response / notification       │
│   方法名: server/discover, tools/call ...  │
├──────────────────────────────────────────┤
│ 传输层（怎么把字节送过去）                   │
│   stdio  |  Streamable HTTP               │
└──────────────────────────────────────────┘
```

**换传输不改变报文含义**。同样一条 `tools/call`，走 stdio 和走 HTTP，body 完全一样，差别只在"谁负责分帧、怎么取消、鉴权放在哪"。

### 3.1 三种消息

| 类型           | 有无 `id`           | 要不要回           | 例子                                 |
| ------------ | ----------------- | -------------- | ---------------------------------- |
| Request      | 有，且**不能为 `null`** | 必须回一个 response | `tools/call`                       |
| Response     | 与请求同 `id`         | —              | `{ result }` 或 `{ error }`         |
| Notification | 没有                | 不回             | `notifications/tools/list_changed` |

`id` 不能为 `null` 是 MCP 对 JSON-RPC 2.0 的**加严**：标准允许 `null` 表示"id 读不出来"，MCP 直接禁掉，避免客户端拿到无法对应的响应。

### 3.2 结果必须带 `resultType`

2026-07-28 起，任何 `result` 都必须有 `resultType` 字段：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { "resultType": "complete", "tools": [] }
}
```

#### 取值全集

|                                              |                                                                                                                                                                  |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 规范只定义了两个值，且类型签名是 "complete"、"input_required" | <span style="color:rgb(154, 56, 215)">ResultType</span>: "complete" | "input_required" | <span style="color:rgb(248, 136, 37)">string</span>，属于开放枚举，将来加新值不需要改类型。 |

| 取值               | 含义                      | 额外字段                                       | 允许出现在哪些方法上                                    |
| ---------------- | ----------------------- | ------------------------------------------ | --------------------------------------------- |
| `complete`       | 请求成功完成，`result` 里就是最终内容 | 由方法自己决定（`tools`、`content` 等）               | 所有方法                                          |
| `input_required` | 服务端需要客户端补充信息才能完成        | `inputRequests` 与 `requestState`，**至少给一个** | 仅 `prompts/get`、`resources/read`、`tools/call` |

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "which_floor": {
        "method": "elicitation/create",
        "params": { "mode": "form", "message": "查哪一层？", "requestedSchema": { "type": "object" } }
      }
    },
    "requestState": "AEAD-protected blob"
  }
}
```

#### 五条硬规则

1. **本版服务端必须带**；反过来，客户端收到旧版服务端（没这个字段）的结果时，**必须按 `complete` 处理**——这是唯一允许缺省的场景。
2. **`input_required` 有方法白名单**。除了 `prompts/get`、`resources/read`、`tools/call` 这三个，服务端在其他任何方法上都 **MUST NOT** 返回它。所以 `tools/list`、`server/discover` 只能回 `complete`。
3. **`inputRequests` 与 `requestState` 至少给一个**。两个都空的 `input_required` 是非法的。
4. **不能请求客户端没声明过的能力**。客户端 `capabilities` 里没有 `elicitation`，服务端就不许在 `inputRequests` 里放 `elicitation/create`。
5. **重试是一次全新请求**：JSON-RPC `id` 必须换新的，`requestState` 必须原样回传（客户端不许解析它），补充的答案放 `inputResponses`。

#### 三个容易混的点

- **`error` 不是 `resultType` 的取值**。响应只有两条互斥分支：`result` 或 `error`。业务失败用 `result` + `isError: true`，协议失败走 `error`，永远不存在 `resultType: "error"`。
- **没有 `partial`、`stream`、`progress` 这些值**。流式是传输层的 SSE 分帧，进度是 `notifications/progress` 通知，都不是结果类型。
- **遇到不认识的 `resultType` 怎么办**：因为枚举开放，客户端应当能容忍未知值；服务端则不应发明非 `complete` 的自定义值——那等于要求客户端理解你私有的返回结构。

#### 对本服务的含义

我们的 7 个工具全是"查一段数据、一次给完"，**只会返回 `complete`**。所以 `dispatch` 出口统一补 `resultType: "complete"`（见 `src/protocol/dispatch.ts`），handler 不用各自记。

`input_required` 在我们这里没有使用场景——它解决的是"参数不够时要问用户"，而我们的模型调用方本来就会补齐参数；真要问，也是在工具的错误文本里说清缺什么，让模型自己重试。

---

## 4. 两种传输

### 4.1 stdio

客户端把服务端当子进程拉起，报文走标准输入输出，一行一条 JSON。

```text
Client ──spawn──> Server
       ──stdin──>  {"jsonrpc":"2.0","id":1,"method":"tools/list"}
       <──stdout── {"jsonrpc":"2.0","id":1,"result":{...}}
       <──stderr── 日志（不能走 stdout）
```

要点：

- **`stdout` 只能放协议消息，日志必须走 `stderr`**。`console.log` 打一行调试信息，客户端就收到非法 JSON 并断开——这是 stdio 实现失败率最高的原因
- 管道天然独占，一个客户端一个进程，**并发问题自动消失**，但两个客户端看到的是两份独立状态
- 继承宿主的工作目录、环境变量和文件权限，**不需要另做鉴权**

### 4.2 Streamable HTTP

单个端点，每次 **POST**，响应可以是 JSON 或请求作用域的 SSE 流。

```text
Client ──POST /mcp──> Server
       <──application/json── 一问一答
       或
       <──text/event-stream── 边算边推（可选）
```

要点：

- 名字里的 "Streamable" 意思是**需要流才流**，不是强制长连接。绝大多数调用是一问一答
- 2026-07-28 起不再有协议级会话，**任何请求可以落到任何实例**，普通轮询负载均衡即可
- 暴露到网络就必须处理鉴权（规范推荐 OAuth 2.1，2026 版还加了 RFC 9207 issuer 校验、从动态注册转向 CIMD 客户端元数据文档）

### 4.3 对比

| 维度   | stdio     | Streamable HTTP   |
| ---- | --------- | ----------------- |
| 通信范围 | 同机父子进程    | 任意网络              |
| 谁启动  | 客户端自动拉起   | 需自行常驻运维           |
| 鉴权成本 | 零（继承系统权限） | 高                 |
| 并发模型 | 天然独占      | 需自行处理             |
| 状态持有 | 进程内存即状态   | 2026-07-28 起必须无状态 |
| 冷启动  | 每次连接重启进程  | 常驻，无冷启动           |
| 传大结果 | 管道直传      | 受超时、体积、代理限制       |
| 团队共享 | 做不到       | 唯一选择              |

一句话记法：**stdio 是"插件"，Streamable HTTP 是"服务"。**

### 4.4 为什么只能 POST

MCP 在传输层是 **RPC(Remote Procedure Call, 远程过程调用)** 风格，不是 **REST(Representational State Transfer, 表述性状态转移)** 风格。REST 把"要做什么"编码在 URL 和 HTTP 动词里；RPC 把"要做什么"编码在 body 的 `method` 字段里。MCP 选了后者，所以只有一个端点。

| 方法          | 为什么不用                               |
| ----------- | ----------------------------------- |
| GET         | 语义上必须安全且幂等，意味着可被缓存、可被预取、可被代理重放      |
| PUT / PATCH | 语义是修改资源，用来描述一次查询是错位，还会误导网关审计规则      |
| DELETE      | 旧代用来终止会话，2026-07-28 删掉会话后打过去是 `405` |
| WebSocket   | 要协议升级、要连接常驻、要粘性路由，等于把刚去掉的有状态性加回来    |

**一个反直觉的点：只读查询也不该用 GET。** HTTP 缓存按 URL 做键，一旦 `tools/call` 能通过 GET 访问，共享缓存或企业代理可能把第一个人的查询结果返回给第二个人。POST 的响应默认不可缓存，天然避开。

好处是：**鉴权、限流、审计、错误映射只需要写一次**，加新工具不动路由。

### 4.5 传输层硬性要求（2026-07-28）

这几条是新手最容易漏的，漏了客户端直接连不上。

| 项                        | 要求                                                                             | 违反的后果            |
| ------------------------ | ------------------------------------------------------------------------------ | ---------------- |
| `MCP-Protocol-Version` 头 | 所有 POST 必带，值与 body `_meta` 中的版本一致                                              | `400` + `-32020` |
| `Mcp-Method` 头           | 所有 POST 必带，值为 body 的 `method`                                                  | `400` + `-32020` |
| `Mcp-Name` 头             | `tools/call`、`resources/read`、`prompts/get` 必带，值为 `params.name` 或 `params.uri` | `400` + `-32020` |
| `Accept` 头               | 客户端应同时列出 `application/json` 与 `text/event-stream`                              | 服务端可能拒           |
| `_meta` 必填字段             | `protocolVersion` 与 `clientCapabilities`                                       | `400` + `-32602` |
| `Origin` 校验              | 服务端必须校验                                                                        | 无效时 `403`        |
| `GET` / `DELETE` 到端点     | 不支持                                                                            | `405`            |
| 无 `id` 的通知               | 受理                                                                             | `202`，无响应体       |
| `Mcp-Session-Id`         | 收到就忽略，不生成也不回显                                                                  | —                |
| 取消                       | 客户端关闭响应流                                                                       | 服务端应停止处理且不再发消息   |

`Mcp-Method` / `Mcp-Name` 是 2026-07-28 新增的**路由头**，目的是让网关、限流器、WAF 不解析 body 就能识别调用。**2025 年的教程里没有这两个头。**

---

## 5. 原语（primitives）

"原语"是协议设计里的常用词，指**最小的、不可再分的构建单元**。协议只定义几种零件，复杂行为由它们组合出来。

MCP 的原语有个特别之处：**按"控制权归谁"划分，而不是按功能。**

| 原语                | 谁提供 | 谁决定何时用       | 类比      |
| ----------------- | --- | ------------ | ------- |
| **Tools**         | 服务端 | **模型**自主决定   | 函数调用    |
| **Resources**     | 服务端 | **应用/客户端**决定 | 文件、只读数据 |
| **Prompts**       | 服务端 | **用户**主动选    | 斜杠命令    |
| **Elicitation**   | 客户端 | 服务端请求用户补充    | 弹窗问一句   |
| **Subscriptions** | 服务端 | 客户端订阅通知      | 变更推送    |

两个已废弃：**Sampling**（服务端借用宿主模型）与 **Logging**（服务端向客户端发日志），2026-07-28 起不建议新实现使用，前者改为直接调用模型厂商接口，后者改为写 `stderr` 或 OpenTelemetry。

### 5.1 为什么非要这样分

因为如果只按功能划分，所有东西都能做成 Tool。MCP 偏要拆开，理由是：**每次 Tool 调用，模型都要做一次决策、消耗一次上下文；而 Resource 是"摆在那儿的材料"，应用可以直接塞进上下文，模型不用做决定。**


具体到一个例子：你想给模型一份"这个数据集的高频属性名清单"。

- 做成 Tool `get_manual`：模型得先想到"我该去取手册"，再调一次。**它可能根本想不到**，于是继续瞎猜字段名。
- 做成 Resource `parquet://manual/properties`：客户端可在会话开始时就加载，或模型需要时直接读，没有"要不要用"的决策负担。

反向也成立：需要**执行**的动作不能做成 Resource，Resource 是只读的，没有执行语义。

---

## 6. 方法命名

### 6.1 是约定，不是随手起

`tools/list`、`tools/call` 这些**不是可自定义的名字，是规范硬编码的 JSON-RPC 方法名，一个字母都不能改。**

```text
JSON-RPC method: "tools/call"     ← 协议规定，固定
      params.name: "filter_rows"  ← 你起的工具名，规范不关心
  params.arguments: { ... }       ← 你定的参数
```

模型通过 `tools/list` 知道你有哪些工具，但调用时走的永远是 `tools/call` 这个通道。**无论你有一百个工具还是一个，方法名只有一个。**

### 6.2 格式规律

- 结构固定为 `对象/动作`，斜杠分隔，**对象常用复数**
- 动作用小写：`list`、`call`、`read`、`get`
- **大小写敏感**，`Tools/List` 无效
- 通知以 `notifications/` 开头

### 6.3 完整方法清单

| 原语            | 方法                                                           | 谁实现         |
| ------------- | ------------------------------------------------------------ | ----------- |
| Discovery     | `server/discover`                                            | 服务端（**必须**） |
| Tools         | `tools/list`、`tools/call`                                    | 服务端         |
| Resources     | `resources/list`、`resources/read`、`resources/templates/list` | 服务端         |
| Prompts       | `prompts/list`、`prompts/get`                                 | 服务端         |
| Subscriptions | `subscriptions/listen`                                       | 服务端         |
| Utilities     | `completion/complete`                                        | 可选          |

一个数据库类服务端，实际只需要写 5 个方法：`server/discover`、`tools/list`、`tools/call`、`resources/list`、`resources/read`。

### 6.4 工具名本身的约定

规范不强制，但社区惯例是 `snake_case`，动词开头。名字要能被模型"望文生义"，并且**同族工具后缀对称**——`get_digest_value` 与 `get_full_value` 放在一起，模型立刻知道是同一个值的两种粒度；若写成 `get_digest` 与 `get_full_value`，一个带 `value` 一个不带，它会犹豫这两个是不是一类东西。

---

## 7. 一次完整调用长什么样

下面三段是 2026-07-28 的写法。注意每个请求都带了四个头，body 里有 `_meta`。

### 7.1 发现

```bash
curl -sS -X POST http://127.0.0.1:8788/mcp \
  -H 'Authorization: Bearer dev-token' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2026-07-28' \
  -H 'Mcp-Method: server/discover' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "server/discover",
    "params": {
      "_meta": {
        "io.modelcontextprotocol/protocolVersion": "2026-07-28",
        "io.modelcontextprotocol/clientInfo": { "name": "curl", "version": "1.0" },
        "io.modelcontextprotocol/clientCapabilities": {}
      }
    }
  }'
```

响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "complete",
    "supportedVersions": ["2026-07-28"],
    "capabilities": { "tools": {}, "resources": {} },
    "instructions": "按 list_datasets → describe_table → filter_rows 的顺序使用。",
    "ttlMs": 300000,
    "cacheScope": "public"
  }
}
```

`server/discover` 是**必须实现**的方法，但它对客户端是**可选调用**——客户端也可以直接发 `tools/list`。它的价值在于一次往返拿到版本、能力、身份和使用说明。

`instructions` 字段值得重视：它是你唯一能对模型说"请按这个顺序用我的工具"的地方。

### 7.2 列工具

```bash
  -H 'Mcp-Method: tools/list' \
  -d '{ "jsonrpc":"2.0","id":2,"method":"tools/list","params":{ "_meta": { ... } } }'
```

```json
{
  "result": {
    "resultType": "complete",
    "tools": [
      {
        "name": "filter_rows",
        "description": "...",
        "inputSchema": { "type": "object" }
      }
    ],
    "ttlMs": 300000,
    "cacheScope": "public"
  }
}
```

`ttlMs` 与 `cacheScope` 让客户端缓存工具清单，也提高上游提示词缓存命中率。工具清单是静态的，给 5 分钟合理。**顺序必须确定**，否则每次缓存都失效。

### 7.3 调工具

```bash
  -H 'Mcp-Method: tools/call' \
  -H 'Mcp-Name: filter_rows' \
  -d '{ "jsonrpc":"2.0","id":3,"method":"tools/call",
        "params": { "name":"filter_rows", "arguments": { "limit": 50 }, "_meta": { ... } } }'
```

```json
{
  "result": {
    "resultType": "complete",
    "content": [{ "type": "text", "text": "..." }],
    "structuredContent": { "rows": [], "meta": {} },
    "isError": false
  }
}
```

`structuredContent` 是可校验的结构化结果，`content` 里的文本是给模型读的兼容形态。两者都给最稳妥。

### 7.4 多轮往返（MRTR）

工具执行到一半需要用户确认（"查哪一层？"）或需要借用宿主的模型时，服务端不占着连接干等，而是返回：

```json
{
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "which_floor": {
        "method": "elicitation/create",
        "params": { "mode": "form", "message": "查哪一层？", "requestedSchema": { "type": "object" } }
      }
    },
    "requestState": "AEAD-protected blob"
  }
}
```

`inputRequests` 是 **map ，不是数组**：键由服务端自己起，值是 `elicitation/create`、`sampling/createMessage`、`roots/list` 三者之一。写成数组是最常见的错误。

客户端把答复按同样的键放进 `inputResponses`，连同原样回传的 `requestState` **重发一次新请求**（`id` 必须换新的，这是两次独立请求）。服务端把需要延续的上下文编码进 `requestState` 就行——**自己不用存任何会话**，这正是 MRTR(Multi Round-Trip Requests, 多轮往返请求) 取代"服务端发请求"的全部动机。

三条约束见 3.2：只能用在三个方法上、`inputRequests` 与 `requestState` 至少给一个、不能请求客户端没声明过的能力。

### 7.5 与 2025 年教程的差异清单

搜到旧教程时，对照这张表改：

| 旧写法                              | 现在                                                 |
| -------------------------------- | -------------------------------------------------- |
| 先 `initialize` 握手                | 不需要，每请求自带 `_meta`                                  |
| 服务端返回 `Mcp-Session-Id`           | 已删除，收到也忽略                                          |
| 请求 `GET /mcp` 开 SSE 长流           | 已删除，`405`                                          |
| 只带 `Content-Type`                | 还要带 `MCP-Protocol-Version`、`Mcp-Method`、`Mcp-Name` |
| `result` 随意结构                    | 必须含 `resultType`                                   |
| 服务端在流上发 `sampling/createMessage` | 改为 MRTR 的 `input_required`                         |
| 取消发 `notifications/cancelled`    | 仅 stdio；HTTP 上关流即取消                                |

---

## 8. JSON-RPC 错误码

有约定，而且分三段管理。新手最容易把"协议错误"和"业务失败"混在一起。

### 8.1 三段区间

`JSON-RPC(JavaScript Object Notation Remote Procedure Call, 基于 JSON 的远程过程调用)` 2.0 把整数空间划成三段：

| 区间                  | 归属                                             |
| ------------------- | ---------------------------------------------- |
| `-32768` ~ `-32000` | **JSON-RPC 保留**，其中 `-32000` ~ `-32099` 留给服务端实现 |
| 其余整数                | 应用自定义                                          |
| 注意                  | 新应用自定义码应分配在 `-32768` ~ `-32000` **之外**         |

MCP 又对保留段做了细分：

| 子区间                 | 用途                                                          |
| ------------------- | ----------------------------------------------------------- |
| `-32000` ~ `-32019` | **legacy**：早期实现自行占用。新代码不得再分配，也不应使用。除 `-32002` 外，接收方不得假设具体含义 |
| `-32020` ~ `-32099` | **MCP 规范专属**。实现只能发出规范定义过的码，且必须严格按定义使用                       |

### 8.2 标准码（来自 JSON-RPC 2.0）

| 码        | 名称               | 含义                                                      |
| -------- | ---------------- | ------------------------------------------------------- |
| `-32700` | Parse error      | body 不是合法 JSON                                          |
| `-32600` | Invalid Request  | 信封不合法：`jsonrpc` 不是 `"2.0"`、`id` 为 `null`、`method` 不是字符串 |
| `-32601` | Method not found | 方法未实现                                                   |
| `-32602` | Invalid params   | 参数不合法，或 `_meta` 缺必填                                     |
| `-32603` | Internal error   | 服务端内部异常                                                 |

### 8.3 MCP 专用码（2026-07-28）

| 码        | 名称                                | 触发场景                |
| -------- | --------------------------------- | ------------------- |
| `-32020` | `HeaderMismatch`                  | 必填头缺失，或头值与 body 不一致 |
| `-32021` | `MissingRequiredClientCapability` | 处理该请求需要客户端未声明的能力    |
| `-32022` | `UnsupportedProtocolVersion`      | 请求的协议版本不在服务端支持列表    |

### 8.4 已废弃、不能再发的码

| 码        | 原用途                                    | 现状                                     |
| -------- | -------------------------------------- | -------------------------------------- |
| `-32002` | resource not found（2025-11-25 及更早）     | 被 `-32602` 取代。客户端仍应接受旧服务端发来的此码；新实现不得发出 |
| `-32042` | URL elicitation required（仅 2025-11-25） | 已废弃                                    |

### 8.5 错误码与 HTTP 状态码是两层，必须同时给

这是最容易写错的地方。**错误码不是 HTTP 状态码的替代品，两者要一一配合。**

| 场景                  | HTTP      | JSON-RPC |
| ------------------- | --------- | -------- |
| body 不是合法 JSON      | `400`     | `-32700` |
| 信封不合法               | `400`     | `-32600` |
| 方法未实现               | **`404`** | `-32601` |
| 参数不合法 / `_meta` 缺必填 | `400`     | `-32602` |
| 服务端内部异常             | `500`     | `-32603` |
| 头缺失或不一致             | `400`     | `-32020` |
| 缺客户端能力              | `400`     | `-32021` |
| 版本不支持               | `400`     | `-32022` |

两个反直觉点：

- **`-32601` 配的是 `404`，不是 `200`。** 网关靠 `404` 判断端点不认这个方法。
- **`-32700` 配的是 `400`。** 规范没为它单列状态码，按 malformed request 统一处理。

如果只回 HTTP 400 而 body 里没有错误码，客户端无法区分"协议层错"还是"业务层错"，重试策略和提示都会乱。

### 8.6 业务失败不要用错误码

SQL 语法错、表不存在、参数值越界——这些**不是协议错误**，应该返回 HTTP `200` + `result.isError = true`：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resultType": "complete",
    "isError": true,
    "content": [
      {
        "type": "text",
        "text": "properties 是 LIST<STRUCT>，不支持 like。请改用 op=\"hasProperty\"。"
      }
    ]
  }
}
```

原因很实际：**错误文本会进入模型上下文，它下一次调用就会照着改。** 这比任何文档都有效，因为它刚好出现在需要的时候。如果只回 `-32602` 和一句 "Invalid params"，模型会开始瞎试，浪费好几轮调用。

判断标准很简单：**错在"报文格式"用错误码，错在"你的数据或参数含义"用 `isError`。**

---

## 9. 模型怎么知道你的工具

模型对工具的全部知识**只来自 `tools/list` 的返回值**。它不读源码，不读 README，也猜不到你没声明的操作符。

所以"模型会不会用对"不是模型的问题，是描述写得够不够。三层保障：

### 9.1 第一层：`enum` 让它发现选项存在

```json
"op": {
  "type": "string",
  "enum": ["eq", "ne", "gt", "hasProperty"],
  "description": "比较操作符。hasProperty 专用于 LIST<STRUCT> 列：value 形如 {\"name\":\"分类编码\",\"equals\":\"010101\"}。对这类列用 like 会失败，因为它是数组不是字符串。"
}
```

- `enum` 是模型发现"有哪些操作"的唯一途径，不列出来它就不会用
- **负向约束比正面说明更有用**。模型最常犯的错不是不知道有 `hasProperty`，而是觉得"我用 `like '%分类编码%'` 也能匹配上吧"，听起来完全合理，实际直接报错

### 9.2 第二层：tool 级 `description` 划清边界

```json
{
  "name": "filter_rows",
  "description": "按条件筛选行，支持属性匹配与范围过滤。适合：找构件、按属性筛、按范围过滤。不适合：分组统计、跨表关联——这些请用 run_sql。"
}
```

那句"不适合……请用 run_sql"是在**做路由**。模型面对两个工具经常选错，明确分工比让它自己权衡可靠。


### 9.3 第三层：错误文本反向纠正

见 8.6。前两层是预防，这一层是补救。

### 9.4 上下文预算

`tools/list` 的全部内容会进模型上下文。工具越多、描述越长，占得越多，还会稀释注意力。分配原则：

| 放哪               | 放什么                     | 为什么                   |
| ------------------ | -------------------------- | ------------------------ |
| tool `description` | 一到两句加分工指引         | 每次会话都在，必须短     |
| 参数 `description` | 含义、枚举、典型值         | 用到时才细看             |
| Resource           | 完整手册、字段全表、示例集 | 按需读取，不占常驻上下文 |
| 错误信息           | 具体怎么改                 | 出现在最需要的时刻       |

注意：**不是所有客户端都会自动读 Resource**。强模型会主动读，弱的不会。所以核心指引仍要留在 `description` 里。

一句话总结这一节：**工具的可发现性不是协议给的，是你写出来的。** 任何 SDK 都不会替你写 description、设计错误文本。

---

## 10. 常见误解清单

| 误解                         | 实际                                                        |
| ---------------------------- | ----------------------------------------------------------- |
| MCP 是一种 API 规范，像 REST | 它是 RPC 风格的能力协商协议，只有一个端点，方法名在 body 里 |
| 每个工具一个 HTTP 路径       | 全部走 `tools/call`，工具名是参数                           |
| 查询用 GET 更符合语义        | 会被缓存串味，且参数是嵌套 JSON，必须 POST                  |
| 报错就返回 HTTP 4xx/5xx      | 两层都要给，`-32601` 配 `404`，业务失败配 `200` + `isError` |
| 用 2025 年的示例应该兼容     | 握手、会话、GET 流都已删除，照抄会连不上                    |
| SSE 是必须的长连接           | 是**可选**的按需流，多数调用走普通 JSON                     |
| Resource 和 Tool 差不多      | 决策权不同：Tool 由模型决定调用，Resource 由应用决定加载    |
| 工具描述写得越详细越好       | 常驻上下文有预算，长内容应放 Resource                       |
| 服务端不能有任何状态         | 协议层无状态；数据、缓存这类共享只读资源不违反              |
| `null` 是合法的 `id`         | JSON-RPC 允许，MCP 明确禁止                                 |
| `resultType` 有 `error` 值   | 只有 `complete` 与 `input_required`；错误是另一条响应分支   |
| `inputRequests` 是数组       | 是 map，键为服务端自起的标识符，值为一种客户端请求          |

---

## 11. 术语表

| 术语     | 全称                                             | 中文释义                 |
| -------- | ------------------------------------------------ | ------------------------ |
| MCP      | Model Context Protocol                           | 模型上下文协议           |
| JSON-RPC | JavaScript Object Notation Remote Procedure Call | 基于 JSON 的远程过程调用 |
| RPC      | Remote Procedure Call                            | 远程过程调用             |
| REST     | Representational State Transfer                  | 表述性状态转移           |
| SSE      | Server-Sent Events                               | 服务器发送事件           |
| stdio    | Standard Input/Output                            | 标准输入输出             |
| MRTR     | Multi Round-Trip Requests                        | 多轮往返请求             |
| SSRF     | Server-Side Request Forgery                      | 服务端请求伪造           |
| TTL      | Time To Live                                     | 存活时间                 |
| P95      | 95th Percentile                                  | 95 分位响应时间          |
| CDN      | Content Delivery Network                         | 内容分发网络             |
| IdP      | Identity Provider                                | 身份提供商               |
| OIDC     | OpenID Connect                                   | 基于 OAuth 的身份认证层  |
| CIMD     | Client ID Metadata Documents                     | 客户端 ID 元数据文档     |
| PoC      | Proof of Concept                                 | 概念验证                 |

---

## 12. 参考

以下链接于 2026-11-11 逐个实测，均返回 HTTP 200。每条末尾的「对应」指本文哪几节引用了它，方便回溯原始措辞。

### 12.1 规范正文（2026-07-28）

- [规范总目录](https://modelcontextprotocol.io/specification/2026-07-28) — 版本入口与全部章节
- [基础层](https://modelcontextprotocol.io/specification/2026-07-28/basic) — 消息类型、`id` 规则、错误码区间、`_meta` 信封；对应 §3、§8.1
- [传输层总览](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports) — 传输绑定的职责边界、元数据镜像原则；对应 §4
- [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http) — 必填请求头、状态码映射、SSE 分帧、取消规则、会话移除；对应 §4.2、§4.5
- [stdio](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio) — 换行分帧与 stderr 约定；对应 §4.1
- [消息模式](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns) — 请求响应、MRTR、订阅通知三种模式；对应 §3、§7.4
- [多轮往返 MRTR](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr) — `input_required`、`inputRequests`、`requestState` 的完整定义与安全要求；对应 §3.2、§7.4
- [取消](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/cancellation) — 两种传输各自的取消机制；对应 §4.5
- [版本与兼容](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning) — 新旧代识别规则与兼容矩阵；对应 §2、§7.5
- [工具](https://modelcontextprotocol.io/specification/2026-07-28/server/tools) — `tools/list`、`tools/call`、`inputSchema`、`outputSchema`、`structuredContent`；对应 §6、§7.2、§7.3、§9
- [资源](https://modelcontextprotocol.io/specification/2026-07-28/server/resources) — `resources/list`、`resources/read`、模板；对应 §5
- [提示](https://modelcontextprotocol.io/specification/2026-07-28/server/prompts) — `prompts/list`、`prompts/get`；对应 §5
- [发现](https://modelcontextprotocol.io/specification/2026-07-28/server/discover) — `server/discover` 的请求、响应与缓存提示；对应 §7.1

### 12.2 发布说明与架构文档

- [2026-07-28 发布说明](https://blog.modelcontextprotocol.io/posts/2026-07-28/) — 本版每项变更的官方动机，是 §2 那张动机表的一手来源
- [架构概览](https://modelcontextprotocol.io/docs/learn/architecture) — 原语定义与一次完整调用（发现 → 列工具 → 调用）的逐步走查；对应 §3、§5、§7

### 12.3 底层标准

- [JSON-RPC 2.0 规范](https://www.jsonrpc.org/specification) — 错误码原始定义、批处理、通知语义；对应 §3.1、§8.1

### 12.4 查阅旧版

本文以 `2026-07-28` 为基准。需要对照旧版时，把 URL 里的版本号换掉即可：

```text
https://modelcontextprotocol.io/specification/<版本号>/
```


