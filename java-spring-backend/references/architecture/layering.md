# 应用分层规范

本文档参考《阿里巴巴 Java 开发手册》的应用分层思想，并结合本项目实际情况制定。

目标：

* 明确各层职责；
* 控制跨层依赖；
* 明确模型分类与 Package 归属；
* 明确异常在各层的转换、记录和对外表达边界；
* 避免业务逻辑和模型职责散落；
* 使用 SOLID 辅助判断职责、依赖和扩展边界；
* 避免为了架构形式增加无意义层级和抽象。

本文负责回答：

> 一个类是什么职责、属于哪个架构边界、应该放在哪个 Package。

Java 语法、Lombok、`class` / `record`、集合、异常实现和日志写法等细节由 Java 编码规范维护；本文只定义异常在各层之间的职责和流转边界。

相关规范：

- [Java 编码](../coding/java.md)
- [Spring](../coding/spring.md)
- [API 设计](../api/api-design.md)
- [MyBatis](../coding/mybatis.md)
- [事务](transactions.md)
- [并发](concurrency.md)

---

# 1. 默认分层

项目默认采用：

```text
Controller / Web
        ↓
      Service
        ↓
      Manager       （按需）
        ↓
   Mapper / DAO
        ↓
     Database
```

简单理解：

```text
Controller   管接口边界
Service      管业务用例和业务流程
Manager      管复用、适配和原子数据操作
Mapper       管数据访问
Database     管数据存储
```

原则：

> 上层可以依赖下层，下层不得反向依赖上层。

Manager 为可选层，不得为了分层形式机械创建。

---

# 2. SOLID 设计原则

分层之外，类、接口和模块设计应参考 SOLID 原则。

SOLID 用于帮助判断职责、依赖和扩展边界，不用于机械增加接口、实现类、设计模式或中间层。

核心原则：

> 先保持职责清晰和依赖合理；只有真实变化、替换、隔离或扩展需求出现时，再增加必要抽象。

## 2.1 S — Single Responsibility Principle（单一职责原则）

一个类、接口或模块应围绕一个明确职责设计，并尽量只有一个主要变化原因。

在当前分层中：

```text
Controller
→ HTTP 接口边界

Service
→ 业务流程和业务用例

Manager
→ 可复用能力、适配和原子数据操作

Mapper
→ 数据库访问
```

例如，不应让 `PlaceService` 同时承担：

```text
业务流程
+ HTTP 响应构造
+ SQL
+ 第三方协议解析
+ 数据库字段转换
```

但单一职责不等于：

```text
一个方法一个类
一个操作一个 Manager
一个转换一个 Converter
```

只有职责确实独立、复杂或需要复用时才拆分。

原则：

> 按变化原因和职责边界拆分，不按代码行数或方法数量机械拆分。

---

## 2.2 O — Open/Closed Principle（开闭原则）

当代码已经存在明确、稳定、持续增加的变化方向时，应优先通过清晰扩展点支持变化，而不是不断扩大核心流程中的条件分支。

可能的扩展方式包括：

* Strategy；
* Handler；
* Factory；
* 模板方法；
* Spring Bean 集合。

例如，不同业务类型确实存在多套稳定审核规则时，可以评估：

```text
PlaceAuditService
        ↓
PlaceAuditStrategy
   ↙          ↘
AStrategy    BStrategy
```

但禁止为了“以后可能扩展”提前设计：

```text
Strategy
Factory
AbstractFactory
大量接口
```

如果当前只有一个实现，或者少量简单条件能够清晰表达业务，应保持简单。

原则：

> 为真实变化建立扩展点，不为猜测中的未来变化提前抽象。

---

## 2.3 L — Liskov Substitution Principle（里氏替换原则）

实现类替换其抽象类型时，不得破坏调用方对原有契约的合理预期。

实现接口或继承父类时，应保持：

* 输入语义；
* 返回语义；
* 异常语义；
* 状态变化；
* 副作用；
* Null 约定。

例如，如果某实现对抽象类型核心方法只能：

```java
@Override
public void audit(...) {
    throw new UnsupportedOperationException();
}
```

