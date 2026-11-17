---
title: "A2A 协议发展与采用困境：规范不慢，为什么用的人少"
date: 2026-11-17 10:00:00 +0800
categories: [人工智能, Agent]
tags: [A2A, MCP, AI Agent, 协议设计, AAIF, 互操作]
---

> **核心观点**：A2A 的规范推进不慢（11 个月到 1.0），设计也比较完整。用的人少，主要有三个原因：一是"跨组织 Agent 协作"这个场景还没大规模出现；二是 MCP 先占了开发者心智，而且能凑合覆盖大部分需求；三是它是双边协议，企业级宿主（Copilot Studio、Gemini Enterprise）已经有了，但开发者天天用的宿主（Claude、ChatGPT、Cursor、Claude Code）都没有原生支持。所以真实的采用主要发生在企业平台和手机系统层，开发者社区看不到。

本文整理 A2A（Agent2Agent）协议的定位、核心概念、交互流程、协议分层与安全模型、版本演进、大事记与背后公司、与 MCP 的分工，重点分析"为什么用的人少"，最后整理可复用的经验和一份"什么时候该用 A2A"的决策清单。技术细节对照 A2A 官方规范（v1.0）、Release Notes、官方迁移指南与公告核实。

MCP 一侧的发展历史见姊妹篇[《MCP 协议演进史（2024—2026）：从借鉴 LSP 到无状态核心》](/posts/mcp-protocol-evolution/)。

起因是看 LoopForge（编排 Claude Code / Codex 等编程 Agent 的自托管平台）时发现，它的多 Agent 协作完全在平台内部用 SOP 引擎和子 Agent 实现，只用了 MCP，没有用 A2A。由此引出一组问题：

1. A2A 发展得怎么样？和 MCP 是什么关系？
2. 为什么 A2A 的规范进展不慢，用的人却很少？
3. 从 A2A 的处境中能学到什么？

**阅读路线**：第一至五节讲协议本身；第六、七节讲发展历程；第八至十一节讲采用困境（重点）；第十二、十三节是经验和决策清单。只关心结论可以直接跳到第九节。

**速览**

| 项目 | 内容 |
|---|---|
| 发起 | Google Cloud，2025-04-09 发布 |
| 治理 | 2025-06 捐给 Linux Foundation → 2026-08 加入 AAIF（与 MCP 同一基金会） |
| 当前版本 | v1.0（2026-03-12），v1.0.1（2026-05-26） |
| 定位 | Agent ↔ Agent 的发现、委派与协作；MCP 管 Agent → 工具 |
| 生态规模 | 官方口径 150+ 组织；核心仓库 22,000+ Star；官方 SDK 覆盖 6 种语言 |
| 主要落地 | 企业 SaaS 互通、三大云托管、鸿蒙系统助手与微信 |

---

## 一、A2A 是什么

### 一句话定义

**A2A（Agent2Agent）是让不同厂商、不同框架开发的、内部互不透明的 AI Agent 之间互相发现、委派任务和交换结果的开放协议。**

### 它要解决的问题

```
没有 A2A：每对 Agent 都要写定制集成
  公司 X 的采购 Agent ──定制代码──► 公司 Y 的报价 Agent
  公司 X 的采购 Agent ──另一套定制代码──► 公司 Z 的物流 Agent
  ……N 个 Agent 两两对接 ≈ N×(N-1)/2 套集成

有了 A2A：每个 Agent 只实现一次协议（共 N 套）
  任意 A2A Client ──标准协议──► 任意 A2A Server（Remote Agent）
```

### 规范中的五条设计原则

| 原则 | 含义 |
|---|---|
| Simple | 复用现成标准：HTTP、JSON-RPC 2.0、SSE |
| Enterprise Ready | 认证、授权、安全、隐私、追踪、监控都对齐企业现有做法 |
| Async First | 天然面向长时间任务和人在回路中的场景 |
| Modality Agnostic | 支持文本、文件、结构化数据、音视频（通过文件引用）、嵌入式 UI |
| Opaque Execution | Agent 之间只按声明的能力协作，**不共享内部思考、计划、工具和记忆** |

最后一条是 A2A 和 MCP 的本质区别：**MCP 的对端是透明的工具，A2A 的对端是不透明的自主 Agent。**

### 先认识一下 MCP（理解 A2A 的必要背景）

A2A 绕不开和 MCP 比较，这里先简单介绍，详细对比见第八节。

- **MCP（Model Context Protocol）**：Anthropic 于 2024-11 开源的协议，规定 Agent（或大模型应用）如何连接外部**工具和数据**，比如让 Claude 调用 GitHub、数据库、文件系统。
- 采用极快：OpenAI（2025-03）、Google（2025-04）、Microsoft（2025-05）相继支持，现在几乎所有主流 AI 产品都能接 MCP 服务端。
- 2025-12 被捐给 Linux Foundation 新成立的 **AAIF（Agentic AI Foundation）**，2026-08 A2A 也加入了这个基金会。

```
MCP：Agent ──调用──► 工具 / 数据源       （垂直：Agent 使用工具）
A2A：Agent ◄──委派 / 协作──► Agent       （水平：Agent 与 Agent 对等协作）
```

---

## 二、核心概念

### 概念总览

| 概念 | 说明 |
|---|---|
| **A2A Client** | 代表用户或其他系统，向远端 Agent 发起请求的应用或 Agent |
| **A2A Server（Remote Agent）** | 暴露 A2A 端点、处理任务并返回结果的 Agent |
| **Agent Card** | Server 发布的 JSON 元数据：身份、能力、技能、端点、认证要求 |
| **Task** | 基本工作单元，有唯一 ID 和生命周期（状态机） |
| **Message** | 一轮通信，`role` 为 user 或 agent，由若干 Part 组成 |
| **Part** | 最小内容单元：文本、文件（URL 或内联字节）或结构化数据 |
| **Artifact** | 任务产出物（文档、图片、结构化数据），由 Part 组成 |
| **Context** | 可选标识，把相关的多个 Task 和 Message 归为一组（类似会话） |
| **Streaming** | 通过流式连接实时推送任务状态和产物增量 |
| **Push Notification** | Server 主动 POST 到 Client 提供的 webhook，用于长任务或断连场景 |
| **Extension** | 在核心规范之外扩展功能或数据的机制 |

### Agent Card（能力名片）

默认发布在 `https://{域名}/.well-known/agent-card.json`。精简示例（据规范 8.5 节改写）：

