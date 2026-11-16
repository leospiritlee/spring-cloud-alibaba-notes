# 第5章 服务调用：从接口到一次远程调用

**一句话**：`@FeignClient` 把一个 Java 接口变成一次 HTTP 调用，`LoadBalancer` 在多个实例里选一台，`@LoadBalanced` 的 `RestTemplate` 是另一种写法；`Dubbo` 与 `@DubboReference` 走的是另一条接缝，但解决的是同一个「别写死地址」的问题。

**为什么需要它**：原书第 6 章搭出三个微服务之后，第 7 章立刻引入注册与发现，第 8 章才讲 `负载均衡`——顺序说明了一件事：只有 `服务发现` 给出一份实例列表，选一台才有意义。你在订单服务里写下 `@FeignClient("server-product")` 时，括号里是服务名不是 IP，这层间接正是第三章那份清单在调用侧的兑现。`OpenFeign` 负责把接口声明变成请求，`Ribbon` 或 `LoadBalancer` 负责选择，`fallback` 负责失败之后给一个可返回的答案。

**核心结构**：三层。第一层是声明：`@FeignClient` 标在接口上，方法上的注解描述路径与参数；`@LoadBalanced` 标在 `RestTemplate` 上，让服务名形式的 URL 能被解析。第二层是选择：`LoadBalancer` 从 `服务发现` 拿到的实例列表里挑一台，`权重` 与 `元数据` 是这一层可参考的信息。第三层是兜底：`fallback` 与 `blockHandler` 都出现在原书第 10 章的容错示例里，原书还明确写了一句 `fallback` 与 `fallbackFactory` 只能选其一。`Dubbo` 侧的 `@DubboReference` 生成的是另一套代理，但它同样依赖注册中心给出的地址。

**关键设计**：把「调用谁」和「怎么调用」分开。`OpenFeign` 只管怎么把接口变成请求，不决定目标机器；`LoadBalancer` 只管选机器，不构造请求；`Sentinel` 的 `@SentinelResource` 只管这次调用要不要被放行，不参与选机器。三件事分别落在第 5、3、7 章，任何一个组件想顺手把另外两件也做了，故障边界就会糊掉。旧卷已经把动态代理与舱壁隔离的机制讲透，这里只写 Alibaba 栈里它们各自的落点。

**拆解**：
### 1. 一个接口怎么变成请求
`@FeignClient` 标注的接口在启动时被生成代理，方法上的路径注解被翻译成 HTTP 请求，服务名交给 `LoadBalancer` 解析成具体地址。原书第 6 章用的是 `server-user`、`server-product`、`server-order` 三个名字，第 7 章把硬编码地址换成注册发现，第 8 章再补上多实例的分担。这条演进链本身就说明了三层各自的必要性。

### 2. 两种写法，同一个选择点
`@LoadBalanced` 修饰的 `RestTemplate` 与 `OpenFeign` 是两种常见写法，区别在声明方式，不在负载均衡发生在哪一侧——都在客户端。`Ribbon` 是旧卷里的那套实现，`LoadBalancer` 是新栈里承接同样职责的接口。这里不复述 `Ribbon` 的源码，只记一条：只要 URL 里写的是服务名而不是 IP，就一定有一层在做实例选择。

### 3. 兜底不是重试
`fallback`、`blockHandler` 与 `fallback` 所属的接口代理，处理的是「这次调用没成功时返回什么」；它们不改变目标实例是否健康，也不会让超时自动消失。原书第 10 章同时提醒 `fallback` 与 `fallbackFactory` 不能同时使用。把兜底当重试，会掩盖第三章里 `健康检查` 本该暴露的问题，也会让第八章里 `DegradeRule` 的统计失真。

### 4. Dubbo 那条接缝
`Dubbo` 与 `@DubboReference` 走 RPC 而非 HTTP 声明式接口，官方生态里由相应的适配把它接进 Spring Cloud 的注册发现体系。本册不复述 `Dubbo` 的 SPI、Filter 链与协议头——那是《Dubbo3源码精读》的内容；这里只保留一条判断：无论走哪种调用方式，`服务发现` 给出的那份实例清单都是共同前提。

**今天怎么做**：找一处 `@FeignClient` 或 `@LoadBalanced` 的 `RestTemplate`，把服务名换成 IP 试一次，观察哪些环节立刻失效；再恢复服务名，回答「实例列表由谁给出、选择规则由谁执行、失败后由谁兜底」。三问答齐，这一章就过了。

<div class="chapend" markdown="1">

**容易误解的地方**：`负载均衡` 不是注册中心的功能，它发生在调用方；`@FeignClient` 里的值也不是主机名，是 `服务注册` 时用的服务名。另一个误读是把 `fallback` 当成限流——限流由 `FlowRule` 决定，兜底只是决定被拒或被熔断之后返回什么。

**记住这一句**：接口负责怎么调，发现负责调谁，兜底负责调不成时怎么办。

</div>

<!-- chapend -->
