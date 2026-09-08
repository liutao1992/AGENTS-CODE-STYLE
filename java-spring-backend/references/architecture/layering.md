# 应用分层规范

本文档参考《阿里巴巴 Java 开发手册》的应用分层思想，并结合本项目实际情况制定。

目标：

* 明确各层职责；
* 控制跨层依赖；
* 避免业务逻辑散落；
* 隔离数据库模型与业务模型；
* 避免过度设计。

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
Controller   管接口
Service      管业务
Manager      管复用、适配和原子数据操作
Mapper       管数据访问
Database     管存储
```

原则：

> 上层可以依赖下层，下层不得反向依赖上层。

Manager 为可选层，不得为了分层形式机械创建。

---

## 1.1 SOLID 设计原则

分层之外，类、接口和模块设计应参考 SOLID 原则。

SOLID 用于帮助判断职责、依赖和扩展边界，不用于机械增加接口、实现类或设计模式。

核心原则：

> 先保持职责清晰和依赖合理；只有真实变化和扩展需求出现时，再增加必要抽象。

### S — Single Responsibility Principle（单一职责原则）

一个类、接口或模块应围绕一个明确职责设计，并尽量只有一个主要变化原因。

在本项目分层中：

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

---

### O — Open/Closed Principle（开闭原则）

对已有稳定逻辑，应优先通过明确扩展点支持新的变化，而不是不断修改核心流程中的大量条件判断。

当业务已经存在多个稳定变化方向时，可以考虑：

* Strategy；
* Handler；
* Factory；
* 模板方法；
* Spring Bean 集合等扩展方式。

例如，当不同场所类型确实存在多套稳定审核规则时，可以评估：

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

如果当前只有一个明确实现，简单条件逻辑能够清晰解决问题，应保持简单。

原则：

> 为真实变化建立扩展点，不为猜测中的未来变化提前抽象。

---

### L — Liskov Substitution Principle（里氏替换原则）

实现类替换其抽象类型时，不得破坏原有调用方对行为的合理预期。

实现接口或继承父类时，应保持契约一致，包括：

* 输入语义；
* 返回语义；
* 异常语义；
* 状态变化；
* 副作用；
* Null 约定。

禁止为了复用继承关系而出现：

```java
@Override
public void audit(...) {
    throw new UnsupportedOperationException();
}
```

如果某个实现无法满足父类型的核心契约，应重新判断抽象是否合理，而不是通过特殊判断修补错误继承关系。

---

### I — Interface Segregation Principle（接口隔离原则）

接口应围绕明确使用场景设计，避免形成所有调用方都依赖的巨大接口。

例如跨模块提供能力时，应避免：

```java
public interface PlaceService {
    // 查询
    // 创建
    // 审核
    // 统计
    // 文件处理
    // 缓存管理
    // 内部维护操作
    // ...
}
```

如果不同调用方只需要其中少量且稳定的能力，可以根据真实边界拆分更聚焦的 Facade / 接口。

但不要因为接口方法稍多就机械拆成大量小接口。

原则：

> 按真实调用边界隔离接口，不按方法数量机械拆分。

---

### D — Dependency Inversion Principle（依赖倒置原则）

高层业务流程不应直接依赖容易变化的底层技术细节。

例如：

```text
Service
   ↓
稳定业务能力 / 抽象
   ↓
第三方 SDK、HTTP 客户端、具体技术实现
```

对于：

* 第三方系统；
* 外部存储；
* 消息系统；
* 可替换算法；
* 多实现技术能力；

如果存在真实替换、隔离或测试需求，可以通过接口或适配层隔离具体实现。

例如：

```text
CaseService
    ↓
FaceRecognitionClient
    ↓
VendorFaceRecognitionClient
```

而不是让业务 Service 到处直接调用第三方 SDK。

但依赖倒置不意味着所有类都必须采用：

```text
XxxService
    ↓
XxxServiceImpl
```

如果只有一个稳定实现，也不存在替换、隔离或多实现需求，不得为了符合 SOLID 机械创建接口和实现类。

本项目中的 Mapper 本身通常已经通过接口形成数据访问边界，也不需要额外包装 Repository / RepositoryImpl 才算满足 DIP。

---

### SOLID 与简单设计的关系

SOLID 是设计判断原则，不是代码结构模板。

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
当前职责是否混乱？
        ↓ 是
考虑 SRP 拆分

是否存在真实且稳定的变化点？
        ↓ 是
考虑 OCP 扩展

抽象实现是否保持一致契约？
        ↓ 否
检查 LSP

接口是否迫使调用方依赖无关能力？
        ↓ 是
考虑 ISP

高层业务是否直接耦合易变技术细节？
        ↓ 是
考虑 DIP 隔离
```