应重新判断抽象关系是否合理，而不是通过特殊判断修补错误继承关系。

---

## 2.4 I — Interface Segregation Principle（接口隔离原则）

接口应围绕明确使用场景设计，避免让调用方或实现方依赖大量无关能力。

如果一个跨模块 Facade / SPI 同时包含大量互不相关能力，并导致调用方只使用很小一部分或实现方被迫实现无意义方法，应考虑按真实边界拆分。

但不要因为：

```text
接口方法稍多
```

就机械拆成大量小接口。

原则：

> 按真实调用边界隔离接口，不按方法数量机械拆分。

---

## 2.5 D — Dependency Inversion Principle（依赖倒置原则）

高层业务流程不应直接耦合容易变化的底层技术细节。

重点关注：

* 第三方系统；
* 外部 HTTP 服务；
* 对象存储；
* 消息系统；
* 可替换算法；
* 多供应商实现。

如果存在真实替换、隔离或测试需求，可以通过接口或适配边界隔离具体实现。

例如：

```text
CaseService
    ↓
FaceRecognitionClient
    ↓
VendorFaceRecognitionClient
```

而不是让业务 Service 直接理解第三方 SDK 的对象、异常和调用协议。

但依赖倒置不意味着所有类都必须采用：

```text
XxxService
    ↓
XxxServiceImpl
```

如果只有一个稳定实现，也不存在替换、隔离或多实现需求，不得为了符合 SOLID 机械创建接口和实现类。

项目中的 Mapper 通常已经通过接口形成数据访问边界，不需要仅为了 DIP 再包装无实际价值的 Repository / RepositoryImpl。

---

## 2.6 SOLID 与简单设计

SOLID 是设计判断原则，不是固定代码模板。

禁止以下机械推导：

```text
SOLID
  ↓
所有 Service 都创建接口
  ↓
所有接口都创建 Impl
  ↓
所有业务都创建 Strategy / Factory
  ↓
所有 Mapper 再包装 Repository
```

正确判断方式：

```text
职责是否真实混乱？
        ↓ 是
考虑 SRP

是否存在真实且稳定的变化方向？
        ↓ 是
考虑 OCP

实现是否破坏已有抽象契约？
        ↓ 是
检查 LSP

调用方或实现方是否被迫依赖无关能力？
        ↓ 是
考虑 ISP

高层业务是否直接耦合易变技术细节？
        ↓ 是
考虑 DIP
```

最终原则：

> SOLID 用来降低真实复杂度，也用来识别错误抽象；不用来制造新的复杂度。

---

# 3. Controller / Web 层

Controller 负责系统接口边界。

主要职责：

* 接收 HTTP 请求；
* 参数绑定；
* 基础参数校验；
* 获取当前用户等请求上下文；
* 调用 Service；
* 返回项目统一响应。

禁止：

* 直接调用 Mapper / DAO；
* 编写 SQL；
* 定义业务事务；
* 承载复杂业务逻辑；
* 直接操作数据库对象完成业务流程。

默认示例：

```java
@PostMapping("/{id}/audit")
public ApiResponse<Void> audit(
        @PathVariable String id,
        @Valid @RequestBody PlaceAuditRequest request) {

    placeService.audit(id, request, currentOperator());
    return ApiResponse.success();
}
```

`ApiResponse<T>` 只是本 Skill 的默认统一响应示例；目标项目已有其他统一响应类型、历史 API 或序列化契约时，以项目现有约定为准。

原则：

> Controller 保持轻量，只表达 HTTP 接口边界。

具体 HTTP 和响应契约读取：

- [spring.md](../coding/spring.md)
- [api-design.md](../api/api-design.md)

---

# 4. Service 层

Service 负责业务用例和业务流程编排。

主要职责：

* 实现业务用例；
* 执行业务校验；
* 编排多个 Manager；
* 简单场景下直接调用 Mapper；
* 组织业务输入和输出；
* 协调多个业务能力。

方法名称应表达明确业务行为。

推荐：

```java
audit(...)
register(...)
approve(...)
reject(...)
bindEquipment(...)
```

避免：

