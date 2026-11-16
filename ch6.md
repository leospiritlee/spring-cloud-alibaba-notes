# 第6章 Gateway：入口那道路由闸

**一句话**：`Spring Cloud Gateway` 用 `Route` 描述一条路由，`Predicate` 决定这次请求走不走这条，`GatewayFilter` 与 `GlobalFilter` 决定进出时怎么改；入口职责全部收在这一层，业务服务不必各自再做一遍。

**为什么需要它**：没有网关时，每个微服务都得自己处理鉴权透传、路径前缀、跨域、限流入口与日志埋点，七道闸里的第一道就散进了所有服务。原书第 12 章把网关概述与核心架构单独成章，第 13 章整合，第 14 章再讲核心技术——它把「断言」「过滤器」「熔断」并列，正是因为这三样分别对应「走不走」「怎么改」「失败怎么办」。你在网关上配错一条 `Predicate` 的症状，往往是所有服务都收不到请求，而不是某一个接口报错。

**核心结构**：一条 `Route` 由标识、目标 URI、`Predicate` 集合与过滤器集合组成。`Predicate` 也叫谓词，原书第 14 章明确写「只有所有的判断结果都通过时，也就是所有的条件判断都返回 true，才会真正的执行路由功能」，并列出基于日期时间、路径、请求头等类型的内置断言工厂。`GatewayFilter` 作用于单条路由，`GlobalFilter` 作用于所有路由；`StripPrefix` 这类过滤器负责改写路径。`Spring Cloud Gateway` 与 `Sentinel` 的适配让入口也能做 `限流`，官方 starter 的 README 与官网示例文档里都能查到。

**关键设计**：把「判定」与「改写」分成两个阶段，且都在请求到达业务服务之前完成。`Predicate` 只读请求属性，决定匹配哪条 `Route`；`GatewayFilter` 与 `GlobalFilter` 才动手改路径、加头、记日志。`GlobalFilter` 的「全局」指的是作用范围，不是执行优先级——原书第 13 章里前后置过滤器的日志顺序，恰好能看出这一点。入口 `限流` 与调用点 `限流` 是两道不同的闸：前者保护整个系统，后者保护某个资源，第八章会把后者拆开。

**拆解**：
### 1. Route 与 Predicate 的分工
`Route` 是配置对象，`Predicate` 是它的匹配条件集合，全部为真才执行路由。原书列出的 `AfterRoutePredicateFactory`、`BeforeRoutePredicateFactory`、`BetweenRoutePredicateFactory` 属于按日期时间判断的一族，另有按路径、按请求头、按方法判断的工厂。排查「请求 404 但服务明明活着」时，先看是不是没有任何一条 `Route` 的 `Predicate` 命中，而不是先去查注册中心。

### 2. 两类过滤器与 StripPrefix
`GatewayFilter` 绑在单条 `Route` 上，`GlobalFilter` 对所有路由生效，`StripPrefix` 负责在转发前剥掉路径前缀。三者都可能改写请求，但作用范围不同；把本该全局做的 `TraceID` 透传写进某一条 `GatewayFilter`，就会出现「只有部分链路能串起来」这种第十一章才会暴露的问题。

### 3. 入口限流与调用点限流
`Spring Cloud Gateway` 通过 `Sentinel` 的 gateway starter 在入口做 `限流`，而 `@SentinelResource` 与 `FlowSlot` 在业务方法处做 `限流`。两者共用同一套 `FlowRule` 概念，但资源名不同、统计位置不同、被拒之后返回的东西也不同。综合报告口径常把「网关限流」与「接口限流」混着说，正文按官方 starter README 与源码里的适配类分别记录，不替来源调和。

### 4. 网关与旧卷的 Zuul
旧卷里 `Zuul` 用过滤器链承载同样的入口职责，这里换成 `Route` + `Predicate` + 过滤器三段。旧卷已经讲过反向代理的分流与过滤器链机制，本册不重复；只补一条：Alibaba 栈里入口这一层是 `Spring Cloud Gateway`，不是 `Nacos` 也不是 `Sentinel`，尽管后两者都能在这里被引用。

**今天怎么做**：挑一条线上 `Route`，把它拆成四栏写下：匹配条件（`Predicate`）、目标 URI、单路由过滤器、全局过滤器。再问一句「如果这条路由全不命中，日志里会出现在哪一层」。四栏加一问，就是入口闸的验收单。

<div class="chapend" markdown="1">

**容易误解的地方**：`Predicate` 不是过滤器，它不改写请求，只做条件判定；`GlobalFilter` 也不等于「最先执行」。另一个常见误读是把 `StripPrefix` 当成路由规则——它只改路径，不决定走哪条 `Route`。

**记住这一句**：断言决定走不走，过滤器决定怎么改，两者都在业务服务之前。

</div>

<!-- chapend -->

# SpringCloudAlibaba精读
