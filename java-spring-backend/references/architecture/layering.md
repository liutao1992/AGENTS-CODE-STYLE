# 应用分层规范

本文档定义应用分层、职责边界、模型分类、Package 归属和 SOLID 设计判断。

本文负责回答：

> 一个类是什么职责、属于哪个架构边界、应该放在哪个 Package。

本文不重复 Java 实现、Spring 注解、HTTP 契约、异常实现、SQL、事务和并发细节。

相关规范：

- [Java 编码](../coding/java.md)
- [Spring](../coding/spring.md)
- [API 设计](../api/api-design.md)
- [MyBatis](../coding/mybatis.md)
- [异常处理与错误边界](error-handling.md)
- [事务](transactions.md)
- [并发](concurrency.md)

---

# 1. 默认分层

项目默认按以下逻辑边界理解：

```text
                    入站适配器
       ┌──────────────┼──────────────┐
 Controller / Web   RPC / Open API   Consumer / Scheduled Task
       └──────────────┼──────────────┘
                      ↓
                   Service
                      ↓
                 Manager（按需）
                 ↙            ↘
          Mapper / DAO      Client / Adapter
               ↓                  ↓
            Database      第三方服务 / 外部系统
                    出站适配器
```

简单理解：

```text
入站适配器
→ 接收外部请求或触发，转换为应用调用

Service
→ 业务用例和业务流程

Manager
→ 可复用的应用能力、组合操作和原子操作

Mapper / DAO
→ 数据库访问

Client / Adapter
→ 第三方协议和技术细节适配
```

原则：

> 入站适配器负责“外部如何进入应用”，Service 负责“业务要做什么”，出站适配器负责“应用如何访问外部资源”。

> 上层可以依赖下层或稳定抽象，下层不得反向依赖上层。

Manager 为可选层，不得为了分层形式机械创建。

上图表达的是逻辑职责，不要求每个项目都创建：

```text
openapi
rpc
consumer
adapter
client
integration
```

等物理 Package。具体目录优先遵循目标项目已有结构。

---

## 1.1 入站适配器

常见入站形式：

```text
HTTP Controller
Open API Endpoint
RPC Endpoint
Message Consumer
Scheduled Task
Command / Job Handler
```

共同职责：

* 接收外部输入或触发；
* 完成协议相关解析和基础校验；
* 获取调用上下文；
* 调用 Service / Facade；
* 转换为当前协议需要的返回或确认结果。

不同入站适配器优先复用业务能力，不互相调用。

推荐：

```text
HTTP Controller ───┐
RPC Endpoint ──────┼→ PlaceService
Message Consumer ──┘
```

避免：

```text
RPC Endpoint → HTTP Controller → Service
Consumer     → Controller
```

原则：

> 复用业务能力应复用 Service / Facade，不复用另一个协议适配器。

---

## 1.2 出站适配器

常见出站形式：

```text
Mapper / DAO
HTTP Client
RPC Client
SDK Adapter
Object Storage Client
Message Producer
External Data Client
```

Mapper / DAO 只是数据库访问边界，不代表所有下游能力都应该实现成 DAO。

例如：

```text
Manager
  ├── PlaceMapper        → Database
  ├── FaceClient         → Face Recognition Service
  ├── StorageClient      → Object Storage
  └── NotificationClient → External Message Service
```

禁止把第三方 HTTP、RPC、对象存储 SDK、消息发送为了“统一下层结构”机械塞入 Mapper / DAO。

Client / Adapter / Gateway / Integration 的具体命名以目标项目现有约定为准。

---

## 1.3 业务核心不感知协议细节

Service 不应直接理解：

```text
HttpServletRequest / ResponseEntity
RPC 框架对象
消息中间件 Record / Message
第三方 SDK Request / Response
第三方 SDK Exception
数据库拼音字段
```

这些协议或技术对象应在对应适配边界完成转换。

原则：

