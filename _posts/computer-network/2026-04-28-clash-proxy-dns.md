---
title: "Clash 代理模式与 DNS 机制：从系统代理到 TUN、fake-ip"
date: 2026-04-28 00:00:00 +0800
categories: [计算机网络, 代理]
tags: [Clash, DNS, TUN, 代理, fake-ip, redir-host]
---

很多人使用 Clash 时，只知道“打开代理就能访问”。但一旦遇到内网域名打不开、`ping` 域名不通、DNS 泄漏、关掉 Clash 后短暂断网，就很容易不知道问题出在哪里。

这些现象大多和三个问题有关：

1. 当前用的是**系统代理**还是 **TUN 模式**。
2. DNS 查询有没有被 Clash 接管。
3. DNS 模式用的是 **fake-ip** 还是 **redir-host**。

本文用尽量短的篇幅把这几件事串起来。

## 先说结论

如果只是想让浏览器、Cursor、VS Code 这类应用走代理，优先使用**系统代理**或应用自己的代理配置。它们副作用小，不改路由，也不接管 DNS。

如果想让尽可能多的应用流量都经过 Clash，才需要开启 **TUN 模式**。TUN 更强，但也更复杂，通常要配合 DNS 劫持使用。这里的“全局”仍然受系统路由、平台限制、VPN/安全软件、应用自带 DoH/DoT 等因素影响，不是绝对意义上的所有数据包。

DNS 模式的选择可以简单记成：

| 场景 | 建议 |
|---|---|
| 长期开 Clash，追求速度并降低 DNS 泄漏风险 | fake-ip |
| 经常开关 Clash，需要调试网络 | redir-host |
| 经常访问公司内网、NAS、路由器域名 | redir-host，或给 fake-ip 配好过滤规则 |
| 只让某个应用走代理 | 不开 TUN，只配应用代理 |

## 系统代理：靠应用自觉

系统代理的本质，是在操作系统里写入一条代理配置，例如告诉应用：

```txt
HTTP/HTTPS 请求请发给 127.0.0.1:7890
```

在 macOS 上，Clash Verge 开启系统代理后，效果类似：

```bash
networksetup -setwebproxy "Wi-Fi" 127.0.0.1 7890
networksetup -setsecurewebproxy "Wi-Fi" 127.0.0.1 7890
```

这类代理模式的关键是：**应用必须主动遵守系统代理设置**。

浏览器、Electron 应用通常会遵守；很多命令行工具、游戏、自己实现网络栈的软件不一定遵守。比如 `curl` 默认不会自动走 macOS 系统代理，通常需要设置环境变量或显式加代理参数。

系统代理的好处是轻量：不改路由表，不创建虚拟网卡，也不劫持 DNS。坏处是覆盖不完整。

## TUN 模式：在网络层接管流量

TUN 模式不再依赖应用是否配合。它会创建一张虚拟网卡；配合 `auto-route` 或手动路由规则时，可以让应用发出的网络包先进入 Clash。

可以把它理解成：

```txt
普通应用 -> 系统路由 -> TUN 虚拟网卡 -> Clash -> 代理或直连
```

所以 TUN 的覆盖范围比系统代理大得多。浏览器、命令行工具、游戏、UDP 流量，只要按系统路由进入 TUN，都有机会被接管。

但代价也很明显：TUN 会影响系统路由和 DNS。如果配置不当，问题通常也会出现在这两层。

## 为什么 TUN 模式需要 DNS 劫持

普通访问域名时，流程是：

```txt
应用访问 google.com
系统 DNS 把 google.com 解析成 IP
应用连接这个 IP
```

问题在于，TUN 层看到的是已经解析后的 IP 包。也就是说，Clash 可能只看到：

```txt
连接 142.250.80.46:443
```

它并不知道这个 IP 原本对应 `google.com`。但 Clash 的很多规则是基于域名写的：

```yaml
- DOMAIN-SUFFIX,google.com,PROXY
- DOMAIN-SUFFIX,baidu.com,DIRECT
```

如果 Clash 不知道域名，域名规则就无法准确生效。

DNS 劫持就是为了解决这个问题：应用查询 DNS 时，Clash 先截获这个查询，建立“域名和连接目标”的关联。之后应用真正发起连接时，Clash 就能利用这个关联匹配规则。

需要注意的是：

```txt
TUN 本身负责捕获流量
dns-hijack 负责把 DNS 查询交给 Clash 处理
dns.enable 负责启用 Clash 内部 DNS 模块
```

