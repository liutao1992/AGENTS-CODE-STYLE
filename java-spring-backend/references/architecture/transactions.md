# 事务规范

本文档定义事务必要性、一致性范围、事务边界、回滚、隔离、传播和数据库并发语义。

本文负责回答：

> 哪些操作必须共同提交或回滚、事务边界应该覆盖什么，以及用什么数据库一致性手段保证正确性。

Spring Proxy、`@Transactional` 和 `TransactionTemplate` 的框架 API 机制读取：

- [Spring](../coding/spring.md)

核心原则：

> 事务由数据一致性需求决定，不由方法名称、Service / Manager 层级、Mapper 数量或注解形式决定。

> 先确定一致性边界，再选择 `@Transactional` 或 `TransactionTemplate`；二者都是实现手段，不是架构分层方式。

> 普通快照读默认不显式开启事务；需要事务时尽量缩短持有时间。

---

# 1. 什么时候需要事务

通常需要事务：

* 多个写操作必须全部成功或全部回滚；
* 多张表修改共同表达一个原子业务动作；
* 查询后修改存在并发一致性要求；
* 使用数据库行锁；
* 多个查询必须共享特定一致性视图；
* 业务明确要求数据库操作形成同一个提交边界。

通常不因为以下情况自动开启事务：

* 单条普通查询；
* 多个相互独立的普通查询；
* 普通统计；
* 仅仅调用多个 Mapper；
* 仅仅方法位于 Service / Manager；
* 仅仅执行了一条 INSERT / UPDATE / DELETE。

原则：

> “访问数据库”不是事务需求；“必须共同保持一致”才是事务需求。

---

# 2. 事务边界由一致性范围决定

先回答：

```text
哪些数据库操作必须作为一个整体成功或失败？
```

再决定事务放在哪里。

常见：

```text
Manager
→ 一个可复用原子数据库能力
```

或者：

```text
Service
→ 跨多个 Manager 的完整业务事务
```

不要机械规定：

```text
所有事务都在 Service
```

也不要机械规定：

```text
所有事务都下沉 Manager
```

原则：

> 谁拥有完整的一致性边界，谁拥有事务；层名本身不是依据。

---

# 3. 谨慎使用 `@Transactional`

不要看到写操作就直接添加：

```java
@Transactional(rollbackFor = Exception.class)
```

新增事务前至少确认：

```text
需要共同提交 / 回滚的操作是什么？
事务从哪里开始、在哪里结束？
事务内是否存在远程调用、文件 IO、等待或长耗时计算？
查询后修改是否存在竞争？
是否需要锁、条件更新或唯一约束？
失败时哪些异常应该导致回滚？
```

`@Transactional` 只实现已经确定的事务边界，不能替代一致性设计。

---

# 4. Service 准备数据，Manager 收口原子能力

当一个事务对应一个可以独立复用的数据库原子能力时，可以将不依赖事务的数据准备放在 Service，再由 Manager 收口事务内操作。

例如：

```java
public void createPlace(PlaceCreateRequest request, Operator operator) {
    PlaceDO place = buildPlace(request, operator);
    placeManager.create(place);
}
```

Manager：

```java
@Transactional(rollbackFor = Exception.class)
public void create(PlaceDO place) {
    placeMapper.insert(place);
    auditRecordMapper.insert(buildCreateRecord(place));
}
```

这种模式表达：

```text
Service
→ 不依赖事务的业务准备 / 流程

Manager
→ 可复用原子数据库能力
```

但这不是强制模板。

如果完整业务用例要求多个 Manager 的写操作共同提交 / 回滚：

```java
@Transactional(rollbackFor = Exception.class)
public void registerCase(...) {
    caseManager.create(...);
    materialManager.register(...);
    personManager.bind(...);
}
```

事务应提升到能够覆盖完整一致性范围的 Service。

不要为了使用 Manager 机械创建无职责中间层。

---

# 5. 事务外与事务内工作

可以放事务外的工作通常包括：

* 不依赖数据库当前事务状态的参数准备；
* 纯内存转换；
* 可提前完成且不要求与数据库原子提交的外部读取；
* 与当前一致性边界无关的对象组装。

必须根据语义留在事务内的工作可能包括：

* 基于事务内查询结果的业务判断；
* 依赖行锁的判断；
* 依赖一致性快照的判断；
* 与写操作共同保证原子性的必要校验；
* 必要数据库写入。

推荐思路：

```text
事务外：准备 / 非必要 IO
        ↓
事务内：必要读取 → 一致性判断 → 必要写入
        ↓
提交
        ↓
事务外：不要求数据库原子提交的后续工作
```

原则：

> 缩短的是非必要事务时间，不是把必要业务规则移出一致性边界。

---

# 6. 合理收敛数据库路径

