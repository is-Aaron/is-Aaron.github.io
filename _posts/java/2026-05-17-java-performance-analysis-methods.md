---
title: "Java 程序性能分析手段：从现象到证据"
date: 2026-05-17 20:00:00 +0800
categories: [Java, 工程实践]
tags: [Java, JVM, 性能分析, JFR, JMC, JMH, GC, async-profiler, Arthas]
mermaid: true
---

Java 程序的性能问题很少只有一个原因。一次接口超时，可能来自 CPU 热点、锁竞争、GC 暂停、线程池耗尽、数据库慢查询、连接池等待，也可能只是上游依赖抖动。真正困难的不是“知道很多优化技巧”，而是能把一个模糊现象拆成可验证的问题，并用合适工具拿到证据。

> **核心观点**：性能分析的顺序应该是：定义问题，建立基线，采集证据，定位瓶颈，实施改动，复测验证。没有测量就优化，通常只是在制造新的不确定性。

## 一、先把“慢”说清楚

“程序很慢”不是一个可分析的问题。开始动工具之前，先把现象拆成具体指标：

| 现象 | 应该追问的问题 |
| --- | --- |
| 接口延迟高 | 是平均延迟高，还是 P95/P99 高？是所有接口，还是少数接口？ |
| 吞吐量上不去 | CPU 已经打满，还是线程、连接、队列、下游服务限制住了？ |
| CPU 高 | 是用户态计算、GC、JIT、系统调用，还是自旋等待？ |
| 内存高 | 是 Java 堆、Metaspace、线程栈、Direct Memory，还是容器 RSS 高？ |
| GC 频繁 | 是分配速率太高、老年代增长、堆太小，还是对象生命周期设计不合理？ |
| 偶发卡顿 | 是 STW、锁竞争、磁盘/网络 IO、DNS、数据库、日志同步写入，还是虚拟化环境抖动？ |

如果无法回答这些问题，第一步不是改代码，而是补齐观测数据。至少要知道问题发生的时间、版本、流量、机器、JDK 参数、容器资源限制、错误率、延迟分位数和依赖服务状态。

## 二、性能分析的基本流程

性能分析不是“打开 profiler 看一眼”，而是一套闭环流程。

```mermaid
flowchart LR
    A["发现现象<br/>告警 / 用户反馈 / 压测结果"] --> B["定义目标<br/>延迟 / 吞吐 / CPU / 内存"]
    B --> C["建立基线<br/>版本 / 流量 / 环境 / 指标"]
    C --> D["采集证据<br/>指标 / 日志 / JFR / Profile / Dump"]
    D --> E["形成假设<br/>CPU / GC / 锁 / IO / 下游依赖"]
    E --> F["定点验证<br/>代码 / SQL / 配置 / 线程池"]
    F --> G["实施优化"]
    G --> H["复测对比<br/>Benchmark / 压测 / 线上观测"]
    H -->|"未达标"| D
    H -->|"达标"| I["沉淀结论<br/>监控 / 文档 / 回归用例"]
```

这套流程有两个关键点：

1. **先看全局，再看局部**：先确认 CPU、内存、GC、线程、IO、依赖服务等大类，再深入到具体方法和代码行。
2. **先证据，后优化**：每次改动前要能说清楚“瓶颈是什么”，改动后要能证明“指标变好了”。

## 三、工具全景：不同问题用不同工具

Java 性能分析工具很多，但它们解决的问题并不一样。

