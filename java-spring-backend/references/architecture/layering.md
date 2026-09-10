# 应用分层规范

本文档定义应用逻辑分层、职责边界、依赖方向、模型分类和**职责 Package**。

本文负责回答：

> 一个类在应用中是什么职责、应该依赖谁、属于哪个职责 Package。

项目 / module 的**物理位置与目录组织**读取：

- [项目与业务模块目录](project-structure.md)

Java 实现、Spring 注解、HTTP 契约、异常、SQL、事务和并发由对应专项 reference 维护，本文不重复实现细节。

核心原则：

> 先确定职责，再确定 Package；隔离易变协议和技术细节，不机械增加调用层级。

---

# 1. 默认逻辑分层

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

职责简化为：

```text
入站适配器
→ 外部如何进入应用

Service
→ 当前业务用例要做什么

Manager（可选）
→ 可复用应用能力、组合操作、原子操作

Mapper / DAO
→ 如何访问数据库

Client / Adapter
→ 如何访问和适配外部技术系统
```

Manager 为可选层。简单业务允许：

```text
Controller → Service → Mapper
```

或：

```text
Controller → Service → Client
```

不得为了“完整分层”机械创建 Manager。

原则：

> 上层可以依赖下层或稳定抽象，下层不得反向依赖上层；层级数量由真实职责决定。

---

## 1.1 入站适配器

常见入站形式：

```text
HTTP Controller
RPC Endpoint
Open API Endpoint
Message Consumer
Scheduled Task
Command / Job Handler
```

共同职责：

* 接收外部输入或触发；
* 解析协议；
* 完成当前协议入口的结构校验；
* 获取可信调用上下文；
* 转换为应用调用；
* 调用 Service / Facade；
* 转换为当前协议需要的输出或确认结果。

不同协议适配器不互相调用来复用业务逻辑。

推荐：

```text
HTTP Controller ───┐
RPC Endpoint ──────┼→ PlaceService
Consumer ──────────┘
```

避免：

```text
RPC Endpoint → HTTP Controller → Service
Consumer → Controller
```

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

数据库访问和外部技术调用都是出站边界，但职责不同：

```text
Mapper / DAO
→ Database

Client / Adapter
→ External System / Vendor Protocol
```

不要为了“统一下层”把 HTTP、RPC、SDK、对象存储、消息发送机械塞入 Mapper / DAO。

---

## 1.3 业务核心不感知协议细节

Service / Manager 原则上不直接依赖：

```text
HttpServletRequest / HttpServletResponse / ResponseEntity
RPC 框架 Request / Context
消息中间件 Record / Message
第三方 SDK Request / Response / Exception
数据库物理字段命名
```

协议与供应商类型在对应入站 / 出站边界转换为应用能够理解的稳定语义。

---

# 2. SOLID 与简单设计

SOLID 用于判断真实职责、替换、扩展和依赖问题，不用于机械增加设计模式。

## 2.1 SRP

一个类应围绕一个主要职责和变化原因。

例如 `PlaceService` 不应同时承担：

```text
业务流程
+ HTTP 响应构造
+ SQL
+ 第三方 SDK 协议解析
```

但 SRP 不等于“一方法一类”或“代码稍多就拆 Manager”。

## 2.2 OCP

只有真实、稳定、持续存在的变化方向出现时才建立 Strategy、Handler、Factory 等扩展点。

不要为猜测中的未来变化提前抽象。

## 2.3 LSP

实现不得破坏抽象类型的输入、返回、Null、异常、副作用和状态变化契约。

## 2.4 ISP

接口按真实消费者 / 实现者边界隔离，不按方法数量机械拆分。

## 2.5 DIP

高层业务不应直接耦合易变供应商和技术协议。真实存在替换、隔离或测试需求时，通过 Client / Adapter / SPI 等稳定边界隔离。

DIP 不意味着：