```json
{
  "name": "GeoSpatial Route Planner Agent",
  "description": "路线规划、交通分析与定制地图生成",
  "supportedInterfaces": [
    {"url": "https://georoute.example.com/a2a/v1",   "protocolBinding": "JSONRPC",   "protocolVersion": "1.0"},
    {"url": "https://georoute.example.com/a2a/grpc", "protocolBinding": "GRPC",      "protocolVersion": "1.0"},
    {"url": "https://georoute.example.com/a2a/json", "protocolBinding": "HTTP+JSON", "protocolVersion": "1.0"}
  ],
  "provider": {"organization": "Example Geo Services Inc."},
  "version": "1.2.0",
  "capabilities": {"streaming": true, "pushNotifications": true, "extendedAgentCard": true},
  "securitySchemes": {
    "google": {"openIdConnectSecurityScheme": {"openIdConnectUrl": "https://accounts.google.com/.well-known/openid-configuration"}}
  },
  "defaultInputModes": ["application/json", "text/plain"],
  "defaultOutputModes": ["application/json", "image/png"],
  "skills": [
    {"id": "route-optimizer-traffic", "name": "Traffic-Aware Route Optimizer",
     "tags": ["maps", "routing"], "examples": ["从 A 到 B 规划一条避开收费站的路线"]}
  ],
  "signatures": [{"protected": "eyJhbGciOi...", "signature": "QFdkNL..."}]
}
```

要点：

- `supportedInterfaces`：同一个 Agent 可以同时提供多种协议绑定，按优先顺序排列，由 Client 挑选。每个接口单独声明 `protocolVersion`，所以同一张名片可以同时声明 0.3 和 1.0 两个版本。
- `skills`：用自然语言加示例描述"能做什么"，粒度比 MCP 的工具粗得多。
- `signatures`：签名 Agent Card（JWS，签名前按 RFC 8785 规范化 JSON），用于跨组织验证身份与元数据未被篡改。
- `extendedAgentCard`：认证后才能拿到的扩展名片，可以向不同客户展示不同能力。

### Task 状态机（v1.0）

下图是常见流转的示意，不是穷举：

```
                         ┌──► COMPLETED
                         ├──► FAILED
SUBMITTED ───► WORKING ──┼──► CANCELED       （以上四个为终态）
                 ▲  │    └──► REJECTED
                 │  │
                 │  ├──► INPUT_REQUIRED ──┐
                 │  └──► AUTH_REQUIRED  ──┤ （中断态）
                 │                        │
                 └────────────────────────┘
                   Client 补充信息 / 完成授权后继续
```

v1.0 中的枚举值写作 `TASK_STATE_SUBMITTED`、`TASK_STATE_WORKING`、`TASK_STATE_INPUT_REQUIRED`、`TASK_STATE_AUTH_REQUIRED`、`TASK_STATE_COMPLETED`、`TASK_STATE_FAILED`、`TASK_STATE_CANCELED`、`TASK_STATE_REJECTED`（另有 `TASK_STATE_UNSPECIFIED` 作为默认值）。0.3 及以前写作 `completed`、`input-required` 这类小写形式。

- **四个终态**（COMPLETED / FAILED / CANCELED / REJECTED）的任务不能再接收消息。REJECTED 既可以在创建任务时发生，也可以在执行中途发生；取消也可以发生在任意非终态。
- **两个中断态**：
  - **INPUT_REQUIRED**：Agent 中途需要补充信息，Client 带着 taskId 再发一条消息即可继续。MCP 里对应的机制是 elicitation（服务端在工具执行中向用户追问），2026-07 版规范改为 MRTR（多轮往返请求）实现。
  - **AUTH_REQUIRED**："任务内授权"，执行到一半才发现需要访问某个受保护资源。A2A 把它直接做进了任务状态机。

### Message 与 Artifact 的区别

| | Message | Artifact |
|---|---|---|
| 是什么 | 一轮对话（问、答、追问、说明） | 任务的交付物 |
| 例子 | "请提供出发时间"、"好的，正在处理" | 一份报告、一张地图、一段 JSON |
| 是否可流式 | 可以 | 可以分块增量推送 |

---

## 三、一次完整交互

### 全流程示意

```
Client Agent                                        Remote Agent
    │                                                    │
 ①  │── GET /.well-known/agent-card.json ──────────────►│  发现：读能力、端点、认证方式
    │◄── Agent Card ─────────────────────────────────────│
    │                                                    │
 ②  │   （按 securitySchemes 走 OAuth / OIDC / mTLS 拿凭证，带外完成）
    │                                                    │
 ③  │── SendMessage / SendStreamingMessage ────────────►│  委派任务
    │◄── Task{id, contextId, state: SUBMITTED/WORKING} ──│
    │                                                    │
 ④  │   获取进度，三选一：                                │
    │   a. 流式：SSE / gRPC 流持续推送状态与产物           │
    │   b. 轮询：GetTask(id)                              │
    │   c. 推送：Remote Agent POST 到 Client 的 webhook   │
    │                                                    │
 ⑤  │◄── state: INPUT_REQUIRED（"需要你确认预算上限"）────│  多轮交互
    │── SendMessage(taskId, 补充信息) ──────────────────►│
    │                                                    │
 ⑥  │◄── state: COMPLETED + Artifacts ───────────────────│  交付
```

### HTTP+JSON 绑定下的最简请求

据规范 6.1 节，补充了 `A2A-Version` 头：

```http
POST /message:send HTTP/1.1
Host: agent.example.com
Content-Type: application/a2a+json
A2A-Version: 1.0
Authorization: Bearer token

{
  "message": {
    "role": "ROLE_USER",
    "parts": [{"text": "What is the weather today?"}],
    "messageId": "msg-uuid"
  }
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/a2a+json

{
  "task": {
    "id": "task-uuid",
    "contextId": "context-uuid",
    "status": {"state": "TASK_STATE_COMPLETED"},
    "artifacts": [{
      "artifactId": "artifact-uuid",
      "name": "Weather Report",
      "parts": [{"text": "Today will be sunny with a high of 75°F"}]
    }]
  }
}
```

说明：

- 简单问答时，Agent 也可以**直接返回一个 Message**，不创建 Task。
- `SendMessage` 默认是阻塞的：等任务进入终态或中断态（INPUT_REQUIRED / AUTH_REQUIRED）才返回；传 `returnImmediately: true` 则立即返回，之后再轮询、订阅或等推送。
- `A2A-Version` 请求头用于版本协商；不带时按 0.3 处理，版本不支持时返回 `VersionNotSupportedError`。
- 1.0.1 起 HTTP 绑定推荐媒体类型 `application/a2a+json`。

