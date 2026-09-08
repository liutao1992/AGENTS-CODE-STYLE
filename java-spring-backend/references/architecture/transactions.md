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

## 2.1 谨慎使用 `@Transactional`

不要因为一个方法位于 Service / Manager，或者方法中存在数据库写操作，就简单添加：

```java
@Transactional(rollbackFor = Exception.class)
```

然后认为事务问题已经解决。

新增事务前必须先回答：

```text
哪些数据库操作必须作为一个整体成功或失败？
事务从哪里开始、在哪里结束？
事务中是否包含不需要锁和一致性保证的工作？
是否存在远程调用、文件 IO、等待或长耗时计算？
查询后修改是否仍然存在并发竞争？
```

事务注解只是实现事务边界的 Spring 机制，不能替代对一致性范围、并发和回滚语义的设计。

原则：

> 先确定一致性边界，再使用 `@Transactional`；不要先加注解，再倒推事务范围。

## 2.2 优先收敛数据库操作

同一个原子业务动作中，如果多个数据库操作可以在不破坏可读性、约束和并发语义的前提下合理合并，应优先减少不必要的数据库往返和事务持有时间。

例如可以根据真实场景评估：

```text
多次逐条 INSERT
→ Batch / 批量写入

先 SELECT 再 UPDATE，且状态条件可直接表达
→ 条件 UPDATE

多个职责相同的重复查询
→ 一次查询后复用结果
```

但“合并数据库操作”不等于：

* 把多个无关业务写进一条巨大 SQL；
* 为了减少 Mapper 调用破坏业务可读性；
* 把不同一致性边界强行合并；
* 绕过必要的唯一约束、锁或状态校验。

事务方法中应尽量只保留与当前一致性边界直接相关的工作。

推荐顺序：

```text
事务外：参数准备 / 不依赖事务的计算 / 可提前完成的外部读取
                    ↓
事务内：必要查询 / 业务一致性判断 / 必要数据库写入
                    ↓
事务提交
                    ↓
事务外：不要求与数据库原子提交的后续工作
```

不能为了“减少业务逻辑”把依赖数据库当前状态、锁或原子性的关键业务判断移出事务；是否移出必须以一致性语义为准。

原则：

> 减少事务内非必要工作，而不是减少必要业务规则；能在更短数据库路径内保证相同语义时，优先选择更短路径。

---

# 3. Manager 层事务

当一个 Manager 内部组合多个数据库写操作，并且这些操作必须作为整体成功或失败时，优先在 Manager 层定义事务。

例如：

```java
@Transactional(rollbackFor = Exception.class)
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
@Transactional(rollbackFor = Exception.class)
public void registerCase(...) {
    caseManager.create(...);
    materialManager.register(...);
    personManager.bind(...);
}
```

原则：

> 事务覆盖业务真正要求原子性的范围，不要为了保险扩大事务。

## 4.1 Service 准备数据，Manager 收口原子数据库操作

如果事务只对应一个可以独立复用的数据库原子能力，优先考虑把不依赖事务的数据准备放在 Service，把真正需要整体提交 / 回滚的数据库操作收口到 Manager。

例如：

```java
public void createPlace(PlaceCreateRequest request, Operator operator) {
    PlaceDO place = buildPlace(request, operator);

    // 这里可以完成不依赖数据库事务的参数准备、轻量转换等工作。
    placeManager.create(place);
}
```

Manager 只覆盖真正需要原子性的数据库操作：

```java
@Transactional(rollbackFor = Exception.class)
public void create(PlaceDO place) {
    placeMapper.insert(place);
    auditRecordMapper.insert(buildCreateRecord(place));
}
```

这种方式的目标是：

```text
Service
→ 业务流程、事务外数据准备

Manager
→ 可复用原子数据库能力
→ 较短事务边界
```

但不要机械规定“所有事务必须下沉 Manager”。如果完整业务用例需要多个 Manager 的写操作共同提交 / 回滚，事务仍应提升到能够覆盖完整一致性范围的 Service。

也不要为了把 Service 参数传给 Manager，机械创建没有真实职责的 DTO / BO。已有 Request、DO、DTO、BO 或少量清晰参数能够正确表达语义时直接使用。