```text
所有 Service → ServiceImpl
所有 Mapper → Repository → RepositoryImpl
```

## 2.6 过度设计

没有真实替换、扩展、复用或隔离需求时，不机械创建：

```text
Interface + Impl
Strategy
Factory
Repository 包装
Adapter
Facade
Manager
```

原则：

> SOLID 用来降低真实复杂度，不用来制造新的复杂度。

---

# 3. Controller / Web 层

Controller 是 HTTP 入站适配器。

主要职责：

* HTTP 参数绑定；
* 结构性参数校验；
* 获取请求相关调用者上下文；
* 调用 Service；
* 完成必要协议转换。

禁止：

* Controller → Mapper / DAO；
* 编写 SQL；
* 定义业务事务；
* 承载业务状态流转；
* 在 Controller 做跨数据源复杂业务拼装；
* 直接使用数据库 DO 完成业务流程。

当前用户、部门、租户、数据权限等如果来自 Web Request、SecurityContext 或请求 ThreadLocal，优先在入站边界取得，并按业务需要显式传递职责明确的：

```text
Operator
CallerContext
项目已有统一调用上下文
```

不要让 Service / Manager 为获取“当前请求用户”反向依赖 Web 对象。

如果项目已有能够安全覆盖多入口的统一上下文机制，沿用现有机制，不创建第二套 Context。

Spring MVC 用法读取 `spring.md`；URL、Method、Request、VO、统一响应读取 `api-design.md`。

原则：

> Controller 只做协议边界必须做的事情；业务决策交给 Service。

---

# 4. Service 层

Service 负责业务用例和业务流程。

主要职责：

* 实现业务用例；
* 执行业务校验；
* 协调多个业务能力；
* 编排 Manager 或稳定出站能力；
* 简单场景下直接调用 Mapper / Client；
* 组织业务输入和输出。

Service 不负责：

* HTTP Status / HTTP 响应协议；
* SQL；
* 数据库字段映射；
* 第三方 SDK 协议细节。

方法名优先表达明确业务行为：

```text
audit
register
approve
reject
bindEquipment
```

避免长期使用无语义：

```text
handle
process
doSomething
```

## 4.1 Service 方法命名

对于普通 CRUD / 查询型业务能力，在目标项目没有更具体稳定约定时，优先使用以下前缀：

```text
获取单个对象 → get
获取多个对象 → list
获取统计数量 → count
新增 / 保存   → save
删除         → remove
修改         → update
```

例如：

```java
PlaceVO getPlace(String id);

List<PlaceVO> listPlaces(PlaceQuery query);

long countPlaces(PlaceQuery query);

void savePlace(PlaceSaveRequest request);

void removePlace(String id);

void updatePlace(PlaceUpdateRequest request);
```

集合方法使用 `list` 前缀。直接表达资源集合时优先使用复数名词，例如：

```text
listPlaces
listCases
listEquipmentItems
```

如果方法重点在查询条件或筛选语义，可以使用：

```text
listByStatus
listByQuery
listAvailablePlaces
```

不要为了满足“复数结尾”而牺牲更明确的业务语义。

`save` 表达 Service 层的新增 / 保存业务动作；如果新增和修改具有不同业务语义，应分别使用职责更明确的方法，不把所有写操作都模糊成 `save`。

真实业务动作优先于 CRUD 模板。例如：

```text
auditPlace
approveCase
rejectCase
registerCase
bindEquipment
```

这些名称已经准确表达业务用例时，不应为了统一前缀机械改成：

```text
updatePlace
saveCase
```

Mapper / DAO 的数据访问方法命名由 `mybatis.md` 维护，Service 不为了与数据库操作一一对应而使用 `insert` / `delete` 等持久化术语。

原则：

> CRUD 型 Service 使用稳定前缀降低理解成本；存在明确业务动作时优先表达业务语义，不让命名模板覆盖真实用例。

## 4.2 Service 拆分