最终原则：

> SOLID 用来降低真实复杂度，不用来制造新的复杂度。

---

# 2. Controller / Web 层

Controller 负责系统接口边界。

主要职责：

* 接收 HTTP 请求；
* 参数绑定；
* 基础参数校验；
* 获取当前用户等请求上下文；
* 调用 Service；
* 返回统一响应。

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

---

# 3. Service 层

Service 负责具体业务逻辑和业务流程编排。

主要职责：

* 实现业务用例；
* 执行业务校验；
* 编排多个 Manager；
* 调用 Manager 或 Mapper；
* 组织业务输入和输出。

方法名称应表达明确的业务行为。

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

* HTTP 处理；
* SQL；
* 数据库字段映射；
* 第三方协议细节。

---

# 4. Manager 层

Manager 为可选的通用业务处理层。

适用于：

* 多个 Mapper / DAO 的组合操作；
* 多表一致性修改；
* 可复用的数据处理；
* 公共查询能力；
* 第三方服务适配；
* 缓存等基础能力封装；
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

> 有复用、一致性或技术隔离需求时再引入 Manager。

---

# 5. Mapper / DAO 层

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

* 权限判断；
* 业务状态流转；
* 复杂业务决策；
* HTTP 处理；
* 业务事务编排。

---

# 6. 分层调用规则

允许：

```text
Controller
    ↓
Service
    ↓
Mapper
```

复杂场景：

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

跨业务模块时，优先调用对方提供的 Service / Facade。

推荐：

```text
CaseService
    ↓
PlaceService
```

避免：

```text
CaseService
    ↓
PlaceMapper
```

---

# 7. 领域模型

领域模型命名参考阿里巴巴 Java 开发规范：

```text
DO
DTO
BO
Query
VO
```

## DO

数据持久化对象。

本项目数据库使用拼音命名，但 Java 模型统一使用英文，因此 DO 不要求机械对应数据库表名。

例如：

```text
数据库表：ryxx
Java：PersonDO
```

数据库：

```text
zjhm
rqsj
lqsj
```

Java：

```java
private String identityNumber;
private LocalDateTime entryTime;
private LocalDateTime exitTime;
```

---

## DTO

用于跨层、跨模块或跨系统的数据传输。

只有存在明确的数据传输职责时才创建。

---

## BO

用于表达业务处理过程中需要的业务对象。

只有存在实际业务价值时才创建。

---

## Query

用于封装查询条件。

查询条件较多时，应使用明确的 Query 对象。

避免：

```java
Map<String, Object>
```

作为普通业务查询条件。

---

## VO

用于展示层或 API 输出。

VO 根据接口需求设计，不根据数据库字段机械生成。

---

# 8. 数据库防腐层

本项目规定：

```text
数据库：拼音
    ↓
Mapper / ResultMap
    ↓
Java：英文
```

例如：

```text
数据库              Java

zjhm       →        identityNumber
xm         →        name
rqsj       →        entryTime
lqsj       →        exitTime
csbh       →        placeCode
fzxbh      →        centerCode
```

通过 MyBatis 显式建立映射：

```xml
<resultMap id="PersonResultMap" type="PersonDO">
    <result property="identityNumber" column="zjhm"/>
    <result property="name" column="xm"/>
    <result property="entryTime" column="rqsj"/>
    <result property="exitTime" column="lqsj"/>
</resultMap>
```

禁止：

```text
zjhm
 ↓
DO.zjhm
 ↓
DTO.zjhm
 ↓
VO.zjhm
```

数据库拼音命名不得向 Java 业务模型扩散。

---

# 9. 模型转换原则

不要机械创建：

```text
DO → DTO → BO → VO
```

只有模型职责发生变化时才进行转换。

原则：

> 不为了少写转换代码破坏边界，也不为了形式完整增加无意义模型。

---

# 10. 事务边界

事务根据数据一致性需求确定，而不是根据所在层级机械决定。

核心原则：

> 没有事务需求，就不要显式开启事务。

---

## 10.1 普通快照读

普通查询默认不显式开启事务。

例如：

```java
public PlaceDetailVO detail(String id) {
    PlaceDO place = placeManager.getById(id);
    List<EquipmentDO> equipment =
            equipmentManager.listByPlaceId(id);

    return buildVO(place, equipment);
}
```