所以准确地说，不是 TUN 天生等于 DNS 劫持，而是 **TUN + dns-hijack** 才具备接管传统 DNS 查询的条件。

`dns-hijack` 主要处理传统 DNS 流量。以 mihomo 为例，不写协议时默认是 UDP，因此想同时覆盖 UDP/TCP 53 端口，配置里应同时写：

```yaml
dns-hijack:
  - "any:53"
  - "tcp://any:53"
```

浏览器或系统启用的 DoH/DoT、Android 私人 DNS，以及部分平台上的局域网 DNS 请求，可能不会被这条规则自动接管，需要单独关闭、改规则或按平台文档处理。

mihomo 还支持通过 sniffer 从 HTTP、TLS、QUIC 流量里嗅探域名，作为 DNS 映射之外的补充机制；为了保持主线清晰，本文只展开 DNS 这条路径。

## 为什么不能简单依赖系统 DNS

在理想网络里，系统 DNS 当然可以直接用。但在真实环境中，尤其是跨境访问时，直接依赖系统 DNS 容易有三个问题。

第一是 **DNS 污染**。某些域名可能被本地 DNS 返回错误 IP。即使后续流量走代理，连接目标已经错了，访问仍然会失败。

第二是 **DNS 泄漏**。页面内容走代理了，但 DNS 查询仍然发给本地运营商 DNS，访问过哪些域名就可能被本地网络看到。

第三是 **CDN 调度不准**。如果流量最终由远端代理服务器访问目标站点，却使用本地 DNS 解析出来的 IP，可能会拿到不适合代理出口位置的 CDN 节点。

因此常见做法是让 Clash 自己分流 DNS：

```yaml
dns:
  enable: true
  default-nameserver:
    - 223.5.5.5
  nameserver:
    - https://doh.pub/dns-query
    - https://dns.alidns.com/dns-query
  fallback:
    - https://dns.google/dns-query
    - https://1.1.1.1/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN
```

`default-nameserver` 用来解析 DoH/DoT 服务器自身的域名，避免“DNS 服务器的域名也需要先解析”的启动问题。`nameserver` 是默认解析服务器；配置 `fallback` 后，mihomo 会结合 `fallback-filter` 判断是否采用 fallback 结果。大致思路是：可信的国内结果可以直接用，可疑或命中过滤条件的结果改用 fallback，尽量降低污染和泄漏风险。

## fake-ip 和 redir-host 的区别

fake-ip 和 redir-host 都是 Clash DNS 模块的增强模式，目的都是让 Clash 在 TUN 模式下知道连接背后的域名。

### fake-ip

fake-ip 的做法是：应用查询域名时，Clash 不直接把真实 IP 返回给应用，而是返回一个保留网段里的假 IP，例如 `198.18.x.x`。

```txt
应用问：google.com 是什么 IP？
Clash 答：198.18.0.5
Clash 记录：198.18.0.5 -> google.com
```

之后应用连接 `198.18.0.5`，Clash 查表就知道它实际想访问的是 `google.com`。

fake-ip 的优点是快，并且能减少应用把真实域名解析绕出 Clash 的机会；但它不是“自动消灭所有 DNS 泄漏”的开关，实际隐私效果仍取决于上游 DNS、规则、代理出口和应用是否绕过 Clash。它的缺点是系统 DNS 缓存里可能留下假 IP，`ping` 这类工具也会看到假 IP，因此经常会出现“域名解析到了 198.18.x.x，但 ping 不通”的现象。

这不是目标网站挂了，而是 fake-ip 的正常表现。

### redir-host

redir-host 的做法更接近传统 DNS：Clash 真的去解析域名，把真实 IP 返回给应用，同时把这次解析结果用于后续规则判断。

```txt
应用问：google.com 是什么 IP？
Clash 解析后答：142.250.80.46
Clash 关联：google.com -> 142.250.80.46
```

redir-host 的优点是兼容性好，系统里看到的也是真实 IP，适合调试网络、经常开关 Clash 的场景。缺点是需要等待上游 DNS，隐私和速度通常不如 fake-ip。另一个限制是，真实 IP 可能被多个域名共享，尤其是 CDN 场景；这时 IP 和域名不一定能稳定一一对应，sniffer 可以作为补充。

两者可以这样对比：