官方的说法是"最简形式下，一次 A2A 交互只需要一个 HTTP 请求"。这对 Server 端成立，但要做出一个**生产可用**的 Agent，仍需处理名片、认证、状态机、流式和推送，第九节会再谈这一点。

### 三种获取结果方式的取舍

| 方式 | 适用场景 | 代价 |
|---|---|---|
| 流式 | 秒级到分钟级任务，需要实时进度 | 需要保持连接 |
| 轮询 | 简单实现、连接不稳定 | 有延迟，浪费请求 |
| 推送（webhook） | 小时到天级的长任务，Client 可能离线 | Client 要暴露可访问的 URL，并校验推送来源 |

---

## 四、协议分层、绑定与方法映射

### 三层结构（v1.0 重构的核心）

```
┌──────────────────────────────────────────────────────────┐
│ 第 1 层：规范数据模型                                       │
│   Task / Message / AgentCard / Part / Artifact / Extension │
│   以 Protocol Buffers 定义，spec/a2a.proto 是唯一权威来源   │
├──────────────────────────────────────────────────────────┤
│ 第 2 层：抽象操作                                          │
│   发送消息 / 流式发送 / 查询任务 / 列出任务 / 取消 / 订阅 …  │
├──────────────────────────────────────────────────────────┤
│ 第 3 层：协议绑定                                          │
│   JSON-RPC │ gRPC │ HTTP+JSON/REST │ 自定义绑定             │
└──────────────────────────────────────────────────────────┘
```

好处：语义只定义一次，新增绑定不改数据模型。代价：实现者要理解三层映射，兼容测试矩阵也更大。

### 方法映射（v1.0，规范 5.3 节）

| 功能 | JSON-RPC 方法 | gRPC 方法 | REST 端点 |
|---|---|---|---|
| 发送消息 | `SendMessage` | `SendMessage` | `POST /message:send` |
| 流式发送 | `SendStreamingMessage` | `SendStreamingMessage` | `POST /message:stream` |
| 查询任务 | `GetTask` | `GetTask` | `GET /tasks/{id}` |
| 列出任务 | `ListTasks` | `ListTasks` | `GET /tasks` |
| 取消任务 | `CancelTask` | `CancelTask` | `POST /tasks/{id}:cancel` |
| 订阅任务 | `SubscribeToTask` | `SubscribeToTask` | `POST /tasks/{id}:subscribe` |
| 推送配置增删查 | `Create/Get/List/DeleteTaskPushNotificationConfig(s)` | 同左 | `/tasks/{id}/pushNotificationConfigs[/{configId}]` |
| 获取扩展名片 | `GetExtendedAgentCard` | `GetExtendedAgentCard` | `GET /extendedAgentCard` |

> 0.3 及以前的 JSON-RPC 方法名是 `message/send`、`tasks/get` 这类斜杠风格，1.0 统一改成了 PascalCase；REST 路径也去掉了 `/v1` 前缀。两者都是破坏性变更。

---

## 五、安全模型与企业特性

| 特性 | 说明 |
|---|---|
| **认证方案声明** | Agent Card 中声明 `securitySchemes`：API Key、HTTP Auth、OAuth 2.0、OpenID Connect、mTLS |
| **OAuth 现代化（1.0）** | 废弃 implicit、password 两种不安全流程，加入 device code，授权码流程支持声明必须使用 PKCE |
| **签名 Agent Card** | JWS 签名，跨组织验证"这张名片确实出自该提供方且未被篡改" |
| **扩展 Agent Card** | 认证后才返回，可以按客户分级展示能力 |
| **任务内授权** | 任务执行中进入 `AUTH_REQUIRED`，Client 补充授权后继续 |
| **多租户（1.0）** | 请求中的 `tenant` 字段，一个端点托管多个 Agent |
| **推送安全** | 规范建议 Agent 校验 webhook 地址以防 SSRF（服务端请求伪造），并对推送做认证；消息中的文件引用也必须校验以防 SSRF |
| **数据访问范围** | `ListTasks` 等操作必须只返回调用方有权看到的任务，且不能泄露无权资源是否存在 |

和 MCP 的对比：**A2A 从第一版起就把认证当成一等公民**（发布公告原话是 "Secure by default"，首版即对齐 OpenAPI 的认证方案）；MCP 首版（2024-11）没有认证，2025-03 才加入基于 OAuth 2.1 的授权框架。这让 A2A 更贴近企业的安全要求，但也提高了个人开发者的上手门槛。

需要注意的是，协议只解决"你是谁"，不解决"你能做什么"：哪个 Agent 可以调用哪个、委派出去的任务带多少权限、审计日志放在哪里，仍然要由部署方的治理层来定。

---

## 六、版本演进

这里只列规范本身的变化，生态事件见第七节。

```
2025-04     首个草案（google/A2A 仓库）：Agent Card、Task、HTTP + JSON-RPC + SSE；
              认证方案对齐 OpenAPI
2025-05~07  0.2.x：
              - 0.2.1（05-27）：认证后扩展名片的开关；referenceTaskIds（任务追问）
              - 0.2.2（06-09）：规范中加入 gRPC 与 REST 定义；扩展机制；
                                修正与 JSON-RPC 2.0 不一致处（破坏性变更）
              - 0.2.4（06-30）：Agent Card 可声明多种传输
              - 0.2.5（06-30）：Agent Card 必须声明协议版本；一个任务支持多个推送配置
                                （两项均为破坏性变更）
2025-07-30  0.3：签名 Agent Card；mTLS 认证方案；扩展名片获取方法；
              well-known 路径由 agent.json 改为 agent-card.json（破坏性变更）；
              Google 在发布博客中把"gRPC 支持"列为 0.3 的关键能力
2026-03-12  1.0：首个稳定版（大量破坏性变更）
              - 规范重构为"数据模型 / 抽象操作 / 协议绑定"三层，proto 成为唯一权威来源
              - Part 统一为单一结构（去掉 TextPart / FilePart / DataPart 和 kind 字段），
                官方迁移指南标为影响最大的变更
              - JSON-RPC 方法名由 message/send 等斜杠风格改为 SendMessage 等 PascalCase
              - 枚举改为 ProtoJSON 格式（TASK_STATE_*、ROLE_*）
              - Agent Card 用 supportedInterfaces 统一声明接口，协议版本下沉到每个接口
              - 新增 ListTasks（过滤 + 游标分页）；多租户（tenant 字段）
              - OAuth 废弃 implicit / password，加入 device code、PKCE
              - 错误统一用 google.rpc.Status 表示；REST 路径去掉 /v1 前缀
              - 确立变更与废弃规则；Agent Card 可同时声明 0.3 和 1.0，方便逐步迁移
2026-05-26  1.0.1：HTTP 绑定推荐 application/a2a+json；修正 TaskStatus 等描述
2026-09     路线图：1.1（任务时间线、事件过滤）、双向流式、结构化的人在回路交互
```

