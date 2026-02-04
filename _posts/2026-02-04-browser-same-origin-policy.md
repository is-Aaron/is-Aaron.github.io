---
title: 浏览器同源策略：Web 安全的第一道防线
date: 2026-02-04 00:00:00 +0800
categories: [Web 安全, 浏览器]
tags: [同源策略, CORS, 浏览器安全, 跨域]
mermaid: true
---

> **核心观点**：同源策略是浏览器的**访问控制策略**——它限制一个源的脚本只能读取同源的资源，防止恶意网站窃取用户在其他网站上的敏感数据。

## 一、什么是"源"

在浏览器中，**源（Origin）** 由三部分组成：

| 组成部分 | 示例                  |
| -------- | --------------------- |
| 协议     | `https://`            |
| 域名     | `example.com`         |
| 端口     | `:443`（HTTPS 默认）  |

**判断规则**：三者完全相同才算同源。

```mermaid
graph TB
    subgraph Origin["https://example.com:443"]
        A["协议: https"]
        B["域名: example.com"]
        C["端口: 443"]
    end
    
    A & B & C --> D{"三者都相同？"}
    D -->|是| Same["✅ 同源"]
    D -->|否| Cross["❌ 跨源"]
```

### 同源判断示例

以 `https://example.com/page.html` 为基准：

| URL                                  | 是否同源 | 原因         |
| ------------------------------------ | -------- | ------------ |
| `https://example.com/other.html`     | ✅ 同源   | 路径不同不影响 |
| `http://example.com/page.html`       | ❌ 跨源   | 协议不同     |
| `https://api.example.com/page.html`  | ❌ 跨源   | 子域名也算不同域名 |
| `https://example.com:8080/page.html` | ❌ 跨源   | 端口不同     |

## 二、为什么需要同源策略

想象没有同源策略的世界：

```mermaid
sequenceDiagram
    participant User as 用户
    participant Evil as 恶意网站<br/>evil.com
    participant Bank as 银行网站<br/>bank.com
    
    User->>Bank: 1. 登录银行，获取 Cookie
    User->>Evil: 2. 访问恶意网站
    Evil->>Bank: 3. JS 发起请求<br/>自动携带 Cookie
    Bank-->>Evil: 4. 返回用户账户信息
    Evil->>Evil: 5. 窃取数据 💀
```

**没有同源策略**：恶意网站的 JS 可以：
- 读取你在银行网站的余额
- 获取你在邮箱的所有邮件
- 窃取任何网站的敏感数据

**有了同源策略**：浏览器会阻止 `evil.com` 的脚本读取 `bank.com` 的响应。

## 三、同源策略限制什么

同源策略主要限制**跨源读取**，而非所有跨源行为：

| 行为类型   | 是否允许 | 示例                          |
| ---------- | -------- | ----------------------------- |
| 跨源写入   | ✅ 允许   | 表单提交、链接跳转            |
| 跨源嵌入   | ✅ 允许   | `<script>`、`<img>`、`<iframe>` |
| 跨源读取   | ❌ 禁止   | AJAX 读取响应、iframe 内容读取 |

### 具体限制

```javascript
// ❌ 禁止：读取跨源 AJAX 响应
fetch('https://api.other.com/data')
  .then(res => res.json())  // 请求会发送，但响应被阻止读取

// ❌ 禁止：读取跨源 iframe 内容
const iframe = document.querySelector('iframe');
iframe.contentDocument;  // 抛出安全错误

// ❌ 禁止：读取跨源 canvas 图像数据
const canvas = document.createElement('canvas');
ctx.drawImage(crossOriginImg, 0, 0);
canvas.toDataURL();  // 污染的 canvas，抛出安全错误
```

## 四、如何合法跨源

### 1. CORS（跨源资源共享）

服务端通过 HTTP 头声明"允许谁来访问"：

```mermaid
sequenceDiagram
    participant Browser as 浏览器<br/>a.com
    participant Server as 服务器<br/>b.com
    
    Note over Browser,Server: 简单请求
    Browser->>Server: GET /api/data
    Server-->>Browser: 响应 + Access-Control-Allow-Origin: a.com
    Browser->>Browser: ✅ 允许读取响应
    
    Note over Browser,Server: 预检请求（复杂请求）
    Browser->>Server: OPTIONS /api/data
    Server-->>Browser: Access-Control-Allow-Origin: a.com<br/>Access-Control-Allow-Methods: POST
    Browser->>Server: POST /api/data
    Server-->>Browser: 响应数据
```

**关键响应头**：

```http
Access-Control-Allow-Origin: https://a.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Credentials: true
```

### 2. JSONP（仅限 GET，已过时）

利用 `<script>` 标签不受同源限制的特性：

```javascript
// 客户端
function handleData(data) {
  console.log(data);
}
// <script src="https://api.com/data?callback=handleData"></script>

// 服务端返回
handleData({"name": "Aaron"});
```

**缺点**：只支持 GET、存在 XSS 风险，已被 CORS 取代。

### 3. 代理服务器

通过同源的后端转发请求：

```mermaid
graph LR
    A["前端<br/>a.com"] -->|同源请求| B["后端<br/>a.com/api"]
    B -->|服务端请求| C["第三方 API<br/>b.com"]
```

服务端没有同源限制，可以自由请求任何 API。

## 五、常见误区

| 误区                        | 真相                                          |
| --------------------------- | --------------------------------------------- |
| 同源策略阻止请求发送        | ❌ 请求会发送，只是**阻止读取响应**            |
| `<img>` 跨源被阻止          | ❌ 嵌入资源允许，但 JS 无法读取图片像素数据    |
| CORS 是前端配置             | ❌ CORS 是**服务端**通过响应头控制的          |
| 同源策略能防止 CSRF         | ❌ 跨源写入（表单提交）是允许的，需额外防护   |

## 六、总结

同源策略的设计哲学：

| 原则       | 实现                                |
| ---------- | ----------------------------------- |
| 默认隔离   | 不同源的脚本不能互相读取数据        |
| 嵌入宽松   | 允许嵌入跨源资源（script、img 等） |
| 读取严格   | 禁止读取跨源响应内容                |
| 服务端授权 | 通过 CORS 让服务端决定谁能访问      |

**一句话总结**：同源策略体现了浏览器的"最小权限原则"——默认禁止跨源读取，除非目标服务器明确授权。
