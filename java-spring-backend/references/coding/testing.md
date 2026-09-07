# 测试规范

本文档定义项目测试规范。

核心原则：

> 测试验证业务行为，而不是为了覆盖率而测试实现细节。

---

## 1. 行为变化需要测试

以下情况应补充或更新测试：

* 新增业务功能；
* 修改业务规则；
* 修改状态流转；
* 修复 Bug；
* 修改关键 SQL；
* 修改权限逻辑；
* 修改事务或并发行为。

---

## 2. Bug 修复

Bug 修复在合理情况下按照：

```text
复现 Bug
    ↓
增加失败测试
    ↓
修复
    ↓
测试通过
```

避免只修改代码而没有回归保护。

---

## 3. 测试必须可重复

测试必须：

* 自动执行；
* 结果确定；
* 可重复；
* 互相独立。

禁止依赖：

* 测试执行顺序；
* 本机绝对路径；
* 上一个测试遗留状态；
* 人工观察日志；
* 无控制的外部网络服务。

---

## 4. 测试命名

名称应描述业务行为。

推荐：

```java
shouldRejectAuditWhenPlaceAlreadyRejected()
```

或者：

```java
audit_shouldRejectWhenPlaceAlreadyRejected()
```

同一模块保持统一。

避免：

```java
test1()
testAudit()
testMethod()
```

---

## 5. AAA 结构

复杂测试建议采用：

```text
Arrange
Act
Assert
```

例如：

```java
@Test
void shouldRejectAuditWhenPlaceAlreadyRejected() {
    // Arrange
    Place place = rejectedPlace();

    // Act
    // Assert
    assertThrows(
            PlaceStatusException.class,
            () -> place.audit(APPROVED));
}
```

注释不是强制，代码清晰时可以省略。

---

## 6. 单元测试

单元测试重点验证：

* 业务规则；
* 状态变化；
* 参数边界；
* 异常分支；
* 重要算法。

不要测试：

* Java Getter / Setter；
* 框架本身已经保证的行为；
* 没有业务价值的简单转发代码。

---

## 7. Mock

只 Mock 当前测试之外的依赖。

不要为了让测试通过而 Mock 被测试对象本身。

避免过度 Mock 导致测试实际上只验证：

```text
调用了某个方法
```

而没有验证真实业务结果。

---

## 8. Service 测试

Service 测试重点验证：

* 业务流程；
* Manager / Mapper 调用条件；
* 异常分支；
* 业务结果。

如果只需要单元测试，可以 Mock 外部依赖。

复杂数据库行为应使用集成测试。

---

## 9. Controller 测试

Controller 测试重点验证：

* HTTP 路由；
* 参数绑定；
* 参数校验；
* HTTP 状态；
* Response 格式；
* 权限边界。

不要在 Controller 测试中重复完整 Service 业务测试。

---

## 10. Mapper / SQL 测试

对于复杂 SQL，应考虑数据库集成测试。

特别是：

* PostgreSQL 特有 SQL；
* 动态 SQL；
* 多表 JOIN；
* 时间范围；
* NULL；
* 聚合；
* 分页；
* 唯一约束；
* 条件 UPDATE。

不要只通过 Mock Mapper 验证 SQL 正确性。

---

## 11. 数据库测试

数据库测试应尽量使用与生产环境一致的数据库类型。

本项目使用 PostgreSQL 时，不应默认使用 H2 验证 PostgreSQL 特有 SQL。

如果 SQL 强依赖 PostgreSQL 行为，优先使用真实 PostgreSQL 测试环境或 Testcontainers。

---

## 12. 时间

测试时间相关逻辑时，避免直接依赖：

```java
LocalDateTime.now()
```

导致测试难以控制。

复杂时间逻辑可以考虑注入：

```java
Clock
```

或项目已有时间抽象。

---

## 13. 并发测试

并发问题不能仅靠：

```java
Thread.sleep(...)
```

证明正确。

涉及并发时应验证：

* 最终数据结果；
* 并发冲突；
* 更新行数；
* 乐观锁；
* 唯一约束；
* 重复执行。

并发详细规则读取：

- [concurrency.md](../architecture/concurrency.md)

---

## 14. 事务测试

事务测试应验证最终数据状态，而不是仅检查存在：

```java
@Transactional
```

需要测试：

* 成功提交；
* 异常回滚；
* 多表一致性；
* 必要的隔离和锁行为。

事务详细规范读取：

- [transactions.md](../architecture/transactions.md)

---

## 15. 测试数据

测试数据应：

* 最小化；
* 含义明确；
* 与测试场景相关。

避免构造大量与测试目标无关的数据。

推荐使用统一 Fixture / Builder，前提是确实能够提高可读性。

---

## 16. 断言

必须使用明确断言。

禁止通过：

```java
System.out.println(...)
```

人工判断测试结果。

断言应关注关键业务结果。

不要为了增加断言数量而验证大量无关字段。

---

## 17. 不得修改测试迎合错误实现

如果已有测试失败，首先判断：

```text
实现错了？
还是
需求真的变了？
```

只有需求明确发生变化时才修改原有断言。

禁止：

* 删除失败测试；
* 注释失败测试；
* 弱化断言；
* 添加无意义等待；

来让代码“通过”。

---

## 18. Codex 测试检查

修改代码后判断：

1. 是否改变了已有行为；
2. 是否存在重要边界条件；
3. 是否属于 Bug 修复；
4. 是否涉及 SQL、事务、权限或并发；
5. 是否已有测试能够覆盖本次变化。

执行测试后必须如实汇报：

```text
执行了哪些测试
哪些通过
哪些未执行
为什么未执行
```

不得声明没有实际执行的测试已经通过。

