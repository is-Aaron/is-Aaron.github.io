---
title: "MCP 协议演进史（2024—2026）：从借鉴 LSP 到无状态核心"
date: 2026-11-16 10:00:00 +0800
categories: [人工智能, Agent]
tags: [MCP, A2A, AI Agent, 协议设计, Streamable HTTP, LSP, AAIF]
---

> **核心观点**：MCP 诞生于"本机桌面应用之间的集成"这个场景，因此借鉴了同样为本地场景设计的 LSP，继承了它的协议骨架：JSON-RPC、stdio、握手、有状态会话、双向请求。它的首个远程传输（HTTP+SSE）犯了分布式 Web 服务的常识性错误。此后约 20 个月的改版，本质上是在偿还"本地协议被当成云端基础设施使用"的技术债：到 2026-07，从 LSP 继承来的握手、会话、服务端主动请求都被拿掉，只留下 JSON-RPC 和 stdio。（对设计者是否"外行"的评价存在分歧，见第八节。）

本文按时间顺序整理 MCP（Model Context Protocol）的发展历史：它从哪里来、每一版改了什么、为什么要改，以及背后的公司与人物；再按专题拆开传输层、状态管理、弊端与 A2A 的关系；最后整理可复用的协议 / API 设计经验。

围绕的几个问题：

1. MCP 是怎么来的？为什么借鉴 LSP？
2. MCP 有什么弊端？为什么改了那么多版？
3. stdio、SSE、Streamable HTTP、MRTR 分别是什么？
4. MCP（尤其是 HTTP+SSE 传输）是不是外行设计的？
5. A2A 发展得怎么样？和 MCP 有什么区别？
6. 从这些设计失误里能学到什么？

---

## 一、全景：五个阶段

| 阶段 | 时间 | 标志事件 | 协议形态 |
|---|---|---|---|
| 前史 | 2016-06 — 2024-06 | LSP、ChatGPT Plugins、Function Calling | — |
| 诞生 | 2024-07 — 2024-11 | 想法提出 → 开源发布（规范 2024-11-05） | 本地优先：stdio + HTTP+SSE；有状态、双向、无认证 |
| 走向远程 | 2025-03 — 2025-06 | Streamable HTTP、OAuth 2.1；OpenAI、Google、Microsoft 相继采纳 | 远程可用，但仍是有状态会话 |
| 治理与生态 | 2025-07 — 2025-12 | 治理模型、官方 Registry、实验性 Tasks、捐给 AAIF | 开放治理；生态开始补救上下文浪费问题 |
| 无状态核心 | 2026-01 — 2026-11 | 维护者多元化、2026-07-28 规范、A2A 加入 AAIF | 请求自描述，无握手、无会话，服务端不再主动发请求 |

四个规范版本一览：

```
2024-11-05  首版：stdio + HTTP+SSE，为 Claude Desktop 接本地插件设计
     │        缺口：远程场景粗糙、无认证
2025-03-26  Streamable HTTP 取代 HTTP+SSE；引入基于 OAuth 2.1 的授权框架；
     │        工具注解；新增 JSON-RPC batching
     │        缺口：OAuth 与企业身份体系难以对接
2025-06-18  MCP 服务端定位为 OAuth 资源服务器、强制 RFC 8707、结构化输出、elicitation
     │        移除不到 3 个月前刚加的 JSON-RPC batching（加了又删）
2025-11-25  一周年版：实验性 Tasks、推荐 CIMD 注册、URL 模式 elicitation、
     │        sampling 支持工具调用；正式确立治理结构（SEP-932）与 SDK 分级
     │        缺口：有状态会话 + 长连接仍难以水平扩展
2026-07-28  无状态核心：去掉 initialize 握手和 Mcp-Session-Id
              MRTR、Mcp-Method/Mcp-Name 请求头路由、列表结果可缓存
              Tasks 成为正式扩展；DCR 废弃改 CIMD；RFC 9207 iss 校验
              废弃 Roots / Sampling / Logging 与旧 HTTP+SSE 传输（≥12 个月过渡期）
```

> 下文日期与事件均来自文末所列官方公告、规范 changelog、当事人访谈或主流媒体；标有"（约）"的为月份级精度。

---

## 二、前史（2016—2024）：工具调用是怎么一路走到 MCP 的

```
2016-06  Microsoft 联合 Red Hat、Codenvy 发布 LSP（语言服务器协议）
           → 解决"M 个编辑器 × N 种语言"的集成问题，后来 MCP 的设计原型
2023-03  OpenAI 推出 ChatGPT Plugins（基于 OpenAPI 描述外部服务）
           → 第一次大规模"让模型调外部服务"的尝试，2024 年逐步下线
2023-06  OpenAI 在 API 中推出 Function Calling
           → 各家模型陆续跟进，"工具调用"成为模型的标准能力
           → 但每个应用都要自己写对接代码，M 个应用 × N 个工具 = M×N 套集成
```

Function Calling 解决了"模型能不能调工具"，但没有解决"工具能不能只写一次、到处可用"。后者正是 MCP 要解决的问题。

---

## 三、诞生（2024-07 — 2024-11）：一个本地集成的痛点

### 起源：从复制粘贴到 M×N 问题

据两位共同发明人在 Latent Space、Software Engineering Daily、a16z 等播客中的讲述，MCP 的起源顺序是这样的：

1. **痛点（2024-07 前后）**：David Soria Parra 当时刚加入 Anthropic 两三个月，负责内部开发者工具。他一边用 Claude Desktop（能画 Artifacts，但无法访问文件系统和外部系统），一边用编辑器 Zed（能访问代码，但没有这些可视化能力），总在两者之间来回复制粘贴。他希望能用"小脚本"去扩展这些应用。
2. **抽象成 M×N 问题**：多个 AI 应用 × 多个集成，每对都要单独写。他认为答案是定一个协议。
3. **借鉴 LSP**：他当时恰好在做一个与 LSP 相关的内部实验（后来没做成），对 LSP 规范很熟。用他的话说，定协议时不想重新发明轮子，先找现成模式来套，"LSP 是一个很好的匹配对象"，因为 LSP 正是解决"编辑器 × 编程语言"这个 M×N 问题的。
4. **两人小项目**：他把想法讲给 Justin Spahr-Summers，Justin 认定这件事重要，主导了 Claude Desktop 侧的首个集成；David 做了 Zed 侧的概念验证。两人用大约一个半月做出协议和首批集成，同时写了 TypeScript、Python、Rust 三种语言的 SDK。

因果关系可以概括为：

```
本地桌面应用之间的集成痛点
   → 抽象成 M×N 问题，需要一个协议
   → 找同类问题的成熟方案：LSP
   → 随之带来 JSON-RPC、stdio、initialize 握手、有状态会话、双向请求
```

**stdio 和"借鉴 LSP"都是"本地场景"这个起点的结果，而不是因为先选了 stdio 才去借鉴 LSP。** 而且首版也不是只有 stdio，同时还定义了 HTTP+SSE 远程传输。

### 从 LSP 借了什么，没借什么

MCP 规范概述中写道："MCP takes some inspiration from the Language Server Protocol"。Justin 在访谈中的说法更具体：LSP 带给 MCP 的核心是同一个原则（用一个公共协议把 M×N 变成 M+N）；具体设计上主要拿了 JSON-RPC 和双向通信，"之后很快走了不同的路"。此外还沿用了 LSP 的一条原则：按"功能在应用里如何呈现"来划分原语，而不只看语义。