```java
handle(...)
process(...)
doSomething(...)
```

Service 不负责：

* HTTP 状态和响应协议；
* SQL；
* 数据库字段映射；
* 第三方协议细节。

原则：

> Service 表达业务流程，不成为 HTTP、SQL 和技术细节的混合层。

---

# 5. Manager 层

Manager 为可选层。

适用于：

* 多个 Mapper / DAO 的组合操作；
* 多表一致性修改；
* 可复用的数据操作；
* 公共查询能力；
* 第三方服务适配；
* 缓存等技术能力封装；
* 需要独立事务保证的原子数据操作。

例如：

```text
PlaceService
      ↓
PlaceManager
   ↙       ↘
PlaceMapper AuditMapper
```

不要因为只有：

```text
Service
   ↓
Mapper
```

就强制增加 Manager。

原则：

> 有真实复用、一致性或技术隔离需求时再引入 Manager。

---

# 6. Mapper / DAO 层

Mapper / DAO 负责数据库访问。

主要职责：

* 查询；
* 新增；
* 更新；
* 删除；
* 批量操作；
* ResultMap；
* SQL。

例如：

```java
PlaceDO getById(String id);

List<PlaceDO> listByQuery(PlaceQuery query);

int insert(PlaceDO place);

int update(PlaceDO place);
```

Mapper 不负责：

* 权限业务判断；
* 业务状态流转；
* 复杂业务决策；
* HTTP 处理；
* 业务事务编排。

MyBatis 基础设施、数据库与 Java 映射等详细规则读取：

- [mybatis.md](../coding/mybatis.md)

---

# 7. 分层调用与依赖规则

简单业务允许：

```text
Controller
    ↓
Service
    ↓
Mapper
```

需要真实复用、一致性操作或技术适配时：

```text
Controller
    ↓
Service
    ↓
Manager
    ↓
Mapper
```

禁止：

```text
Controller → Mapper
Mapper     → Service
Manager    → Controller
```

也禁止为了减少代码直接从业务层绕过已有边界访问底层实现。

原则：

> 依赖方向保持单向，职责边界比调用方便更重要。

---

## 7.1 分层异常处理规约

异常也必须遵循分层边界。

核心目标不是让每一层都 `catch` 一次，而是做到：

```text
底层技术异常
    ↓ 必要时转换
业务 / 应用边界
    ↓ 补充业务上下文
Web / API 边界
    ↓ 转换为稳定的对外错误契约
客户端
```

总体原则：

> 异常只在真正拥有“转换、补充上下文、恢复或对外表达”职责的层处理；同一个异常不要在每一层重复打印日志。

### 7.1.1 Mapper / DAO 层

DAO / Mapper 面对的底层异常类型通常很多，不应在每个数据访问方法中对各种数据库异常进行细粒度 `catch`。

在 Spring + MyBatis 项目中，应优先复用框架已有的数据访问异常翻译机制，不要机械给所有 Mapper 外面再包一层：

```java
try {
    ...
} catch (Exception ex) {
    throw new DAOException(ex);
}
```

只有在以下情况确实存在时，才在 DAO / 数据访问适配边界进行异常转换：

* 当前项目已经有统一的 DAO / DataAccess 异常体系；
* 接入的底层库不会被 Spring 正确翻译；
* 需要隔离第三方驱动、SDK 或存储实现的异常类型；
* 上层不应直接依赖某个具体持久化技术的异常。

如果确实需要统一捕获多个不可细分的底层异常，可以在该技术边界使用较宽的捕获方式并转换为项目已有的数据访问异常，但必须：

```text
保留原始 cause
不吞异常
不改变成功/失败语义
```

例如概念上：

```java
try {
    return storageClient.load(id);
} catch (Exception ex) {
    throw new DataAccessException("Failed to load data", ex);
}
```

这里的 `DataAccessException` 只表示“项目已有的统一数据访问异常”这一角色，不要求新建同名类型。

DAO / Mapper 层通常不重复记录完整异常日志，因为上层拥有更多业务上下文；如果底层已经记录一次，上层再次记录同一堆栈通常只会产生重复日志和噪声。

禁止：

