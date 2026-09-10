---
name: backend-code-review
description: 审查 Java、Spring Boot、MyBatis、PostgreSQL 后端代码或变更，按需加载团队规范，检查正确性、分层、API、数据库、事务、并发、安全和测试，并输出有定位与证据的 findings；纯审查默认只读。
---

# Backend Code Review

评估指定后端变更是否正确、是否破坏目标项目契约，以及是否违反适用团队规范。

本 Skill 只负责：

```text
审查范围
审查流程
规范路由
证据要求
严重程度
输出格式
```

具体 Java、分层、API、Spring、MyBatis、SQL、数据库、事务、并发和测试规则直接读取 `java-spring-backend/references`，不在本文件维护第二套规范。

---

## 1. 边界与权限

- 先阅读目标项目适用的 `AGENTS.md`。
- 纯审查默认只读，不自动修复、格式化、重命名、安装依赖、更新快照、发布评论或创建独立代理。
- 开发任务调用本 Skill 自检时，修复权限来自原开发任务，并且只限原任务范围。
- 与 `java-spring-backend` 保持同级目录，直接读取它的 references，不执行开发 Skill 的实施流程。
- references 缺失时应明确说明规范检查受限，不得宣称完成了完整团队规范验证。

---

## 2. 确定审查范围

优先使用用户明确指定的：

```text
文件
目录
commit
commit range
PR
branch diff
```

如果用户只说“审查当前变更”而没有指定比较对象，应覆盖当前工作区可识别变更：

```text
unstaged changes
+
staged changes
+
与当前任务相关的 untracked files
```

实际使用 Git 时，应先确认工作区状态，再分别读取工作区和暂存区差异；不要只执行单一 `git diff` 后就假定范围完整。

未指定范围时：

* 不默认审查全仓库；
* 不猜测远端基线或比较分支；
* 不把无关历史问题混成本次 finding。

如果无法确定比较基线，应说明实际审查了什么。

---

## 3. 审查流程

1. **确认范围。** 明确本次审查对象及是否包含 staged / unstaged / untracked。
2. **阅读完整差异。** 不只看单行 patch；读取受影响方法、调用者、模型、SQL、配置和相关测试。
3. **建立项目上下文。** 搜索类似实现、已有契约、构建配置和可用近期 Git 历史；不存在时如实说明。
4. **选择规范。** 按路由表只加载实际涉及的 references，不递归加载全部规范。
5. **沿数据流验证。** 确认输入从哪里来、经过哪些层、最终影响什么状态或契约。
6. **区分新旧问题。** 优先报告本次引入或加剧的问题；既有无关问题不混入 findings。
7. **必要验证。** 运行与范围相关、不会无授权改写代码或共享环境的已有测试、静态检查和架构检查。
8. **形成结果。** 合并同根因问题，按严重程度排序，并说明实际验证范围。

---

## 4. 规范路由

| 涉及领域 | 加载规范 | 主要判断 |
| --- | --- | --- |
| Java 实现 | [Java](../java-spring-backend/references/coding/java.md) | 命名、类设计、常量、Enum、魔法值、POJO 默认值、参数、Null、集合、异常实现、日志、格式 |
| 项目物理目录 / module | [项目结构](../java-spring-backend/references/architecture/project-structure.md) | 业务模块位置、公共目录、物理组织 |
| 分层 / 模型 / 职责 Package / SOLID | [分层](../java-spring-backend/references/architecture/layering.md) | 职责、依赖、模型边界、跨模块、过度设计 |
| Spring Framework | [Spring](../java-spring-backend/references/coding/spring.md) | MVC、Validation、DI、Bean、Proxy、Advice |
| HTTP API | [API](../java-spring-backend/references/api/api-design.md) | URL、Method、Request/VO、响应、错误、兼容、分页、幂等 |
| 异常跨层 | [异常处理](../java-spring-backend/references/architecture/error-handling.md) | 转换、cause、日志归属、对外泄漏 |
| MyBatis / MyBatis-Plus | [MyBatis / MyBatis-Plus](../java-spring-backend/references/coding/mybatis.md) | Mapper/DAO、BaseMapper、Wrapper、XML 业务常量、绑定、ResultMap、TypeHandler、集合契约 |
| SQL | [SQL](../java-spring-backend/references/database/sql.md) | 正确性、范围、注入、安全、PostgreSQL、性能证据 |
| 数据库 Schema | [数据库设计](../java-spring-backend/references/database/database-design.md) | 类型、Null、约束、索引、Migration、兼容 |
| 事务 / 锁 / 一致性 | [事务](../java-spring-backend/references/architecture/transactions.md) | 必要性、范围、回滚、传播、隔离、竞态 |
| 并发 / 异步 | [并发](../java-spring-backend/references/architecture/concurrency.md) | 收益、线程池、上下文、异常、共享状态、资源容量 |
| Bug / 行为变化 / 测试 | [测试](../java-spring-backend/references/coding/testing.md) | 回归保护、边界、断言、集成验证 |
| 权限 / 租户 / 数据范围 / 安全 | 目标项目已有安全规范、契约和实现 | 越权、隔离、数据泄漏、凭证安全 |

多领域只取必要并集。例如：

