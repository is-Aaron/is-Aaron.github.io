---
layout: post
title: "Docker 网络模式演进史：从端口映射到跨主机 Overlay"
date: 2026-03-19
categories: systems
tags: [Docker, 网络, 容器化, libnetwork, overlay, bridge]
---

当我们在 `docker run` 时加上 `--network host` 或 `--network none`，背后其实是一段跨越十余年的网络抽象演进史。Docker 从最初「能跑就行」的简陋端口映射，一路演进出 bridge、host、none、overlay、macvlan 等多种模式，每一次新模式的诞生都对应着容器技术在实际场景中遇到的新问题。理解这段历史，不仅能解释为什么会有这么多网络选项，更能帮助我们在不同场景下做出正确选择。

## 🕰️ 史前时代：端口映射的天下（2013–2014）

Docker 于 2013 年 3 月开源发布时，网络功能极其朴素。当时的核心理念只有两件事：**让容器能访问外网**，以及**让宿主机能访问容器内的服务**。

### 默认 bridge：唯一的网络

早期 Docker 没有「网络模式」的概念，所有容器默认接入一个名为 `bridge` 的虚拟网桥（对应宿主机上的 `docker0`）。每个容器会获得一个独立的 Network Namespace，拿到一个虚拟 IP（通常是 `172.17.0.0/16` 网段），通过 NAT 访问外网。容器之间可以互相 `ping` 通，但**没有服务发现**，你必须自己记住每个容器的 IP。

要暴露服务给宿主机或外部访问，唯一的方式是 **`-p` 端口映射**：

```bash
docker run -p 8080:80 nginx
```

这条命令的本质是：在宿主机上监听 8080 端口，将流量转发到容器的 80 端口。对开发者来说足够简单，但存在明显的局限：

*   **跨主机通信**：几乎不可能。不同宿主机上的容器处在不同的 `docker0` 网段，没有路由，无法直接通信。
*   **端口冲突**：同一宿主机上跑多个 Web 容器时，必须为每个容器分配不同的宿主机端口（8080、8081、8082…），编排困难。
*   **链接（linking）**：后来 Docker 引入了 `--link`，允许容器通过**名称**访问另一个容器，但这只是往 `/etc/hosts` 里写死 IP，容器重启后 IP 变化，链接就失效，且只能单向链接，体验糟糕。

这一时期，Docker 的网络设计哲学可以概括为：**先解决「能跑」的问题，复杂场景交给用户自己折腾**。

## 🏗️ 奠基时代：libnetwork 与可插拔驱动（2015 年）

2015 年，Docker 在网络上迎来转折。**libnetwork** 项目于年初诞生（2 月），目标是重新设计 Docker 的网络架构；同年 3 月，Docker 收购了专注于容器软件定义网络的初创公司 **SocketPlane**，并将其技术整合进 libnetwork。

### 从「硬编码」到「可插拔」

之前的网络逻辑是写死在 Docker Engine 里的，想要支持新的网络拓扑（比如跨主机、与现有 SDN 集成）几乎不可能。libnetwork 引入了 **Container Network Model (CNM)**，核心思想是：

*   **网络（Network）** 是一个抽象，应用开发者只需声明「这几个服务要在同一个网络里互通」。
*   **如何实现** 由 **驱动（Driver）** 决定。Docker 内置若干驱动，第三方（如 VMware NSX、Cisco ACI）可以开发自己的驱动，实现不同的底层网络拓扑。

这套设计体现了 Docker 的哲学：**Batteries included, but swappable**——开箱即用，但关键部件可以替换。

### Docker 1.9：新网络的正式发布

2015 年 11 月，Docker 1.9 正式发布新的网络子系统。用户第一次可以通过 `docker network create` 创建**自定义网络**，并通过 `--network` 将容器接入指定网络。同时，内置了多种网络驱动，对应我们今天熟悉的多种「网络模式」。

| 驱动        | 诞生背景                       | 核心能力                         |
| :---------- | :----------------------------- | :------------------------------- |
| **bridge**  | 延续早期设计，但支持用户自定义 | 单机内容器互通，可配置子网、网关 |
| **host**    | 早期就存在，1.9 纳入统一抽象   | 直接使用宿主机网络栈，零 NAT     |
| **none**    | 早期就存在，1.9 纳入统一抽象   | 完全无网络，仅有 loopback        |
| **overlay** | libnetwork 新引入              | 跨主机互通，配合 Swarm 集群      |