> 业务层依赖业务语义，不依赖某个 Web、RPC、MQ、数据库或供应商协议的具体对象。

---

# 2. SOLID 设计原则

SOLID 用于判断职责、依赖和扩展边界，不用于机械增加接口、实现类、设计模式或中间层。

核心原则：

> 先保持职责清晰和依赖合理；只有真实变化、替换、隔离或扩展需求出现时，再增加必要抽象。

## 2.1 SRP — 单一职责

类、接口或模块应围绕一个明确职责设计，并尽量只有一个主要变化原因。

例如 `PlaceService` 不应同时承担：

```text
业务流程
+ HTTP 响应构造
+ SQL
+ 第三方协议解析
+ 数据库字段转换
```

但 SRP 不等于“一方法一类”或“代码稍多就拆 Manager”。

> 按变化原因和职责边界拆分，不按代码行数或方法数量机械拆分。

---

## 2.2 OCP — 开闭原则

当代码已经存在明确、稳定、持续增加的变化方向时，可以通过 Strategy、Handler、Factory、模板方法或 Spring Bean 集合等建立扩展点。

只有存在真实变化需求时才引入扩展机制。

禁止因为“以后可能扩展”提前创建大量 Strategy / Factory / AbstractFactory / Interface。

> 为真实变化建立扩展点，不为猜测中的未来变化提前抽象。

---

## 2.3 LSP — 里氏替换

实现类替换抽象类型时不得破坏调用方合理预期，包括：

* 输入语义；
* 返回语义；
* Null 约定；
* 异常语义；
* 状态变化；
* 副作用。

如果实现对抽象类型核心能力只能抛 `UnsupportedOperationException`，应重新检查抽象关系。

---

## 2.4 ISP — 接口隔离

接口应围绕真实调用边界设计，避免让调用方或实现方依赖大量无关能力。

不要根据方法数量机械拆接口。

> 按真实消费者和实现者边界隔离，不按形式拆分。

---

## 2.5 DIP — 依赖倒置

高层业务流程不应直接耦合容易变化的底层技术细节，例如：

* 第三方 SDK；
* 外部 HTTP / RPC；
* 对象存储；
* 消息系统；
* 多供应商实现；
* 可替换算法。

存在真实替换、隔离或测试需求时，通过稳定 Client / Adapter / SPI 等边界隔离。

DIP 不意味着：

```text
所有 Service → ServiceImpl
所有 Mapper → Repository → RepositoryImpl
```

Mapper 接口通常已经构成数据访问边界，不需要仅为 DIP 再包装一层 Repository。

---

## 2.6 SOLID 与简单设计

SOLID 既用于识别设计风险，也用于识别错误抽象。

禁止机械推导：

```text
SOLID
→ 所有 Service 创建接口
→ 所有业务创建 Strategy / Factory
→ 所有 Mapper 再包装 Repository
```

最终原则：

> SOLID 用来降低真实复杂度，不用来制造新的复杂度。

---

# 3. Controller / Web 层

Controller 属于 HTTP 入站适配器。

主要职责：

* HTTP 参数绑定；
* 基础参数校验；
* 获取请求上下文；
* 调用 Service；
* 返回项目统一 HTTP 响应。

禁止：

* 直接调用 Mapper / DAO；
* 编写 SQL；
* 定义业务事务；
* 承载业务状态流转；
* 直接操作数据库对象完成业务流程。

HTTP 注解、Bean Validation、统一异常处理等 Spring 机制读取：

- [spring.md](../coding/spring.md)

URL、Method、响应包装和兼容契约读取：

- [api-design.md](../api/api-design.md)

原则：

> Controller 保持轻量，只表达 HTTP 入站边界。

---

# 4. Service 层

Service 负责业务用例和业务流程。

主要职责：

* 实现业务用例；
* 执行业务校验；
* 编排 Manager 或稳定出站能力；
* 简单场景下直接调用 Mapper / Client；
* 组织业务输入和输出；
* 协调多个业务能力。

