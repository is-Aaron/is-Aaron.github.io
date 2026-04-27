---
title: "Nginx Header 命名：横杠与下划线的踩坑记录"
date: 2026-04-27 00:00:00 +0800
categories: [Web, Nginx]
tags: [HTTP, Header, Nginx, 反向代理, Web开发]
---

这个坑我踩过不止一次：明明请求里带了自定义 header，到了 Nginx 或后端却取不到。最后发现问题不在业务代码，而在 **HTTP header 名和 Nginx 变量名不是同一个命名世界**。

最短结论是：

```txt
HTTP header 名：工程实践中用横杠 -
Nginx 变量名：用下划线 _
```

也就是说，请求里应该发：

```http
X-Request-ID: abc123
X-User-ID: 10001
X-Trace-ID: trace-xxx
```

而不是：

```http
X_Request_ID: abc123
X_User_ID: 10001
X_Trace_ID: trace-xxx
```

但在 Nginx 配置里读取这些 header 时，又要写成：

```nginx
$http_x_request_id
$http_x_user_id
$http_x_trace_id
```

这就是容易混淆的地方：**HTTP 里用横杠，Nginx 变量里用下划线。**

## HTTP Header 本身推荐用横杠

HTTP header 的真实名字是发在请求或响应里的字段名，例如：

```http
Content-Type: application/json
Authorization: Bearer xxx
User-Agent: curl/8.0
X-Request-ID: abc123
X-Forwarded-For: 203.0.113.10
```

这些名字在工程实践中基本都用横杠 `-` 分隔单词。标准 header 是这样，自定义 header 也应该沿用这个习惯：

```http
X-Client-Version: 1.2.3
X-Tenant-ID: tenant-a
X-Device-ID: device-001
```

严格说，从 HTTP 语法角度看，下划线 `_` 并不是绝对非法字符。HTTP field name 的语法是 `token`，而 `token` 允许 `_`。但 RFC 9110 在讨论新字段命名时也明确提醒：为了互操作性，新字段名应该限制在字母、数字、`-` 和 `.` 这类字符里，并且下划线 `_` 在经过非 HTTP 网关接口时可能有问题。

所以这不是“语法上完全不能写”的问题，而是“工程链路里很容易出事”的问题。很多代理、网关和服务器会把带下划线的 header 当成特殊情况处理，甚至直接忽略。Nginx 就是最典型的例子。

另外，上面的示例沿用了常见的 `X-...` 写法，因为很多现有系统里都能看到 `X-Request-ID`、`X-Forwarded-For` 这类名字。但如果是全新设计一个只属于自己业务的 header，现代规范并不鼓励继续发明新的 `X-` 前缀字段。更好的做法是使用清晰的业务或产品前缀，例如：

```http
Acme-User-ID: 10001
Acme-Client-Version: 1.2.3
Acme-Device-ID: device-001
```

所以在真实项目里，不要依赖下划线 header：

```http
X_Client_Version: 1.2.3
X_Tenant_ID: tenant-a
X_Device_ID: device-001
```

这类写法可能在某个本地环境里能跑，一上 Nginx、负载均衡、API 网关或服务网格，就开始出现“明明传了但是后端收不到”的问题。

另外，HTTP header 名本身是大小写不敏感的。也就是说，下面这些名字在语义上指的是同一个 header：

```http
X-Request-ID
x-request-id
X-REQUEST-ID
```

实际工程里也不要依赖大小写。HTTP/2 构造消息时要求 field name 转成小写；HTTP/3 编码前也要求 field name 转成小写，出现大写 field name 的消息会被视为 malformed。所以服务端代码应该做大小写不敏感匹配，而不是只认某一种大小写写法。

## Nginx 读取 Header 时用下划线变量

Nginx 通过 `$http_` 前缀读取客户端请求里的 header。转换规则是：

```txt
header 名转小写
横杠 - 转下划线 _
前面加 $http_
```

例如：

```txt
HTTP Header        Nginx 变量
X-Request-ID   ->  $http_x_request_id
X-User-ID      ->  $http_x_user_id
X-Trace-ID     ->  $http_x_trace_id
Content-Type   ->  $http_content_type
Authorization  ->  $http_authorization
User-Agent     ->  $http_user_agent
```

