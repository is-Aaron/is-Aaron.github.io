---
title: 一致性哈希：分布式系统数据分片的基石
date: 2026-02-05 12:00:00 +0800
categories: [分布式系统, 算法]
tags: [一致性哈希, 分布式缓存, 负载均衡, 虚拟节点]
mermaid: true
---

> **核心观点**：一致性哈希通过**哈希环**将数据和节点映射到同一空间，使得节点增删时**仅影响相邻区间的数据**，避免全局数据迁移，是分布式系统动态扩缩容的关键技术。

## 一、问题：传统哈希的致命缺陷

假设用 `hash(key) % N` 将数据分配到 N 个节点：

```mermaid
graph LR
    subgraph Before[" 🟢 3 个节点时 "]
        direction LR
        K1["🔑 key=7"] -->|"7 % 3 = 1"| N1["📦 Node 1"]
        K2["🔑 key=8"] -->|"8 % 3 = 2"| N2["📦 Node 2"]
        K3["🔑 key=9"] -->|"9 % 3 = 0"| N0["📦 Node 0"]
    end
    
    Before =====>|"➕ 新增 1 个节点"| After
    
    subgraph After[" 🔴 4 个节点时 "]
        direction LR
        K1a["🔑 key=7"] -->|"7 % 4 = 3"| N3a["📦 Node 3"]
        K2a["🔑 key=8"] -->|"8 % 4 = 0"| N0a["📦 Node 0"]
        K3a["🔑 key=9"] -->|"9 % 4 = 1"| N1a["📦 Node 1"]
    end
    
    style Before fill:#d3f9d8,stroke:#1e7b34,color:#1e7b34
    style After fill:#ffe3e3,stroke:#c92a2a,color:#c92a2a
    style N1 fill:#d0ebff,stroke:#1864ab,color:#1864ab
    style N2 fill:#d0ebff,stroke:#1864ab,color:#1864ab
    style N0 fill:#d0ebff,stroke:#1864ab,color:#1864ab
    style N3a fill:#ffe0e0,stroke:#c92a2a,color:#c92a2a
    style N0a fill:#ffe0e0,stroke:#c92a2a,color:#c92a2a
    style N1a fill:#ffe0e0,stroke:#c92a2a,color:#c92a2a
```

**问题**：节点数从 3 变成 4，**几乎所有数据的映射都变了**，需要大规模迁移。

## 二、解决方案：哈希环

一致性哈希的核心是将节点和数据映射到一个**首尾相连的环形空间**（通常是 `0 ~ 2³²-1`）：

```mermaid
graph LR
    subgraph Ring[" 🔄 哈希环 （0 ~ 2³²-1）"]
        direction LR
        
        subgraph Nodes[" 服务器节点 "]
            A["🖥️ Node A<br/>hash = 1000"]
            B["🖥️ Node B<br/>hash = 4000"]
            C["🖥️ Node C<br/>hash = 7000"]
        end
        
        subgraph Data[" 数据对象 "]
            D1["📄 Data 1<br/>hash = 500"]
            D2["📄 Data 2<br/>hash = 2500"]
            D3["📄 Data 3<br/>hash = 5000"]
        end
    end
    
    D1 -.->|"⟳ 顺时针"| A
    D2 -.->|"⟳ 顺时针"| B
    D3 -.->|"⟳ 顺时针"| C
    
    style Ring fill:#f1f3f5,stroke:#495057,color:#212529
    style Nodes fill:#d3f9d8,stroke:#1e7b34,color:#1e7b34
    style Data fill:#d0ebff,stroke:#1864ab,color:#1864ab
    style A fill:#b2f2bb,stroke:#1e7b34,stroke-width:2px,color:#1e7b34
    style B fill:#b2f2bb,stroke:#1e7b34,stroke-width:2px,color:#1e7b34
    style C fill:#b2f2bb,stroke:#1e7b34,stroke-width:2px,color:#1e7b34
    style D1 fill:#a5d8ff,stroke:#1864ab,stroke-width:2px,color:#1864ab
    style D2 fill:#a5d8ff,stroke:#1864ab,stroke-width:2px,color:#1864ab
    style D3 fill:#a5d8ff,stroke:#1864ab,stroke-width:2px,color:#1864ab
```

