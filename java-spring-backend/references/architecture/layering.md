# 应用分层规范

本文档参考《阿里巴巴 Java 开发手册》的应用分层思想，并结合本项目实际情况制定。

目标：

* 明确各层职责；
* 控制跨层依赖；
* 避免业务逻辑散落；
* 使用 SOLID 辅助判断职责、依赖和扩展边界；
* 避免为了架构形式增加无意义层级和抽象。

模型分类、数据库映射、事务和并发分别由对应专项规范维护，本文不重复展开。

相关规范：

- [Java 模型与编码](../coding/java.md)
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

而不是让业务 Service 到处理解第三方 SDK 的对象、异常和调用协议。

但依赖倒置不意味着所有类都必须采用：

```text
XxxService
    ↓
XxxServiceImpl
```

如果只有一个稳定实现，也不存在替换、隔离或多实现需求，不得为了符合 SOLID 机械创建接口和实现类。

本项目 Mapper 通常已经通过接口形成数据访问边界，不需要仅为了 DIP 再包装无实际价值的 Repository / RepositoryImpl。

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

推荐：

```java
@PostMapping("/{id}/audit")
public AjaxResult audit(
        @PathVariable String id,
        @Valid @RequestBody PlaceAuditRequest request) {

    placeService.audit(id, request, currentOperator());
    return AjaxResult.success();
}
```

原则：

> Controller 保持轻量，只表达 HTTP 接口边界。

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

# 9. 模型边界

模型分类与 Package 归属统一由 Java 编码规范维护：

- [java.md](../coding/java.md)

本文只强调：

* 不直接把数据库 DO 当作 HTTP 输出；
* 不为了分层形式机械创建 DTO / BO / Converter；
* 模型职责发生变化时再进行转换；
* 数据库模型与 Java/API 的隔离由对应 Java、MyBatis 和数据库规范负责。

不要机械创建：

```text
Request → DTO → BO → DO → BO → VO
```

原则：

> 模型跟随真实职责，不跟随形式化层级。

---

# 10. 事务与并发边界

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

# 11. 不要过度分层

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

原则：

> 先保持简单，真实复杂度出现后再增加对应抽象。

---

# 12. Codex 编码检查

编码前：

1. 查看当前模块已有分层和调用关系。
2. 找到至少一个类似实现。
3. 判断逻辑属于 Controller、Service、Manager 还是 Mapper。
4. 判断是否已经存在可复用能力。
5. 新增或调整类、接口、抽象时，确认存在真实职责、变化、替换或隔离需求。
6. 涉及模型时读取 Java 规范，不在本文自行推导模型体系。
7. 涉及事务、锁或一致性时读取事务规范。
8. 涉及并发或跨线程时读取并发规范。

编码后检查：

* Controller 是否直接调用 Mapper；
* Controller 是否存在复杂业务逻辑；
* Service 是否混入 HTTP、SQL 或第三方协议细节；
* Mapper 是否包含业务判断；
* 是否跨模块直接访问其他模块 Mapper；
* 是否创建无实际价值的 Manager / Facade / Repository / Converter；
* 一个类是否同时承担多个明显不同职责，违反 SRP；
* 新增扩展点是否来自真实变化需求，而不是为了套用 OCP；
* 子类或实现是否改变抽象类型核心契约，违反 LSP；
* 接口是否迫使调用方或实现方依赖大量无关能力，违反 ISP；
* 高层业务是否直接耦合易变第三方或技术细节，需要按 DIP 建立稳定边界；
* 是否以 DIP 为理由机械创建 `Interface + Impl`；
* 是否以 SOLID 为理由增加无实际价值的 Strategy / Factory / Repository 包装。

优先保持现有合理架构，不为了遵守规范进行无意义重构。

最终原则：

> 分层用于明确职责和依赖方向，SOLID 用于辅助判断真实设计问题；两者都不能成为过度设计的理由。
