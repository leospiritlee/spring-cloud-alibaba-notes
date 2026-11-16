# 第8章 Sentinel 规则：五类规则与滑动窗口

**一句话**：`FlowRule`、`DegradeRule`、`ParamFlowRule`、`SystemRule`、`AuthorityRule` 五种规则共用同一套 `滑动窗口` 统计，但分别挡流量、挡不稳的依赖、挡热点参数、挡整机过载、挡来源；`WarmUp` 与 `RateLimiterController` 再决定超限之后怎么排队。

**为什么需要它**：同一个「被拒」在日志里长得一模一样，根因却可能差很远：QPS 超阈值、下游慢调用比例升高、某个商品 ID 被反复刷、机器负载已经到顶、调用方不在白名单里。五种规则对应五种不同的现场，统一写成「限流没配好」就会把问题带到错误的那一层。原书第 11 章「Sentinel核心技术与配置规则」里逐项列出配置字段，第 10 章的整合示例里还留着一份 `flow-rule.json`：`resource`、`count`、`grade`、`strategy`、`controlBehavior`、`warmUpPeriodSec`、`maxQueueingTimeMs`、`clusterMode` 这些字段，正好是这一章要拆开的东西。

**核心结构**：规则对象与管理者成对出现。`FlowRule` 由 `FlowRuleManager` 加载，`DegradeRule` 由 `DegradeRuleManager` 加载，`ParamFlowRule`、`SystemRule`、`AuthorityRule` 各有对应管理者。控制行为上，`直接拒绝` 是最干脆的一种，`WarmUp` 处理 `冷启动`，`RateLimiterController` 处理 `匀速排队`；`令牌桶` 与 `漏桶` 是中文解析里讨论流控算法时反复引用的两种模型，该解析还明确写了 `Sentinel` 中流控设计是参考 Guava RateLimiter 的。`热点参数限流` 由 `ParamFlowSlot` 执行，`慢调用比例` 与 `异常比例` 是 `DegradeSlot` 的两类判据。

**关键设计**：把「判据」与「超限后的行为」拆开。`FlowRule` 的 `grade` 与 `count` 回答「按什么算、算到多少算超」，`controlBehavior` 回答「超了之后是直接拒绝、预热放行还是排队等」；`DegradeRule` 的 `慢调用比例` 与 `异常比例` 回答「什么时候熔断」，`熔断降级` 之后走 `fallback` 或 `blockHandler` 回答「返回什么」。原书第 10 章还提醒 `fallback` 与 `fallbackFactory` 只能选其一。把这三层混在一次配置里，就会出现「阈值明明调大了却还是被拒」——问题常在行为而不在阈值。

**拆解**：
### 1. FlowRule 挡的是流量形状
`FlowRule` 按 `grade`（如线程数或 QPS）与 `count` 设阈值，`strategy` 决定按资源、按关联还是按链路，`controlBehavior` 决定超限行为。`直接拒绝` 立刻返回，`WarmUp` 让 `冷启动` 期间阈值从低慢慢抬到设定值，`RateLimiterController` 配合 `maxQueueingTimeMs` 做 `匀速排队`。原书示例里 `warmUpPeriodSec` 与 `maxQueueingTimeMs` 同时出现在同一份 JSON 中，正说明行为字段与判据字段是并列的两组。

### 2. DegradeRule 挡的是不稳的依赖
`DegradeRule` 用 `慢调用比例` 或 `异常比例` 判断下游是否已经不稳，触发 `熔断降级` 后在一段时间内不再发起真实调用。`DegradeRuleManager` 负责加载。它与 `FlowRule` 的差别不在阈值大小，而在保护对象：前者保护自己不被慢依赖拖垮，后者保护资源不被超量请求压垮。旧卷已经把熔断器的滑动窗口与舱壁机制讲透，这里只记 Alibaba 栈里它落在 `DegradeSlot` 与这两个比例上。

### 3. 热点、系统与授权三类
`ParamFlowRule` 由 `ParamFlowSlot` 做 `热点参数限流`，把统计粒度下沉到参数值，适合「某个商品 ID 被刷爆」这类场景；`SystemRule` 由 `SystemSlot` 看整机负载，是入口之外的兜底；`AuthorityRule` 由 `AuthoritySlot` 管调用来源。三者共用同一条 `SlotChain`，但 `资源` 粒度、统计位置与拒绝理由都不同。综合报告口径常把这三类统统归入「限流规则」，正文按源码里的 Slot 名分开记录。

### 4. WarmUp、匀速排队与两种桶
中文解析用 Guava 26.0-jre 的 SmoothBursty 与 SmoothWarmingUp 两种模式做对照，说明 `WarmUp` 与 `RateLimiterController` 分别应对「系统刚启动还没热」与「请求突刺想摊平」两种流量形状；`令牌桶` 与 `漏桶` 是理解它们的模型，不是 `Sentinel` 里要直接配置的类。本册不复述算法的数学推导，只保留一条判断：选 `直接拒绝` 还是选排队，取决于你的调用方能不能等，而不是取决于阈值填几。

**今天怎么做**：拿一份线上 `flow-rule.json`，把字段分成三栏写下：判据（`grade` / `count` / `strategy`）、行为（`controlBehavior` / `warmUpPeriodSec` / `maxQueueingTimeMs`）、作用范围（`resource` / `clusterMode`）。再看一眼被拒日志里是 `FlowSlot` 还是 `DegradeSlot` 抛出的，然后才决定改哪一栏。

<div class="chapend" markdown="1">

**容易误解的地方**：`慢调用比例` 不是平均响应时间，`异常比例` 也不是错误总数；`WarmUp` 不是「阈值慢慢调大」的手工操作，它是规则里的一种控制行为。另一个误读是把 `热点参数限流` 当成普通的 `FlowRule` 加个参数——它由 `ParamFlowSlot` 单独执行，统计粒度不同。

**记住这一句**：五种规则共用统计，但挡的东西不同；先分清被谁拒，再动阈值。

</div>

<!-- chapend -->