至此，Docker 有了清晰的「网络模式」概念：**模式 = 驱动类型**。选择不同的 `--network`，本质是选择不同的驱动来「实现」容器之间的连通性。

## 🌐 单机网络三剑客：bridge、host、none

在单机场景下，三种模式至今仍是最常用的选择，它们代表了三种不同的隔离与性能权衡。

### bridge：默认与平衡

bridge 是**默认模式**。当你 `docker run` 时不指定 `--network`，容器会接入默认的 bridge 网络。你也可以创建自定义 bridge 网络：

```bash
docker network create mynet
docker run --network mynet --name web nginx
docker run --network mynet --name app alpine
```

同一个自定义网络内的容器，可以通过 **容器名** 互相解析（Docker 内置了嵌入式的 DNS）。 bridge 模式下，每个容器有独立 IP，通过虚拟网桥与宿主机和外网通信，适合绝大多数「多容器协同」的单机应用。

### host：性能优先，放弃隔离

当使用 `--network host` 时，容器**不再拥有独立的 Network Namespace**，直接共享宿主机的一套网络栈。这意味着：

*   **无 NAT 开销**：数据包不需要经过端口映射，延迟更低，吞吐更高。
*   **端口直用**：容器内监听 80 端口，宿主机上就是 80 端口，无需 `-p` 映射。
*   **代价**：失去了网络隔离。多个容器不能同时监听同一端口；容器可以直接看到宿主机的所有网络接口和连接。

典型场景：**高性能代理、监控采集、或需要绑定宿主机特定网卡的场景**（如某些负载均衡实现）。

### none：极致隔离

`--network none` 让容器只有一块 loopback 接口，无法访问外网，也无法被其他容器或宿主机直接访问。适用于**离线批量计算、安全沙箱**等对网络零依赖的场景。

## 🚀 跨主机时代：overlay 与 Swarm（2015–2016）

单机 bridge 无法解决「容器分布在不同物理机上」的问题。Docker 的答案是 **overlay 网络**，其底层基于 **VXLAN（Virtual Extensible LAN）**，在物理网络之上构建一个虚拟的二层网络。

### overlay 的诞生背景

2015 年，Docker 力推 **Docker Swarm** 作为原生编排方案（2016 年 Docker 1.12 起由内置的 **Swarm Mode** 取代）。Swarm 允许将多台机器组建成一个集群，容器可以被调度到任意节点上。但问题来了：**节点 A 上的 Web 容器如何访问节点 B 上的数据库容器？**

overlay 驱动应运而生。当你在 Swarm 集群中创建一个 overlay 网络时，所有加入该网络的容器，无论落在哪台物理机上，都处在同一个逻辑网段中。它们可以通过容器名互相解析，就像在同一台机器上一样。

```bash
docker swarm init   # overlay 依赖 Swarm，需先初始化
docker network create -d overlay my_overlay_net
# 在 Swarm 中部署服务时指定该网络
```

### overlay 的局限与生态

overlay 解决了「有」的问题，但在大规模、多租户、与现有 SDN 深度集成的场景下，企业往往会选择 **Kubernetes + CNI** 或第三方网络插件（如 Calico、Cilium）。这些方案通常比 Docker 原生的 overlay 更灵活。但 overlay 作为 Docker 自带的跨主机方案，在小型集群、快速 PoC 场景中仍有价值。

## 🏢 企业级扩展：macvlan 与 ipvlan（2016 年后）

随着容器从开发测试环境走向生产，一些企业遇到了新需求：**容器需要像物理机或虚拟机一样，直接出现在既有网络里**。比如：

*   迁移 legacy 应用：原来跑在 VM 上，有固定 IP，防火墙、监控都按 IP 做规则。容器化后希望「透明替代」，网络行为不变。
*   合规与审计：某些网络策略要求流量可追溯，需要容器有独立的 MAC 或明确 IP，而不是躲在 NAT 后面。

### macvlan：容器即「物理机」