所以准确地说，MCP **借鉴的是 LSP 的协议骨架，原语（Tools、Resources、Prompts 等）则是自己设计的**：

| LSP 设计 | MCP 首版 | 在 LSP 里为什么合理 |
|---|---|---|
| JSON-RPC 2.0 | 沿用 | 简单通用 |
| stdio 传输 | 沿用 | 编辑器与语言服务器同机 |
| initialize 握手 + 能力协商 | 沿用 | 一个窗口对应一个服务器，生命周期一致 |
| 有状态会话 | 沿用 | 需要记住打开的文件和索引 |
| 双向请求 | 沿用 | 服务器要主动推诊断、请求编辑 |
| 无认证 | 同样未定义 | 本机进程间无需认证 |
| 语言相关的功能（补全、跳转、诊断……） | 不适用，自行设计 Tools / Resources / Prompts / Sampling / Roots | — |

问题在于场景错位：

```
设计时的场景                      实际被拿来做的事
本机 App ↔ 本地进程               云端 Agent ↔ 远程 SaaS
一对一，生命周期一致               多对多，负载均衡后多实例
信任边界在本机                     跨公司、跨网络
个人开发者                         企业：SSO、审计、多租户
几个插件                           几十个服务端、上百个工具
```

**在 LSP 里是优点的设计，放到云上全变成了负担。** 后面每一次大改版，几乎都能在这张表里找到原因。

### 首版规范（2024-11-05）与开源发布（2024-11-25）

- **传输**：stdio（客户端把服务端作为子进程启动，换行分隔的 JSON-RPC）和 HTTP+SSE（用于远程，细节见第七节）。
- **服务端原语**：Tools（模型调用的动作）、Resources（应用控制的数据）、Prompts（用户触发的模板）。
- **客户端原语**：Sampling（服务端借用客户端的模型）、Roots（告知服务端可访问的目录）。
- **其他**：Logging 等工具类能力；没有定义认证。

2024-11-25，Anthropic 正式开源 MCP。早期采用者有 Block、Apollo，开发工具 Zed、Replit、Codeium、Sourcegraph。

### 两位设计者

- **David Soria Parra**：PHP 5.4 / 5.5 版本发布负责人，Mercurial 贡献者，此后约十年在 Facebook / Meta 做源代码管理、开发者工具与静态分析；MCP 的想法由他提出。
- **Justin Spahr-Summers**：ReactiveCocoa 共同作者；据 David 本人说法，是 Justin 认定这件事重要并推动落地。他自己说此前主要做产品和基础设施，LSP 对他来说是新东西。

两人都是开发者工具与协议方向的资深工程师。

---

## 四、走向远程（2025-03 — 2025-06）：补传输、补认证、被行业采纳

### 2025-03-26 规范：第一次大修

- **Streamable HTTP 取代 HTTP+SSE**（PR #206，Justin Spahr-Summers 提出，距首版仅 4 个月）。
- **基于 OAuth 2.1 的授权框架**：首次定义认证。
- **工具注解**：描述工具是否只读、是否有破坏性等。
- **JSON-RPC batching**：在 PR #206 的讨论中被社区提出，Justin 坦言"差点忘了 JSON-RPC 还有这个能力"，随后单独加入。
- 其他：音频内容、参数补全（completions）能力、进度通知带描述信息。

### 行业采纳：竞争对手相继加入

| 日期 | 事件 | 主导方 / 关键人物 |
|---|---|---|
| 2025-03-19 | Copilot Studio 支持 MCP（第一家大厂产品级支持） | Microsoft |
| 2025-03-26 | **OpenAI 宣布采纳 MCP**（Agents SDK 起步）——竞争对手采纳，MCP 开始成为行业标准 | OpenAI / Sam Altman |
| 2025-04-09 | **Google 在 Cloud Next 发布 A2A**，50+ 合作伙伴 | Google Cloud / Rao Surapaneni、Todd Segal 等 |
| 2025-04-09 | 同日，Google DeepMind 宣布 Gemini 支持 MCP | Google DeepMind / Demis Hassabis |
| 2025-05-07 | Azure AI Foundry、Copilot Studio 支持 A2A | Microsoft |
| 2025-05-19 | Build 2025：Windows 11 原生支持 MCP；Microsoft 与 GitHub 加入 MCP 指导委员会 | Microsoft |
| 2025-05-21 | Responses API 支持远程 MCP；OpenAI 加入 MCP 指导委员会 | OpenAI |

### 2025-06-18 规范：补授权、加交互

- **移除 JSON-RPC batching**：加入不到 3 个月就删掉。
- **结构化工具输出**、工具结果中的**资源链接**。
- **授权加固**：MCP 服务端被定位为 OAuth 资源服务器（通过受保护资源元数据发现授权服务器）；客户端必须实现 RFC 8707 Resource Indicators，防止恶意服务端骗取令牌。授权规范的共同作者 Den Delimarsky（时任 Microsoft）主导引入了 RFC 8707。
- **Elicitation**：服务端可以在交互中向用户追问信息。
- HTTP 传输下后续请求必须携带 `MCP-Protocol-Version` 请求头。

同月，A2A 被捐给 Linux Foundation（2025-06-23），创始成员包括 AWS、Cisco、Google、Microsoft、Salesforce、SAP、ServiceNow。

**这一阶段留下的缺口**：OAuth 方案与企业的 SSO / IdP 体系难以对接；会话和长连接依然存在。

---

## 五、治理与生态（2025-07 — 2025-12）：从两人项目到中立基金会

| 日期 | 事件 | 主导方 / 关键人物 |
|---|---|---|
| 2025-07-16 | Bedrock AgentCore 预览版，支持 MCP 与 A2A（2025-10-13 正式发布） | AWS |
| 2025-07-30 | A2A v0.3：签名 Agent Card、mTLS | A2A 社区 |
| 2025-07-31 | **MCP 正式引入治理模型**：SEP 提案流程（参考 Python PEP、Rust RFC），Maintainer / Core Maintainer / Lead Maintainer 三级角色 | MCP 核心团队 / David Soria Parra |
| 2025-08（约） | **IBM 的 ACP 并入 A2A**，Agent 间通信协议基本统一 | IBM Research |
| 2025-09-08 | **官方 MCP Registry 预览版上线**：公开 MCP 服务端的目录与 API，支持公有 / 私有子 Registry | MCP Registry 工作组 |
| 2025-09（约） | Google 联合 PayPal 等发布 AP2（基于 A2A 的 Agent 支付协议）；OpenAI 在 ChatGPT 开发者模式中支持 MCP | Google、PayPal；OpenAI |
| 2025-09-26 | Cloudflare 提出 **Code Mode**：让模型写代码调用 MCP 工具，减少上下文消耗 | Cloudflare |
| 2025-10（约） | OpenAI DevDay 发布 Apps SDK，**基于 MCP 构建** | OpenAI |
| 2025-10（约） | Anthropic 推出 Agent Skills（Claude Code 中的技能机制） | Anthropic |
| 2025-11-04 | Anthropic 发文 "Code execution with MCP"，推荐以代码方式调用工具 | Anthropic |
| 2025-11-24 | Anthropic API 推出 Tool Search 与 Programmatic Tool Calling，支撑大规模 MCP 部署 | Anthropic |
| 2025-11-25 | MCP 一周年规范（见下） | MCP 核心团队 / Den Delimarsky 牵头发布 |
| 2025-12-09 | **Linux Foundation 成立 AAIF**：Anthropic 捐 MCP、Block 捐 goose、OpenAI 捐 AGENTS.md | Anthropic、Block、OpenAI 联合创始 |
| 2025-12（约） | Agent Skills 作为开放标准发布 | Anthropic |