```text
DAO catch
→ log.error
→ throw
→ Manager 再 log.error
→ Service 再 log.error
→ Web 再 log.error
```

原则：

> DAO 负责数据访问和必要的技术异常隔离，不负责重复记录业务失败现场。

---

### 7.1.2 Manager 层

Manager 与 Service 同进程部署时，异常处理方式通常接近 DAO / 应用内部能力层：

* 能恢复的异常可以在职责范围内处理；
* 需要改变抽象语义时可以转换异常；
* 无法处理的异常继续向 Service / 应用边界传播；
* 不因为“经过 Manager”就重复打印同一异常堆栈。

Manager 进行异常转换时，必须建立真实的抽象边界。

例如第三方能力：

```text
VendorSdkException
        ↓
FaceRecognitionException
        ↓
Service
```

可以避免 Service 直接理解供应商 SDK 的异常类型。

但禁止仅为了分层形式：

```text
RuntimeException
    ↓
ManagerException
    ↓
ServiceException
```

机械逐层包装。

如果 Manager 本身被独立部署成远程服务，则它已经成为一个独立应用边界，应按照 Service / API 边界的方式完成日志记录和对外异常转换，而不能假设上层仍与它共享同一进程日志。

原则：

> 同进程 Manager 不重复制造日志和异常层级；独立部署时按独立服务边界处理。

---

### 7.1.3 Service 层

Service 最了解当前业务用例、业务动作和关键参数，因此通常是补充异常业务上下文的重要位置。

对于无法在当前业务用例内恢复的非预期异常，应确保系统最终能够记录足够的“案发现场”，例如：

```text
业务动作
资源标识
关键业务编号
安全的输入摘要
当前处理阶段
异常 cause
TraceId / RequestId（项目存在时）
```

例如日志语义应接近：

```text
审核场所失败，placeId=xxx，operatorId=xxx，当前阶段=更新审核记录
```

而不是只有：

```text
操作失败
```

但“Service 必须记录日志”不等于每个 Service 方法都机械：

```java
catch (Exception ex) {
    log.error(..., ex);
    throw ex;
}
```

如果项目已经由统一异常处理器、AOP 或应用边界集中记录未处理异常，并且能够获得足够业务上下文，Service 不应再次重复记录相同堆栈。

已知且可预期的业务异常，例如：

```text
参数不合法
状态不允许
资源不存在
权限不足
```

也不应机械按系统故障记录为 `error`；日志级别和是否记录应以项目已有日志规范和实际影响为准。

Service 记录参数时必须遵守安全规则，不得直接打印：

* 密码；
* Token；
* 密钥；
* 完整身份证件；
* 生物特征；
* 其他敏感数据。

原则：

> 异常日志应在最了解业务上下文且能够避免重复的位置记录一次，并保留足够排查信息。

---

### 7.1.4 Controller / Web 层

Web 层是应用的 HTTP 边界，异常不能以 Java 堆栈、内部异常类型或未处理容器错误的形式直接暴露给客户端。

“Web 层不继续往上抛异常”在 Spring MVC 中应理解为：

> 异常必须在 Web 应用边界内被统一收口和转换，而不是要求每个 Controller 手写 `try/catch`。

推荐通过项目已有的：

```text
@RestControllerAdvice
@ExceptionHandler
HandlerExceptionResolver
统一错误页面机制
```

处理异常。

普通 Controller 不应机械写成：

```java
try {
    service.execute();
} catch (Exception ex) {
    ...
}
```

否则容易出现：

* Controller 重复异常转换；
* 返回结构不一致；
* 重复打印日志；
* 吞掉事务回滚所需异常；
* 不同接口形成不同错误语义。

对于传统服务端页面渲染场景，如果异常会导致页面无法正常渲染，应由 Web 错误处理机制返回友好的错误页面或提示，而不是向用户展示技术堆栈。

对于 REST API，应转换为稳定的错误响应契约。

原则：

> Controller 可以把异常交给应用内统一 Web 异常处理器，但异常不得裸露越过 HTTP 边界。

---

### 7.1.5 开放 API / REST 接口

