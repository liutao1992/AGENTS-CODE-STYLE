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

编码后检查：

* Controller 是否直接调用 Mapper；
* Controller 是否存在复杂业务逻辑；
* Mapper 是否包含业务判断；
* 是否创建了无实际价值的 Manager / DTO / BO；
* 是否使用 Map 替代明确 Query；
* 数据库拼音是否泄漏到 Java 模型；
* 是否跨模块直接访问其他模块 Mapper；
* 普通快照读是否被无意义地包进事务；
* 事务范围是否超过真正需要保证一致性的范围。

优先保持现有合理架构，不为了遵守规范进行无意义重构。