注意，这里的下划线只是 Nginx 变量命名方式，不代表 HTTP header 本身也应该写下划线。

还有一个边界要记住：Nginx 对少数字段也提供了专门变量。例如官方文档单独列出了：

```nginx
$content_length
$content_type
```

它们分别对应请求里的 `Content-Length` 和 `Content-Type`。尤其是 `Content-Length`，排查时优先用 `$content_length`，不要想当然写成 `$http_content_length`。

同一套转换规则也会出现在其他 Nginx header 变量里：

```txt
客户端请求 Header：$http_<name>
发给客户端的响应 Header：$sent_http_<name>
上游服务返回的 Header：$upstream_http_<name>
```

例如：

```txt
X-Request-ID        ->  $http_x_request_id
Content-Type        ->  $sent_http_content_type
X-Upstream-Token    ->  $upstream_http_x_upstream_token
```

## `proxy_set_header` 里最容易看反

最容易踩坑的是这种配置：

```nginx
location / {
    proxy_set_header X-Request-ID $http_x_request_id;
    proxy_set_header X-User-ID $http_x_user_id;
    proxy_set_header X-Trace-ID $http_x_trace_id;

    proxy_pass http://backend;
}
```

这里左边和右边的语义完全不同。

左边是真正要转发给后端的 HTTP header 名：

```nginx
X-Request-ID
X-User-ID
X-Trace-ID
```

所以左边用横杠。

右边是 Nginx 从客户端请求中读取到的变量：

```nginx
$http_x_request_id
$http_x_user_id
$http_x_trace_id
```

所以右边用下划线。

可以用一句话记住：

```txt
proxy_set_header 左边是 HTTP header 世界，用 -
proxy_set_header 右边是 Nginx 变量世界，用 _
```

如果把左边也写成下划线：

```nginx
proxy_set_header X_User_ID $http_x_user_id;
```

后端收到的就会是 `X_User_ID`，这已经不是推荐的 HTTP header 命名了。更糟的是，如果中间还有一层 Nginx 或网关，它可能会继续把这个 header 丢掉。

还有一个小细节：如果 `proxy_set_header` 的值是空字符串，Nginx 不会把这个 header 传给上游。所以如果客户端没有传 `X-Request-ID`，下面这行不会凭空给后端生成一个请求 ID：

```nginx
proxy_set_header X-Request-ID $http_x_request_id;
```

如果你希望统一由 Nginx 生成请求 ID，可以显式使用 Nginx 自己的 `$request_id`：

```nginx
proxy_set_header X-Request-ID $request_id;
```

注意，这会覆盖客户端原本传来的 `X-Request-ID`。如果你想“客户端传了就沿用，没传才由 Nginx 兜底生成”，可以用 `map` 先做一个中间变量。`map` 要放在 `http` 上下文里：

```nginx
map $http_x_request_id $request_id_for_upstream {
    default $http_x_request_id;
    ""      $request_id;
}

server {
    location / {
        proxy_set_header X-Request-ID $request_id_for_upstream;
        proxy_pass http://backend;
    }
}
```

## Nginx 默认可能忽略下划线 Header

假设客户端真的发了这个请求：

```http
GET /api/profile HTTP/1.1
Host: example.com
X_User_ID: 10001
```

Nginx 默认配置下，带下划线的 header 会被标记成 invalid header，然后通常会被忽略。相关配置主要有两个：

```nginx
underscores_in_headers off;
ignore_invalid_headers on;
```

它们的含义可以简单理解为：

- `underscores_in_headers off`：默认关闭。请求 header 名里出现下划线时，Nginx 会把这个 header 标记为 invalid。
- `ignore_invalid_headers on`：默认开启。遇到 invalid header 时忽略它。

因此，当客户端发的是：

```http
X_User_ID: 10001
```

你在 Nginx 里读：

```nginx
$http_x_user_id
```

可能就是空的。

如果确实要兼容历史客户端发来的下划线 header，可以显式开启：

```nginx
underscores_in_headers on;
```

这个配置的上下文是 `http` 或 `server`。如果有多个虚拟主机，尤其要注意默认 server 的配置影响。通常它只是兼容方案，不是推荐方案。更好的做法是统一把 header 改成横杠：

```http
X-User-ID: 10001
```

