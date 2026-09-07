# 事务规范

本文档定义项目中的事务使用规则。

核心原则：

> 事务由数据一致性需求决定，而不是由方法名称、调用层级或 Mapper 数量决定。

> 普通快照读默认不显式开启事务。

> 事务范围应尽可能小，只覆盖真正需要保证一致性的数据库操作。

---

# 1. 什么时候需要事务

通常需要事务：

* 多个写操作必须全部成功或全部回滚；
* 多张表修改必须保持一致；
* 查询后修改存在并发一致性要求；
* 使用数据库锁；
* 多个查询必须共享明确的一致性快照。

通常不需要显式事务：

* 单条普通查询；
* 多个相互独立的普通查询；
* 多个 Manager 的普通快照读；
* 普通统计查询；
* 不要求一致性快照的详情查询。

原则：

> 不因为访问数据库就自动开启事务。

---

# 2. 事务边界

事务边界由一致性范围决定。

通常：

```text
Manager
    ↓
可复用的原子数据操作
```

或者：

```text
Service
    ↓
跨多个 Manager 的完整业务事务
```

不要机械规定所有事务只能存在某一层。

---

# 3. Manager 层事务

当一个 Manager 内部组合多个数据库写操作，并且这些操作必须作为整体成功或失败时，优先在 Manager 层定义事务。

例如：

```java
@Transactional
public void auditPlace(...) {
    placeMapper.update(...);
    auditRecordMapper.insert(...);
    operationLogMapper.insert(...);
}
```

例如以下操作：

```text
修改场所状态
+
新增审核记录
+
新增操作记录
```

共同构成一个原子操作。

此时适合由 Manager 管理事务。

---

# 4. Service 层事务

Service 主要负责业务流程编排。

如果一个完整业务用例需要协调多个 Manager，并且这些 Manager 中的数据库写操作必须整体提交或回滚，可以将事务提升到 Service。

例如：

```java
@Transactional
public void registerCase(...) {
    caseManager.create(...);
    materialManager.register(...);
    personManager.bind(...);
}
```

原则：

> 事务覆盖业务真正要求原子性的范围，不要为了保险扩大事务。

---

# 5. 普通快照读

普通快照读默认不显式开启事务。

例如：

```java
public PlaceDetailVO detail(String placeId) {
    PlaceDO place = placeManager.getById(placeId);
    List<EquipmentDO> equipments =
            equipmentManager.listByPlaceId(placeId);

    return buildVO(place, equipments);
}
```

即使：

* 查询多个表；
* 调用多个 Mapper；
* 调用多个 Manager；

只要这些查询允许按照数据库正常快照语义独立执行，并且业务不要求所有查询共享同一个数据快照，就无需增加：

```java
@Transactional
```

也不要因为这是查询方法，就机械添加：

```java
@Transactional(readOnly = true)
```

原则：

> “查询”本身不是开启事务的理由。

---

# 6. 多查询一致性快照

如果多个查询必须基于同一个一致性数据视图，则需要专门设计事务。

例如：

```text
查询 A
   ↓
业务计算
   ↓
查询 B
```

如果 A 和 B 必须基于同一数据库状态，需要考虑：

* 事务边界；
* PostgreSQL 快照语义；
* 事务隔离级别。

注意：

PostgreSQL 默认 `READ COMMITTED` 隔离级别下，同一事务中的不同 SQL 仍可能看到不同时间点已经提交的数据。

因此：

```java
@Transactional(readOnly = true)
```

并不天然意味着：

```text
多个 SELECT 共享完全相同的数据快照
```

如果业务明确要求一致性快照，应结合隔离级别进行设计。

---

# 7. 查询后修改

以下模式需要关注并发一致性：

```text
查询
 ↓
判断
 ↓
修改
```

例如：

```java
PlaceDO place = placeMapper.getById(id);

if (place.canAudit()) {
    placeMapper.updateStatus(id, AUDITED);
}
```

多个请求可能同时读取到相同状态，并同时通过判断。

仅仅增加：

```java
@Transactional
```

并不一定能够解决问题。

应根据场景考虑：

* 条件更新；
* 乐观锁；
* 悲观锁；
* 唯一约束；
* 合适的事务隔离级别。

---

# 8. 优先使用条件更新

简单状态竞争优先考虑通过 SQL 条件保证并发正确性。

例如：

```sql
UPDATE place
SET zt = #{targetStatus}
WHERE id = #{id}
  AND zt = #{expectedStatus}
```

然后检查更新行数。

如果：

```text
affectedRows == 0
```

说明状态已经发生变化。

对于简单状态流转，这通常比：

```text
先 SELECT
↓
Java 判断
↓
UPDATE
```

更可靠。

---

# 9. 乐观锁

冲突概率较低，并且允许冲突检测后失败或有限重试时，可以使用乐观锁。

常见方式：

```text
version
```

例如：