Service 变大只是信号。只有出现能够独立命名、独立变化的业务用例时才按业务能力拆分，例如：

```text
OrderQueryService
OrderCreateService
OrderDeliveryService
```

不要仅因行数 / 方法数创建：

```text
OrderHelperService
OrderCommonService
OrderValidatorService
```

除非它们确实具有独立稳定职责。

原则：

> Service 按业务用例和变化原因拆，不按文件长度机械拆。

---

# 5. Manager 层

Manager 是**可选应用能力层**。

适合：

* 多个 Mapper / Client 的有意义组合；
* 可复用数据操作；
* 多表原子操作；
* 缓存 + 数据访问的应用级组合；
* 多个外部能力的应用级组合；
* 多个 Service 用例都需要的复杂数据组装。

Manager 的重点是应用级复用、组合和原子能力，不是第三方协议适配。

例如：

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
PlaceService → FaceRecognitionClient
```

不要为了“Service 不能调用 Client”创建纯转发 Manager。

事务是否在 Manager 由一致性范围决定，读取 `transactions.md`；不要反过来因为想加事务才创造 Manager。

原则：

> Manager 因真实应用能力存在而存在，不因层级形式或注解存在而存在。

---

# 6. Mapper / DAO 层

Mapper / DAO 是数据库出站适配边界，负责：

* SELECT；
* INSERT；
* UPDATE；
* DELETE；
* 参数与结果映射；
* SQL 执行。

不负责：

* 权限业务判断；
* 业务状态流转；
* 完整业务流程；
* 第三方服务调用；
* 业务事务编排。

MyBatis 具体规则读取 `mybatis.md`；SQL 本身读取 `sql.md`。

---

# 7. Client / Adapter 层

Client / Adapter 是外部技术系统的出站适配边界，与 Mapper / DAO 平级，不属于 Mapper 的子层。

负责：

* HTTP / RPC / SDK 调用；
* Vendor 认证和协议参数；
* Vendor Request / Response 转换；
* 外部错误码和异常隔离；
* 外部 nullable / 特殊值等协议差异归一化；
* 超时、连接、序列化等当前集成需要的技术细节。

不负责：

* 当前应用业务用例编排；
* HTTP Controller 响应；
* 多个业务状态流转；
* 为了减少 Service 代码而承接无关业务判断。

命名可以根据项目已有约定使用：

```text
Client
Adapter
Gateway
Integration
```

不要机械把一种命名迁移成另一种。

原则：

> Client / Adapter 隔离易变技术协议；Manager 组合应用能力；Service 表达业务用例。

---

# 8. 调用与依赖规则

默认推荐：

```text
入站适配器 → Service
Service → Manager / Mapper / Client
Manager → Mapper / Client
Mapper → Database
Client / Adapter → External System
```

禁止：

```text
Controller → Mapper
Mapper → Service
Client → Service
下层 → Controller
```

简单场景允许跳过可选层，但不得穿透不应暴露的技术细节。

例如：

```text
Service → Mapper
```

可以；

```text
Controller → Mapper
```

不可以。

---

# 9. 模型分类与 Package 归属

模型按职责分类，不把所有数据统一命名为 DTO。

默认模型体系：

```text
Request → 外部接口操作输入
Query   → 查询条件和过滤语义
DTO     → 应用内部数据传输
BO      → 业务处理中的中间结果 / 组合语义
DO      → 数据库持久化模型
VO      → 具体业务接口 / 视图输出
```

模型职责和 Package 不要求一一对应。`Request` 与 `Query` 语义不同，但都属于入站请求模型，默认共享 `request` Package：

```text
Request → <module>.request
Query   → <module>.request
DTO     → <module>.dto
BO      → <module>.bo
DO      → <module>.domain
VO      → <module>.vo
```

例如：

```text
module.place.request.PlaceSaveRequest
module.place.request.PlaceAuditRequest
module.place.request.PlaceQuery
module.place.dto.PlaceDTO
module.place.bo.PlaceAuditBO
module.place.domain.PlaceDO
module.place.vo.PlaceDetailVO
```

不要因为模型类型叫 `Query` 就机械创建：

```text
module.place.query
```

模型职责与“项目物理目录”仍然是两件事：

```text
业务模块位置
→ project-structure.md