开放接口必须把内部异常转换为客户端能够稳定理解的：

```text
错误码
错误信息
必要的请求追踪标识
```

统一 HTTP 响应默认可以使用项目约定的错误结构；本 Skill 默认响应包装为 `ApiResponse<T>`，但具体错误字段、错误码和序列化契约以目标项目已有实现为准。

对外错误信息不得直接暴露：

* Java 异常类名；
* SQL；
* 数据库表名；
* 堆栈；
* 文件路径；
* 内部服务器地址；
* 第三方密钥和凭证；
* 其他内部技术细节。

例如：

```text
SQLException
DuplicateKeyException
VendorSdkTimeoutException
```

不应直接成为稳定 API 错误协议。

应该由统一异常映射层转换为项目已有的业务错误码和安全错误信息。

错误码和错误信息的详细契约读取：

- [api-design.md](../api/api-design.md)
- [spring.md](../coding/spring.md)

---

### 7.1.6 异常转换必须保留根因

跨抽象边界转换异常时必须保留原始 cause。

推荐：

```java
throw new StorageAccessException("Failed to load attachment", ex);
```

避免：

```java
throw new StorageAccessException("Failed to load attachment");
```

导致原始堆栈丢失。

同时禁止无意义逐层包装：

```text
SQLException
→ DAOException
→ ManagerException
→ ServiceException
→ ApiException
```

只有异常语义确实跨越一个新的抽象边界时才转换。

原则：

> 转换异常是为了隔离抽象，不是为了证明代码经过了多少层。

---

### 7.1.7 异常日志只记录一次完整现场

同一个失败链路通常只需要在一个拥有足够上下文的位置记录一次完整异常堆栈。

推荐思路：

```text
DAO / Manager
→ 不重复打印

Service / 应用边界
→ 补充业务上下文

统一异常处理器
→ 如果项目在这里集中记录，则 Service 不再重复打印

Web / API
→ 转换为安全错误响应
```

是否最终由 Service、全局异常处理器、AOP 或其他应用边界记录，以目标项目现有日志架构为准。

禁止为了“保险”让每层都：

```java
log.error(..., ex);
throw ex;
```

否则会导致：

* 一次请求产生多份相同堆栈；
* 日志量被放大；
* 告警重复；
* 真正关键业务上下文被噪声淹没。

---

### 7.1.8 禁止的异常处理方式

禁止：

```java
catch (Exception ex) {
}
```

吞掉异常。

禁止：

```java
catch (Exception ex) {
    log.error("failed", ex);
    return null;
}
```

把失败伪装成成功或普通空值。

禁止为了形式统一机械：

```text
每层 catch
每层 log
每层 wrap
```

禁止在没有实际抽象边界时创建：

```text
DAOException
ManagerException
ServiceException
```

平行异常体系。

禁止为了隐藏异常而破坏事务回滚、API 错误语义或调用方判断逻辑。

最终原则：

> 底层负责必要的技术异常隔离，中间层补充真正有价值的业务语义，应用边界记录一次足够的失败现场，Web / API 层统一转换为安全稳定的对外错误契约。

---

# 8. 跨模块调用

跨业务模块时，优先调用目标模块提供的 Service / Facade。

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

原因：

* 直接访问其他模块 Mapper 会绕过其业务规则；
* 调用方会耦合目标模块数据库实现；
* 数据访问边界和权限边界容易被破坏。

但不要为了形式机械创建 Facade。

如果目标模块已有清晰且稳定的 Service，并且能够表达跨模块能力，可以直接复用。

---

# 9. 模型分类与 Package 归属

模型属于应用架构边界的一部分，因此统一在本文定义。

默认模型体系：

```text
Request
→ 接口输入模型

Query
→ 查询条件模型

DTO
→ 应用内部数据传输模型

BO
→ 业务处理模型

DO
→ 持久化数据模型

VO
→ 视图输出模型
```

不同模型职责不同，不得因为“都是保存字段的 Java 类”就统一放入 `dto`。

Java 层面的 Lombok、`class` / `record`、Getter / Setter 等实现方式读取：

- [java.md](../coding/java.md)

---

