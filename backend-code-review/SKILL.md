---
name: backend-code-review
description: 审查 Java、Spring Boot、MyBatis、PostgreSQL 后端代码或变更，按需加载团队规范，检查正确性、分层、SOLID、API、数据库、事务、并发、安全和必要测试，并输出有定位与证据的发现；纯审查默认只读。
---

# Backend Code Review

评估指定后端变更是否正确、符合目标项目和团队规范，并指出可操作的问题。

本 Skill 负责：

```text
审查流程
证据要求
严重程度
输出格式
规范路由
```

不维护第二套 Java、分层、SOLID、API、MyBatis、SQL、事务或测试规范；具体规则直接读取 `java-spring-backend/references`。

## 边界与权限

- 先阅读目标项目适用的 `AGENTS.md`。
- 纯审查默认只读，不自动修复、重命名、格式化、更新快照、安装依赖、发布评论或创建独立代理。
- 开发任务调用本 Skill 自检时，修复权限来自原开发任务，并且只限原任务范围；本 Skill 不扩大权限。
- 与 `java-spring-backend` 保持同级目录，直接读取其 references，不加载开发 Skill 的实施流程。
- 配套 references 缺失时应明确说明规范检查受限；仍可完成有证据的正确性审查，但不得宣称完整验证团队规范。

---

## 审查流程

1. **确定范围。** 优先使用用户明确指定的文件、差异、提交或 PR。未指定时只检查当前可识别的变更范围；不默认审查全仓库，也不猜测比较分支。
2. **建立上下文。** 阅读完整相关差异、受影响方法与调用者、模型、契约、测试、构建配置和可用近期历史；搜索类似实现。没有历史或测试时如实记录。
3. **选择规范。** 按下表只加载实际涉及的 references；新增领域取并集，不递归读取全部规范。
4. **验证发现。** 沿调用链和数据流确认触发条件、影响、项目例外和兼容性；区分本次引入的问题和既有问题。
5. **必要验证。** 只运行项目已有、与审查范围相关且不会无授权改写代码或共享环境的测试、静态检查和架构检查。
6. **形成结果。** 合并同根因问题，按严重程度排序；优先报告本次引入或加剧的问题，无关历史问题不混入本次 findings。

---

## 规范路由

| 涉及领域 | 加载规范 | 主要检查 |
| --- | --- | --- |
| Java 实现 | [Java](../java-spring-backend/references/coding/java.md) | 命名、类设计、Lombok、Null、集合、异常实现、日志、格式、注释 |
| 分层、模型、Package、SOLID、跨模块、入站/出站 | [分层](../java-spring-backend/references/architecture/layering.md) | 职责、依赖方向、模型归属、技术组件归属、过度设计 |
| Spring 框架使用 | [Spring](../java-spring-backend/references/coding/spring.md) | DI、Bean、Validation、Proxy、Advice、Spring 注解机制 |
| HTTP API | [API](../java-spring-backend/references/api/api-design.md) | URL、Method、Request/VO、统一响应、错误契约、兼容、分页、幂等 |
| 异常跨层流转 | [异常处理](../java-spring-backend/references/architecture/error-handling.md) | 转换边界、cause、日志归属、重复记录、对外泄漏 |
| MyBatis 映射 | [MyBatis](../java-spring-backend/references/coding/mybatis.md) | Mapper、XML、ResultMap、TypeHandler、参数绑定、动态 SQL |
| SQL | [SQL](../java-spring-backend/references/database/sql.md) | SQL 正确性、范围、安全、PostgreSQL、分页、N+1、性能 |
| 数据库 Schema | [数据库设计](../java-spring-backend/references/database/database-design.md) | 命名、类型、Null、约束、索引、Migration、兼容 |
| 事务、锁、一致性 | [事务](../java-spring-backend/references/architecture/transactions.md) | 事务必要性、范围、传播、隔离、回滚、锁、竞态 |
| 并发、异步、线程池 | [并发](../java-spring-backend/references/architecture/concurrency.md) | 收益、线程池、上下文、共享状态、异常、容量、事务跨线程 |
| Bug、行为变化、测试 | [测试](../java-spring-backend/references/coding/testing.md) | 回归保护、AIR、BCDE、断言、Mock、集成测试、实际验证 |
| 权限、租户、数据范围、安全 | 目标项目已有安全规范、契约和实现 | 是否绕过认证/隔离、扩大数据范围、泄漏或记录敏感凭证 |

只加载与当前问题有关的规范。例如：

```text
只改 ResultMap
→ MyBatis

只改 SQL
→ SQL

Controller URL / 返回契约
→ API + 必要的 Spring

新增 VO Package
→ 分层 + 必要的 Java
```

不要因为代码使用 Java 就自动加载所有 references。

---

## 发现成立条件

只报告有以下至少一种证据支持的问题：

* 可复现的错误行为；
* 明确调用链 / 数据流风险；
* 已有 API、数据库、测试或业务契约被破坏；
* 目标项目现有实现形成的稳定约定被本次变更无依据绕过；
* 适用 reference 中明确规则被违反，并且没有项目例外；
* 安全、权限、数据完整性或并发风险有具体触发条件。

不得仅凭：