如果某个业务校验必须依赖事务内读取、行锁、唯一约束结果或一致性快照，它仍应位于事务边界内，不能为了让 Manager “只剩 Mapper 调用”而错误移到 Service 的事务外。

原则：

> 能在 Service 准备的数据放事务外；依赖数据库一致性、锁和原子性的判断与写入留在事务内。

## 4.2 `TransactionTemplate` 作为精确事务边界的另一种实现

除了 `@Transactional`，Spring 的 `TransactionTemplate` 也可以用于显式控制事务代码块。

适合优先评估 `TransactionTemplate` 的场景包括：

* 一个方法中只有一小段数据库操作需要事务；
* 事务前后存在明显的远程调用、文件 IO、复杂计算或数据准备，希望明确排除在事务外；
* 使用 `@Transactional` 会因为自调用或代理边界导致事务语义不直观；
* 需要在代码中非常明确地看出事务从哪里开始、在哪里结束；
* 目标项目已经稳定使用编程式事务处理这类局部事务边界。

例如：

```java
public void createPlace(PlaceCreateRequest request, Operator operator) {
    PlaceDO place = buildPlace(request, operator);

    transactionTemplate.executeWithoutResult(status -> {
        placeMapper.insert(place);
        auditRecordMapper.insert(buildCreateRecord(place));
    });
}
```

也可以由 Service 完成事务外准备，再调用一个使用 `TransactionTemplate` 的 Manager 原子能力。

`TransactionTemplate` 的优势是事务范围在代码中直接可见，并且不依赖 `@Transactional` 的方法代理调用方式；但它不是默认比声明式事务“更高级”或“更安全”。

如果一个公开方法本身就是清晰、稳定的完整事务边界，优先使用声明式：

```java
@Transactional(rollbackFor = Exception.class)
public void create(...) {
    ...
}
```

通常更简单。

如果只有局部代码块需要事务，或者需要精确缩短事务持有时间，再优先评估：

```java
transactionTemplate.execute(...)
transactionTemplate.executeWithoutResult(...)
```

使用 `TransactionTemplate` 时必须注意：

* `rollbackFor` 是 `@Transactional` 的配置，不适用于 `TransactionTemplate`；
* 回调中异常如果被捕获并吞掉，事务不会因为“曾经出现异常”自动知道应该回滚；
* 需要回滚时应让失败异常正确传播，或者在项目确有恢复逻辑时明确使用 `status.setRollbackOnly()`；
* 不要在事务回调内放入 HTTP / RPC、文件上传、长时间等待等非必要工作；
* 传播、隔离、超时等配置仍要符合真实事务语义和项目现有 `PlatformTransactionManager` 配置；
* 不要为了绕开分层问题，让 Service 因使用 `TransactionTemplate` 就大量直接承担 Mapper 编排；类职责仍按分层规范判断。

原则：

> `@Transactional` 适合清晰的方法级事务边界；`TransactionTemplate` 适合需要显式、局部、精确控制的事务代码块。二者都只是实现手段，一致性边界才是事务设计本身。

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
@Transactional(rollbackFor = Exception.class)
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

@Transactional(rollbackFor = Exception.class)
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

如果选择 `TransactionTemplate`，事务由模板调用显式创建，不依赖同一个 Bean 的 `@Transactional` 代理自调用；但不能为了绕开代理问题就机械改成编程式事务，仍应优先判断哪种实现更符合职责和可读性。

---

# 14. 异常与回滚

禁止在事务中捕获异常后直接吞掉：