**观察**：

- 1.0 之前的 0.2.2、0.2.5、0.3 都有破坏性变更，1.0 又是一次大改，早期实现者迁移成本很高。
- 1.0 最大的进步是**确立变更与废弃规则**：改名时旧名保留并标记废弃，至少到下一个大版本才删除；并提供多版本共存的 Agent Card。作为对比，MCP 到 2026-07 版规范才正式确立"至少 12 个月废弃期"的政策。

---

## 七、大事记、关键人物与背后公司

### 大事记

| 日期 | 事件 | 主导方 |
|---|---|---|
| 2025-04-09 | **Google 在 Cloud Next 发布 A2A**，50+ 合作伙伴（Atlassian、Box、Cohere、Intuit、LangChain、MongoDB、PayPal、Salesforce、SAP、ServiceNow、UKG、Workday，以及 Accenture、BCG、Deloitte、McKinsey 等咨询公司） | Google Cloud |
| 2025-04-09 | 同日 Google DeepMind 宣布 Gemini 支持 MCP | Google DeepMind |
| 2025-05-07 | Azure AI Foundry、Copilot Studio 支持 A2A | Microsoft |
| 2025-06-23 | **捐给 Linux Foundation**，仓库迁到 a2aproject/A2A；创始成员 AWS、Cisco、Google、Microsoft、Salesforce、SAP、ServiceNow | Google |
| 2025-07-30 | v0.3 发布；Google 随后（08-01）宣布 ADK 原生支持 A2A、AI Agent Marketplace 可售卖 A2A Agent，并称支持组织已超过 150 家 | A2A 社区、Google Cloud |
| 2025-08 | **IBM 的 ACP 并入 A2A** | IBM Research |
| 2025-09 | Google 联合 PayPal 等发布 AP2（Agent 支付协议，可作为 A2A 的扩展使用） | Google、PayPal 等 |
| 2025-11-11 | Bedrock AgentCore Runtime 支持托管 A2A Server（AgentCore 于 2025-10-13 正式发布） | AWS |
| 2026-03-12 | **v1.0 稳定版**；TSC 由 AWS、Cisco、Google、IBM Research、Microsoft、Salesforce、SAP、ServiceNow 代表组成 | A2A TSC |
| 2026-04-09 | 一周年：150+ 组织支持，三大云原生集成，供应链、金融、保险、IT 运维有生产部署 | Linux Foundation |
| 2026-05-26 | v1.0.1 | A2A 社区 |
| 2026-08-17 | **加入 AAIF**（成为托管项目），与 MCP、goose、AGENTS.md 同属一个基金会 | AAIF |
| 2026-10-01 | 发布官方 A2A CLI 及配套 Agent Skill，面向 Claude Code、Cursor、Codex 等编程 Agent | A2A 社区 |

### 关键人物

| 人物 | 所属 | 角色 |
|---|---|---|
| Rao Surapaneni | Google Cloud（VP & GM, Business Application Platform） | A2A 发布公告和 0.3 发布博客的第一作者，多次在 LF、AAIF 公告中代表 Google 发言 |
| Todd Segal | Google Cloud（发布时为 Principal Engineer，后为 Distinguished Engineer） | 发布公告共同作者 |
| Miku Jha | Google Cloud（Director, AI/ML Partner Engineering） | 发布公告共同作者，负责合作伙伴生态 |
| Michael Vakoc | Google Cloud（Product Manager） | 发布公告共同作者 |
| Philip Stephens | Google Cloud（Senior Staff Software Engineer, Business Applications Platform） | 0.3 发布博客共同作者 |
| Manik Surtani | AAIF（CTO） | 代表 AAIF 接纳 A2A |
| Jim Zemlin | Linux Foundation（执行董事） | 主导成立 AAIF |

> 观察：A2A 的发布公告署名来自 Google Cloud 业务应用平台和合作伙伴工程团队，而不是研究团队；首批合作伙伴里还有大量咨询公司。这说明它从一开始就是**面向企业应用集成**的产品战略，而不是面向个人开发者的工具。

### 背后公司

| 公司 | 角色 | 动机 / 看点 |
|---|---|---|
| **Google** | 发起者；捐给 LF；推出 AP2；ADK、Agent Engine、Cloud Run、GKE 原生支持；Gemini Enterprise（原 Agentspace）可注册 A2A Agent 供员工使用 | 工具层跟随 MCP，Agent 协作层自己定标准 |
| **Microsoft** | 创始成员、TSC；Azure AI Foundry 可暴露 A2A 端点、发现外部 Agent；Copilot Studio 可把外部 A2A Agent 接为子 Agent | MCP 和 A2A 两边都押注 |
| **AWS** | 创始成员、TSC；Bedrock AgentCore 托管 A2A Server | 两边都深度参与 |
| **Salesforce / SAP / ServiceNow** | 创始成员、TSC；在自家产品之间用 A2A 打通工作流（Agentforce、Joule、AI Agent Fabric 等） | 企业 SaaS 是 A2A 最主要的推动力 |
| **Cisco** | 创始成员、TSC；另有 AGNTCY 项目 | 网络与基础设施视角 |
| **IBM** | ACP 并入 A2A；TSC（IBM Research）；BeeAI 支持 A2A | 主动放弃自有协议 |
| **Atlassian** | 首批合作伙伴；在自家产品间使用 A2A | Rovo Agent 之间的协作 |
| **华为** | 鸿蒙系统助手小艺与应用内 Agent 之间统一使用 A2A | **系统级、手机规模**的落地 |
| **腾讯（微信）** | 首批通过 A2A 对接华为及其他安卓厂商系统助手的大型应用之一，支持通过 AI 助手发消息、打语音或视频电话，并需要双重授权 | 超级 App 与系统助手之间的标准接口 |
| **PayPal** | 与 Google 合作 AP2 | Agent 电商支付 |
| **框架方** | LangGraph、CrewAI、Pydantic AI、AG2、IBM BeeAI 支持 A2A | 框架层面的支持已经具备 |

**明显的缺席者**：截至本文整理时，Anthropic、OpenAI 都没有在 Claude、ChatGPT、Claude Code 中原生支持调用外部 A2A Agent（社区已有相应的功能请求），两者都是 MCP 的主要推动方。

---

## 八、A2A 与 MCP 的分工