模型语义与职责 Package
→ 本文
```

目标项目已有清晰且稳定的 Package 结构时优先沿用，不为了本默认批量迁移历史代码。

---

## 9.1 Request

Request 表达外部调用者提交的操作型接口输入。

例如：

```text
PlaceSaveRequest
PlaceAuditRequest
```

默认放入：

```text
<module>.request
```

Request 不应承载客户端无法可信提供的服务端身份信息，例如当前 Operator / Tenant 权限上下文。

---

## 9.2 Query

Query 表达查询条件和过滤语义。

例如：

```text
PlaceQuery
CaseQuery
```

Query 在模型语义上仍然独立于普通 Request，但默认与 Request 一起放入：

```text
<module>.request
```

通过 `*Query` 类名表达其查询职责，不单独建立 `query` Package。

普通查询条件增多时优先使用 Query，而不是无界方法参数或 `Map<String, Object>`。

---

## 9.3 DTO

DTO 用于应用内部明确的数据传输边界。

适合：

* 多层之间传递一组稳定数据；
* 内部能力输入 / 输出与 Request、DO、VO 均不等价；
* 一个操作形成明确内部数据契约。

不要因为“不知道叫什么”就统一使用 DTO。

---

## 9.4 BO

BO 表达业务处理中有独立语义的中间结果、计算结果或组合对象。

只有真实业务处理中间语义存在时创建，不机械为每个 Service 方法建立 BO。

---

## 9.5 DO

DO 表达数据库持久化结构。

DO 字段使用 Java 英文业务语义；数据库物理字段可通过 MyBatis 显式映射。

DO 不直接作为外部 API 输出。

---

## 9.6 VO

VO 表达具体业务接口或视图输出，例如：

```text
PlaceVO
PlaceDetailVO
PlaceStatsVO
```

统一 HTTP 外层包装不是 VO；具体规则由 `api-design.md` 维护。

不要为了区分“业务输出”和“HTTP 输出”机械再创建一个同职责模型层。

---

## 9.7 模型转换

只有职责、契约或数据语义真实变化时才转换。

避免无价值链路：

```text
DO → DTO → BO → VO
```

如果某层模型没有独立职责，可以直接跳过。

简单一次性 DO → VO 字段映射不要求机械创建 Converter / Assembler；复杂、多处复用或有业务转换规则时再提取明确映射能力。

---

# 10. 跨模块调用

同一应用内部跨模块调用优先通过对方稳定的 Service / Facade 能力。

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

跨模块不应穿透对方 Mapper、Client 或内部 Manager，仅因为处于同一 JVM 就无边界访问内部实现。

是否需要 Facade 由真实模块边界和公共调用面决定，不机械创建。

---

# 11. 职责判断流程

新增或移动类时依次判断：

```text
它解决哪个业务 / 技术问题？
        ↓
是入站、业务用例、应用能力、数据库访问还是外部技术适配？
        ↓
是否已有同职责实现？
        ↓
依赖方向是否单向？
        ↓
如果是模型，属于 Request / Query / DTO / BO / DO / VO 哪种职责？
        ↓
根据职责映射到 Package；Request / Query 默认都进入 request
        ↓
再结合 project-structure.md 确定业务模块物理位置
```

最终原则：

> `project-structure.md` 决定“放在哪个业务模块和物理目录”，`layering.md` 决定“这个类逻辑上是什么、应该依赖谁”；Request 与 Query 默认共享 `request` Package，通过类名区分语义。职责先于 Package，Package 先于文件创建。