**macvlan** 驱动允许为每个容器分配一个**独立的 MAC 地址**，从物理网卡上虚拟出多块「网卡」，每块绑定一个容器。从交换机或路由器的视角看，这些容器就像直接挂在局域网上的物理设备。

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my_macvlan
```

限制包括：**需要 Linux 内核 3.9 及以上**（官方建议 4.0+）；父接口需为物理网卡或其 VLAN 子接口（如 `eth0.10`）；某些云主机或虚拟化环境对 MAC 数量有限制。

### ipvlan：共享 MAC 的变体

**ipvlan** 与 macvlan 类似，但**多个容器共享同一块物理网卡的 MAC 地址**，仅 IP 不同。在 MAC 地址数量受限的环境（如某些公有云）中，ipvlan 是更稳妥的选择。ipvlan 支持 L2、L3 及 L3S 等模式，提供更细粒度的组网能力。

---

## 📖 当下一览：Docker 网络模式详解（自上而下，由浅入深）

以下从顶层抽象到具体实现，逐层拆解当前 Docker 的网络模式，便于按需查阅与选型。

### 第一层：顶层视角——容器看到了什么？

从容器内部看，它不关心自己用的是 bridge 还是 overlay。它只知道：**有一块网卡、一个 IP、一个网关、一份路由表和 DNS 配置**。Docker 通过不同驱动，在背后实现这些「表象」。

| 驱动        | 范围   | 隔离程度               | 容器获得                  | 典型场景                    |
| :---------- | :----- | :--------------------- | :------------------------ | :-------------------------- |
| **bridge**  | 单机   | 独立 Network Namespace | 虚拟 IP，经 NAT 访问外网  | 多容器协同、常规 Web 应用   |
| **host**    | 单机   | 无网络隔离             | 宿主机网络栈              | 高性能代理、监控、端口直绑  |
| **none**    | 单机   | 完全隔离               | 仅 loopback               | 离线计算、安全沙箱          |
| **overlay** | 跨主机 | 独立 Namespace         | overlay 内虚拟 IP         | Swarm 集群、多节点互通      |
| **macvlan** | 单机   | 独立 Namespace         | 独立 MAC + IP，直连物理网 | 容器需像物理机出现在局域网  |
| **ipvlan**  | 单机   | 独立 Namespace         | 共享 MAC，独立 IP         | 同上，但 MAC 数量受限时使用 |

此外还有 **container** 模式（`--network container:<name>`）：复用另一容器的网络栈，二者共享 IP 和端口，常用于 sidecar 等紧耦合场景。

### 第二层：按场景与约束分类

**按通信范围：**

*   **单机配置型**：bridge、host、none、macvlan、ipvlan。网络在单台宿主机上创建；bridge、host 可经 NAT 或直连访问外网，macvlan/ipvlan 容器挂在物理网中，同一物理网段内可跨主机互通。
*   **跨主机型**：overlay。需 Swarm 初始化，Docker 原生支持容器跨物理机互通。

**按隔离与性能权衡：**

| 模式    | 网络隔离 | NAT 开销         | 端口映射           | 服务发现（容器名解析） |
| :------ | :------- | :--------------- | :----------------- | :--------------------- |
| bridge  | 有       | 有               | 需要（对外暴露时） | 自定义网络支持         |
| host    | 无       | 无               | 不需要             | 不适用                 |
| none    | 最强     | 无               | 无                 | 无                     |
| overlay | 有       | 无（overlay 内） | 按需               | 支持                   |
| macvlan | 有       | 无               | 不需要             | 支持                   |
| ipvlan  | 有       | 无               | 不需要             | 支持                   |

### 第三层：逐模式深入

#### bridge（默认与自定义）

**原理**：基于 Linux `bridge`，宿主机上有 `docker0`（默认 bridge）或用户创建的虚拟网桥。容器通过 veth pair 连到网桥，获得虚拟 IP，经 iptables NAT 访问外网。

**默认 bridge vs 自定义 bridge**：

| 特性       | 默认 bridge        | 自定义 bridge      |
| :--------- | :----------------- | :----------------- |
| 容器名解析 | 不支持             | 支持（嵌入式 DNS） |
| 端口映射   | 需要（对外暴露时） | 需要（对外暴露时） |
| 隔离       | 同网段互通         | 可按网络分组隔离   |
| 自定义子网 | 默认 172.17.0.0/16（需改 daemon.json 调整） | 可指定 `--subnet`  |

**常用配置**：

```bash
# 创建自定义 bridge，指定子网
docker network create -d bridge --subnet 172.28.0.0/16 --gateway 172.28.0.1 mynet