```
         垂直（Agent → 工具）                水平（Agent ↔ Agent）
        ┌────────────────────┐            ┌────────────────────┐
        │        MCP         │            │        A2A         │
        │  对端：透明的工具    │            │  对端：不透明的 Agent │
        │  粒度：函数级        │            │  粒度：任务级         │
        │  关系：使用者-工具   │            │  关系：对等协作方      │
        └────────────────────┘            └────────────────────┘
```

| 维度 | MCP | A2A |
|---|---|---|
| 交互单位 | 一次工具调用 | 一个有生命周期的 Task |
| 能力描述 | 工具的名称、参数 schema | Agent Card 中的技能（自然语言 + 示例） |
| 上下文占用 | 每个工具的描述都要进模型上下文，工具一多就膨胀 | 只暴露 Agent 级的能力摘要，细节留在对方内部 |
| 中途交互 | elicitation（2025-06 起），2026-07 改为 MRTR 机制 | `INPUT_REQUIRED` / `AUTH_REQUIRED` 状态（首版即有） |
| 长任务 | Tasks（2025-11 实验性，2026-07 成为正式扩展） | 原生支持，流式 / 轮询 / 推送三种获取方式 |
| 认证 | 首版没有，2025-03 加入 OAuth 2.1 授权框架 | 首版即"Secure by default" |
| 治理 | AAIF（2025-12 起） | LF（2025-06）→ AAIF（2026-08） |
| 开发者门槛 | 低 | 较高 |
| 生态规模 | 极大，主流 AI 产品几乎都支持 | 以企业平台、云平台和手机系统为主 |

**常见组合**：对外用 A2A 暴露 Agent 能力，Agent 内部用 MCP 调用工具。

Cisco 的一篇博客给了一个好记的类比：MCP 像二层网络，细节全可见、直接可达，但规模有上限；A2A 像三层路由边界，只对外通告汇总后的能力，把细节挡在边界内。系统小的时候只用 MCP 就够了，Agent 多了才需要 A2A 这一层。

---

## 九、为什么 A2A 用的人少（重点）

### 先校正判断

| 维度 | 事实 |
|---|---|
| 规范进度 | **不慢**：11 个月到 1.0，并在 1.0 中确立了正式的废弃规则（MCP 到 2026-07 才确立） |
| 机构支持 | **很强**：150+ 组织、三大云、主要企业 SaaS、AAIF |
| 开发者关注 | **不算少**：核心仓库 22,000+ Star，官方 SDK 从 1 种语言扩展到 6 种 |
| 开发者使用 | **冷清**：2025-08 一位开发者在 A2A 官方仓库讨论区写道，"几乎每个客户端都支持 MCP，但你很难找到一个支持 A2A 的客户端"；Cisco 博客（2026-01）形容 A2A 的采用是 "slow burn"（慢慢升温），并提到业内有"A2A 正在悄悄淡出"的说法 |
| 真实采用 | **集中在看不见的地方**：企业 SaaS 内部打通、云平台托管、手机系统层集成，都是平台方替用户接好的，不会出现在 GitHub、Hacker News 和技术博客上 |

还有一个耐人寻味的数据点：0.3 发布时（2025-08），Google 已经说支持组织超过 150 家；到一周年（2026-04），官方口径仍然是"150+"。合作伙伴名单的增长在早期就基本到顶了，之后的进展主要体现在落地深度上。

所以问题应该是：**机构都在支持、开发者也在关注，为什么开发者很少真正用？** "没人用"有一部分是"看不见"，剩下的原因可以分成需求侧、供给侧和环境三类。

### 需求侧：场景少，现有方案够用

**① 要解决的场景还很少见（最根本）**

A2A 只在"**不同所有者、互不透明的 Agent 之间互相委派**"时才明显有价值。而现在大多数多 Agent 系统是：

```
同一团队、同一框架、同一进程
主 Agent ──► 子 Agent A（共享上下文）
         ──► 子 Agent B（共享上下文）
```

- Claude Code 子 Agent、LangGraph、CrewAI 都是框架内编排，不需要网络协议。
- "该不该拆多 Agent"本身有争论。Cognition 在《Don't Build Multi-Agents》（2025-06）中主张优先做好单 Agent 的上下文工程；10 个月后的跟进文章（2026-04）总结出真正有效的形态是"一个写入者 + 若干只贡献智能的 Agent"，并认为"任意 Agent 网络互相协商"的无结构群体大多是干扰项。他们的经理 Devin 协调子 Devin，用的是内部 MCP，不是 A2A。
- 模型越强、上下文越长，单个 Agent 加工具能做的事越多，拆多 Agent 的需求反而下降。

**② MCP 已经覆盖了大部分需求**

- 把对方的 Agent 包装成一个 MCP 工具，就能满足大多数委派需求。
- MCP 持续补能力：Tasks（长任务）、MRTR（工具执行中途向用户追问），和 A2A 的差距越来越小。
- 开发者的现实想法：有了 MCP，为什么还要学第二个协议？

### 供给侧：双边协议，开发者宿主缺位

**③ 双边协议，又缺少开发者能直接用的宿主**

| | MCP | A2A |
|---|---|---|
| 生效条件 | 一边有服务端就有用，客户端早已遍地 | 两边都必须是 A2A Agent |
| 第一次体验 | 写个服务端接进 Claude Desktop，十分钟见效 | 写好一个 A2A Agent，然后没有对端可连 |
| 企业宿主 | 普遍支持 | Copilot Studio、Gemini Enterprise 等已支持 |
| 开发者宿主 | Claude、ChatGPT、Cursor、VS Code | 均未原生支持 |

这是典型的**双边网络效应"鸡和蛋"问题**。MCP 借 Claude Desktop 解决了它，A2A 在企业侧找到了宿主，但在开发者每天打开的工具里没有入口，所以开发者既看不到它，也没机会顺手用一下。

**④ 门槛高、早期变动大**

- 概念多：Agent Card、Task 状态机、Message / Part / Artifact、推送、多种安全方案。
- 三种协议绑定，加上自定义绑定，兼容矩阵大。
- 0.2.2、0.2.5、0.3、1.0 都有破坏性变更；1.0 发布时官方表示接下来的重点是补齐多语言 1.0 SDK。

### 环境：非技术障碍与时机

**⑤ 跨组织协作的主要障碍不在技术**

| 障碍 | 协议能做的 | 协议做不了的 |
|---|---|---|
| 信任 | 签名 Agent Card 证明身份 | 证明对方"靠谱" |
| 计费 | — | 定价、结算（AP2 在尝试） |
| 责任 | — | 对方 Agent 出错谁负责 |
| 合规 | 认证、授权 | 数据能否交给第三方 Agent |
| 发现 | well-known 名片 | 全球目录（你得先知道对方的域名） |