Service 不负责：

* HTTP 状态和响应协议；
* SQL；
* 数据库字段映射；
* 第三方协议对象和 SDK 细节。

方法名优先表达明确业务行为，例如：

```text
audit
register
approve
reject
bindEquipment
```

避免含义模糊的 `handle`、`process`、`doSomething`。

原则：

> Service 表达业务用例，不成为 HTTP、SQL 和供应商技术细节的混合层。

---

# 5. Manager 层

Manager 为可选的应用能力层。

适用于：

* 多个 Mapper / Client 的组合操作；
* 可复用数据操作；
* 多表原子操作；
* 多个外部能力的组合；
* 缓存与数据访问的应用级组合；
* 需要复用的复杂数据组装。

Manager 的重点是**应用级复用和编排**，不是直接承担第三方协议细节。

典型：

```text
PlaceService
    ↓
FaceRecognitionManager
    ↓
FaceRecognitionClient
    ↓
Vendor HTTP / SDK
```

简单场景允许：

```text
PlaceService
    ↓
FaceRecognitionClient
```

不要为了“Service 不能调用 Client”机械创建 Manager。

原则：

> Client / Adapter 负责技术适配，Manager 负责有真实价值的应用级复用、组合和原子能力。

---

# 6. Mapper / DAO 层

Mapper / DAO 负责数据库访问：

* SELECT；
* INSERT；
* UPDATE；
* DELETE；
* ResultMap；
* SQL 执行。

Mapper 不负责：

* 权限业务判断；
* 业务状态流转；
* 完整业务决策；
* HTTP / RPC 处理；
* 第三方服务调用；
* 业务事务编排。

MyBatis 具体使用读取：

- [mybatis.md](../coding/mybatis.md)

SQL 规则读取：

- [sql.md](../database/sql.md)

---

## 6.1 Client / Adapter

Client / Adapter 负责与数据库之外的外部技术系统交互，例如：

```text
HTTP / RPC
第三方 SDK
对象存储
消息系统
外部数据服务
```

主要职责：

* 协议调用；
* 供应商 Request / Response 转换；
* 认证签名等技术要求；
* 技术错误转换；
* 屏蔽供应商字段和 SDK 类型。

Client / Adapter 不承担完整业务流程，也不因为只被一个模块使用就自动变成 Manager。

---

# 7. 分层调用与依赖规则

简单业务允许：

```text
Controller
    ↓
Service
    ↓
Mapper / Client
```

存在真实复用、组合或原子能力时：

```text
Controller
    ↓
Service
    ↓
Manager
   ↙     ↘
Mapper  Client
```

禁止：

```text
Controller → Mapper
Mapper     → Service
Client     → Service
Manager    → Controller
RPC Endpoint → Controller
Consumer     → Controller
```

原则：

> 依赖方向保持单向，职责边界比调用方便更重要。

---

## 7.1 分层异常处理规约

本文只定义异常处理属于跨层边界问题，不重复具体实现。

总体方向：

```text
底层技术异常
    ↓ 必要时隔离
应用 / 业务边界
    ↓ 补充业务上下文
Web / API 边界
    ↓ 转换为安全稳定的错误契约
```

详细规则统一读取：

- [异常处理与错误边界](error-handling.md)

Java `catch`、`throw`、日志写法读取 `java.md`；Spring Web 收口机制读取 `spring.md`；错误码和 HTTP 响应读取 `api-design.md`。

---

# 8. 跨模块调用

跨业务模块优先调用目标模块提供的 Service / Facade。

推荐：

```text
CaseService
    ↓
PlaceService / PlaceFacade
```

避免：

```text
CaseService
    ↓
PlaceMapper
```

直接访问其他模块 Mapper 会绕过业务规则并耦合数据库实现。