即使：

* 查询多个表；
* 调用多个 Mapper；
* 调用多个 Manager；

只要这些查询只是普通快照读，并且业务不要求它们共享同一个一致性快照，就没有必要增加：

```java
@Transactional
```

也不要机械增加：

```java
@Transactional(readOnly = true)
```

单纯“这是查询方法”不是开启事务的理由。

---

## 10.2 Manager 层事务

当 Manager 内部包含多个数据库写操作，并要求：

```text
全部成功
或
全部回滚
```

时，应在 Manager 层建立事务。

例如：

```java
@Transactional
public void auditPlace(...) {
    placeMapper.update(...);
    auditRecordMapper.insert(...);
    operationLogMapper.insert(...);
}
```

这种场景适合作为一个独立原子操作。

---

## 10.3 Service 层事务

如果一个完整业务用例需要协调多个 Manager，并要求这些写操作整体成功或失败，可以将事务提升到 Service。

例如：

```java
@Transactional
public void registerCase(...) {
    caseManager.create(...);
    materialManager.register(...);
    personManager.bind(...);
}
```

事务边界应该覆盖：

> 真正需要保持原子性的数据修改范围。

---

## 10.4 查询需要事务的例外

普通快照读默认不开启显式事务。

只有存在明确需求时，查询才建立事务边界，例如：

### 多条查询必须共享同一个数据库快照

例如：

```text
查询 A
   ↓
业务计算
   ↓
查询 B
```

如果 A 和 B 必须基于同一个数据库快照，需要明确设计事务及隔离级别。

不能仅因为添加：

```java
@Transactional(readOnly = true)
```

就默认认为多个查询一定获得了相同快照。

---

### 查询需要锁

例如：

```sql
SELECT ...
FOR UPDATE
```

此类查询本身具有事务和锁语义，应运行在明确的事务中。

---

### 查询后修改要求并发一致性

例如：

```text
读取状态
   ↓
判断
   ↓
修改状态
```

如果要求整个过程保持一致，应通过事务以及：

* 乐观锁；
* 悲观锁；
* 唯一约束；
* 合适的隔离级别；

保证正确性。

---

## 10.5 禁止扩大事务范围

禁止：

* 因为调用多个 Manager 就开启事务；
* 因为调用多个 Mapper 就开启事务；
* 给所有查询方法统一加 `@Transactional(readOnly = true)`；
* 给所有 Service 方法统一加事务；
* 给所有 Manager 方法统一加事务；
* 为了“保险”扩大事务范围。

事务中应尽量避免：

* HTTP / RPC 调用；
* 文件上传；
* 长时间等待；
* 大量耗时计算；
* 无必要的异步操作。

复杂事务、事务传播、隔离级别、锁和异步规则读取：

- [transactions.md](transactions.md)
- [concurrency.md](concurrency.md)

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
Domain
Repository
RepositoryImpl
Converter
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

1. 查看当前模块已有分层。
2. 找到类似实现。
3. 判断逻辑属于 Controller、Service、Manager 还是 Mapper。
4. 判断是否已经存在可复用实现。
5. 涉及事务时，先判断是否真的存在原子性、一致性或锁需求。
6. 新增或调整类、接口、抽象时，检查是否存在真实的职责或变化需求，不得为了 SOLID 机械增加抽象。

编码后检查：

* Controller 是否直接调用 Mapper；
* Controller 是否存在复杂业务逻辑；
* Mapper 是否包含业务判断；
* 是否创建了无实际价值的 Manager / DTO / BO；
* 是否使用 Map 替代明确 Query；
* 数据库拼音是否泄漏到 Java 模型；
* 是否跨模块直接访问其他模块 Mapper；
* 普通快照读是否被无意义地包进事务；
* 事务范围是否超过真正需要保证一致性的范围；
* 一个类是否同时承担多个明显不同的职责，违反 SRP；
* 新增扩展点是否来自真实变化需求，而不是为了套用 OCP；
* 子类或实现是否改变了抽象类型的核心契约，违反 LSP；
* 接口是否迫使调用方依赖大量无关能力，违反 ISP；
* 高层业务是否直接耦合易变的第三方或底层技术细节，应该通过稳定边界隔离；
* 是否以 DIP 为理由机械创建 `Interface + Impl`；
* 是否以 SOLID 为理由增加无实际价值的 Strategy / Factory / Repository 包装。

优先保持现有合理架构，不为了遵守规范进行无意义重构。