同一原子操作中，如果在不改变语义的前提下可以减少数据库往返，可以评估：

```text
逐条 INSERT
→ Batch

先 SELECT 再 UPDATE，且状态条件可以在 SQL 中表达
→ 条件 UPDATE

重复读取同一结果
→ 一次读取后复用
```

但不要为了减少 Mapper 调用：

* 创建巨大 SQL；
* 合并无关业务；
* 绕过唯一约束、锁或状态校验；
* 把不同一致性边界强行放入一个大事务。

---

# 7. `@Transactional` 与 `TransactionTemplate`

二者都只是 Spring 事务实现方式，不决定 Service / Manager 职责。

选择依据是事务边界表达方式，而不是“哪一层应该用哪个 API”。

## 7.1 方法级完整边界

一个公开方法本身就是清晰、稳定的完整事务边界时，声明式事务通常更简单：

```java
@Transactional(rollbackFor = Exception.class)
public void create(...) {
    ...
}
```

## 7.2 局部精确边界

一个方法只有局部代码块需要事务，或希望显式缩短事务持有范围时，可以评估：

```java
transactionTemplate.execute(...)
transactionTemplate.executeWithoutResult(...)
```

例如，**拥有该原子能力的 Manager** 可以：

```java
public void create(PlaceDO place) {
    transactionTemplate.executeWithoutResult(status -> {
        placeMapper.insert(place);
        auditRecordMapper.insert(buildCreateRecord(place));
    });
}
```

如果完整一致性边界由 Service 拥有，Service 也可以使用 `TransactionTemplate` 包住该完整业务事务。

因此不要推导：

```text
TransactionTemplate → 必须放 Service
```

或：

```text
@Transactional → 必须放 Manager
```

原则：

> 先确定谁拥有一致性边界，再决定该方法用声明式还是编程式事务。

## 7.3 使用 `TransactionTemplate` 时

注意：

* `rollbackFor` 只属于 `@Transactional`，不适用于 `TransactionTemplate`；
* 回调内捕获并吞掉异常后，事务不会因为“曾发生异常”自动知道应回滚；
* 需要回滚时应让失败正确传播，或在项目真实恢复逻辑中明确 `setRollbackOnly()`；
* 传播、隔离、超时等仍必须符合业务语义；
* 不在回调内放入无必要 HTTP / RPC、文件 IO、等待和长耗时计算；
* 不为了绕过 Proxy 或分层问题把所有声明式事务机械改成模板事务。

Spring API 和 Proxy 细节统一读取 `spring.md`。

---

# 8. `rollbackFor` 与回滚语义

Spring 默认通常对 `RuntimeException` 和 `Error` 回滚，对普通 Checked Exception 不自动回滚。

本 Skill 的默认约定只适用于：

```text
新建一个业务事务边界
或
当前任务明确要求调整事务 / 回滚语义
```

且目标项目没有更具体约定时，默认使用：

```java
@Transactional(rollbackFor = Exception.class)
```

这不是普通代码修改时的迁移规则。

例如已有：

```java
@Transactional
public void audit(...) {
    ...
}
```

用户只修改普通业务逻辑、没有要求改变事务语义时，不得因为“本 Skill 默认”顺手改成：

```java
@Transactional(rollbackFor = Exception.class)
```

因为这可能改变 Checked Exception 的回滚行为。

以下情况优先遵循项目现有契约：

* 项目已有统一事务注解；
* 已明确 `rollbackFor` / `noRollbackFor`；
* 某类 Checked Exception 按业务明确不应回滚；
* 历史公共方法已经形成稳定事务语义；
* 修改会造成兼容风险。

原则：

> 默认规则用于设计新的事务边界，不用于无授权改变已有事务语义。

---

# 9. 普通快照读

普通查询默认不因为：

```text
查询多个表
调用多个 Mapper
调用多个 Manager
```

就显式开启事务。

也不要因为方法叫：

```text
get / find / list / query / select
```

就机械添加：

```java
@Transactional(readOnly = true)
```

只有真实需要事务边界时才使用读事务。

---

# 10. 多查询一致性快照

如果多个查询必须基于同一一致性数据视图，需要同时考虑：

* 事务边界；
* PostgreSQL 快照语义；
* 隔离级别。

PostgreSQL 默认 `READ COMMITTED` 下，同一事务中的不同 SQL 仍可能看到不同时间点已经提交的数据。

因此：

```java
@Transactional(readOnly = true)
```

不天然等于“整个方法所有 SELECT 使用完全相同快照”。

明确要求固定快照时应根据实际隔离级别设计。

---

# 11. 查询后修改与竞争

典型风险：

```text
SELECT
↓
Java 判断
↓
UPDATE
```

多个请求可能同时读到相同旧状态。