但不要为了形式机械创建 Facade；已有 Service 能稳定表达跨模块能力时直接复用。

---

# 9. 模型分类与 Package 归属

模型职责与 Package 统一在本文定义。

默认模型体系：

```text
Request → 接口输入          → <module>.request
Query   → 查询条件          → <module>.query
DTO     → 应用内部数据传输  → <module>.dto
BO      → 业务处理中间对象  → <module>.bo
DO      → 持久化数据        → <module>.domain
VO      → 具体业务视图输出  → <module>.vo
```

不同模型职责不同，不得因为“都是保存字段的 Java 类”就统一放入 `dto`。

Java 的 Lombok、`class` / `record`、Getter / Setter 等实现方式读取：

- [java.md](../coding/java.md)

---

## 9.1 Request

Request 表达接口输入，例如：

```text
PlaceCreateRequest
PlaceUpdateRequest
PlaceAuditRequest
```

默认：

```text
<module>.request
```

Request 可以承载接口结构性校验，不承担数据库持久化、查询结果或视图输出职责。

---

## 9.2 Query

Query 表达查询条件，例如：

```text
PlaceQuery
PersonQuery
CaseQuery
```

默认：

```text
<module>.query
```

Query 可以在 Controller → Service → Mapper 之间传递查询条件。

Query 不因为 Mapper 使用就属于 `mapper`，也不机械归入 `dto`。

---

## 9.3 DTO

DTO 表达应用内部真实存在的数据传输，例如 Service 与 Manager、模块应用接口之间的数据交换。

默认：

```text
<module>.dto
```

DTO 不是所有数据对象的兜底目录。没有真实 DTO 职责时，该 Package 可以不存在。

---

## 9.4 BO

BO 表达业务处理过程中确实需要独立存在的中间业务对象。

默认：

```text
<module>.bo
```

禁止机械：

```text
Request → DTO → BO → DO → BO → VO
```

简单业务允许：

```text
Request → Service → DO → VO
```

---

## 9.5 DO

DO 表达数据库持久化模型。

默认：

```text
<module>.domain
```

Java DO 使用英文业务语义。数据库物理命名和 Java 映射规则由数据库设计与 MyBatis 规范维护：

- [database-design.md](../database/database-design.md)
- [mybatis.md](../coding/mybatis.md)

DO 不要求机械复制数据库表名。

---

## 9.6 VO

VO 表达提供给视图层或客户端的具体业务输出数据，例如：

```text
PlaceVO
PlaceStatsVO
PlaceTreeNodeVO
```

默认：

```text
<module>.vo
```

具体业务输出优先使用 VO，不机械新增：

```text
PlaceResponse
PlaceStatsResponse
place.response.*
```

但不得为了统一命名无授权重命名已发布历史 API 模型。

统一 HTTP 响应包装属于 API 契约，不属于业务模型分类；具体规则读取：

- [api-design.md](../api/api-design.md)

---

## 9.7 通用模型

只有真正与具体业务模块无关、能够跨模块稳定复用的结构才进入公共区域。

具体业务模型即使多个 Controller 使用，也仍属于对应业务模块。

公共 Package 名称和统一响应类型以目标项目已有约定为准。

---

## 9.8 模型归属判断

创建模型前先问：

```text
接口输入？        → Request
查询条件？        → Query
内部数据传输？    → DTO
业务处理中间对象？→ BO
数据库持久化？    → DO
具体业务输出？    → VO
```

Package 由真实职责决定，不由当前任务所在目录决定。

---

## 9.9 创建模型前必须搜索

新增模型前必须先搜索：

1. 是否已有相同或类似模型；
2. 当前模块已有模型 Package；
3. 是否可以复用；
4. 当前对象真实职责；
5. 是否真的需要新的模型类型。

流程：

```text
确定职责
→ 搜索已有模型
→ 判断是否需要新增
→ 确定模型类型与 Package
→ 读取 java.md 确定 Java 实现
→ 创建文件
```

