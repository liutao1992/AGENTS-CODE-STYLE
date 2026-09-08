# 应用分层规范

本文档参考《阿里巴巴 Java 开发手册》的应用分层思想，并结合本项目实际情况制定。

目标：

* 明确各层职责；
* 控制跨层依赖；
* 明确模型分类与 Package 归属；
* 避免业务逻辑和模型职责散落；
* 使用 SOLID 辅助判断职责、依赖和扩展边界；
* 避免为了架构形式增加无意义层级和抽象。

本文负责回答：

> 一个类是什么职责、属于哪个架构边界、应该放在哪个 Package。

Java 语法、Lombok、`class` / `record`、集合、异常和日志等实现细节由 Java 编码规范维护。

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
9. 涉及事务、锁或一致性时读取事务规范。
10. 涉及并发或跨线程时读取并发规范。

编码后检查：

* Controller 是否直接调用 Mapper；
* Controller 是否存在复杂业务逻辑；
* Service 是否混入 HTTP、SQL 或第三方协议细节；
* Mapper 是否包含业务判断；
* 是否跨模块直接访问其他模块 Mapper；
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

> 先确定职责，再确定 Package；分层和模型边界负责“类是什么、放哪里”，Java 规范负责“类怎么写”，SOLID 只用于解决真实设计问题。