### 2025-11-25 规范：一周年版

- **实验性 Tasks**：可追踪的长任务，支持轮询与延后获取结果。
- **授权**：推荐使用 CIMD（客户端 ID 元数据文档）注册客户端；支持 OpenID Connect Discovery；通过 `WWW-Authenticate` 增量申请权限。
- **交互**：URL 模式 elicitation；sampling 支持工具调用。
- **传输**：允许服务端主动断开 SSE 流、客户端轮询恢复（SEP-1699），缓解长连接压力。
- **治理**：正式确立治理结构（SEP-932）、工作组 / 兴趣组、SDK 分级。

### 这一阶段的两条主线

1. **治理开放化**：从"两个人在一个房间里讨论"走向 SEP 流程，再走向中立基金会。Anthropic 在捐赠公告中说明治理模型保持不变。
2. **生态自救上下文问题**：工具定义全量注入导致的 token 浪费（见第十节）没有在协议层解决，于是出现了 Code Mode、代码执行、Tool Search、Skills 等一系列"绕开 MCP 原样用法"的方案。

**这一阶段留下的缺口**：有状态会话 + 长连接仍难以水平扩展。

---

## 六、无状态核心（2026）：拆掉从 LSP 继承的状态

| 日期 | 事件 | 主导方 / 关键人物 |
|---|---|---|
| 2026-03-12 | **A2A v1.0 稳定版**；技术指导委员会来自 AWS、Cisco、Google、IBM Research、Microsoft、Salesforce、SAP、ServiceNow | A2A TSC |
| 2026-04-02 | MCP Dev Summit 北美（纽约，04-02—03），约 1200 人参会 | AAIF |
| 2026-04-08 | MCP 维护者团队扩充：Den Delimarsky（此时已在 Anthropic）升任 Lead Maintainer，AWS 的 Clare Liguori 加入 Core Maintainer | MCP 治理团队 |
| 2026-04-09 | A2A 一周年：150+ 组织支持，三大云原生集成 | Linux Foundation |
| 2026-05-26 | A2A v1.0.1 | A2A 社区 |
| 2026-07-28 | **MCP 无状态核心规范**（见下） | MCP 核心团队 / David Soria Parra、Den Delimarsky |
| 2026-08-17 | **A2A 加入 AAIF**（Growth Stage 项目），与 MCP 同属一个基金会 | AAIF |
| 2026-10-01 | A2A 发布官方 CLI | A2A 社区 |
| 2026-10-22 | Anthropic 把 Agent Skills 捐给 AAIF，David Soria Parra 任 Lead Maintainer 之一 | Anthropic |

### 2026-07-28 规范：从"双向有状态"变成"请求 / 响应无状态"

官方原话：MCP 正在"从双向有状态协议转变为请求 / 响应式的无状态协议"。

- **去掉握手与会话**：取消 `initialize` / `initialized` 交换和 `Mcp-Session-Id`。每个请求在 `_meta` 中自带协议版本、客户端身份和能力；需要提前了解服务端能力时，可选调用 `server/discover`。任意请求都能落到负载均衡后的任意实例上。
- **服务端不再主动发起请求**：原来由服务端发起的 `elicitation/create`、`sampling/createMessage`、`roots/list` 改用 MRTR（见第九节）。
- **请求头路由**：Streamable HTTP 请求必须携带 `Mcp-Method`、`Mcp-Name`，网关、限流、WAF 不必解析请求体。
- **列表结果可缓存**：`tools/list` 等返回 `ttlMs`、`cacheScope`，顺序确定，便于客户端缓存工具目录、稳定提示词缓存。
- **扩展框架**：Tasks 成为正式扩展 `io.modelcontextprotocol/tasks`，与 MCP Apps、企业托管授权（EMA）等并列。
- **授权加固**：RFC 9207 `iss` 校验；客户端凭据绑定签发方；DCR 正式废弃，改用 CIMD。
- **废弃政策**：至少 12 个月的废弃窗口。Roots、Sampling、Logging 和旧 HTTP+SSE 传输被废弃。
- **SDK**：TypeScript、Python、Go、C# 四个 Tier 1 SDK 同步支持。

### 回看：从 LSP 继承的设计，最后去了哪里

| 继承自 LSP 的设计 | 结局 | 时间 |
|---|---|---|
| JSON-RPC 2.0 | 保留 | — |
| stdio 传输 | 保留，一直是标准传输 | — |
| 无认证 | HTTP 传输补上 OAuth 2.1，之后反复加固 | 2025-03 起 |
| initialize 握手 | 删除，改为每个请求自描述 | 2026-07 |
| 有状态会话 | 删除，业务状态改为显式句柄 | 2026-07 |
| 双向请求（服务端主动发请求） | 删除，改为 MRTR | 2026-07 |

**为什么改了这么多版（根因）：**

| 原因 | 说明 |
|---|---|
| 起点定位太小 | 为"本机 App ↔ 本地进程"设计，被拿去做云端、企业级基础设施 |
| 认证是后补的 | 首版无认证；直接套用 OAuth 又与企业 SSO/IdP 不兼容，DCR 最终被废弃，企业场景另起 EMA 扩展 |
| 采用速度远超设计速度 | 成千上万服务端已上线，只能先补丁，再攒成大版本还债 |
| 开放治理后决策变慢 | 走 SEP 流程，各厂商诉求不同 |
| 部分问题协议层无解 | 提示注入源于模型无法区分"数据"与"指令" |

---

## 七、专题：传输层演进

### 几个概念不在一个层次

```
┌─────────────────────────────────────────────┐
│ MCP 协议（tools/call、tools/list ...）       │
├─────────────────────────────────────────────┤
│ MCP 传输：stdio │ HTTP+SSE（旧） │ Streamable HTTP（新） │ ← MCP 规范定义
├─────────────────────────────────────────────┤
│ 流式技术：SSE（text/event-stream）           │ ← Web 标准
├─────────────────────────────────────────────┤
│ HTTP                                         │
└─────────────────────────────────────────────┘
```

类比：SSE 是"分批寄送"这种包装方式；Streamable HTTP 是某公司的收发货流程，规定"大件可以分批寄"。

```
2024-11  stdio ────────────────────────────────────────────────► 一直保留
2024-11  HTTP+SSE ──► 2025-03 被 Streamable HTTP 取代 ──► 2026-07 正式废弃（12 个月过渡）
2025-03  Streamable HTTP（有会话）──► 2025-11 SSE 流可断开轮询 ──► 2026-07 无会话、请求头路由
```

### stdio：唯一一直没被推翻的传输

- **首版即有，直接继承自 LSP**：客户端把服务端作为子进程启动，向它的 stdin 写入、从它的 stdout 读出 JSON-RPC 消息；每条消息以换行分隔，消息内不能有换行；stderr 可用于输出日志。
- **历次大改中一直作为标准传输保留**：被推翻的都是远程传输。原因是它的场景（同机、一对一、生命周期一致、信任边界在本机）正好是 LSP 设计成立的场景，不存在负载均衡、多实例、长连接这些问题。
- **2026-07-28 的协议层变化同样作用于 stdio**：协议语义在所有传输上一致，不再有 `initialize` 握手，每个请求在 `_meta` 里自带协议版本与客户端能力；服务端不再主动发起 JSON-RPC 请求；取消请求通过 `notifications/cancelled` 通知完成。规范还建议：基于可靠字节流（Unix domain socket、TCP）的自定义传输直接复用 stdio 的分帧方式。
- **它的问题在安全和共享，而不在扩展**：客户端配置里写的就是要执行的命令，"配置即命令"（见第十节）；服务端只能跑在本机，无法直接共享给团队或云端 Agent，这也是需要远程传输的原因。