| 问题类型 | 首选手段 | 辅助手段 |
| --- | --- | --- |
| CPU 热点 | JFR、async-profiler 火焰图 | `top -H`、`jcmd Thread.print` |
| 内存分配过高 | JFR Allocation、async-profiler alloc | GC 日志、对象直方图 |
| 堆内存泄漏 | Heap Dump、MAT/VisualVM | `jcmd GC.class_histogram`、多次快照对比 |
| Native Memory 增长 | Native Memory Tracking | RSS、容器内存指标、Direct Memory 指标 |
| GC 暂停或频繁 GC | GC 日志、JFR GC 事件 | `jstat`、堆使用趋势 |
| 线程阻塞或死锁 | `jcmd Thread.print -l`、JFR monitor/park 事件 | 多次线程 dump、Arthas `thread` |
| 单个方法慢 | Arthas `trace` / `watch` | 日志埋点、JFR 执行采样 |
| 微小代码片段对比 | JMH | `perf stat`、JIT 编译日志 |
| 端到端链路慢 | APM / Trace / 日志关联 | 数据库慢查询、连接池指标、网关指标 |

工具选择的原则是：**先用低侵入、全局视角的工具缩小范围，再用高精度或高侵入工具做定点验证。**

## 四、第一层：系统指标和 JVM 基础指标

很多 Java 性能问题不在 Java 代码里。先看系统和 JVM 指标，可以避免一开始就陷进方法级别细节。

### 系统层指标

常见系统指标包括：

- CPU 使用率：区分用户态、系统态、iowait、steal time。
- Load Average / 运行队列：判断可运行任务或不可中断等待任务是否堆积；Linux 的 load average 不只代表 CPU 等待，也可能被 D 状态 I/O 等待拉高。
- 上下文切换：过高可能表示线程过多、锁竞争或阻塞唤醒频繁。
- 内存：关注 RSS、Swap、容器 memory limit、OOM kill。
- 磁盘和网络：日志同步写、磁盘抖动、网络重传都会放大接口延迟。

Linux 上定位 Java 进程高 CPU 时，常见做法是先找进程，再找线程：

```bash
top -H -p <pid>
printf "%x\n" <thread_id>
jcmd <pid> Thread.print -l
```

`top -H` 看到的是十进制线程 ID，而 Java 平台线程 dump 里的 `nid` 通常是十六进制，转换后才能对应起来。JDK 21 之后如果大量使用虚拟线程，要注意操作系统线程与 Java 虚拟线程不是一一对应关系；`top -H` 只能帮助定位承载虚拟线程的 carrier/platform thread。需要观察虚拟线程本身时，优先看 JFR 的 `jdk.VirtualThreadPinned`、`jdk.VirtualThreadSubmitFailed` 等事件；如果要观察虚拟线程启动和结束，还需要显式开启默认关闭的 `jdk.VirtualThreadStart` 和 `jdk.VirtualThreadEnd`。也可以用 `jcmd <pid> Thread.dump_to_file -format=json /tmp/threads.json` 生成包含虚拟线程的线程转储。

### JVM 层指标

JVM 基础指标至少包括：

- Heap 使用量、各代或各 region 使用情况。
- Non-Heap、Metaspace、Code Cache。
- GC 次数、GC 暂停时间、GC 后堆大小。
- 线程数、线程状态、线程池队列。
- 类加载数量、JIT 编译活动。
- 分配速率和晋升速率。

可以用 `jcmd` 查看进程启动参数和 JVM 状态：

```bash
jcmd <pid> VM.command_line
jcmd <pid> VM.flags
jcmd <pid> VM.system_properties
jstat -gcutil <pid> 1s 10
```

`jcmd` 需要在目标 JVM 所在机器上、以相同有效用户和用户组执行。`jstat` 适合快速观察 GC 趋势，但 JDK 文档把它标为 experimental/unsupported，输出和行为可能随版本变化；不要只凭一两行输出下结论。GC 问题需要结合 GC 日志、JFR 和业务流量一起看。

## 五、JFR 与 JMC：生产诊断的第一选择

JFR（JDK Flight Recorder，历史上也常称 Java Flight Recorder）是 JDK 自带的事件记录机制，能记录方法采样、对象分配、GC、锁、线程、文件 IO、Socket IO、异常、类加载等事件。JMC（JDK Mission Control）用于打开 `.jfr` 文件做可视化分析；现代 JDK 中 JMC 通常需要单独安装，不一定随普通 JDK 一起提供。

