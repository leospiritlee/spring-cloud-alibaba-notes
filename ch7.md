# 第7章 Sentinel 骨架：一次 entry 穿过 SlotChain

**一句话**：`Sentinel` 把「这次调用要不要放行」拆成一条 `SlotChain`：前面几个 `ProcessorSlot` 建上下文与资源树，`StatisticSlot` 记数，后面的 `FlowSlot`、`DegradeSlot` 才做判定；顺序就是它的设计。

**为什么需要它**：你给一个接口加了 `@SentinelResource`，压测时却发现限流没生效。最容易怀疑的是规则写错，但更常见的是「这个 `资源` 根本没被埋点」或者「统计还没发生就做了判定」。中文源码解析里那句定位说得最准：`Sentinel` 的定位是流量控制、`熔断降级`，你应该把它理解为一个第三方 Jar 包——它不接管你的请求，只在你显式进入的那个点上做一次检查。所以要看懂它，得先看一次 `SphU` 调用从头到尾穿过了哪几个 `ProcessorSlot`。

**核心结构**：入口是 `SphU`（另有布尔式的 `SphO`），调用返回一个 `Entry`；内部由 `CtSph` 组织，为 `资源` 找到或创建一条 `SlotChain`。`ContextUtil` 建立调用上下文，`NodeSelectorSlot` 与 `ClusterBuilderSlot` 分别挂起 `DefaultNode` 与 `ClusterNode`，形成 `簇点链路`；`StatisticSlot` 用 `LeapArray` 里的 `WindowWrap` 与 `MetricBucket` 做 `滑动窗口` 统计；`LogSlot` 记日志；`FlowSlot`、`DegradeSlot`、`AuthoritySlot`、`ParamFlowSlot`、`SystemSlot` 各按自己的规则检查。链上每个 `ProcessorSlot` 都只做一件事，且都依赖前序 Slot 留下的结构。

**关键设计**：把「统计」放在「判定」之前。`StatisticSlot` 先记录这一秒的通过数与阻塞数，后面的 `FlowSlot` 才能拿 `滑动窗口` 里的数去比 `FlowRule` 的阈值；`DegradeSlot` 也才有 `慢调用比例` 或 `异常比例` 可算。反过来，如果判定在前，`簇点链路` 上就永远看不到被拒绝的那部分调用。源码里 `SlotChain` 的构造顺序与官方 Wiki 的 How-it-works 页、Sentinel工作主流程 一文相互印证；本册用本地 1.8.x 源码里的类名为准，两篇中文解析基于 1.7.0 的说法按各自来源标注，不替来源调和。

**拆解**：
### 1. 从 SphU 到 Entry
`SphU` 的 entry 方法接受 `资源` 名，返回 `Entry`，`CtSph` 负责为这个名字找到对应的 `SlotChain` 并逐个触发 `ProcessorSlot`。`SphO` 提供布尔式入口，语义相同、返回不同。`Entry` 必须在 finally 里 exit，否则 `StatisticSlot` 的计数不闭合，后面的判定会一直看到「有进无出」的窗口。这是最容易被忽略、也最容易让规则「时灵时不灵」的一处。

### 2. 上下文与资源树怎么建起来
`ContextUtil` 给出调用上下文，`NodeSelectorSlot` 把 `资源` 挂成 `DefaultNode`，`ClusterBuilderSlot` 汇总出 `ClusterNode`，三者一起构成 `簇点链路`。同一份 `资源` 在不同调用入口下会有不同的 `DefaultNode`，但共享一个 `ClusterNode`——这正是「按入口分别看」与「按资源整体看」能同时成立的原因。控制台里的树状视图，画的就是这层结构。

### 3. 滑动窗口里到底存了什么
`LeapArray` 是环形数组，`WindowWrap` 是其中一个时间窗的包装，`MetricBucket` 存这一窗内的通过、阻塞、异常、响应时间等计数。`滑动窗口` 让统计随时间推进复用旧格子，而不是每秒新建一整份数据。看懂这三层的包含关系，才能解释为什么阈值配在「每秒」上时，边界附近会出现看似多放过一个请求的现象——那是窗口边界，不是规则失效。

### 4. 判定 Slot 各挡什么
`FlowSlot` 按 `FlowRule` 做 `限流`，`DegradeSlot` 按 `DegradeRule` 做 `熔断降级`，`AuthoritySlot` 按 `AuthorityRule` 管来源，`ParamFlowSlot` 按 `ParamFlowRule` 做 `热点参数限流`，`SystemSlot` 按 `SystemRule` 看整体负载。`LogSlot` 只记不挡。五个判定 Slot 共用前序 `StatisticSlot` 的统计，但判据与拒绝原因互不相同；排查「为什么被拒」时，先看是哪一个 Slot 抛出的，而不是先去改 `FlowRule` 的阈值。

**今天怎么做**：在一处 `@SentinelResource` 方法上，按顺序列出它这次 entry 会经过的 `ProcessorSlot`，并标出「建结构」「记数」「做判定」三类分别对应哪几个。再回答一句：如果 `Entry` 没有 exit，`滑动窗口` 的哪一格会失真。

<div class="chapend" markdown="1">

**容易误解的地方**：`Sentinel` 不是代理整个 HTTP 请求的框架，没被 `SphU` 或 `@SentinelResource` 包住的代码不会进入 `SlotChain`；`簇点链路` 也不是调用链追踪，它是 `资源` 在这条 Slot 链上形成的树。另一个误读是把 `StatisticSlot` 当成「事后记日志」——它在判定之前，是判定的数据前提。

**记住这一句**：先建树、再记数、后判定，SlotChain 的顺序就是限流的语义。

</div>

<!-- chapend -->