### SSE（Server-Sent Events）基础

- Web 标准：一个 HTTP 响应不关闭，服务端持续推送文本消息。
- 格式：`data:` / `event:` / `id:` / `retry:`，空行分隔；断线后客户端带 `Last-Event-ID` 续传。
- 单向（服务端 → 客户端），纯文本，普通 HTTP，代理友好。大模型 API 的流式输出几乎都用它。

| | SSE | WebSocket |
|---|---|---|
| 方向 | 单向 | 双向 |
| 协议 | 普通 HTTP | HTTP 升级为独立协议 |
| 数据 | 文本 | 文本 / 二进制 |
| 断线重连 | 标准自带 | 需自行实现 |

### 旧：HTTP+SSE 传输（2024-11）

```
① GET /sse            建立全局长连接，服务端通过 endpoint 事件告知 POST 地址
② POST /messages      发请求，HTTP 响应不带结果（SDK 实现通常返回 202 Accepted）
③ 结果以 message 事件从 ① 的长连接推回来
```

问题：必须保持高可用长连接；请求与结果分离导致需要粘性会话或 Redis 转发；不支持断线恢复；不适合 Serverless。设计成因与失误复盘见第八节。

### 新：Streamable HTTP（2025-03 起）

- 单一地址（如 `/mcp`），客户端发出的消息一律走 POST。2026-07 之前还可用 GET 另开一条 SSE 流接收服务端主动消息；2026-07 起，变更通知改为客户端按需订阅的 `subscriptions/listen` 流。
- 服务端自行决定：直接返回 `application/json`，或把**这一次响应**升级为 `text/event-stream` 流式推送，推完即关。
- 2025-03 版本允许服务端用 `Mcp-Session-Id` 维持会话；2026-07 起去掉会话，完全无状态。

| | HTTP+SSE（旧） | Streamable HTTP（新） |
|---|---|---|
| 地址 | 2 个 | 1 个 |
| SSE 角色 | 全局长连接，所有回复都走它 | 可选，仅用于单次响应的流式返回 |
| POST 响应 | 202，不带结果 | 直接包含结果 |
| 扩展 / Serverless | 困难 | 友好 |

> 注意：**被取代的是"HTTP+SSE"这个 MCP 传输方式，不是 SSE 技术本身。**

### 为什么当时不用 WebSocket（PR #206 中的官方理由）

1. 简单的"RPC 式"工具调用，每次建 WebSocket 开销过大；
2. 浏览器 WebSocket 无法附加 `Authorization` 等请求头；
3. 只有 GET 能升级为 WebSocket，POST 需要两步升级；
4. 不希望官方传输方式过多，避免客户端/服务端兼容组合爆炸。

这些理由在 PR 评论区也有反对意见，例如认为前三条可以通过常见变通手段解决。

---

## 八、专题：HTTP+SSE 设计失误复盘

### 它为什么长这样：把 stdio 的两根管道原样搬上 HTTP

```
stdio：  客户端 ──stdin──► 服务端      客户端 ◄──stdout── 服务端
HTTP：   客户端 ──POST /messages──►   客户端 ◄══GET /sse══ 服务端
```

需要双向通信（服务端要主动发 sampling、通知），SSE 只能单向，于是"SSE 下行 + POST 上行"成了最直接的映射。

### 失误清单（均为 Web 后端常识）

| 失误 | 常识 |
|---|---|
| 请求与结果分走两条连接 | 负载均衡下必然需要粘性会话或消息总线 |
| 强制高可用长连接 | 与水平扩展、Serverless、代理超时冲突 |
| 不支持断线恢复 | SSE 标准自带 `Last-Event-ID`，却没用上 |
| 首版远程场景无认证 | 跨网络调用必须有认证 |
| 修正稿第一版由客户端生成会话 ID | 评审当场指出不安全，应由服务端生成并签名 |

PR #206 的 Motivation 一节自己承认了前三条的核心：不支持恢复（Does not support resumability）、要求高可用长连接（Requires the server to maintain a long-lived connection with high availability）、服务端消息只能走 SSE（Can only deliver server messages over SSE）。

### 谁设计、谁修正

- **HTTP+SSE（2024-11-05）**：出自 MCP 两位共同设计者 David Soria Parra、Justin Spahr-Summers 与 Anthropic 团队。首版在内部设计后开源，**具体由谁负责这一节没有公开记录**。
- **Streamable HTTP（2025-03）**：由 Justin Spahr-Summers 提出（PR #206），吸收了 Shopify、Pydantic、Cloudflare、Vercel、LangChain 等社区成员的意见。距 HTTP+SSE 发布仅 4 个月。

### 算不算"外行设计"

对这个问题有两种看法，均保留：

**看法 A：区分范围**

- 就 **HTTP+SSE 远程传输这一部分**而言，是外行水平的设计，Web 后端常识性错误明显。
- 但 MCP 协议主体（消息模型、能力协商、工具/资源抽象、stdio）是合格设计，后续多次大改基本未推翻。
- 因此：**"HTTP+SSE 是外行水平的设计"成立；"MCP 整体是外行设计的"范围过宽。**
- 设计者是协议与开发者工具方面的内行，在大规模分布式 Web 服务方面属于外行；首版规范流程又缺少这方面的内行把关，这是**流程问题**。

**看法 B：直接定性**

- 有盲区就是外行，外行就是外行，不需要"有一定道理"之类的折中措辞。
- 一个后来承载全行业远程调用的协议，在关键部分犯常识性错误，就应该直接称为外行设计。

措辞可以争，事实层面两种看法一致：**HTTP+SSE 犯了领域常识性错误，原因是设计者的专长不覆盖这个领域，且没有找该领域的人评审。**

---

## 九、专题：MRTR 与"状态"的拆分

### 为什么工具调用中途需要用户输入

| 场景 | 缺什么 | 旧机制 |
|---|---|---|
| 新建付费资源 | 用户确认费用 | elicitation |
| 执行删除数据的 SQL | 用户确认 | elicitation |
| 缺少必要参数 | 用户补参 | elicitation |
| 服务端需要模型能力 | 借客户端的模型生成 | sampling |
| 文件工具 | 允许访问的目录 | roots |

### 旧做法：在服务端挂起等待

```
客户端 ── tools/call ──────────────► 服务端实例 A（执行到一半）
客户端 ◄── elicitation/create ───────  │ 调用挂在 A 的内存里
       （用户思考 30 秒）              │ 连接必须一直开着
客户端 ── 回答 ────────────────────►  │ 必须回到 A
客户端 ◄── 结果 ──────────────────────┘
```

问题本质：**"这次调用执行到哪了"这个状态被藏在了服务器内存 + 长连接里。**

### MRTR：把问题返回给客户端，由客户端带着答案重发

```
第 1 轮：客户端 ── tools/call ──► 任意实例 A
         客户端 ◄── resultType: "input_required" + 需要回答的问题（A 不保留任何东西）
第 2 轮：客户端 ── tools/call + inputResponses ──► 任意实例 B
         客户端 ◄── 最终结果
```

与 HTTP 401 → 带凭证重发、"需要验证码 → 带验证码重新提交"是同一模式。

### 2026-07 后 MCP 对"状态"的拆分