**定位规则**：数据从自身位置**顺时针查找**，遇到的第一个节点就是其归属节点。

## 三、核心优势：最小化数据迁移

### 节点下线

```mermaid
graph LR
    subgraph Before[" 📊 Node B 下线前 "]
        direction TB
        A1["🖥️ Node A"]
        B1["🖥️ Node B"]
        C1["🖥️ Node C"]
        D1["📄 Data 1"]
        D2["📄 Data 2"]
        D3["📄 Data 3"]
        A1 --- D1
        B1 --- D2
        C1 --- D3
    end
    
    Before ==>|"💥 Node B 宕机"| After
    
    subgraph After[" 📊 Node B 下线后 "]
        direction TB
        A2["🖥️ Node A"]
        C2["🖥️ Node C"]
        D1a["📄 Data 1 ✅"]
        D2a["📄 Data 2 🔄"]
        D3a["📄 Data 3 ✅"]
        A2 --- D1a
        C2 --- D2a
        C2 --- D3a
    end
    
    style Before fill:#d0ebff,stroke:#1864ab,color:#1864ab
    style After fill:#fff3bf,stroke:#e67700,color:#e67700
    style B1 fill:#ffe0e0,stroke:#c92a2a,stroke-width:2px,stroke-dasharray: 5 5,color:#c92a2a
    style D2a fill:#ffe066,stroke:#e67700,stroke-width:2px,color:#945a00
    style A1 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style A2 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style C1 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style C2 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
```

**只有 Node B 的数据迁移到 Node C**，其他数据不受影响。

### 节点上线

```mermaid
graph LR
    subgraph Before[" 📊 新增 Node D 前 "]
        direction TB
        A1["🖥️ Node A"]
        B1["🖥️ Node B"]
        D1["📄 Data 1, 2, 3"]
        D2["📄 Data 4, 5"]
        A1 --- D1
        B1 --- D2
    end
    
    Before ==>|"➕ 加入 Node D"| After
    
    subgraph After[" 📊 新增 Node D 后 "]
        direction TB
        A2["🖥️ Node A"]
        D2a["🖥️ Node D"]
        B2["🖥️ Node B"]
        D1a["📄 Data 1 ✅"]
        D2b["📄 Data 2, 3 🔄"]
        D3a["📄 Data 4, 5 ✅"]
        A2 --- D1a
        D2a --- D2b
        B2 --- D3a
    end
    
    style Before fill:#d0ebff,stroke:#1864ab,color:#1864ab
    style After fill:#d3f9d8,stroke:#1e7b34,color:#1e7b34
    style D2a fill:#69db7c,stroke:#1e7b34,stroke-width:3px,color:#1e7b34
    style D2b fill:#ffe066,stroke:#e67700,stroke-width:2px,color:#945a00
    style A1 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style A2 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style B1 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style B2 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
```

**只有部分数据从 Node A 迁移到新节点 D**。

## 四、虚拟节点：解决数据倾斜

当节点数较少时，可能出现分布不均：