**⑥ 时机与叙事不利**

- 比 MCP 晚约 4 个半月（2024-11-25 → 2025-04-09），开发者注意力已被占据。
- 一发布就被写成"A2A vs MCP"，官方反复强调互补（1.0 公告专门用一节澄清），但第一印象已经形成。
- 早期由 Google 单方主导，其他厂商可能存在中立性顾虑（推测）；捐给 LF、进入 AAIF 后这个障碍基本消除。

### 一句话总结

> A2A 用的人少，不是因为规范慢或设计差，而是：**场景未成熟 + MCP 先占心智且能覆盖大部分需求 + 双边协议在开发者工具里缺宿主**。它目前的采用集中在企业平台和手机系统层。

---

## 十、历史类比：提前到来的互操作标准

| 历史标准 | 目标 | 结局 | 与 A2A 的相似点 |
|---|---|---|---|
| **KQML / FIPA-ACL**（1990s–2000s） | 多 Agent 系统之间的通信语言 | 学术界热，工业界几乎没用起来 | 思想上的前身（无直接继承关系）：Agent 间通信、能力描述 |
| **UDDI**（2000，Microsoft、IBM 等） | 全球 Web 服务注册与发现 | 公共注册中心 2006 年关闭 | 类似"Agent Card + 全球目录"的发现机制 |
| **WS-\* / SOAP** | 企业级 Web 服务标准，功能全、重量级 | 被更简单的 REST 取代 | 企业优先、规范完整、门槛高 |

**规律**：为"未来场景"提前设计的互操作标准，通常要等场景真正出现才会起来，否则会被更简单的方案取代。

**A2A 的不同之处**：背后有真实的 LLM Agent 需求、三大云和大型 SaaS 的支持，以及已经出现的规模化落地（鸿蒙、微信）。另外，A2A 1.0 主动向 REST 风格靠拢（无状态、HTTP+JSON 绑定、复用现有网关和负载均衡），吸取了 SOAP 的教训。所以它更可能是"慢热"，而不是重蹈 FIPA 的覆辙，但这一点还需要观察。

---

## 十一、落地场景、新变化与起飞条件

### 已经在用的场景

| 场景 | 例子 | 为什么必须用 A2A 这类协议 |
|---|---|---|
| 系统助手 ↔ 应用内 Agent | 鸿蒙小艺把长任务交给 App Agent、通过 Agent 操作 App UI、请求上下文推荐 | 系统厂商和 App 开发者是不同主体，Agent 互不透明 |
| 超级 App ↔ 多家系统助手 | 微信对接华为及其他安卓厂商助手，发消息、打电话，双重授权 | 一个接口对接多家厂商 |
| 企业 SaaS 互通 | Salesforce、SAP、ServiceNow、Atlassian 在产品间打通工作流 | 跨产品、跨厂商 |
| 企业助手接入外部 Agent | Copilot Studio、Gemini Enterprise 把第三方 A2A Agent 接给员工使用 | Agent 由不同供应商开发和运营 |
| 跨云 Agent 互调 | Google、Azure、AWS 平台上的 Agent 互相通信 | 不同云、不同框架 |
| Agent 电商 | 购物 Agent 与商家 Agent 在商品发现、定价、履约中协作，AP2 负责支付授权 | 买卖双方是不同主体 |

### 正在发生的变化：补开发者这一边

A2A 社区显然意识到了第九节的问题。2026-09 的官方路线图把"A2A CLI 与编程 Agent 集成"列为近期重点，原话是要 "bridge the adoption gap in coding harness tools"（弥合在编程 Agent 工具中的采用缺口）。

2026-10-01 发布的官方 A2A CLI 做了两件事：

- **让任何能执行命令的东西都成为 A2A Client**：shell 脚本、CI 流水线，以及通过配套 Agent Skill 驱动 CLI 的 Claude Code、Cursor、Codex。这等于绕过"宿主不原生支持"的问题，用命令行把开发者工具接上 A2A。
- **降低做 Server 的门槛**：`a2a server --exec` 可以把任意读 stdin、写 stdout 的程序包装成 A2A Server（官方说明仅用于学习和测试）。

这正是第十二节"经验 2"说的做法：自己先把双边中的一边补上。效果如何还要观察。

### 起飞条件

| 条件 | 进展 |
|---|---|
| 主流 Agent 宿主支持调用外部 A2A Agent | 企业宿主已有（Copilot Studio、Gemini Enterprise）；开发者 / 消费级宿主（ChatGPT、Claude、Claude Code、Cursor）未原生支持，目前靠 A2A CLI + Skill 间接接入 |
| 系统助手调用 App Agent 成为普遍形态 | 鸿蒙、微信已开始，最有希望 |
| Agent 电商、支付规模化 | AP2 早期 |
| Agent 目录 / 市场解决发现与信任 | Google AI Agent Marketplace、AWS Marketplace 可售卖 A2A Agent，但仍限于各自平台内 |
| MCP 与 A2A 互通或桥接，开发者不必二选一 | 同属 AAIF，社区有桥接项目，官方无合并计划 |

---

## 十二、经验与启示

### 经验 1：标准要跟着场景走，而不是领先场景太多

A2A 的设计不差，但"跨组织 Agent 协作"的场景还没成熟。FIPA、UDDI 都是前车之鉴。

> **做法**：设计标准或平台能力前，先确认有足够多的真实用户**今天**就在为这个问题付出代价。

### 经验 2：双边协议必须先解决"鸡和蛋"

MCP 一边（客户端）先被 Claude Desktop 铺满，另一边（服务端）写完立刻有用；A2A 两边都要从零开始，直到 2026-10 才用官方 CLI 补开发者这一边。

> **做法**：设计双边协议时，要么自己先提供一边（官方宿主、参考客户端），要么让单边就能产生价值。

### 经验 3：开发者体验决定扩散速度

"十分钟见效" vs "学完一堆概念后没有对端可连"。

> **做法**：衡量一个协议或 SDK 时，把"从零到看到第一个效果要多久"当作核心指标。

### 经验 4：企业优先的采用是隐形的

A2A 有生产落地，但都在平台内部，开发者看不到，于是被认为"没人用"。

> **做法**：评估一项技术的采用情况时，不要只看 GitHub 和社区热度，还要看云平台、企业产品和终端设备里的集成。

### 经验 5：已经有被广泛接受的方案时，新方案的增量价值要足够大

A2A 很多能力 MCP 也能凑合实现，于是大多数人选择不学第二个协议。