JFR 的优势是覆盖面广、侵入性低、适合线上短时间采集。它不一定给出“某一行代码就是问题”的答案，但能快速回答：

1. CPU 或执行采样主要落在哪些调用栈？
2. 哪些类型分配最多？
3. GC 是否造成明显暂停？
4. 哪些线程阻塞、等待或锁竞争严重？
5. 慢请求期间有没有文件 I/O、Socket I/O、异常风暴、monitor 等线程停顿事件？

启动时采集：

```bash
java \
  -XX:StartFlightRecording:filename=/tmp/app.jfr,duration=60s,settings=profile \
  -jar app.jar
```

对已经运行的进程采集，可以让记录在固定时长后自动停止并写出文件：

```bash
jcmd <pid> JFR.start name=profile settings=profile duration=60s filename=/tmp/app.jfr
```

也可以手动开始和停止，适合等待问题复现后再结束：

```bash
jcmd <pid> JFR.start name=profile settings=profile
jcmd <pid> JFR.check name=profile
jcmd <pid> JFR.stop name=profile filename=/tmp/app.jfr
```

如果是持续运行的记录，只想导出当前缓冲区而不停止记录，可以使用：

```bash
jcmd <pid> JFR.dump name=profile filename=/tmp/app.jfr
```

实践建议：

- 线上优先采集问题发生窗口，不要只采集空闲期。
- `settings=default` 更保守，`settings=profile` 信息更多但开销也更高。
- JFR 记录的是事件和采样，不是完整执行轨迹；不要把采样比例误读为绝对时间。
- 不同 JDK 版本的事件和字段会变化，分析时以目标运行时版本为准。

## 六、CPU 分析：用火焰图找热点路径

CPU 高时，最常用的是采样型 profiler。采样型 profiler 会定期记录线程调用栈，采样次数越多的调用栈，说明它越常出现在 CPU 执行现场。

火焰图的读法很简单：

- 横向宽度表示采样占比，不表示时间先后顺序。
- 纵向表示调用栈，下面是调用者，上面是被调用者。
- 最宽的栈通常是优先分析对象。
- 如果热点都在 GC、锁、自旋或系统调用里，说明不一定是业务计算代码慢。

async-profiler 是 Java 生态常用的低开销采样 profiler。它可以分析 CPU、分配、锁、wall-clock 等维度，并能输出 HTML 火焰图。

```bash
asprof -d 30 -e cpu -f /tmp/cpu.html <pid>
asprof -d 30 -e alloc -f /tmp/alloc.html <pid>
asprof -d 30 -e wall -t -f /tmp/wall.html <pid>
```

几个判断点：

1. **CPU 火焰图宽，不等于函数本身慢**：可能是它调用的下游方法消耗 CPU，也可能是 CPU 时间花在 native、JVM 或内核代码里；阻塞等待通常要看 wall-clock、线程 dump 或 JFR 阻塞事件。
2. **Wall-clock 火焰图适合看等待**：接口延迟高但 CPU 不高时，wall 模式比 CPU 模式更有用。
3. **采样要在稳定负载下做**：压测未预热、流量波动、机器争抢都会影响结果。
4. **需要区分 Java 栈和 native 栈**：加解密、压缩、正则、序列化、JNI、内核调用都可能出现在热点里。

## 七、内存分析：区分分配多、存活多和 native 多

内存问题至少分三类：

1. **分配速率高**：对象创建太频繁，导致 GC 压力大，但对象未必泄漏。
2. **存活对象增长**：对象被长期引用，可能是真正的堆泄漏。
3. **非堆或 native 内存增长**：Direct Buffer、Metaspace、线程栈、Code Cache、JNI 或第三方 native 库导致 RSS 增长。

### 看分配速率

分配速率高时，优先用 JFR Allocation 或 async-profiler alloc：