然后在 Nginx 中读取：

```nginx
$http_x_user_id
```

## 为什么不要混用横杠和下划线

还有一个容易被忽略的问题：Nginx 变量会把横杠转换成下划线。

这意味着下面两个 header 在变量层面很容易变得混乱：

```http
X-User-ID: 10001
X_User_ID: 10002
```

它们看起来是两个不同的 HTTP header 名，但转换到 Nginx 变量时，都可能对应到：

```nginx
$http_x_user_id
```

这会带来歧义，也可能带来安全风险。比如认证、用户 ID、租户 ID、请求来源这类 header，如果命名不统一，就可能出现“上游看的是一个值，下游看的是另一个值”的问题。

这不是纯理论问题。RFC 9110 在安全考虑里也提到过类似风险：如果某些接口把 `Transfer_Encoding` 和 `Transfer-Encoding` 映射到同一个变量名，处理不当就可能引出请求走私类问题。Nginx 的 `$http_` 变量同样会把横杠转换成下划线，所以不要让横杠版和下划线版 header 同时存在于你的协议约定里。

所以团队内应该统一约定：

```txt
自定义 HTTP header 尽量限制在字母、数字、横杠，必要时加点号。
字段名开头尽量使用字母。
多词分隔统一使用横杠。
不要设计或依赖带下划线的 HTTP header。
```

## 排查时怎么确认

遇到“后端收不到 header”时，可以按这个顺序排查。

第一步，确认客户端实际发出的 header 名是不是横杠：

```bash
curl -v \
  -H 'X-User-ID: 10001' \
  -H 'X-Request-ID: req-001' \
  http://example.com/api/profile
```

重点看请求里是不是：

```http
X-User-ID: 10001
X-Request-ID: req-001
```

第二步，在 Nginx 日志里临时打印变量。注意 `log_format` 只能放在 `http` 上下文里，`access_log` 可以放在 `http`、`server` 或 `location` 上下文里：

```nginx
http {
    log_format header_debug
        'request_id=$http_x_request_id '
        'user_id=$http_x_user_id '
        'uri=$request_uri';

    server {
        access_log /var/log/nginx/header_debug.log header_debug;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

如果这里为空，说明 Nginx 没读到请求 header。常见原因就是客户端发成了下划线，或者上游链路已经把 header 丢掉。

第三步，检查转发给后端时的名字：

```nginx
proxy_set_header X-User-ID $http_x_user_id;
proxy_set_header X-Request-ID $http_x_request_id;
```

左边应该是横杠 header 名，右边应该是下划线变量名。

## 最后记法

以后看到 Nginx header 配置，可以直接按这三行检查：

```txt
HTTP 请求/响应里的 header 名：X-User-ID
Nginx 里读取这个 header：$http_x_user_id
Nginx 转发给后端时的 header 名：X-User-ID
```

完整配置示例：

```nginx
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_set_header X-Request-ID $http_x_request_id;
    proxy_set_header X-User-ID $http_x_user_id;
    proxy_set_header X-Trace-ID $http_x_trace_id;

    proxy_pass http://backend;
}
```

一句话总结：

```txt
HTTP header 名里分隔单词用横杠，避免下划线。
Nginx 内部变量用下划线。
不要把 Nginx 变量名反推成 HTTP header 名。
```

## 参考资料

- [RFC 9110: HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)，Field Names、Tokens、Considerations for New Field Names、Security Considerations。
- [RFC 9113: HTTP/2](https://www.ietf.org/rfc/rfc9113.html)，HTTP Fields。
- [RFC 9114: HTTP/3](https://www.ietf.org/rfc/rfc9114.html)，HTTP Fields。
- [Nginx ngx_http_core_module](https://nginx.org/en/docs/http/ngx_http_core_module.html)，`$http_`、`$sent_http_`、`underscores_in_headers`、`ignore_invalid_headers`。
- [Nginx ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)，`$upstream_http_`。
- [Nginx ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header)，`proxy_set_header`。
- [Nginx ngx_http_map_module](https://nginx.org/en/docs/http/ngx_http_map_module.html)，`map`。
- [Nginx ngx_http_log_module](https://nginx.org/en/docs/http/ngx_http_log_module.html)，`log_format`、`access_log`。