| 状态类型 | 例子 | 旧做法 | 新做法 |
|---|---|---|---|
| 连接/会话状态 | 协议版本、客户端能力 | 握手后存在会话里 | 取消，每个请求自带 |
| 调用中途状态 | 等用户确认 | 服务端内存 + 长连接 | MRTR，客户端带回 |
| 长任务状态 | 20 分钟构建 | 一直连着 | Tasks 扩展，任务 ID + 轮询/订阅 |
| 业务状态 | 事务、购物车 | 隐式绑在会话 ID 上 | 工具返回显式句柄，模型作为参数传回 |

### MRTR 的代价

- 工具可能被执行两次 → **询问之前不能产生副作用**，必须"先确认后执行"，且幂等。
- 前半段计算昂贵时会重复计算 → 返回中间结果句柄，或改用 Tasks。
- 服务端编程模型改变：从"函数中途 await 用户回答"变为"可重复调用、每次检查信息是否齐全"。

---

## 十、专题：MCP 至今仍有的弊端

### 上下文浪费（最受诟病）

```
用户问问题之前，上下文里已经有：
┌──────────────────────────────────────┐
│ GitHub MCP: 40 个工具定义             │
│ Jira MCP:   30 个工具定义             │  ← 可能占掉几万 token
│ Slack MCP:  20 个工具定义             │
├──────────────────────────────────────┤
│ 用户的问题                            │
└──────────────────────────────────────┘
```

- 工具定义一次性全量注入；中间结果必须经过模型。Roadie 实测同任务 MCP 比 CLI 多耗 4~32 倍 token。
- 应对（多数来自生态而非协议本身，见第五节）：Code Mode（让模型写代码调工具）、按需加载工具（Tool Search）、Skills、直接用 CLI。2026-07 的列表缓存只缓解了重复拉取，没有解决全量注入。

### 安全：设计上默认信任

| 攻击 | 原理 |
|---|---|
| 工具投毒 | 工具描述 / 返回结果中藏指令，用户看不见、模型会执行 |
| Rug Pull | 先正常获批，后替换为恶意（Cursor MCPoison，CVE-2025-54136） |
| 跨服务器工具覆盖 | 恶意服务端影响模型对可信服务端的使用 |
| stdio 配置即命令 | 配置暴露或可注入即可执行任意命令 |
| 传统 Web 漏洞 | 路径穿越、SSRF 等，调用方换成了不知疲倦的 Agent |

CSA（2026-05）：规范本身对工具投毒、Rug Pull、跨服务器工具覆盖没有原生防御。

### 规范频繁变动、实现参差

- JSON-RPC batching 在 2025-03 加入、2025-06 移除；Sampling / Roots / Logging 首版即有，约 20 个月后（2026-07）被废弃。
- 很多客户端只实现了 Tools，同一服务端换客户端效果不同。

### 生态质量不稳定

- 大量"REST API 一对一包装"的低质量服务端，工具粒度过细、描述差。
- 官方 Registry 起步晚（2025-09 才上线预览版），缺少统一签名与信誉体系，供应链风险类似早期 npm。

### 与 A2A 边界模糊

- Tasks、MRTR 加入后，MCP 能力与 A2A 重叠扩大；权限与审计留给客户端/网关自行实现。

---

## 十一、专题：MCP 与 A2A，从竞争到分层

**两者已从"竞争"变成"分层互补"，并归到同一个中立基金会（AAIF / Linux Foundation）治理。** MCP 管"Agent 调工具"，是事实标准；A2A 管"Agent 之间协作"，2026 年 1.0 稳定，但主要用在跨厂商、跨组织的企业场景。

### A2A 时间线

```
2025-04  Google 发布 A2A（50+ 合作伙伴）
2025-06  捐给 Linux Foundation
2025-07  v0.3：签名 Agent Card、mTLS
2025-08  IBM ACP 并入 A2A
2025-12  Linux Foundation 成立 AAIF，MCP 捐入
2026-03  A2A v1.0（首个稳定版：多协议绑定、版本协商、多租户、签名 Agent Card、OAuth 现代化）
2026-04  一周年：150+ 组织，Google / Azure / AWS 原生集成
2026-05  v1.0.1
2026-08  A2A 加入 AAIF，与 MCP 同属一个基金会
2026-10  官方 CLI
```

落地案例：华为鸿蒙小艺 ↔ 应用内 Agent、微信对接安卓厂商系统助手、Google Cloud + PayPal 的 AP2 支付协议。

### 分层示意

```
        ┌──────────────┐    A2A（Agent ↔ Agent）  ┌──────────────┐
        │  Agent A     │◄────────────────────────►│  Agent B     │
        │ (公司 X)     │  发现：Agent Card         │ (公司 Y)     │
        └──────┬───────┘  委派：Task + 状态机      └──────┬───────┘
               │ MCP（Agent → 工具）                     │ MCP
        ┌──────┴───────┐                         ┌──────┴───────┐
        │ GitHub/DB/…  │                         │ ERP/支付/…   │
        └──────────────┘                         └──────────────┘
```

AAIF 的分层：AGENTS.md（指令与上下文）/ goose（运行时）/ **MCP（Agent → 工具）** / agentgateway（流量治理）/ **A2A（Agent ↔ Agent）**。

### 核心对比

| 维度 | MCP | A2A |
|---|---|---|
| 对端 | 透明的工具 / 资源 | 不透明的自主 Agent |
| 核心抽象 | Tools、Resources、Prompts | Agent Card、Task、Message、Artifact |
| 长任务 | Tasks 扩展（2026-07 才正式） | 原生 Task 生命周期 |
| 传输 | stdio、Streamable HTTP | JSON-RPC、gRPC、HTTP+JSON |
| 生态 | 极大，开发者门槛低 | 150+ 组织，企业为主，门槛较高 |

### 选型

| 场景 | 选择 |
|---|---|
| 给 Agent 接工具 / 数据 / 内部 API | MCP |
| 公司内同一框架的多 Agent 编排 | 框架自带机制，或把子 Agent 包成 MCP 工具 |
| 跨团队 / 跨厂商互相委派、需要发现与鉴权 | A2A |
| 对外开放 Agent 能力 | 外部 A2A + 内部 MCP |

---

## 十二、关键人物与背后公司

### 关键人物

**MCP 方面**

| 人物 | 所属 | 角色与贡献 |
|---|---|---|
| David Soria Parra | Anthropic | MCP 共同发明人，**想法提出者**；Lead Maintainer；主导治理模型；曾任 PHP 5.4/5.5 发布负责人，在 Meta 做了约十年开发者工具 |
| Justin Spahr-Summers | Anthropic（时任） | MCP 共同发明人，**推动落地**，主导 Claude Desktop 首个集成；提出 Streamable HTTP（PR #206）；现为 Lead Maintainer Emeritus（荣誉） |
| Den Delimarsky | 前 Microsoft CoreAI，现 Anthropic | **授权规范共同作者**，把 RFC 8707 引入规范；牵头 2025-11-25 版发布；2026-04 起任 Lead Maintainer |
| Nick Cooper | OpenAI | MCP Core Maintainer；在 AAIF 成立公告中代表 OpenAI 发言（捐赠 AGENTS.md） |
| Clare Liguori | AWS | 2026-04 起任 MCP Core Maintainer；负责 Kiro、Strands Agents SDK，参与 Tasks 与执行模型设计 |
| Mike Krieger | Anthropic（时任 CPO） | AAIF 成立时代表 Anthropic 发言："MCP 起初是为解决我们内部团队的问题" |
| Sam Altman | OpenAI（CEO） | 2025-03 宣布 OpenAI 全线支持 MCP，是 MCP 成为行业标准的转折点 |
| Demis Hassabis | Google DeepMind（CEO） | 2025-04 宣布 Gemini 支持 MCP |