```bash
asprof -d 30 -e alloc -f /tmp/alloc.html <pid>
```

关注“谁分配最多”，而不是只看“当前谁占用最多”。例如一个接口每次创建大量临时 `String`、`byte[]`、JSON 对象，可能不会造成泄漏，但会显著增加 GC CPU 和暂停频率。

### 看堆中存活对象

对象直方图可以快速看当前堆里哪些类型实例多：

```bash
jcmd <pid> GC.class_histogram
```

这个命令会检查 Java 堆，官方影响等级是 High，耗时取决于堆大小和内容；生产环境不要在高峰期随手执行。如果需要把不可达对象也纳入统计，可以使用 `-all`，但这样更接近“当前所有对象快照”，不等于 GC 后仍存活对象。

如果需要更完整的引用链分析，可以导出 heap dump：

```bash
jcmd <pid> GC.heap_dump /tmp/app.hprof
```

注意：heap dump 文件可能非常大，生成过程可能造成明显停顿。JDK 的 `GC.heap_dump` 默认会请求一次 Full GC，除非指定 `-all` 导出包含不可达对象的 dump。生产环境导出前要确认磁盘空间、影响窗口和权限策略。分析 heap dump 时，不要只看最大对象，还要看 GC Roots 和引用链：谁把对象留住，才是泄漏问题的关键。

### 看 native memory

如果 Java 堆不高但进程 RSS 持续上涨，要怀疑 native memory。JDK 提供 Native Memory Tracking（NMT），但它通常需要在进程启动时开启：

```bash
java -XX:NativeMemoryTracking=summary -jar app.jar
jcmd <pid> VM.native_memory summary scale=MB
jcmd <pid> VM.native_memory baseline
jcmd <pid> VM.native_memory summary.diff scale=MB
```

NMT 可以帮助区分 Java Heap、Class、Thread、Code、GC、Compiler、Internal、Symbol、Arena 等 JVM 内部内存类别。它默认关闭，只能在启动时打开，不能通过 `jcmd` 对一个未开启 NMT 的进程再启动或重启；Oracle 文档提示开启 NMT 会带来约 5%-10% 的性能开销。它不是万能内存探针，对第三方 native 库、JDK 类库自身 native 分配、操作系统页缓存、容器统计口径仍需结合系统工具判断。

## 八、GC 分析：看暂停，也看分配

GC 问题常被误解为“换个垃圾回收器”或“把堆调大”。实际上，GC 分析至少要回答四个问题：

1. GC 暂停是否真的解释了业务延迟？
2. 对象分配速率是否过高？
3. 老年代或 old region 是否持续增长？
4. Full GC、to-space exhausted、humongous allocation、metaspace 扩容等事件是否出现？

JDK 9 之后可以使用统一日志参数输出 GC 日志：

```bash
-Xlog:gc*,safepoint:file=/var/log/app/gc-%p.log:time,uptime,level,tags:filecount=10,filesize=100M
```

看 GC 日志时，重点不是把每个字段都背下来，而是建立几个判断：

- **GC 后堆大小是否回落**：不回落通常说明存活对象多。
- **Young GC 是否过于频繁**：可能是分配速率高或新生代空间不足。
- **Old 区是否持续增长**：可能有长生命周期对象积累。
- **暂停是否与接口延迟尖刺对齐**：如果不对齐，瓶颈可能在别处。
- **GC CPU 是否显著**：低延迟 GC 也可能以更多 CPU 换暂停时间。

JFR 的 GC 视图和 GC 日志可以互补：JFR 更适合和线程、方法、IO、锁事件放在同一时间线里看；GC 日志更适合做长期趋势分析和离线归档。

## 九、线程、锁与阻塞分析

Java 服务吞吐下降但 CPU 不高，常见原因是线程在等待：