```text
Controller URL 变化
→ API + 必要的 Spring

只调整 Mapping 注解
→ Spring

新增 VO Package
→ 分层 + Java

魔法值 / 大而全常量类 / 固定值域 / POJO 默认值
→ Java

MyBatis List<T> 后出现 Null 兜底
→ MyBatis + Java

MyBatis-Plus Mapper 未继承 BaseMapper，或出现 Wrapper 条件构建
→ MyBatis

Mapper XML 写死业务状态 / 类型编码
→ MyBatis；如果同时判断 SQL 正确性再加 SQL

TransactionTemplate 局部事务
→ 事务 + Spring

CompletableFuture 数据库操作
→ 并发 + 必要的事务
```

---

## 5. Finding 成立条件

只报告有明确证据支持的问题。至少满足一类：

* 可复现错误行为；
* 明确调用链或数据流风险；
* 已有 API、数据库、权限、事务或业务契约被破坏；
* 目标项目稳定实现被本次变更无依据绕过；
* 适用 reference 的明确规则被违反且不存在项目例外；
* 安全、数据完整性或并发风险有具体触发条件。

不得仅凭：

```text
个人偏好
“更优雅”
以后可能扩展
某种架构流派
代码看起来不舒服
```

形成 finding。

如果缺少关键契约无法确认，标记为“待确认”或说明证据不足，不把推测写成已确认缺陷。

---

## 6. 项目契约优先

reference 中的默认推荐不能自动覆盖目标项目已有稳定约定。

形成规范类 finding 前，必须检查当前项目是否已有明确：

```text
API / 序列化契约
模型与 Package 约定
module 结构
Spring MVC 风格
Validation 边界
异常和日志体系
集合 Null 契约
事务 / rollback 约定
分页结构
数据库命名与迁移机制
```

如果本次变更只是延续稳定历史契约，不因 Skill 默认风格不同形成 finding。

同样，不为了“统一”要求用户顺手迁移无关历史代码。

---

## 7. 容易误判的问题

以下问题必须先确认关键前提，再报告。

### 7.1 重复结构校验

详细规则读取 `spring.md`。

只有同时确认：

1. 当前调用路径已经经过可信结构校验边界；
2. 业务层判断与该结构约束语义相同，而不是独立业务规则；
3. 不存在其他未校验入口要求当前方法承担公共输入契约；

才报告“重复结构校验”。

不要只因同一字段同时出现 Bean Validation 和手写判断就机械报错。

### 7.2 集合 Null 防御

通用规则读取 `java.md`；MyBatis 来源同时读取 `mybatis.md`。

只有确认：

1. 返回值是集合；
2. 当前来源明确保证非 Null；
3. 项目自定义代理 / 插件 / 遗留实现没有改变契约；
4. 当前判断不是兼容已知 nullable 第三方来源；

才报告无意义 Null 防御。

不要泛化成“所有集合都不允许 Null”。

### 7.3 SOLID / 架构问题

详细规则读取 `layering.md`。

必须指出具体职责、依赖或契约风险，不报告：

```text
建议遵循 SOLID
建议抽接口
建议加 Strategy / Factory
```

这种没有触发条件的泛化评论。

### 7.4 性能问题

没有真实数据、执行计划、容量或调用频率证据时，不把“可能更快”写成已确认性能缺陷。

SQL 结构上明确存在 N+1、无界查询、错误索引假设等风险时，应说明触发条件和证据范围。

### 7.5 事务问题

详细规则读取 `transactions.md`。

不能因为存在两个 Mapper、存在写操作或没有 `@Transactional` 就自动报告事务问题。必须指出哪些操作需要共同成功/回滚，以及当前实现如何破坏该一致性需求。

---

## 8. 严重程度

### P0 — 阻断 / 灾难性

例如：

* 明确导致大范围数据破坏；
* 严重认证绕过或敏感数据暴露；
* 生产核心能力必然不可用。

### P1 — 高优先级

例如：

* 主要功能错误；
* 数据一致性或权限边界被破坏；
* 已发布关键 API 明确不兼容；
* 高概率并发错误或错误事务提交。

### P2 — 中优先级

例如：

* 特定条件下的功能错误；
* 可维护性问题已经导致真实误用风险；
* 明确违反稳定项目边界，未来修改容易产生错误。

### P3 — 低优先级

例如：

* 局部清晰度或一致性问题；
* 有真实维护成本，但当前不影响主要行为。

严重程度依据实际影响和触发概率，不依据规则名称或代码风格决定。

---

## 9. Finding 写法

每条 finding 应包含：

```text
[P1/P2/P3] 简洁标题
位置：文件:行号或最小相关范围
问题：具体哪里错误
触发：什么输入 / 调用路径 / 状态下发生
影响：会导致什么结果
依据：项目契约、代码路径、测试或适用 reference
建议：最小方向，不必展开成完整重构方案
```

优先给出能够验证的因果链：

```text
变更
→ 触发条件
→ 错误行为
→ 用户 / 数据 / 契约影响
```

不要把大段教程写进 finding。

---

## 10. 审查输出

有 findings 时：

1. 先按严重程度列 findings；
2. 再列必要的待确认事项；
3. 最后简短说明验证范围和未执行检查。

没有 findings 时明确说明：

> 在实际审查范围内未发现有足够证据支持的缺陷。

同时仍应说明：

* 审查了什么；
* 实际运行了什么验证；
* 哪些验证未运行以及原因。

最终原则：

> Review Skill 负责证明“这里为什么是问题”；领域 reference 负责定义“正确规则是什么”。