```sql
UPDATE ...
SET
    ...,
    version = version + 1
WHERE id = #{id}
  AND version = #{version}
```

更新行数为 `0` 表示发生并发修改。

禁止无限重试。

---

# 10. 悲观锁

确实需要数据库行锁时，可以使用：

```sql
SELECT ...
FOR UPDATE
```

此类查询必须位于明确事务中。

使用前应考虑：

* 锁持有时间；
* 死锁风险；
* 加锁顺序；
* 并发吞吐；
* 是否可以使用条件更新或乐观锁替代。

不要默认使用悲观锁解决并发问题。

---

# 11. readOnly 事务

不要因为方法名称是：

```text
get
find
list
query
select
```

就自动添加：

```java
@Transactional(readOnly = true)
```

普通快照读默认无需显式事务。

只有确实需要事务边界，同时事务主要用于读取时，才考虑只读事务。

---

# 12. 事务传播

默认优先使用 Spring 默认传播行为：

```java
Propagation.REQUIRED
```

没有明确业务语义时，不要随意使用：

```text
REQUIRES_NEW
NESTED
NOT_SUPPORTED
NEVER
```

尤其是：

```java
REQUIRES_NEW
```

会开启独立事务。

内层事务一旦提交，即使外层事务后续回滚，已经提交的数据通常也不会随外层回滚。

因此只有明确要求独立提交时才能使用。

---

# 13. Spring 事务自调用

Spring 常见事务机制基于代理。

例如：

```java
public void process() {
    audit();
}

@Transactional
public void audit() {
}
```

同一个对象内部直接调用：

```java
audit();
```

可能绕过 Spring Proxy。

因此不得仅因为方法存在：

```java
@Transactional
```

就认为事务一定生效。

涉及自调用时，应检查实际 Bean 调用关系和代理行为。

---

# 14. 异常与回滚

禁止在事务中捕获异常后直接吞掉：

```java
@Transactional
public void execute() {
    try {
        mapper.update(...);
    } catch (Exception e) {
        log.error("error", e);
    }
}
```

异常被吞掉后，事务可能继续提交。

需要捕获异常时，应明确：

* 是否继续抛出；
* 是否转换异常；
* 当前事务是否应该回滚。

如果项目使用 Checked Exception，并要求其触发回滚，应根据项目异常体系明确配置。

不要机械给所有事务添加：

```java
rollbackFor = Exception.class
```

---

# 15. 避免长事务

事务应尽可能短。

事务中尽量避免：

* HTTP / RPC；
* 文件上传；
* 第三方接口调用；
* 长时间计算；
* 阻塞等待；
* `sleep`。

推荐：

```text
准备参数
   ↓
开始事务
   ↓
必要数据库操作
   ↓
提交
```

避免：

```text
开始事务
   ↓
远程调用
   ↓
复杂计算
   ↓
等待
   ↓
数据库操作
   ↓
提交
```

---

# 16. 事务与线程切换

Spring 事务通常绑定当前线程。

切换线程后，不应假设事务自动传播。

例如：

```java
@Transactional
public void process() {
    mapper.update(...);

    CompletableFuture.runAsync(() -> {
        mapper.insert(...);
    });
}
```

这里两个数据库操作默认不能认为处于同一个事务中。

原则：

> 事务中的数据库操作不得为了性能直接拆分到不同线程执行。

涉及线程池、异步执行或 `CompletableFuture` 时，同时读取：

- [concurrency.md](concurrency.md)

---

# 17. 禁止事项

禁止：

* 给所有 Service 方法统一添加事务；
* 给所有 Manager 方法统一添加事务；
* 给所有查询方法统一添加 `@Transactional(readOnly = true)`；
* 因为调用多个 Mapper 就开启事务；
* 因为调用多个 Manager 就开启事务；
* 为了“保险”扩大事务范围；
* 在 Controller 中定义业务事务；
* 在 Mapper 中编排业务事务；
* 假设事务自动跨线程；
* 通过吞异常让事务继续提交。

---

# 18. Codex 事务判断流程

```text
只是普通快照读？
    ↓ 是
不显式开启事务
```

否则：

```text
是否存在多个写操作必须整体成功？
    ↓ 是
需要事务
```

继续判断：

```text
是否属于一个 Manager 的原子能力？
    ↓ 是
Manager 事务
```

否则：

```text
是否需要多个 Manager 整体一致？
    ↓ 是
Service 事务
```

完成后检查：

* 这个事务是否真的必要；
* 普通快照读是否被无意义地加入事务；
* 事务范围是否过大；
* 是否存在远程调用位于事务中；
* 是否吞掉异常；
* 是否存在事务自调用失效；
* 是否错误使用 `REQUIRES_NEW`；
* 查询后更新是否存在竞争条件；
* 是否错误认为跨线程共享事务。

最终原则：

> 没有一致性需求，就不要增加事务；需要事务时，只覆盖真正需要保证一致性的范围。