- 线程池耗尽，任务排队。
- 数据库连接池、HTTP 连接池等待。
- 锁竞争、类加载锁、日志锁。
- `synchronized`、`ReentrantLock`、`CountDownLatch`、`Future.get()` 等等待。
- 下游 RPC、数据库、缓存、文件 IO 慢。

线程 dump 是最直接的入口：

```bash
jcmd <pid> Thread.print -l
```

如果服务大量使用虚拟线程，可以导出包含平台线程和虚拟线程的 JSON 线程转储：

```bash
jcmd <pid> Thread.dump_to_file -format=json /tmp/threads.json
```

这类线程转储适合观察大量虚拟线程的栈，但它不包含传统线程 dump 中的对象地址、锁、JNI 统计和堆统计等信息。

分析线程 dump 不要只看一次。更好的方式是在问题窗口连续抓 3 到 5 次，每次间隔 5 到 10 秒。如果同一批线程持续停在同一个栈上，才更有诊断价值。下面这些是 JVM 线程状态，不等同于操作系统线程状态。

常见状态含义：

| 状态 | 含义 | 关注点 |
| --- | --- | --- |
| `RUNNABLE` | JVM 视角下可运行，可能正在执行，也可能在等待 CPU 或其他操作系统资源 | 高 CPU 线程、native IO、忙等 |
| `BLOCKED` | 等待进入 monitor 锁 | 谁持有锁、锁保护的临界区是否过大 |
| `WAITING` | 无限期等待通知 | 线程池、队列、`LockSupport.park`、`Future.get` |
| `TIMED_WAITING` | 带超时等待 | sleep、poll、网络超时、连接池等待 |

JDK 的线程 dump 可以发现 Java monitor 死锁。对于更细的锁竞争，JFR 的 `jdk.JavaMonitorEnter`、`jdk.JavaMonitorWait`、`jdk.ThreadPark` 等事件通常更好用，因为它们带有时间和调用栈。

## 十、Arthas：线上定点诊断

Arthas 适合在无法重启、无法立刻加日志的线上环境做定点排查。它可以查看线程、反编译类、观察方法入参返回值、统计方法耗时、追踪调用链，也可以通过 profiler 命令生成火焰图。

常用命令示例：

```bash
dashboard
thread -n 5
trace com.example.order.OrderService createOrder '#cost > 100'
watch com.example.order.OrderService createOrder '{params, returnObj, throwExp}' -x 2
monitor -c 5 com.example.order.OrderService createOrder
```

使用 Arthas 要克制：

- `watch` 可能打印敏感数据，生产环境要控制表达式和输出深度。
- `trace` 会增强目标方法，命中高频方法时可能带来额外开销。
- `trace` 默认只展开当前增强方法的一层调用，复杂深层路径要逐步缩小范围或配合动态 trace。
- 排查结束后要退出会话，并确认没有持续 profiler 或增强命令残留。
- 对核心交易链路，优先在低峰期或单台实例上操作。

Arthas 的价值在于回答具体问题，例如“这个方法到底被谁调用”“这个参数是不是异常”“这个方法内部哪一步耗时最多”。它不替代监控、JFR 和系统级 profiler。

## 十一、JMH：微基准不要手写计时器

如果要比较两个小函数、两种集合、两段序列化逻辑的性能，不要用 `System.currentTimeMillis()` 或简单循环计时。JVM 有 JIT 编译、逃逸分析、死代码消除、常量折叠、CPU 缓存、分支预测等因素，手写计时器很容易测错。

JMH（Java Microbenchmark Harness）是 OpenJDK 维护的 Java 微基准框架。它会处理预热、fork、测量轮次、结果统计等问题。

一个极简示例：

```java
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.State;

@State(Scope.Thread)
public class ParseBenchmark {
    private String value = "123456";

    @Benchmark
    public int parseInt() {
        return Integer.parseInt(value);
    }
}
```

写 JMH 时要注意：