---

## 9.10 模型转换原则

只有职责或边界真实发生变化时才转换模型。

不为了少写转换代码破坏模型边界，也不为了形式完整增加无意义 DTO / BO / Converter / Assembler。

---

# 10. 技术基础设施的 Package 归属

Package 由组件自身职责决定，不由当前使用它的业务模块决定。

例如通用 MyBatis 技术组件可以按项目约定归入：

```text
TypeHandler   → common.mybatis.handler
Interceptor   → common.mybatis.interceptor
Plugin        → common.mybatis.plugin
Configuration → common.mybatis.config
```

因此 `JsonStringListTypeHandler` 即使当前只被 `PlaceMapper` 使用，也不应仅因此放入 `place.mapper`。

原则：

> “当前模块需要这个类”不等于“这个类属于当前模块”。

具体技术组件的 Package 仍以对应领域规范和目标项目已有结构为准。

---

# 11. 事务与并发边界

事务边界由一致性需求决定，不由层级名称机械决定。

本文只保留分层判断：

```text
可复用原子数据操作
→ Manager 可以成为事务边界

跨多个 Manager 的完整业务写入需要整体提交/回滚
→ Service 可以成为事务边界
```

最终仍以真实一致性范围为准。

详细规则：

- [transactions.md](transactions.md)
- [concurrency.md](concurrency.md)

---

# 12. 不要过度分层

简单业务优先：

```text
Controller → Service → Mapper / Client
```

存在真实复用、组合、原子性或技术隔离需求时再引入 Manager、Adapter、Facade 等边界。

不要为了架构形式机械增加：

```text
Manager
Domain Service
Repository / RepositoryImpl
Converter
Assembler
Factory
ServiceInterface / ServiceImpl
Strategy
Adapter
DTO / BO
```

原则：

> 先保持简单，真实复杂度出现后再增加对应抽象。

---

# 13. Codex 编码检查

编码前：

1. 查看当前模块已有分层和调用关系。
2. 搜索至少一个类似实现。
3. 判断属于入站适配、Service、Manager、Mapper、Client / Adapter、模型还是技术基础设施。
4. 新增类先确定职责，再确定 Package。
5. 新增模型按本文判断 Request / Query / DTO / BO / DO / VO，再读取 Java 规范确定实现方式。
6. 第三方集成先区分技术适配和应用编排：协议细节属于 Client / Adapter，真实复用组合才考虑 Manager。
7. 涉及异常读取 `error-handling.md`。
8. 涉及事务读取 `transactions.md`。
9. 涉及并发读取 `concurrency.md`。

编码后检查：

* 入站适配器是否直接访问 Mapper；
* Controller / RPC Endpoint / Consumer 是否互相调用而不是复用 Service；
* Service 是否混入 HTTP、SQL、第三方 SDK 类型或协议细节；
* Manager 是否只是为了包一层 Client / Mapper 而存在；
* Client / Adapter 是否承担完整业务流程；
* Mapper 是否包含业务判断或第三方调用；
* 是否跨模块直接访问其他模块 Mapper；
* Request / Query / DTO / BO / DO / VO 是否按真实职责归类；
* Package 是否由职责决定；
* 具体业务输出是否错误新增为 `*Response` / `response`；
* 是否把所有模型机械放入 `dto`；
* 技术基础设施是否错误放入业务 Package；
* 一个类是否同时承担多个明显不同职责；
* 新增扩展点是否来自真实变化需求；
* 实现是否破坏抽象契约；
* 接口是否迫使调用方依赖无关能力；
* 高层业务是否直接耦合易变技术细节；
* 是否以 SOLID 为理由机械增加接口、Strategy、Factory 或 Repository 包装。

最终原则：

> 先确定职责，再确定 Package；分层规范负责“类是什么、放哪里”，专项规范负责“具体怎么实现”。