```text
个人偏好
“更优雅”的写法
可能以后会扩展
某种架构流派
```

形成 finding。

如果缺少关键业务契约导致无法确认，应列为“待确认”，不要把推测写成已确认缺陷。

---

## 项目约定与默认规范

references 中存在默认推荐时，必须先检查目标项目是否已有明确约定。

例如：

```text
ApiResponse<T>
VO / Request 命名
Package 结构
record / class
异常体系
日志框架
分页结构
```

如果目标项目已有稳定契约或历史 API，以项目为主，不得仅为了迁移到 Skill 默认风格形成 finding。

同样，不得把历史代码中的全仓库不一致扩大成本次审查问题；差异审查优先判断本次变更是否新增或加剧问题。

---

## SOLID 与架构发现

SOLID 的详细定义和判断统一读取 `layering.md`，本 Skill 不复制第二套定义。

SOLID finding 必须指出具体证据，例如：

```text
SRP → 哪些独立职责实际混在一个类中
OCP → 哪个真实、稳定变化点反复修改核心流程
LSP → 哪个输入/返回/异常/副作用契约被破坏
ISP → 哪些调用方或实现方被迫依赖无关能力
DIP → 哪个高层业务直接耦合易变技术协议
```

禁止泛化评论：

```text
“建议遵循 SOLID”
“建议抽接口”
“建议加 Strategy + Factory”
```

同时检查错误套用 SOLID 的过度设计。没有真实替换、扩展、隔离或复用需求时，不应机械要求：

```text
ServiceInterface + ServiceImpl
Strategy
Factory
Repository + RepositoryImpl
Adapter
大量单方法接口
```

> SOLID finding 需要具体设计风险；设计模式不是默认修复答案。

---

## 模型与 Package 审查

模型职责和 Package 的唯一详细事实来源是 `layering.md`。

Review 只需要确认：

* 本次模型是否按真实职责分类；
* Package 是否由职责而不是当前目录决定；
* 具体业务输出是否无依据新增为 `*Response` / `response` 而不是项目约定的 VO；
* DO 是否直接泄漏到 HTTP 边界；
* 技术基础设施是否错误放入业务 Package；
* 是否机械增加无真实职责的 DTO / BO / Converter / Assembler。

不要在本 Skill 中重新定义 Request / Query / DTO / BO / DO / VO 的完整规则。

---

## 异常与日志审查

异常跨层流转读取 `error-handling.md`；Java `catch` / `throw` 和日志写法读取 `java.md`；Web Advice 读取 `spring.md`；错误码和响应读取 `api-design.md`。

重点只检查实际问题：

* catch 后吞异常或把失败伪装成成功；
* 转换异常丢失 cause；
* Mapper → Manager → Service → Advice 重复打印同一堆栈；
* 技术异常无必要泄漏到业务或 API；
* 对外暴露 SQL、堆栈、内部类型、路径或敏感信息。

---

## SQL / MyBatis 审查边界

MyBatis 负责框架映射：

```text
Mapper
XML
#{}/ ${}
ResultMap
TypeHandler
动态 SQL
```

SQL 规范负责：

```text
SELECT / JOIN
UPDATE / DELETE 范围
NULL / 时间范围
分页 / 排序
N+1 / Batch
PostgreSQL / EXPLAIN
```

只改 MyBatis 映射时不机械加载 SQL；修改实际 SQL 时加载两者需要的最小集合。

---

## 测试审查

测试 finding 应围绕真实回归风险，不以覆盖率百分比本身代替质量判断。

重点检查：

* Bug 修复是否缺少可行的回归保护；
* 高风险业务规则、权限、SQL、事务、并发是否缺少必要验证；
* 新测试是否自动化、独立、可重复；
* 是否为了让错误实现通过而删除测试、弱化断言或扩大 Mock；
* 是否声称执行了实际没有运行的验证。

测试详细规则统一读取 `testing.md`。

---

## 严重程度

按实际影响判断，不因为规范使用“必须”就机械升级等级：

* **P0**：已确认会阻断系统运行、造成广泛严重损害或紧急安全事件。
* **P1**：明确重大功能、安全、权限、数据完整性或高影响兼容风险。
* **P2**：局部功能缺陷、可靠性问题或明确且有实际维护影响的规范违规。
* **P3**：影响较低但有证据的可维护性、可读性或一致性问题。

规范问题如果没有实际高影响，一般不应仅因“违反必须规则”自动提升到 P1。

---

## 输出格式

先给出按严重程度排序的 findings，再给待确认事项和验证范围。

每个 finding 包含：

- **严重程度与标题**；
- **位置**：文件和尽可能精确的行号；
- **证据与影响**：触发条件、调用关系、适用契约或规范；
- **最小修复建议**：说明应该调整什么，不顺带设计无关架构。

同一根因只报告一次。

如果没有已确认问题，明确写：

```text
未发现已确认的问题
```

并说明：

* 审查范围；
* 实际加载的规范；
* 实际执行的验证；
* 未验证项和局限。

不得把“本次没有发现”描述成“代码一定没有缺陷”。

最终原则：

> Review Skill 负责审查过程和证据质量；references 负责规则本身。只报告能够定位、解释和产生实际行动价值的问题。