仅增加事务不一定解决竞争。

根据场景评估：

* 条件 UPDATE；
* 乐观锁；
* 悲观锁；
* 唯一约束；
* 合适隔离级别。

简单状态竞争优先评估条件更新：

```sql
UPDATE place
SET zt = #{targetStatus}
WHERE id = #{id}
  AND zt = #{expectedStatus}
```

然后检查受影响行数。

---

# 12. 乐观锁与悲观锁

## 12.1 乐观锁

冲突概率较低且允许检测冲突时，可以使用 `version` 等机制。

更新行数为 0 表示发生竞争；重试必须有限且保证操作语义允许重试。

## 12.2 悲观锁

确实需要数据库行锁时可以使用：

```sql
SELECT ... FOR UPDATE
```

必须位于明确事务中，并考虑：

* 锁持有时间；
* 死锁；
* 加锁顺序；
* 吞吐；
* 是否可由条件更新 / 乐观锁替代。

不要默认使用悲观锁解决所有竞争。

---

# 13. 事务传播

默认优先 Spring 常规：

```text
REQUIRED
```

没有明确业务语义时，不随意使用：

```text
REQUIRES_NEW
NESTED
NOT_SUPPORTED
NEVER
```

尤其 `REQUIRES_NEW` 会创建独立事务，其提交可能不会随着外层事务回滚。

只有业务确实要求独立提交时才使用。

---

# 14. 异常与事务

事务内不要捕获失败后吞掉：

```java
try {
    mapper.update(...);
} catch (Exception ex) {
    log.error("failed", ex);
}
```

否则事务可能继续提交。

捕获异常时必须明确：

* 是否恢复；
* 是否继续抛出；
* 是否转换并保留 cause；
* 当前事务是否仍应回滚。

异常跨层规则读取 `error-handling.md`。

---

# 15. 避免长事务

事务中尽量避免：

* HTTP / RPC；
* 文件上传；
* 第三方调用；
* 长时间计算；
* 阻塞等待 / sleep；
* 与一致性边界无关的大量对象转换；
* 可以提前完成的外部读取。

但依赖事务内状态、锁或一致性视图的必要判断不能机械移出。

---

# 16. Spring Proxy 与自调用

`@Transactional` 是否经过 Spring Proxy、同对象自调用是否绕过代理、方法可见性等属于 Spring 机制，统一读取：

- [spring.md](../coding/spring.md)

本文只要求：

> 不能仅因为代码上存在 `@Transactional` 就认定实际调用路径已经处于目标事务中。

不在事务规范重复维护完整 Spring Proxy 教程。

---

# 17. 事务与线程切换

Spring 常规事务通常绑定当前线程。

不得假设：

```text
当前线程事务
→ 自动传播到 CompletableFuture / @Async / 自建线程
```

事务内数据库操作不得为了性能直接拆到多个线程并假设仍属于同一事务。

线程、Executor、上下文传播读取 `concurrency.md`。

---

# 18. 禁止事项

禁止：

* 给所有 Service / Manager 方法统一加事务；
* 给所有查询方法统一加 `readOnly`；
* 因为调用多个 Mapper / Manager 就开启事务；
* 为保险扩大事务范围；
* 在 Controller / Mapper 编排业务事务；
* 为了事务机械创建 Manager；
* 把 `TransactionTemplate` 当成分层模式；
* 普通业务修改时无授权改变已有 `rollbackFor` 语义；
* 事务中吞异常；
* 假设事务自动跨线程；
* 为减少 Mapper 调用创建巨大 SQL / 巨大事务。

---

# 19. Codex 事务判断流程

```text
是否存在明确一致性需求？
    ↓ 否
不新增事务
```

如果有：

```text
哪些操作必须共同提交 / 回滚？
        ↓
谁拥有完整一致性边界？
        ↓
Manager 原子能力 / Service 完整业务事务
        ↓
哪些工作可以安全移到事务外？
        ↓
是否需要条件更新 / 锁 / 唯一约束 / 特定隔离级别？
        ↓
选择 @Transactional 或 TransactionTemplate
        ↓
明确回滚语义
```

检查：

* 事务是否真的必要；
* 边界是否完整且足够小；
* 是否为了事务机械改变分层；
* 是否存在无必要远程调用、IO、等待；
* 关键业务判断是否被错误移出事务；
* 新事务边界的 `rollbackFor` 是否符合默认与项目约定；
* 已有事务语义是否被无授权改变；
* 是否存在竞态、锁、传播、快照问题；
* 是否错误假设跨线程事务传播。

最终原则：

> 一致性边界决定事务所有者；`@Transactional` 和 `TransactionTemplate` 只决定实现形式。没有一致性需求不加事务，需要事务时只覆盖真正必须共同保持一致的范围。