```mermaid
graph TB
    subgraph Problem[" ⚠️ 问题：负载不均衡 "]
        direction LR
        N1["🖥️ Node A<br/>━━━━━━━ 70%"]
        N2["🖥️ Node B<br/>━━ 20%"]
        N3["🖥️ Node C<br/>━ 10%"]
    end
    
    Problem ==>|"💡 引入虚拟节点"| Solution
    
    subgraph Solution[" ✅ 方案：虚拟节点均匀分布 "]
        direction LR
        subgraph VA[" Node A "]
            A1["A#1"]
            A2["A#2"]
            A3["A#3"]
        end
        subgraph VB[" Node B "]
            B1["B#1"]
            B2["B#2"]
            B3["B#3"]
        end
        subgraph VC[" Node C "]
            C1["C#1"]
            C2["C#2"]
            C3["C#3"]
        end
    end
    
    style Problem fill:#ffe0e0,stroke:#c92a2a,color:#c92a2a
    style Solution fill:#d3f9d8,stroke:#1e7b34,color:#1e7b34
    style N1 fill:#ffa8a8,stroke:#c92a2a,stroke-width:2px,color:#a51d1d
    style N2 fill:#ffc078,stroke:#d9480f,color:#a53c00
    style N3 fill:#ffe066,stroke:#e67700,color:#945a00
    style VA fill:#d3f9d8,stroke:#1e7b34,color:#1e7b34
    style VB fill:#d3f9d8,stroke:#1e7b34,color:#1e7b34
    style VC fill:#d3f9d8,stroke:#1e7b34,color:#1e7b34
    style A1 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style A2 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style A3 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style B1 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style B2 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style B3 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style C1 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style C2 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
    style C3 fill:#b2f2bb,stroke:#1e7b34,color:#1e7b34
```

| 特性     | 无虚拟节点           | 有虚拟节点                |
| -------- | -------------------- | ------------------------- |
| 数据分布 | 可能严重不均         | 趋于均匀                  |
| 故障影响 | 全部迁移到下一个节点 | 分散迁移到多个节点        |
| 内存开销 | 低                   | 需维护虚拟节点映射        |
| 常见配置 | -                    | 100~200 个/节点（ketama） |

## 五、代码示例（Go）

```go
type ConsistentHash struct {
    ring     map[uint32]string // hash -> node
    sorted   []uint32          // 有序的 hash 值
    replicas int               // 每个节点的虚拟节点数
}

func (c *ConsistentHash) Add(node string) {
    for i := 0; i < c.replicas; i++ {
        hash := c.hash(fmt.Sprintf("%s#%d", node, i))
        c.ring[hash] = node
        c.sorted = append(c.sorted, hash)
    }
    sort.Slice(c.sorted, func(i, j int) bool {
        return c.sorted[i] < c.sorted[j]
    })
}

func (c *ConsistentHash) Get(key string) string {
    hash := c.hash(key)
    // 二分查找第一个 >= hash 的节点
    idx := sort.Search(len(c.sorted), func(i int) bool {
        return c.sorted[i] >= hash
    })
    if idx == len(c.sorted) {
        idx = 0 // 回绕到环首
    }
    return c.ring[c.sorted[idx]]
}
```

## 六、实际应用

| 系统              | 实现方式                                      |
| ----------------- | --------------------------------------------- |
| **Memcached**     | 客户端实现一致性哈希（ketama 算法）           |
| **Cassandra**     | Murmur3Partitioner + 虚拟节点（vnodes）       |
| **Nginx/Tengine** | Nginx 内置 `hash ... consistent` 参数；Tengine 提供专用 `consistent_hash` 模块 |
| **Amazon Dynamo** | 一致性哈希的经典实现，影响了众多后续系统      |

> **注意**：Redis Cluster 使用 16384 个**固定哈希槽**（Hash Slots），虽然也能最小化数据迁移，但这是**预分配的分片机制**，与一致性哈希的环形空间动态映射是不同的设计。

## 七、总结

| 对比项       | 传统哈希 `% N`        | 一致性哈希         |
| ------------ | --------------------- | ------------------ |
| 节点变化影响 | 全局重新映射          | 仅影响相邻区间     |
| 数据迁移量   | N/(N+1)（约 75%~90%） | ~1/N（约 10%~25%） |
| 扩展性       | 差                    | 优秀               |
| 实现复杂度   | 简单                  | 中等               |

**一句话总结**：一致性哈希通过环形空间 + 顺时针查找的设计，将节点变化的影响从**全局**收敛到**局部**，是分布式系统实现弹性伸缩的核心算法。

---

**参考资料**：
- Karger et al. [*"Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web"*](https://dl.acm.org/doi/10.1145/258533.258660) (STOC 1997) — 一致性哈希的原始论文