> **做法**：推出新方案前，问一句"用户能不能用现有方案凑合解决"。如果能，新方案的增量价值要非常明显，或者干脆做成现有方案的扩展。

### 经验 6：合并胜过分裂

IBM 把 ACP 并入 A2A，Google 和 Anthropic 发起的协议最终进入同一个基金会，避免了 Agent 协议的碎片化。

> **做法**：发现有同类标准时，优先考虑合并或分层共存，而不是另起炉灶。

### 经验 7：中立治理是企业采用的前提

A2A 刚发布时由 Google 单方主导；捐给 LF 时，AWS、Cisco、Microsoft、Salesforce、SAP、ServiceNow 一起成为创始成员，其中不乏 Google 的直接竞争对手。

> **做法**：希望竞争对手也采用的标准，要尽早交给中立组织。

### 经验 8：尽早确立版本与废弃规则

A2A 1.0 明确了"旧名保留到下一个大版本"和多版本共存的 Agent Card，减轻了迁移痛苦。

> **做法**：协议一旦有外部使用者，就公布版本协商机制和废弃政策。

### 经验 9：协议解决不了商业问题

跨组织协作的主要障碍是信任、计费、责任、合规，这些需要合同和行业规则。

> **做法**：设计跨组织集成时，把商业和法律层面的问题与协议设计分开梳理，不要指望协议本身解决它们。

---

## 十三、什么时候该用 A2A：决策清单

```
要对接的对端是什么？
├─ 工具 / API / 数据源 ─────────────────────────────► MCP
└─ 另一个自主 Agent
     ├─ 同一团队、同一框架、可共享上下文 ─────────────► 框架内编排（子 Agent）
     ├─ 同一组织、不同团队，只需"调一下拿结果" ────────► 把对方 Agent 包成 MCP 工具
     └─ 不同组织 / 不同厂商 / 互不透明
          ├─ 需要发现、签名身份、多种认证方式 ─────────► A2A
          ├─ 需要小时到天级的长任务和推送 ────────────► A2A
          └─ 需要中途补充信息或授权（多轮交互）─────────► A2A
```

自查问题：

- [ ] 对端 Agent 是否由**另一个主体**开发和运营？
- [ ] 是否**不能**也不应共享内部上下文、工具和记忆？
- [ ] 是否需要标准化的**发现**和**身份验证**？
- [ ] 任务是否**长时间运行**，需要状态追踪、取消、推送？
- [ ] 对端是否**已经支持** A2A（否则双边成本很高）？
- [ ] 计费、责任、合规等**非技术问题**是否已经有方案？

前三项都是"是"时，A2A 才明显优于"把 Agent 包成 MCP 工具"。

---

## 术语表

| 术语 | 解释 |
|---|---|
| A2A（Agent2Agent） | Agent 之间发现、委派、协作的开放协议，Google 发起，现属 AAIF |
| MCP（Model Context Protocol） | Agent 连接工具与数据的协议，Anthropic 发起，现属 AAIF |
| AAIF（Agentic AI Foundation） | Linux Foundation 旗下基金会，托管 MCP、A2A、goose、AGENTS.md、agentgateway 等项目 |
| A2A Client / Server | 发起请求的一方 / 暴露 A2A 端点的远端 Agent |
| Agent Card | Agent 的能力名片（JSON），默认位于 `/.well-known/agent-card.json` |
| Extended Agent Card | 认证后才能获取的扩展名片 |
| Signed Agent Card | 带 JWS 签名的 Agent Card，用于验证身份与完整性 |
| Skill | Agent Card 中声明的一项能力，含描述、标签、示例和输入输出模态 |
| Task | A2A 的基本工作单元，有唯一 ID 和状态机 |
| Context | 把相关 Task 和 Message 归为一组的标识，类似会话 |
| Message / Part / Artifact | 一轮通信 / 最小内容单元 / 任务交付物 |
| 终态 / 中断态 | 任务结束、不再接收消息的状态 / 暂停等待 Client 补充信息或授权的状态 |
| INPUT_REQUIRED / AUTH_REQUIRED | 任务中途需要补充信息 / 需要补充授权的状态 |
| Push Notification | Server 主动 POST 到 Client webhook 的任务更新 |
| Protocol Binding | 抽象操作到具体协议的映射：JSON-RPC、gRPC、HTTP+JSON |
| JSON-RPC 2.0 | 基于 JSON 的远程过程调用格式 |
| gRPC | Google 开源的高性能 RPC 框架，基于 HTTP/2 和 Protocol Buffers |
| Protocol Buffers（proto） | Google 的结构化数据定义与序列化格式，A2A 1.0 用它作为规范的唯一权威来源 |
| ProtoJSON | Protocol Buffers 的标准 JSON 映射，A2A 1.0 的枚举写法（如 `TASK_STATE_COMPLETED`）来自它 |
| SSE（Server-Sent Events） | 服务端通过一个 HTTP 响应持续推送消息的 Web 标准，A2A 用于流式更新 |
| Webhook | Client 提供的回调 URL，Server 有事件时主动 POST 过去 |
| A2A-Version | 版本协商用的请求头 |
| application/a2a+json | A2A 的 HTTP 媒体类型 |
| A2A CLI | A2A 官方命令行工具，可作为 Client 调用任意 A2A Agent，也可把脚本包装成 Server |
| TSC（Technical Steering Committee） | A2A 技术指导委员会 |
| OAuth 2.0 / OIDC | 授权框架 / 基于 OAuth 的身份认证层（OpenID Connect） |
| mTLS | 双向 TLS，客户端和服务端互相验证证书 |
| PKCE | OAuth 授权码流程的防截获扩展 |
| JWS | JSON Web Signature，JSON 数据的签名格式 |
| SSRF | 服务端请求伪造：诱使服务端去访问攻击者指定的内网或敏感地址 |
| elicitation（MCP） | MCP 服务端在工具执行中向用户追问确认或参数的机制 |
| MRTR（MCP） | Multi Round-Trip Requests，MCP 2026-07 版规范中以"返回问题 + 客户端带答案重发"实现中途追问的机制 |
| Tasks（MCP） | MCP 的长任务扩展，2025-11 实验性引入，2026-07 成为正式扩展 |
| ACP（Agent Communication Protocol） | IBM 提出的 Agent 通信协议，2025-08 并入 A2A（注意与 OpenAI 的 Agentic Commerce Protocol 同名） |
| AP2（Agent Payments Protocol） | Google 发起的 Agent 支付授权协议，可作为 A2A 的扩展使用 |
| AGNTCY | Cisco 发起并贡献给 LF 的 Agent 协作基础设施项目 |
| ADK（Agent Development Kit） | Google 开源的 Agent 开发框架，0.3 起原生支持 A2A |
| Gemini Enterprise | Google 的企业 AI 助手平台（原 Agentspace），可注册 A2A Agent 供员工使用 |
| Copilot Studio | Microsoft 的 Agent 构建平台，可把外部 A2A Agent 接为子 Agent |
| KQML / FIPA-ACL | 1990s–2000s 的 Agent 通信语言标准，A2A 思想上的前身 |
| UDDI | 2000 年代的 Web 服务注册与发现标准，公共注册中心已关闭 |
| 双边网络效应 | 平台两边的用户互相依赖，一边越多另一边越有价值，冷启动困难 |