## 9.1 Request

Request 表达接口输入。

例如：

```text
PlaceCreateRequest
PlaceUpdateRequest
PlaceAuditRequest
```

默认 Package：

```text
<module>.request
```

例如：

```text
place.request.PlaceCreateRequest
place.request.PlaceUpdateRequest
place.request.PlaceAuditRequest
```

典型边界：

```text
HTTP Request
      ↓
Request
      ↓
Controller
      ↓
Service
```

Request 可以承载接口结构性校验，但不承担：

* 数据库存储职责；
* 查询结果职责；
* 视图输出职责；
* 复杂业务行为。

接口校验详细规则读取：

- [spring.md](../coding/spring.md)
- [api-design.md](../api/api-design.md)

---

## 9.2 Query

Query 表达查询条件。

例如：

```text
PlaceQuery
PersonQuery
CaseQuery
```

默认 Package：

```text
<module>.query
```

例如：

```text
place.query.PlaceQuery
```

Query 可以在：

```text
Controller
    ↓
Service
    ↓
Mapper
```

之间传递查询条件。

Query 不属于 DTO，也不得因为 Mapper 使用它就放入：

```text
<module>.mapper
```

禁止：

```text
place.dto.PlaceQuery
place.mapper.PlaceQuery
```

应使用：

```text
place.query.PlaceQuery
```

普通业务查询条件较多时，可使用 Query 对象；具体参数规则读取 Java 和 MyBatis 规范。

---

## 9.3 DTO

DTO 表达应用内部的数据传输。

默认 Package：

```text
<module>.dto
```

DTO 只有在确实存在独立的数据传输职责时才创建，例如：

```text
Service
   ↓
Manager
```

或者：

```text
模块 A
   ↓
模块 B 的应用接口
```

不得把 DTO 当作所有数据对象的统一目录。

禁止仅因为类包含字段就机械创建：

```text
PlaceCreateDTO
PlaceResponseDTO
PlaceQueryDTO
```

如果模块没有真实 DTO 需求：

```text
<module>.dto
```

可以不存在。

---

## 9.4 BO

BO 表达业务处理过程中形成的业务对象。

默认 Package：

```text
<module>.bo
```

例如：

```text
AuditResultBO
PlaceRegistrationBO
```

只有业务逻辑确实需要独立于 Request、Query、DO、VO 的中间业务模型时，才创建 BO。

不得机械执行：

```text
Request
   ↓
DTO
   ↓
BO
   ↓
DO
   ↓
BO
   ↓
VO
```

简单业务允许：

```text
Request
   ↓
Service
   ↓
DO
   ↓
VO
```

原则：

> 没有真实职责，就不要增加中间模型。

---

## 9.5 DO

DO 表达数据库持久化数据。

例如：

```text
PlaceDO
PersonDO
CaseDO
```

默认 Package：

```text
<module>.domain
```

例如：

```text
place.domain.PlaceDO
```

DO 使用 Java 英文业务语义。

数据库可以使用拼音：

```text
csxx
csbh
csmc
```

Java 使用英文：

```text
PlaceDO
placeCode
placeName
```

数据库与 Java 之间通过 MyBatis 建立显式映射：

```text
Database（拼音）
       ↓
Mapper / ResultMap
       ↓
DO（英文）
```

DO 不要求机械复制数据库表名。

例如数据库表：

```text
ryxx
```

Java 可以使用：

```text
PersonDO
```

而不是：

```text
RyxxDO
```

映射详细规则读取：

- [mybatis.md](../coding/mybatis.md)

---

## 9.6 VO

VO 表达提供给视图层或客户端的具体业务输出数据。

例如：

```text
PlaceVO
PlaceStatsVO
PlaceTreeNodeVO
```

默认 Package：

```text
<module>.vo
```

例如：

```text
place.vo.PlaceVO
place.vo.PlaceStatsVO
place.vo.PlaceTreeNodeVO
```

典型边界：

```text
Service
   ↓
VO
   ↓
Controller
   ↓
ApiResponse<VO>
```

具体业务返回模型优先使用 VO 表达。

例如，如果：

```text
PlaceResponse
```