| 对比项 | fake-ip | redir-host |
|---|---|---|
| 返回给应用的 IP | 假 IP | 真实 IP |
| 解析速度 | 快 | 相对慢 |
| `ping` 域名 | 常见为假 IP，不通 | 通常是正常 IP |
| 关闭 Clash 后 | 可能短暂受 DNS 缓存影响 | 影响较小 |
| 兼容性 | 少数应用可能有问题 | 更好 |
| DNS 泄漏风险 | 通常更低，但取决于完整配置 | 更依赖上游 DNS 和规则配置 |

## fake-ip 下内网域名为什么容易出问题

fake-ip 最大的坑，通常出现在内网域名上。

比如公司内网有一个域名：

```txt
gitlab.internal.example
```

这个域名只有公司 DNS 才认识，公共 DNS 不认识。如果 Clash 给它分配了 fake-ip，后续又不知道该用哪个内网 DNS 去解析真实地址，就会出现内网服务打不开的问题。

解决方法有两个。

第一个是把内网域名排除出 fake-ip：

```yaml
dns:
  enhanced-mode: fake-ip
  fake-ip-filter:
    - "*.local"
    - "*.lan"
    - "*.internal"
    - "*.corp"
    - "*.home.arpa"
```

第二个是按域名指定 DNS：

```yaml
dns:
  enhanced-mode: fake-ip
  nameserver:
    - https://doh.pub/dns-query
  nameserver-policy:
    "*.internal.example": "10.0.0.1"
    "*.corp.example": "10.0.0.1"
```

如果你经常在公司网络、VPN、家用 NAS、路由器管理域名之间切换，redir-host 往往更省心；如果坚持使用 fake-ip，就要把内网域名规则补全。

## TUN 会不会让 Clash 自己的流量绕回自己

开启 TUN 后，系统流量会进入虚拟网卡。那 Clash 自己发出的 DNS 查询和代理连接，会不会也被路由回 TUN，形成循环？

正常情况下不会。关键配置是：

```yaml
tun:
  auto-route: true
  auto-detect-interface: true
```

`auto-route` 负责自动设置让流量进入 TUN 的路由；`auto-detect-interface` 负责自动选择出站接口。它们配合起来可以避免 Clash 出站流量被错误送回 TUN。多网卡、VPN、公司安全客户端同时存在时，自动检测不一定总是选对接口，这时应考虑手动指定出口接口。

真正可能出问题的情况通常是多网卡、VPN、Docker、接口切换，或者关闭接口自动检测后又没有手动指定正确接口。

## 实用配置参考

只让某个应用走代理时，可以不开 TUN，不开系统代理，只在应用里配置：

```json
{
  "http.proxy": "http://127.0.0.1:7890"
}
```

大多数长期使用 Clash 的场景，可以使用 TUN + fake-ip：

```yaml
tun:
  enable: true
  stack: mixed
  auto-route: true
  auto-detect-interface: true
  dns-hijack:
    - "any:53"
    - "tcp://any:53"

dns:
  enable: true
  default-nameserver:
    - 223.5.5.5
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - "*.local"
    - "*.lan"
    - "*.internal"
    - "*.home.arpa"
  nameserver:
    - https://doh.pub/dns-query
    - https://dns.alidns.com/dns-query
  fallback:
    - https://dns.google/dns-query
    - https://1.1.1.1/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN
```

如果经常开关 Clash，或者经常排查网络问题，可以改用 redir-host：

```yaml
tun:
  enable: true
  stack: system
  auto-route: true
  auto-detect-interface: true
  dns-hijack:
    - "any:53"
    - "tcp://any:53"

dns:
  enable: true
  default-nameserver:
    - 223.5.5.5
  enhanced-mode: redir-host
  nameserver:
    - https://doh.pub/dns-query
    - https://dns.alidns.com/dns-query
  fallback:
    - https://dns.google/dns-query
    - https://1.1.1.1/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN
```

## 总结

Clash 的很多问题，本质上不是“代理没开好”，而是代理层级不同：

- 系统代理是应用层配置，轻量，但依赖应用自觉。
- TUN 是网络层接管，覆盖完整，但会牵涉路由和 DNS。
- DNS 劫持是 TUN 下识别域名规则的关键。
- fake-ip 更快，通常更不容易发生传统 DNS 泄漏，但对内网域名和调试工具不够友好。
- redir-host 更符合传统网络直觉，适合经常切换和排查问题。

最干净的选择往往是：能不用 TUN 就不用 TUN；确实需要全局接管时，再认真配置 DNS、fake-ip 过滤和接口检测。