1. 优先使用 JMH 官方 Maven archetype 或正确配置 annotation processor；只引入 `jmh-core` 并不能保证基准能正确生成和运行。
2. 使用 `@Warmup`、`@Measurement`、`@Fork` 控制预热和测量。
3. 避免被死代码消除，必要时使用 `Blackhole`。
4. 不要把微基准结论直接等同于线上结论。
5. 关注误差范围，不要为 1% 以内且不稳定的差异做复杂改造。
6. 对用户可感知性能，最终仍要通过端到端压测或线上灰度验证。

微基准适合回答“这段代码本身哪个更快”，不适合回答“整个服务为什么慢”。后者要用压测、JFR、APM、日志和依赖指标综合分析。

## 十二、链路分析：别把所有慢都归因于 JVM

在微服务系统里，一个 Java 接口慢，经常不是 Java 方法本身慢，而是依赖链路慢：

- 网关排队或限流。
- RPC 客户端连接池不足。
- 数据库连接池等待。
- 慢 SQL、锁等待、索引失效。
- Redis 热 key、网络抖动、序列化开销。
- 日志同步写入或磁盘 IO 抖动。

因此线上服务应该具备三类基础观测能力：

1. **指标**：请求量、错误率、延迟分位数、线程池、连接池、GC、CPU、内存。
2. **日志**：请求 ID、关键业务字段、耗时分段、异常栈、下游调用结果。
3. **Trace**：跨服务调用链、每一跳耗时、错误位置、数据库和缓存跨度。

如果 trace 显示 900ms 都花在数据库查询上，那么继续优化 Java 循环通常没有意义。性能分析的目标不是证明代码有问题，而是找到系统中真正限制目标指标的那一段。

## 十三、常见误区

1. **只看平均延迟**：平均值会掩盖 P95/P99 尖刺，用户体感通常被尾延迟决定。
2. **在空闲环境采样**：没有代表性负载，profile 只会告诉你空闲时在干什么。
3. **只抓一次线程 dump**：一次 dump 是快照，多次 dump 才能看持续阻塞。
4. **把内存高等同于泄漏**：高缓存、高堆上限、GC 策略、RSS 口径都可能让内存看起来高。
5. **看到 GC 就调堆**：GC 可能是结果，不是原因；真正原因可能是分配过快。
6. **微基准替代压测**：JMH 能测函数，不能代表端到端链路。
7. **只改代码不复测**：没有前后对比，就无法证明优化有效。
8. **忽略安全和权限**：heap dump、Arthas watch、JFR 事件都可能包含敏感数据。

## 十四、实战排查清单

遇到 Java 服务性能问题，可以按下面顺序推进：

1. 明确问题：哪个接口、哪个时间段、哪个版本、哪些机器、什么指标异常。
2. 看监控：CPU、内存、GC、线程、连接池、错误率、P95/P99、下游依赖。
3. 看日志和 trace：确认慢在哪一跳，是否有异常重试、超时、限流、慢 SQL。
4. 采集 JFR：覆盖问题窗口，先从时间线、热点方法、分配、GC、锁、IO 看全貌。
5. 高 CPU 用 CPU 火焰图；低 CPU 高延迟用 wall-clock、线程 dump 和 trace。
6. 内存问题先区分 heap、non-heap、native，再选择 heap dump、alloc profile 或 NMT。
7. 对单个方法疑点，用 Arthas `trace` / `watch` 定点验证。
8. 形成改动后，用相同负载、相同指标复测，并记录优化前后数据。

## 总结

Java 性能分析的核心不是熟记某个命令，而是建立从现象到证据的推理链。系统指标告诉你资源是否紧张，JVM 指标告诉你运行时是否异常，JFR 给出时间线和事件全景，火焰图揭示热点调用栈，heap dump 和 NMT 区分不同内存来源，线程 dump 和锁事件解释阻塞，JMH 则用于验证微观代码改动。