其他现任 Core Maintainer：Peter Alexander、Caitie McCaffrey、Kurtis Van Gent、Paul Carleton（所属单位未核实）。

**A2A 方面**

| 人物 | 所属 | 角色与贡献 |
|---|---|---|
| Rao Surapaneni | Google Cloud（VP & GM, Business Application Platform） | A2A 发布公告第一作者，A2A 在 Google 侧的主要负责人与对外发言人 |
| Todd Segal | Google Cloud（Principal Engineer） | A2A 发布公告共同作者 |
| Miku Jha、Michael Vakoc | Google Cloud | A2A 发布公告共同作者（合作伙伴工程、产品） |

**基金会方面**

| 人物 | 所属 | 角色 |
|---|---|---|
| Jim Zemlin | Linux Foundation（执行董事） | 主导成立 AAIF |
| Manik Surtani | AAIF（CTO），此前任 Block 开源负责人 | AAIF 成立时代表 Block 发言；2026-08 代表 AAIF 接纳 A2A |

### 背后公司及其角色

| 公司 | 在 MCP 中的角色 | 在 A2A 中的角色 | 看点 |
|---|---|---|---|
| **Anthropic** | **发起者**，长期主导规范；捐给 AAIF | — | 用开源协议给 Claude 建生态，再捐出去换取中立性；之后又把 Agent Skills 捐给 AAIF |
| **Google** | Gemini、Vertex AI 采纳；AAIF 白金成员 | **发起者**；捐给 LF；推出 AP2 | 同一天发 A2A、宣布支持 MCP：工具层跟随，Agent 层自己定标准 |
| **OpenAI** | 最重要的早期采纳者；加入指导委员会；Apps SDK 基于 MCP | — | AAIF 联合创始（捐 AGENTS.md）；另有一个与 IBM ACP **同名**的 ACP（Agentic Commerce Protocol，商业支付） |
| **Microsoft** | 最早的大厂产品级支持；Windows 原生支持；加入指导委员会 | 创始成员、TSC；Foundry 原生支持 | 两边都押注；Den Delimarsky 来自 Microsoft |
| **AWS** | AgentCore 托管 MCP；贡献 Tasks 扩展；Clare Liguori 任 Core Maintainer | 创始成员、TSC；AgentCore 托管 A2A | 两边都深度参与 |
| **Block** | 从首版起就是早期采用者与贡献者；goose 是 MCP 参考实现 | — | AAIF 联合创始（捐 goose） |
| **IBM** | AAIF 金牌成员 | ACP 并入 A2A；TSC（IBM Research） | 主动放弃自有协议，并入 A2A |
| **Cisco** | AAIF 金牌成员 | 创始成员、TSC；AGNTCY 项目 | — |
| **Salesforce / SAP / ServiceNow** | AAIF 金牌成员（Salesforce、SAP） | 创始成员、TSC；在自家产品中用 A2A 打通工作流 | 企业 SaaS 是 A2A 的主要推动力 |
| **Cloudflare / Bloomberg** | AAIF 白金成员；Cloudflare 较早支持远程 MCP、提出 Code Mode，Agents SDK 首日支持 2026-07 新规范 | — | — |
| **华为 / 腾讯（微信）** | — | 鸿蒙小艺用 A2A 连接应用内 Agent；微信对接安卓厂商系统助手 | A2A 在移动端的规模化落地 |
| **PayPal** | — | 与 Google 合作 AP2 | Agent 电商支付 |
| **Linux Foundation / AAIF** | MCP 托管方（2025-12 起） | A2A 托管方（2025-06 LF，2026-08 起 AAIF） | 中立治理，让竞争对手能在同一个标准上合作 |

### 格局观察

1. **竞争对手采纳是标准成立的标志。** MCP 真正成为标准，不是 Anthropic 发布那天，而是 OpenAI（2025-03）、Google（2025-04）、Microsoft（2025-05）相继采纳的那两个月。
2. **"捐给中立基金会"是一种推广策略。** Anthropic 捐 MCP、Google 捐 A2A，都是为了让竞争对手愿意放心采用；代价是放弃单方面控制权。
3. **分层共存，而不是赢家通吃。** Google 在工具层直接采纳 MCP，在 Agent 协作层另立 A2A；两者最终进入同一个基金会，形成分层格局。
4. **主动合并比各自为战更有利。** IBM 把 ACP 并入 A2A，避免了 Agent 间通信协议的分裂。
5. **维护者多元化是协议成熟的标志。** MCP 维护者从 Anthropic 两人，扩展到 Microsoft、OpenAI、AWS 等多家公司的工程师；授权、Tasks 等关键改进分别由 Microsoft 出身的 Den Delimarsky 和 AWS 推动。治理规则也明确：维护者席位属于个人而非公司。
6. **注意名字冲突。** "ACP" 至少指两个不同协议：IBM 的 Agent Communication Protocol（已并入 A2A）和 OpenAI 的 Agentic Commerce Protocol（商业支付）。查资料时要看清上下文。

---

## 十三、从 MCP 早期设计失误中学到的经验

### 1. 借鉴成熟方案时，要同时检查它背后的场景假设

LSP 很成功，但它成立的前提是"同机、一对一、生命周期一致、可信边界在本机"。MCP 借来了协议骨架，却没有把这些前提一起列出来核对。

> **做法**：借鉴任何方案前，先写下"原方案成立的前提条件"，再逐条核对自己的场景是否满足。

### 2. 按"最终会被怎么用"来设计，而不是"第一个用户怎么用"

MCP 首版为 Claude Desktop 设计，规范里却写了远程传输，等于承诺了一个没认真设计过的场景。

> **做法**：规范里出现的每种模式都要按"会被大规模使用"来评审；没想清楚的就标为实验性，或者先不写进规范。

### 3. 传输层保持无状态，状态要显式化

MCP 用约 20 个月才走到"协议层零状态，状态按类型显式管理"（见第九节的状态拆分表）。

> **做法**：设计之初就把状态分类——会话、调用中途、长任务、业务——每一类给出显式载体（随请求携带、任务 ID、显式句柄），不要藏在连接或内存里。

### 4. 请求和它的结果必须在同一个往返里

HTTP+SSE 最根本的错误就是把二者拆开。

> **做法**：任何"请求从 A 进、结果从 B 出"的设计，都要立即问一句：多实例部署下 B 怎么知道 A 收到了什么？

### 5. 需要中途补充信息时，用"返回 + 重发"，不要"挂起等待"

MRTR 的思路适用于所有 API：需要确认、补参、二次验证时，返回 `input_required` 类的状态，让调用方带上答案重发。

> **前提**：询问之前不能产生副作用，操作必须幂等。

### 6. 安全和认证要一开始就是一等公民

后补认证导致 OAuth 方案反复修改，工具投毒至今没有协议级防御。

> **做法**：凡是跨越信任边界（跨进程、跨网络、跨组织、把外部文本喂给模型）的地方，首版就要定义认证、授权和不可信输入的处理方式。

### 7. 限制选项数量，避免兼容组合爆炸

MCP 拒绝把 WebSocket 加为官方传输的理由之一，就是避免客户端 × 服务端 × 传输方式的组合爆炸。

> **做法**：标准里每多一个可选项，兼容测试矩阵就翻倍。宁可少而精。

### 8. 先定废弃政策，再做破坏性变更