实际表达的是场所接口业务数据，新代码应优先使用：

```text
PlaceVO
```

并放入：

```text
place.vo
```

不应机械放入：

```text
place.dto
place.response
```

原则：

> VO 表达具体业务视图数据；统一 HTTP Response 包装不是业务模型分类。

`ApiResponse<T>` 是统一响应包装的默认推荐；目标项目已有其他响应契约时，以项目为主。

---

## 9.7 通用模型

只有真正与具体业务模块无关，并且能够被多个模块复用的结构，才能进入 `common`。

例如通用：

```text
ApiResponse<T>
分页包装
通用错误结构
```

可以按照目标项目现有公共 Web 模型目录组织。

而：

```text
PlaceVO
PlaceStatsVO
PlaceTreeNodeVO
```

具有明确 Place 业务语义，因此必须保留在业务模块中。

不得因为多个 Controller 都可能使用，就移动到 `common`。

原则：

> 通用结构进入 common，具体业务模型留在对应业务模块。

具体公共 Package 名称以目标项目已有约定为准，不为了本 Skill 新建第二套公共目录。

---

## 9.8 模型归属判断

创建模型前必须先判断：

```text
这个对象是什么？
        ↓
接口输入？
→ Request

查询条件？
→ Query

内部数据传输？
→ DTO

业务处理中间对象？
→ BO

数据库持久化？
→ DO

视图输出？
→ VO
```

不得根据：

```text
它只是保存字段
```

直接判断：

```text
它就是 DTO
```

也不得根据：

```text
当前正在开发 Place
```

推导：

```text
所有新类都放 place 下当前正在使用的 Package
```

Package 由类和模型的职责决定，不由当前任务所在目录决定。

---

## 9.9 创建模型前必须搜索

新增：

```text
Request
Query
DTO
BO
DO
VO
```

之前必须先搜索：

1. 是否已经存在相同或类似模型；
2. 当前模块是否已经存在统一 Package；
3. 是否能够复用已有模型；
4. 当前模型实际承担什么职责；
5. 是否真的需要新的模型类型。

禁止先把模型统一创建到：

```text
dto
```

然后再根据使用方式调整。

创建流程：

```text
确定职责
    ↓
Request / Query / DTO / BO / DO / VO？
    ↓
搜索已有模型
    ↓
判断是否需要新增
    ↓
确定 Package
    ↓
读取 java.md 确定 Java 实现方式
    ↓
创建文件
```

---

## 9.10 模型转换原则

不要机械创建：

```text
Request → DTO → BO → DO → BO → VO
```

只有职责或边界真正发生变化时才进行转换。

简单业务允许：

```text
Request
   ↓
Service
   ↓
DO
   ↓
VO
```

原则：

> 不为了少写转换代码破坏模型边界，也不为了形式完整增加无意义模型。

---

# 10. 技术基础设施的 Package 归属

Package 由组件自身职责决定，不由当前使用它的业务模块决定。

例如 MyBatis 通用技术组件：

```text
TypeHandler
→ common.mybatis.handler

Interceptor
→ common.mybatis.interceptor

Plugin
→ common.mybatis.plugin

Configuration
→ common.mybatis.config
```

因此：

```text
JsonStringListTypeHandler
```

即使当前只被 `PlaceMapper` 使用，也不应因此放入：

```text
place.mapper
```

原则：

> “当前模块需要这个类”不等于“这个类属于当前模块”。

新增技术基础设施前必须搜索项目已有同类组件和 Package 约定。

具体 MyBatis 规则读取：

- [mybatis.md](../coding/mybatis.md)

---

# 11. 事务与并发边界

事务边界由数据一致性需求决定，不由 Controller / Service / Manager / Mapper 的层级名称机械决定。

本文不重复事务、锁、隔离级别和跨线程细节。

涉及事务时必须读取：

- [transactions.md](transactions.md)

涉及异步、线程池或跨线程执行时读取：

- [concurrency.md](concurrency.md)

只保留以下分层判断：

```text
可复用原子数据操作
→ Manager 可以作为事务边界

跨多个 Manager 的完整业务写入需要整体提交/回滚
→ Service 可以作为事务边界
```

