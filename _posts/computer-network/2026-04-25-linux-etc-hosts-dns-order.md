---
title: "Linux 域名解析：/etc/hosts 与 DNS 的先后顺序"
date: 2026-04-25 00:00:00 +0800
categories: [计算机网络, Linux]
tags: [Linux, DNS, hosts, 域名解析, nsswitch]
---

在 Linux 系统里，当我们访问一个域名时，例如：

```bash
curl https://example.com
ssh myserver
ping localhost
```

程序真正需要的并不是 `example.com` 或 `myserver` 这样的名字，而是一个可以发送网络包的 IP 地址。把“名字”转换成“IP 地址”的过程，就叫**名称解析**或**域名解析**。

很多人会把这个过程直接等同于 DNS，但在 Linux 上，DNS 只是其中一种解析来源。系统通常会先经过本机的名称服务规则，再决定是查本地文件、查 DNS，还是查其他名称服务。

## `/etc/hosts` 是什么

`/etc/hosts` 是一个本地静态映射文件，用来手工指定“主机名或域名 -> IP 地址”的关系。它不依赖 DNS 服务器，写在这里的记录由本机自己使用。

一个典型的 `/etc/hosts` 片段如下：

```txt
127.0.0.1       localhost
192.168.1.10    myserver
10.0.0.5        db.internal
```

含义分别是：

- 访问 `localhost` 时解析到 `127.0.0.1`。
- 访问 `myserver` 时解析到 `192.168.1.10`。
- 访问 `db.internal` 时解析到 `10.0.0.5`。

它常见的用途有几类：

1. **本机基础解析**：例如 `localhost`、本机 hostname。
2. **内网机器起别名**：把难记的 IP 写成短名字，例如 `ssh myserver`。
3. **临时测试域名指向**：网站迁移、灰度环境、测试新服务器时，让某个域名先指向指定 IP。
4. **屏蔽特定域名**：把某些域名指向 `127.0.0.1` 或 `0.0.0.0`，让本机无法正常访问它们。

需要注意的是，`/etc/hosts` 是精确名称映射，不支持通配符。也就是说，不能写一条 `*.example.com` 来匹配所有子域名；不同名字需要分别写。

## 它和 DNS 谁先查

在很多 Linux 发行版上，解析顺序不是由 `/etc/hosts` 自己决定的，而是由 `/etc/nsswitch.conf` 中的 `hosts:` 配置决定。

可以用下面的命令查看：

```bash
grep '^hosts:' /etc/nsswitch.conf
```

常见的简化形式如下：

```txt
hosts: files dns
```

这表示：

1. 先查 `files`，也就是 `/etc/hosts`。
2. 再查 `dns`，也就是 DNS 服务器。

所以，如果 `/etc/hosts` 里写了：

```txt
1.2.3.4 example.com
```

并且 `nsswitch.conf` 里是 `hosts: files dns`，那么程序解析 `example.com` 时通常会先得到 `1.2.3.4`，不会再继续向 DNS 服务器查询这个名字。

如果配置反过来：

```txt
hosts: dns files
```

那就是先查 DNS，再查 `/etc/hosts`。这种配置不如 `files dns` 常见，但它说明了一个关键点：**“hosts 一定优先于 DNS”不是协议规定，而是系统解析策略的结果。**

## 常见的复杂配置

现代 Linux 桌面或服务器上，还可能看到类似这样的配置：

```txt
hosts: files mdns4_minimal [NOTFOUND=return] dns
```

这行的意思大致是：

- `files`：先查 `/etc/hosts`。
- `mdns4_minimal`：再尝试本地网络中的 mDNS 解析，常见于 `.local` 名称。
- `[NOTFOUND=return]`：如果前一个模块明确返回“没找到”，就停止后续查询。
- `dns`：在前面的模块没有让查询提前返回时，最后再查普通 DNS。

中间模块和条件不同，实际行为也会不同。因此排查域名解析问题时，不要只看 `/etc/hosts`，还要看 `/etc/nsswitch.conf`。

## 如何验证解析结果

推荐优先用 `getent`，因为它会走系统的 NSS 解析链路：

```bash
getent hosts example.com
getent ahosts example.com
```

如果想看 DNS 服务器返回什么，可以用 `dig` 或 `nslookup`：

```bash
dig example.com
nslookup example.com
```

这里要特别区分：

- `getent hosts`：查询 NSS 的 `hosts` 数据库，会参考 `/etc/nsswitch.conf`。
- `getent ahosts`：通过 `getaddrinfo()` 查询，通常更接近 `curl`、`ssh` 这类现代程序实际拿到地址的方式。
- `dig` / `nslookup`：主要直接问 DNS，通常不会体现 `/etc/hosts` 的影响。

所以当你发现 `dig example.com` 返回的是一个 IP，而 `curl example.com` 实际连到另一个 IP 时，很可能就是 `/etc/hosts`、NSS 顺序、系统 DNS 缓存或应用自身缓存产生了差异。

## 修改时的注意事项

修改 `/etc/hosts` 通常需要 root 权限：

```bash
sudo vim /etc/hosts
```

写法一般是：

```txt
IP地址    规范主机名    可选别名
```

例如：

```txt
192.168.1.10    myserver    myserver.local
```

几个实践建议：

- 一行可以写一个 IP 和多个名字，但不要把无关域名混在一起。
- 删除临时测试记录时要及时清理，否则以后可能出现“为什么我访问的不是线上地址”的问题。
- 如果系统启用了 `systemd-resolved`、`nscd`、浏览器 DNS 缓存等机制，修改后可能需要清缓存或重启相关服务。
- 对生产环境排障时，应同时记录 `/etc/hosts`、`/etc/nsswitch.conf`、`resolv.conf` 或 `resolvectl status` 的信息。

## 小结

可以把 Linux 的名称解析理解成一条链路：

```txt
应用程序请求解析域名
  -> glibc / NSS 根据 /etc/nsswitch.conf 决定查询顺序
  -> files 查询 /etc/hosts
  -> dns 查询 DNS 服务器
  -> 返回 IP 给应用程序
```

常见的核心顺序是：

```txt
hosts: files dns
```

也就是先查 `/etc/hosts`，再查 DNS。但准确答案永远要以当前机器的 `/etc/nsswitch.conf` 为准。