真正可靠的性能优化应该能回答三个问题：瓶颈在哪里，为什么这是瓶颈，改完之后数据是否变好。只要坚持这个闭环，性能分析就会从“凭经验猜”变成一项可复盘、可协作、可验证的工程活动。

## 术语表

- **APM**：Application Performance Monitoring，应用性能监控，通常包含指标、日志、链路追踪和告警。
- **CPU Profile**：CPU 采样分析结果，用于判断哪些调用栈消耗了最多 CPU 时间。
- **Flame Graph**：火焰图，用宽度展示采样占比，用纵向层级展示调用栈。
- **GC**：Garbage Collection，垃圾回收，自动回收不再可达对象占用的内存。
- **Heap Dump**：堆转储，记录某一时刻 Java 堆中对象和引用关系的快照。
- **JFR**：JDK Flight Recorder，JDK 自带的运行时事件记录工具；历史上也常称 Java Flight Recorder。
- **JIT**：Just-In-Time Compilation，即时编译，JVM 在运行时把热点字节码编译成本地机器码。
- **JMC**：JDK Mission Control，用于分析 JFR 文件的可视化工具。
- **JMH**：Java Microbenchmark Harness，OpenJDK 维护的 Java 微基准测试框架。
- **NMT**：Native Memory Tracking，JVM 原生内存跟踪能力，用于分析部分 native memory 分类。
- **P95 / P99**：延迟分位数，P99 表示 99% 的请求延迟不超过该值。
- **RSS**：Resident Set Size，进程当前驻留在物理内存中的页面大小。
- **STW**：Stop-The-World，JVM 暂停应用线程执行某些运行时操作的阶段。

## 参考文献

- Oracle, [Java Development Kit Tool Specifications](https://docs.oracle.com/en/java/javase/26/docs/specs/man/index.html)
- Oracle, [The jcmd Command](https://docs.oracle.com/en/java/javase/26/docs/specs/man/jcmd.html)
- Oracle, [The jstat Command](https://docs.oracle.com/en/java/javase/26/docs/specs/man/jstat.html)
- Oracle, [The java Command](https://docs.oracle.com/en/java/javase/26/docs/specs/man/java.html)
- Oracle, [Thread.State API](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/lang/Thread.State.html)
- Oracle, [Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- Oracle, [Diagnostic Tools](https://docs.oracle.com/en/java/javase/26/troubleshoot/diagnostic-tools.html)
- Oracle, [Troubleshoot Performance Issues Using Flight Recorder](https://docs.oracle.com/en/java/javase/26/troubleshoot/troubleshoot-performance-issues-using-jfr.html)
- Oracle, [Native Memory Tracking](https://docs.oracle.com/en/java/javase/25/vm/native-memory-tracking.html)
- Oracle, [JDK Mission Control Documentation](https://docs.oracle.com/en/java/java-components/jdk-mission-control/)
- Linux man-pages, [proc_loadavg(5)](https://man7.org/linux/man-pages/man5/proc_loadavg.5.html)
- OpenJDK, [JMH: Java Microbenchmark Harness](https://github.com/openjdk/jmh)
- async-profiler, [async-profiler README](https://github.com/async-profiler/async-profiler)
- async-profiler, [Profiling Modes](https://github.com/async-profiler/async-profiler/blob/master/docs/ProfilingModes.md)
- Alibaba Arthas, [Arthas Documentation](https://arthas.aliyun.com/en/doc/)
- Alibaba Arthas, [thread Command](https://arthas.aliyun.com/en/doc/thread.html)
- Alibaba Arthas, [trace Command](https://arthas.aliyun.com/en/doc/trace.html)
- Alibaba Arthas, [watch Command](https://arthas.aliyun.com/en/doc/watch.html)
- Alibaba Arthas, [monitor Command](https://arthas.aliyun.com/en/doc/monitor.html)
- Alibaba Arthas, [profiler Command](https://arthas.aliyun.com/en/doc/profiler.html)