batching 加入不到 3 个月就移除、Sampling 约 20 个月后被废弃，对生态伤害很大；2026-07 才补上"至少 12 个月废弃期"。

> **做法**：协议或 API 一旦有外部使用者，就要先公布版本与废弃政策（多长过渡期、如何协商版本）。

### 9. 评审要覆盖设计涉及的每一个专业领域

MCP 设计者擅长协议与工具，远程部分却没有经过分布式 Web 专家评审；修正时靠的是 Shopify、Cloudflare、Vercel 等团队。

> **做法**：设计文档按领域拆分，每个领域至少找一位该领域的内行评审（网络、安全、存储、前端……）。**意识到自己在某个领域是外行，本身就是专业能力。**

### 10. 上下文 / 资源预算要纳入设计

MCP 没考虑"挂几十个服务端"时的 token 成本，导致后来出现 Code Mode、按需加载、"用 CLI 替代 MCP"等补救。

> **做法**：设计面向 LLM 的接口时，把"占用多少上下文"当作和延迟、吞吐同等重要的指标。

### 11. "Worse is Better"可以用，但要清楚自己在借什么债

MCP 靠"十分钟写一个服务端"的低门槛赢了生态，代价是近 2 年的改版、迁移成本和安全事件，由生态共同承担。

> **做法**：主动选择简化时，写下"欠了哪些债、计划什么时候还"，并在规范里把未完成的部分明确标为不稳定。

### 12. 快速承认错误比坚持错误设计更重要

HTTP+SSE 发布 4 个月即被设计者本人推翻重做，并公开写明旧方案的问题。这是 MCP 最终能成为事实标准的重要原因之一。

### 协议 / API 设计自查清单

设计任何对外协议、SDK、平台 API（包括内部平台工具、MCP 服务端）时逐项检查：

- [ ] 借鉴的方案，其前提条件是否在我的场景中成立？
- [ ] 规范中的每种模式是否都按大规模使用评审过？不成熟的是否标为实验性？
- [ ] 协议层是否无状态？各类状态是否都有显式载体？
- [ ] 请求与其结果是否在同一个往返中？多实例部署是否需要粘性会话？
- [ ] 需要中途补充信息时，是否采用"返回 + 重发"？操作是否幂等、询问前是否无副作用？
- [ ] 跨信任边界的地方是否首版就有认证、授权与不可信输入处理？
- [ ] 可选项是否已最小化？兼容矩阵是否可控？
- [ ] 是否有版本协商与废弃政策？
- [ ] 是否请每个涉及领域的内行评审过？
- [ ] 若面向 LLM：上下文占用是否可控？工具描述是否精简、可按需加载？
- [ ] 主动做的简化是否记录了技术债和偿还计划？

### 使用 MCP 的实践建议

- 只接可信来源的服务端，并锁定版本，防止 Rug Pull。
- 不要一次挂太多服务端；工具要精简、描述写好；优先按需加载或 Code Mode。
- 高频、确定性强的操作，CLI + Skill 往往比 MCP 更省 token。
- 企业场景前置 MCP 网关，统一鉴权、审计、限流。
- 有写操作的工具开启人工确认。
- 自己写 MCP 服务端时：校验路径、禁止 SSRF、不在返回值中带凭据——2026 年的大量 CVE 都是传统 Web 漏洞。

---

## 术语表

| 术语 | 解释 |
|---|---|
| MCP（Model Context Protocol） | Agent / LLM 应用连接外部工具与数据的协议，由 Anthropic 发起，现属 AAIF |
| A2A（Agent2Agent） | Agent 之间发现、委派、协作的协议，由 Google 发起，现属 AAIF |
| AAIF（Agentic AI Foundation） | Linux Foundation 旗下基金会，托管 MCP、A2A、AGENTS.md、goose、Agent Skills 等 |
| LSP（Language Server Protocol） | 微软提出的编辑器与语言服务之间的协议，MCP 的设计原型 |
| M×N 问题 | M 个客户端与 N 个提供方两两对接需要 M×N 套集成；公共协议把它降为 M+N |
| JSON-RPC 2.0 | 基于 JSON 的远程过程调用格式，LSP 与 MCP 均使用 |
| stdio 传输 | 客户端以子进程启动服务端，通过标准输入输出、按换行分隔传递 JSON-RPC 消息 |
| SSE（Server-Sent Events） | Web 标准的服务端推送机制，`text/event-stream` |
| Last-Event-ID | SSE 断线重连时携带的请求头，用于从断点续传 |
| WebSocket | 通过 HTTP 升级建立的全双工通信协议 |
| HTTP+SSE 传输 | MCP 2024-11 的远程传输，两个端点、全局长连接，已废弃 |
| Streamable HTTP | MCP 2025-03 起的远程传输，单端点 POST，按需流式响应 |
| MRTR（Multi Round-Trip Requests） | 无状态前提下实现"调用中途需要补充输入"的机制 |
| Elicitation | 服务端在工具执行中向用户追问确认或参数 |
| Sampling | 服务端借用客户端的大模型生成内容（已废弃） |
| Roots | 客户端告知服务端可访问的目录范围（已废弃） |
| Tasks | MCP 长任务扩展，任务 ID + 轮询/订阅 |
| DCR / CIMD | OAuth 动态客户端注册 / 客户端 ID 元数据文档（后者取代前者） |
| EMA（Enterprise Managed Authorization） | MCP 的企业托管授权扩展，对接企业身份体系 |
| RFC 8707 / RFC 9207 | OAuth 资源指示符（限定令牌用于哪个资源）/ 授权响应中的签发方标识（防授权服务器混淆） |
| SEP | Specification Enhancement Proposal，MCP 规范改进提案流程 |
| MCP Registry | 官方的公开 MCP 服务端目录与 API，支持公有 / 私有子 Registry |
| 粘性会话（Sticky Session） | 负载均衡把同一客户端固定路由到同一实例 |
| 幂等（Idempotent） | 同一操作执行多次与执行一次效果相同 |
| 工具投毒（Tool Poisoning） | 在工具描述或返回中藏恶意指令诱导模型执行 |
| Rug Pull | 先以正常面貌获得信任，再替换为恶意行为 |
| Code Mode | 让模型写代码调用工具以减少上下文消耗，由 Cloudflare 提出 |
| Agent Skills | Anthropic 提出的技能标准：一个含 SKILL.md 的文件夹，教 Agent 完成特定任务，2026-10 捐给 AAIF |
| Worse is Better | 先发布简单够用的方案往往比追求完美更易成功的设计理念 |
| Function Calling | 大模型按 JSON Schema 输出函数名与参数、由应用执行的能力，OpenAI 于 2023-06 推出 |
| ChatGPT Plugins | OpenAI 2023-03 推出的插件机制，基于 OpenAPI 描述外部服务，2024 年下线 |
| goose | Block 开源的本地优先 Agent 框架，MCP 参考实现之一，AAIF 创始项目 |
| AGENTS.md | OpenAI 提出的仓库级 Agent 指令文件约定，AAIF 创始项目 |
| AP2（Agent Payments Protocol） | Google 联合 PayPal 等推出的、基于 A2A 的 Agent 支付授权协议 |
| ACP（两个含义） | ① IBM 的 Agent Communication Protocol，已并入 A2A；② OpenAI 的 Agentic Commerce Protocol，用于 Agent 电商支付 |
| AGNTCY | Cisco 发起并贡献给 Linux Foundation 的 Agent 协作基础设施项目 |
| TSC（Technical Steering Committee） | 技术指导委员会，A2A 的技术决策机构 |
| Lead / Core Maintainer | MCP 治理中的最高决策者（类似 BDFL）/ 核心维护者 |
| 指导委员会（Steering Committee） | MCP 早期由多家公司代表组成的方向决策组；现由全体维护者组成 Steering Group |