---

## 参考文献

**A2A 规范与官方文档**

1. A2A 官方规范（v1.0）：<https://a2a-protocol.org/latest/specification/>
2. A2A 官方文档：What's New in A2A Protocol v1.0（0.3 → 1.0 迁移指南）：<https://a2a-protocol.org/latest/whats-new-v1/>
3. A2A 官方文档：What is A2A? / Key Concepts / Life of a Task：<https://a2a-protocol.org/latest/topics/what-is-a2a/>
4. A2A 官方文档：A2A and MCP：<https://a2a-protocol.org/latest/topics/a2a-and-mcp/>
5. A2A 官方路线图（2026-09-15 更新）：<https://a2a-protocol.org/latest/roadmap/>
6. A2A GitHub Releases（v0.2.x、v0.3.0、v1.0.0、v1.0.1 变更记录）：<https://github.com/a2aproject/A2A/releases>
7. A2A v0.3.0 规范（JSON-RPC 方法名 message/send 等）：<https://a2a-protocol.org/v0.3.0/specification/>

**版本发布与官方博客**

8. Google Developers Blog：Announcing the Agent2Agent Protocol (A2A)（2025-04-09）：<https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/>
9. Google Cloud Blog：Announcing a complete developer toolkit for scaling A2A agents on Google Cloud（2025-08-01，v0.3 发布，作者 Rao Surapaneni、Philip Stephens）：<https://cloud.google.com/blog/products/ai-machine-learning/agent2agent-protocol-is-getting-an-upgrade>
10. A2A Blog：A2A Protocol Ships v1.0（2026-03-12）：<https://a2a-protocol.org/latest/blog/2026/03/12/a2a-protocol-ships-v10-production-ready-standard-for-agent-to-agent-communication/>
11. Linux Foundation：A2A Protocol Surpasses 150 Organizations（2026-04-09）：<https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year>
12. AAIF：A2A joins AAIF's open agentic stack（2026-08-17）：<https://aaif.io/blog/a2a-joins-aaif>
13. A2A Blog：A New Chapter for A2A: Joining the AAIF（2026-08-27）：<https://a2a-protocol.org/latest/blog/2026/08/27/a-new-chapter-for-a2a-joining-the-agentic-ai-foundation/>
14. A2A Blog：Meet the A2A CLI（2026-10-01）：<https://a2a-protocol.org/latest/blog/2026/10/01/introducing-a2a-cli>

**平台集成**

15. AWS Machine Learning Blog：Introducing agent-to-agent protocol support in Amazon Bedrock AgentCore Runtime（2025-11）：<https://aws.amazon.com/blogs/machine-learning/introducing-agent-to-agent-protocol-support-in-amazon-bedrock-agentcore-runtime>
16. Google Cloud 文档：Register and manage A2A agents（Gemini Enterprise）：<https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent>
17. Microsoft Learn：Connect to an agent over the Agent2Agent (A2A) protocol（Copilot Studio）：<https://learn.microsoft.com/en-us/microsoft-copilot-studio/add-agent-agent-to-agent>
18. Google Cloud Blog：Announcing Agent Payments Protocol (AP2)：<https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol>
19. Claude Code Issue #104852：Native A2A Protocol v1.0 Client Support（社区功能请求）：<https://github.com/anthropics/claude-code/issues/104852>

**分析与评论**

20. A2A GitHub Discussion #955：How do A2A and MCP relate?（2025-08，含"很难找到支持 A2A 的客户端"的评论）：<https://github.com/a2aproject/A2A/discussions/955>
21. Cisco Blogs（Rob Barton）：MCP and A2A: A Network Engineer's Mental Model for Agentic AI（2026-01-29）：<https://blogs.cisco.com/ai/mcp-and-a2a-a-network-engineers-mental-model-for-agentic-ai>
22. Credal：A2A Protocol Explained: What It Is, A2A vs MCP and Is It Dead?（2026-03-06，2026-11 更新；结论是"没死，只是更安静"；作者方销售 MCP 网关产品）：<https://credal.ai/blog/what-happened-to-a2a-protocol>
23. Cognition（Walden Yan）：Don't Build Multi-Agents（2025-06-12）：<https://cognition.com/blog/dont-build-multi-agents>
24. Cognition（Walden Yan）：Multi-Agents: What's Actually Working（2026-04-22）：<https://cognition.com/blog/multi-agents-working>
25. Turing Post：A2A v1.0 Guide for 2026：<https://www.turingpost.com/p/a2a>
26. Hidekazu Konishi：Tool Use and Agent Protocol History and Timeline：<https://hidekazu-konishi.com/entry/tool_use_and_agent_protocol_history_and_timeline.html>

**MCP 与基金会**

27. Linux Foundation：Formation of the Agentic AI Foundation（2025-12-09）：<https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation>
28. Anthropic：Introducing the Model Context Protocol（2024-11-25）：<https://www.anthropic.com/news/model-context-protocol>
29. MCP 规范 2025-03-26 变更记录（加入 OAuth 2.1 授权框架）：<https://modelcontextprotocol.io/specification/2025-03-26/changelog>
30. MCP 规范 2025-11-25 变更记录（实验性 Tasks）：<https://modelcontextprotocol.io/specification/2025-11-25/changelog>
31. MCP Blog：The 2026-07-28 Specification（MRTR、Tasks 扩展、12 个月废弃政策）：<https://blog.modelcontextprotocol.io/posts/2026-07-28/>

**历史标准**

32. Wikipedia：Foundation for Intelligent Physical Agents（FIPA）：<https://en.wikipedia.org/wiki/Foundation_for_Intelligent_Physical_Agents>
33. Wikipedia：Knowledge Query and Manipulation Language（KQML）：<https://en.wikipedia.org/wiki/Knowledge_Query_and_Manipulation_Language>
34. Wikipedia：Web Services Discovery / UDDI：<https://en.wikipedia.org/wiki/Web_Services_Discovery>