但最终仍以真实一致性范围为准。

---

# 12. 不要过度分层

简单业务优先：

```text
Controller
    ↓
Service
    ↓
Mapper
```

需要复用、一致性处理或第三方适配时：

```text
Controller
    ↓
Service
    ↓
Manager
    ↓
Mapper
```

不要为了架构形式机械增加：

```text
Manager
Domain Service
Repository
RepositoryImpl
Converter
Assembler
Factory
```

也不要以 SOLID 为理由机械增加：

```text
ServiceInterface
ServiceImpl
Strategy
Factory
Adapter
```

也不要为了模型形式完整机械增加：

```text
DTO
BO
Converter
Assembler
```

原则：

> 先保持简单，真实复杂度出现后再增加对应抽象。

---

# 13. Codex 编码检查

编码前：

1. 查看当前模块已有分层和调用关系。
2. 找到至少一个类似实现。
3. 判断逻辑属于 Controller、Service、Manager、Mapper、模型还是技术基础设施。
4. 判断是否已经存在可复用能力。
5. 新增类前先确定职责，再确定 Package。
6. 新增模型时按本文判断 Request / Query / DTO / BO / DO / VO，再读取 Java 规范确定实现方式。
7. 新增技术组件时按技术职责确定 Package，不按当前业务使用者归属。
8. 新增或调整类、接口、抽象时，确认存在真实职责、变化、替换或隔离需求。
9. 涉及异常处理时，确认由哪一层负责转换、补充业务上下文、记录日志和对外表达，避免每层重复 catch / log / wrap。
10. 涉及事务、锁或一致性时读取事务规范。
11. 涉及并发或跨线程时读取并发规范。

编码后检查：

* Controller 是否直接调用 Mapper；
* Controller 是否存在复杂业务逻辑；
* Service 是否混入 HTTP、SQL 或第三方协议细节；
* Mapper 是否包含业务判断；
* 是否跨模块直接访问其他模块 Mapper；
* DAO / Mapper 是否无必要地捕获所有异常并重复打印日志；
* Manager 是否在没有新抽象语义时机械包装异常或重复打印同一堆栈；
* Service / 应用边界是否保留足够的业务失败上下文；
* 同一异常链是否在多个层级重复 `log.error(..., ex)`；
* 异常转换时是否保留原始 cause；
* Web / API 是否通过统一异常边界转换为稳定错误码和安全错误信息；
* 是否把 Java 堆栈、SQL、内部异常类型或敏感参数直接暴露给客户端；
* 是否通过 catch 后返回 `null` / 默认值等方式把失败伪装成成功；
* Request / Query / DTO / BO / DO / VO 是否按真实职责归类；
* Package 是否由职责决定，而不是由当前任务目录决定；
* 具体业务输出是否错误使用 `*Response` / `response` 包而不是 VO；
* 是否把所有数据模型机械放入 `dto`；
* 是否创建无实际价值的 DTO / BO / Converter / Manager / Facade / Repository；
* 技术基础设施是否错误放入业务 Mapper 等 Package；
* 数据库拼音是否越过 Mapper 映射边界泄漏到 Java 业务模型；
* 一个类是否同时承担多个明显不同职责，违反 SRP；
* 新增扩展点是否来自真实变化需求，而不是为了套用 OCP；
* 子类或实现是否改变抽象类型核心契约，违反 LSP；
* 接口是否迫使调用方或实现方依赖大量无关能力，违反 ISP；
* 高层业务是否直接耦合易变第三方或技术细节，需要按 DIP 建立稳定边界；
* 是否以 DIP 为理由机械创建 `Interface + Impl`；
* 是否以 SOLID 为理由增加无实际价值的 Strategy / Factory / Repository 包装。

优先保持现有合理架构，不为了遵守规范进行无意义重构。

最终原则：

> 先确定职责，再确定 Package；分层和模型边界负责“类是什么、放哪里”，异常边界负责“哪里转换、哪里记录、哪里对外表达”，Java 规范负责“类怎么写”，SOLID 只用于解决真实设计问题。