```java
@Transactional(rollbackFor = Exception.class)
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

Spring 默认通常对 `RuntimeException` 和 `Error` 回滚，对普通 Checked Exception 不自动回滚。

为了让新增事务的回滚边界在代码中更明确，本 Skill 默认约定：**新建或显著修改的业务事务方法，使用 `@Transactional(rollbackFor = Exception.class)`。**

例如：

```java
@Transactional(rollbackFor = Exception.class)
public void createPlace(...) {
    ...
}
```

但以下情况优先遵循目标项目现有事务契约，不得为了统一注解无授权修改：

* 项目已经统一封装事务注解；
* 项目明确使用更具体的 `rollbackFor` / `noRollbackFor`；
* 某类 Checked Exception 按业务契约明确不应回滚；
* 历史公共方法已经形成稳定回滚语义，修改会产生兼容风险。

不得通过：

```text
所有方法都加 @Transactional(rollbackFor = Exception.class)
```

来替代对事务必要性和事务范围的判断。`rollbackFor = Exception.class` 只解决“已确定需要事务后如何明确回滚范围”的问题。

使用 `TransactionTemplate` 时没有 `rollbackFor` 参数。回调中的失败应按模板事务语义正确传播；如果代码捕获异常并选择不再抛出，但业务仍要求当前事务回滚，必须显式决定是否调用：

```java
status.setRollbackOnly();
```

不能既吞掉异常，又默认认为模板事务会自动回滚。

原则：

> 先确认需要事务，再显式定义回滚语义；声明式事务默认使用 `Exception.class`，但项目已有更具体契约时以项目为准；编程式事务则通过异常传播或显式 rollback-only 表达回滚。

---

# 15. 避免长事务

事务应尽可能短。

事务中尽量避免：

* HTTP / RPC；
* 文件上传；
* 第三方接口调用；
* 长时间计算；
* 阻塞等待；
* `sleep`；
* 与当前一致性边界无关的数据准备和对象转换；
* 可以在事务外完成的大量内存计算。

推荐：

```text
准备参数
   ↓
开始事务
   ↓
必要数据库操作和一致性判断
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

如果某个业务判断必须基于事务内查询结果、数据库锁或同一一致性视图，则应保留在事务中，不能为了缩短事务机械移出。

---

# 16. 事务与线程切换

Spring 事务通常绑定当前线程。

切换线程后，不应假设事务自动传播。

例如：

```java
@Transactional(rollbackFor = Exception.class)
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
* 通过吞异常让事务继续提交；
* 用 `rollbackFor = Exception.class` 掩盖不清晰的事务边界；
* 为减少 Mapper 调用把无关业务强行合并成巨大 SQL 或巨大事务；
* 为了避开 `@Transactional` 代理规则就无条件改用 `TransactionTemplate`；
* 使用 `TransactionTemplate` 后把大量无关业务、远程调用或等待一起包进事务回调。

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
Service 准备事务外数据
→ Manager 收口事务
```

否则：

```text
是否需要多个 Manager 整体一致？
    ↓ 是
Service 事务
```

确定事务边界后，再选择实现方式：

```text
整个公开方法就是稳定事务边界？
    ↓ 是
@Transactional(rollbackFor = Exception.class)

只有局部代码块需要事务，或需要显式精确控制边界？
    ↓ 是
评估 TransactionTemplate
```

继续检查：

```text
能否通过 Batch / 条件更新 / 合理合并数据库操作缩短路径？
    ↓
事务内是否存在可移出的远程调用、IO、等待或长耗时计算？
    ↓
回滚语义是否明确？
```

完成后检查：

* 这个事务是否真的必要；
* 普通快照读是否被无意义地加入事务；
* 事务范围是否过大；
* 是否可以合理减少数据库往返；
* 是否为了合并操作制造巨大 SQL 或混合不同业务边界；
* 是否存在远程调用、文件 IO、阻塞等待或非必要复杂计算位于事务中；
* 必须依赖事务状态的业务判断是否被错误移到事务外；
* 是否适合由 Service 事务外准备数据，再由 Manager 收口原子数据库操作；
* `@Transactional` 与 `TransactionTemplate` 的选择是否来自真实边界，而不是个人偏好；
* `rollbackFor` 是否符合项目约定，新增声明式业务事务是否默认明确为 `Exception.class`；
* `TransactionTemplate` 回调中的异常是否被错误吞掉，是否需要显式 rollback-only；
* 是否存在事务自调用失效；
* 是否错误使用 `REQUIRES_NEW`；
* 查询后更新是否存在竞争条件；
* 是否错误认为跨线程共享事务。

最终原则：

> 没有一致性需求，就不要增加事务；需要事务时，优先缩短数据库路径和事务持有时间，只覆盖真正需要保证一致性的范围，并根据边界清晰度选择 `@Transactional` 或 `TransactionTemplate`，明确回滚语义。