---

## 参考文献

**规范与官方博客**

1. MCP 规范 2024-11-05 Transports：<https://modelcontextprotocol.io/specification/2024-11-05/basic/transports>
2. MCP 规范 2025-03-26 变更记录（Streamable HTTP、OAuth 2.1、batching 加入）：<https://modelcontextprotocol.io/specification/2025-03-26/changelog>
3. MCP 规范 2025-03-26 Transports：<https://modelcontextprotocol.io/specification/2025-03-26/basic/transports>
4. MCP 规范 2025-06-18 变更记录（移除 batching、RFC 8707、elicitation）：<https://modelcontextprotocol.io/specification/2025-06-18/changelog>
5. MCP 规范 2025-11-25 变更记录（实验性 Tasks、CIMD、治理结构）：<https://modelcontextprotocol.io/specification/2025-11-25/changelog>
6. MCP 规范概述（提及受 LSP 启发）：<https://modelcontextprotocol.io/specification/2025-11-25>
7. MCP 2026-07-28 规范发布博客：<https://blog.modelcontextprotocol.io/posts/2026-07-28/>
8. MCP 规范 2026-07-28 Transports（stdio 与 Streamable HTTP 两种标准绑定、向后兼容）：<https://modelcontextprotocol.io/specification/2026-07-28/basic/transports>
9. MCP PR #206：[RFC] Replace HTTP+SSE with new "Streamable HTTP" transport：<https://github.com/modelcontextprotocol/modelcontextprotocol/pull/206>
10. MCP Blog：Building to Last: A New Governance Model for MCP（2025-07-31）：<https://blog.modelcontextprotocol.io/posts/2025-07-31-governance-for-mcp/>
11. MCP Blog：Introducing the MCP Registry（2025-09-08）：<https://blog.modelcontextprotocol.io/posts/2025-09-08-mcp-registry-preview/>
12. MCP Blog：MCP joins the Agentic AI Foundation（2025-12-09）：<https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation/>
13. MCP Blog：Expanding the MCP Maintainer Team（2026-04-08）：<https://blog.modelcontextprotocol.io/posts/2026-04-08-maintainer-update/>
14. MCP Governance and Stewardship（现任维护者名单）：<https://modelcontextprotocol.io/community/governance>
15. Language Server Protocol 规范：<https://microsoft.github.io/language-server-protocol/>

**起源与当事人访谈**

16. Anthropic：Introducing the Model Context Protocol：<https://www.anthropic.com/news/model-context-protocol>
17. Latent Space：The Creators of Model Context Protocol（2025-04-03，David Soria Parra、Justin Spahr-Summers）：<https://www.latent.space/p/mcp>
18. Software Engineering Daily：Anthropic MCP with David Soria Parra（文字稿）：<https://softwareengineeringdaily.com/wp-content/uploads/2025/04/SED1836-Anthropic.txt>
19. a16z：MCP Co-Creator on the Next Wave of LLM Innovation：<https://a16z.com/podcast/mcp-co-creator-on-the-next-wave-of-llm-innovation/>
20. AI Engineer：David Soria Parra 简介：<https://ai.engineer/speakers/david-soria-parra>
21. OpenUK：David Soria Parra 谈 MCP 起源：<https://openuk.uk/thought-leadership/davidsoriaparra-aiopennessllama2/>

**行业采纳、基金会与 A2A**

22. TechCrunch：OpenAI adopts rival Anthropic's standard（2025-03-26）：<https://techcrunch.com/2025/03/26/openai-adopts-rival-anthropics-standard-for-connecting-ai-models-to-data/>
23. TechCrunch：Google to embrace Anthropic's standard（2025-04-09）：<https://techcrunch.com/2025/04/09/google-says-itll-embrace-anthropics-standard-for-connecting-ai-models-to-data/>
24. Google Developers Blog：Announcing the Agent2Agent Protocol (A2A)（2025-04-09，作者 Rao Surapaneni 等）：<https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/>
25. Hidekazu Konishi：Tool Use and Agent Protocol History and Timeline（各厂商采纳日期汇总）：<https://hidekazu-konishi.com/entry/tool_use_and_agent_protocol_history_and_timeline.html>
26. Wikipedia：Model Context Protocol：<https://en.wikipedia.org/wiki/Model_Context_Protocol>
27. Linux Foundation：Formation of the Agentic AI Foundation（2025-12-09）：<https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation>
28. Anthropic：Donating MCP to the Agentic AI Foundation：<https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation>
29. Block：Block, Anthropic, and OpenAI Launch the Agentic AI Foundation：<https://block.xyz/inside/block-anthropic-and-openai-launch-the-agentic-ai-foundation>
30. Anthropic：Donating Agent Skills to the Agentic AI Foundation（2026-10-22）：<https://www.anthropic.com/news/donating-agent-skills-to-the-agentic-ai-foundation>
31. A2A v1.0 发布公告：<https://a2a-protocol.org/latest/blog/2026/03/12/a2a-protocol-ships-v10-production-ready-standard-for-agent-to-agent-communication/>
32. A2A GitHub Releases：<https://github.com/a2aproject/A2A/releases>
33. Linux Foundation：A2A 一周年公告：<https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year>
34. AAIF：A2A joins AAIF（2026-08-17）：<https://aaif.io/blog/a2a-joins-aaif>
35. A2A Blog：A New Chapter for A2A: Joining the Agentic AI Foundation（2026-08-27）：<https://a2a-protocol.org/latest/blog/2026/08/27/a-new-chapter-for-a2a-joining-the-agentic-ai-foundation/>
36. Credal：What happened to A2A Protocol?：<https://credal.ai/blog/what-happened-to-a2a-protocol>

**上下文成本与安全**

37. Cloudflare：Code Mode: the better way to use MCP（2025-09-26）：<https://blog.cloudflare.com/code-mode/>
38. Anthropic Engineering：Code execution with MCP（2025-11-04）：<https://www.anthropic.com/engineering/code-execution-with-mcp>
39. Anthropic Engineering：Introducing advanced tool use（2025-11-24）：<https://www.anthropic.com/engineering/advanced-tool-use>
40. Roadie：Why Your MCP Server Might Be Eating Your Context Window：<https://roadie.io/blog/mcp-server-context-window/>
41. Stacklok：MCP vs CLI Tools: Why Security Changes the Answer：<https://stacklok.com/blog/mcp-vs-cli-tools-why-security-changes-the-answer/>
42. Cloud Security Alliance：MCP Security Crisis：<https://labs.cloudsecurityalliance.org/research/csa-research-note-mcp-security-crisis-20260504-csa-styled/>
43. Invariant Labs：Tool Poisoning Attacks：<https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks>
44. AuthZed：MCP 安全事件时间线：<https://authzed.com/blog/timeline-mcp-breaches>
45. Bonfy：MCP Security Statistics 2026：<https://blog.bonfy.ai/mcp-security-statistics-2026-what-the-vulnerability-data-reveals-about-ai-agent-risk>

**Web 基础**

46. WHATWG HTML 标准：Server-sent events：<https://html.spec.whatwg.org/multipage/server-sent-events.html>
47. MDN：Using server-sent events：<https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events>
48. Richard Gabriel：The Rise of "Worse is Better"：<https://www.dreamsongs.com/RiseOfWorseIsBetter.html>