# 容器加入网络，指定 IP（可选）
docker run --network mynet --ip 172.28.0.10 --name web nginx
```

#### host

**原理**：容器共享宿主机的 Network Namespace，无独立网卡和 IP。监听端口即宿主机端口。

**注意**：`-p` / `--publish` 在 host 模式下无效；多容器不能同时监听同一端口；Linux 专用（Docker Desktop 4.34+ 可选支持）。

#### none

**原理**：只有 loopback 接口，无对外网络。适用于完全不需网络的容器。

#### overlay

**原理**：基于 VXLAN 在物理网络之上构建逻辑二层网络。数据包经封装后经 UDP 4789 在节点间传输。需 Swarm 提供控制平面。

**关键点**：

*   单节点也可用 overlay（`docker swarm init` 后创建）。
*   若要让 `docker run` 的独立容器接入，需加 `--attachable`。
*   单机容器数过多（约 1000+）时可能出现稳定性问题。

```bash
docker swarm init
docker network create -d overlay --attachable --subnet 10.0.9.0/24 my_overlay
```

#### macvlan

**原理**：从物理网卡虚拟出多块「子接口」，每块有独立 MAC，容器直接出现在物理网络中。需 Linux 内核 3.9+，父接口通常为物理网卡。

**限制**：macvlan 容器**不能直接访问宿主机**（内核限制），如需互通需额外配置（如宿主机也建 macvlan 接口）。

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my_macvlan
```

#### ipvlan

**原理**：与 macvlan 类似，但**共享父接口 MAC**，仅 IP 不同。适合 MAC 数量受限的环境（如部分公有云）。内核建议 4.2+。

**模式**：`l2`（默认，同二层）、`l3`（父接口作路由）、`l3s`（L3 带源校验）。

```bash
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  my_ipvlan
```

#### container 模式

**原理**：通过 `--network container:<容器名或ID>` 复用已有容器的网络栈。两容器共享同一网络命名空间，即共享网络接口、IP 与端口。

**典型用途**：sidecar、调试（如 `redis-cli` 连本机 Redis）。

```bash
docker run -d --name redis redis --bind 127.0.0.1
docker run --rm -it --network container:redis redis redis-cli -h 127.0.0.1
```

### 第四层：选型决策

**快速选型指引**：

| 需求                       | 推荐模式         |
| :------------------------- | :--------------- |
| 单机多容器协作，常规应用   | bridge           |
| 追求极致性能，可接受无隔离 | host             |
| 完全无网络                 | none             |
| 多机 Swarm 集群互通        | overlay          |
| 容器需像物理机出现在物理网 | macvlan / ipvlan |
| 与某容器共享网络           | container        |

**多网络连接**：一个容器可接入多个网络，实现不同平面（如对外 bridge + 对内 internal）。通过 `docker network connect` 或多次 `--network` 指定。

**端口发布**：bridge 和 overlay 需对外暴露时，使用 `-p host_port:container_port`。host 和 macvlan/ipvlan 一般不需端口映射，容器直接监听端口。

---

## 📋 总结：按时间线看网络模式的演进

| 时期          | 核心驱动力           | 主要模式 / 能力                       |
| :------------ | :------------------- | :------------------------------------ |
| **2013–2014** | 单机容器能跑、能暴露 | 默认 bridge + 端口映射，`--link` 补丁 |
| **2015**      | 可插拔、跨主机编排   | libnetwork、bridge/host/none、overlay |
| **2016+**     | 企业网络融合、合规   | macvlan、ipvlan                       |

从「端口映射的天下」到「声明式网络抽象 + 多种驱动并存」，Docker 网络模式的演进史，本质上是**容器从单机玩具走向分布式基础设施**的缩影。理解了这段历史，我们在面对 `--network bridge`、`host`、`overlay` 或 `macvlan` 时，就能更清楚它们因何而生、何时该用——这不是随意堆砌的选项，而是不同时代、不同场景下的必然产物。
