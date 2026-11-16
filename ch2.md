# 第2章 工程起点：空目录里先立起什么

**一句话**：Alibaba 栈的所有能力都要挂在「这个服务叫什么、去哪儿取配置」之上，所以第一步不是接组件，而是用 `Maven` 的 `父工程` 与 `bootstrap.yml` 把引导阶段立起来。

**为什么需要它**：本地原书第 4、5 章从「项目准备」和「完成通用模块的开发」开始，先建 `父工程`、再拆工具类与实体类模块、再建数据表，第 6 章才搭出用户、商品、订单三个微服务。这个顺序不是教程的啰嗦，而是因为 `Nacos Config` 要在应用启动的引导阶段就把配置取回来；等到 `application.yml` 被读取时，`spring.cloud.nacos.config` 已经必须就位。很多人第一次接 `Nacos` 时遇到的「配了不生效」，根因常常不是配置写错，而是写在了错误的阶段或错误的作用域里。

**核心结构**：三层文件各管一段。`pom.xml` 在 `父工程` 里统一版本与依赖，`模块划分` 决定哪些能力是公共的；`bootstrap.yml`（或 `bootstrap.properties`）放引导期必需的内容：`spring.application.name`、`spring.profiles.active`、`spring.cloud.nacos.config` 的地址与命名空间；`application.yml` 放业务自己的参数。`@SpringBootApplication` 标出启动类，`Nacos Config` 在引导阶段把远端配置合进环境，之后 `@Value` 与 `@RefreshScope` 才有值可读。综合报告口径把这条归纳为「引导配置必须最先就位、配置有覆盖优先级、刷新有作用域」，三条都来自旧卷已讲过的机制，这里只补 Alibaba 栈的落点。

**关键设计**：把「启动期」与「运行期」切开。启动期只需要三件事：服务名、环境、配置源地址；其余全部交给 `配置中心` 下发。这样 `namespace` 与 `group` 的切换就不需要改代码，只需要换引导配置。反过来，如果把该引导期就位的项塞进 `application.yml`，`NacosPropertySource` 就没机会在 Bean 创建之前进入环境，`@Value` 取到的会是空值或者本地默认值。原书第 21、22 章把 `Nacos` 核心概念与项目整合分成两章讲，正是这个先后顺序。

**拆解**：
### 1. 父工程先统一版本
`Maven` 的 `父工程` 里锁定 Spring Boot、Spring Cloud 与 `spring-cloud-alibaba` 三者的配套版本，子模块只声明依赖不写版本。原书在「技术选型」与「模块划分」两节里把这件事放在建表之前。版本不配套时，症状往往不是编译失败，而是某个 starter 的自动配置类压根没被加载——这类问题在旧卷里已经作为「版本矩阵」问题讲过，这里不再复述，只提醒 Alibaba 栈同样吃这一条。

### 2. 引导配置必须在 bootstrap 阶段
`bootstrap.yml` 在 `application.yml` 之前加载，`spring.application.name` 与 `spring.profiles.active` 决定最终拼出哪个 `dataId`，`spring.cloud.nacos.config` 决定去哪台 `Nacos` 取。`NacosPropertySource` 进入环境之后，业务里的 `@Value` 才能解析出远端值。写错阶段的典型症状是：控制台里配置明明改了，应用日志里却还是本地默认值，而且没有任何报错。

### 3. 模块划分决定公共能力放在哪一层
原书把工具类、实体类各自拆成模块，再让三个微服务依赖它们。这不只是目录好看：`@SentinelResource` 的 `blockHandler`、`Seata` 的 `undo_log` 表结构、链路追踪的 `TraceID` 透传，都是跨服务复用的东西，放在公共模块里才能保证七道闸在每个服务上装的是同一套。公共层少放一样，排查时就会多一个「为什么只有这个服务没有」的分支。

### 4. 配置有覆盖优先级，刷新有作用域
远端配置与本地配置同时存在时按优先级覆盖，`@RefreshScope` 只刷新被它标注的 Bean 的作用域，不是把整个应用重启一遍。`@NacosValue` 与 `@Value` 是两种读取方式，`配置监听` 负责感知变化，`配置推送` 负责把变化送到实例。把这四件事混成「配置自动生效」一句话，就会在灰度发布时搞不清到底哪批实例拿到了新值。

**今天怎么做**：打开任一服务的 `bootstrap.yml`，逐项回答三个问题：服务名是什么、环境是什么、配置从哪里取。再打开 `application.yml`，确认里面没有本该在引导阶段就位的项。三问答不出来，先别急着接流控与事务。

<div class="chapend" markdown="1">

**容易误解的地方**：`bootstrap.yml` 不是「更早就读的 application.yml」，它是另一套引导上下文；`@RefreshScope` 也不是万能刷新，未标注的 Bean 仍可能持有旧值。另一个误读是把 `模块划分` 当成代码整洁问题——在 Alibaba 栈里它直接决定七道闸能不能一致地装到每个服务上。

**记住这一句**：组件接不上，先查引导阶段；配置不生效，先查作用域。

</div>

<!-- chapend -->
