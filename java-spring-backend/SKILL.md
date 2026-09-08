---
name: java-spring-backend
description: 按团队后端规范开发、修复和重构 Java、Spring Boot、MyBatis、PostgreSQL 代码。用于需要实现 Java 后端、判断模型与 Package、应用分层、API、数据库映射、事务、并发或测试边界的任务；纯代码审查使用 backend-code-review。
---

# Java Spring Backend

将当前任务转成最小、符合目标项目现有设计的代码变更。

核心方式：

> 先判断任务涉及什么，再只加载对应 references；不要因为任务使用 Java / Spring 就一次性读取全部规范。

## 开始前

- 阅读目标项目适用的 `AGENTS.md`，遵守当前用户要求和执行权限。
- 纯代码审查转到配套的 [backend-code-review](../backend-code-review/SKILL.md)，不执行本文件的修改流程。
- 两个 Skill 保持同级目录；链接相对所在文档解析，不以目标项目当前工作目录为基准。
- 独立用于其他项目时仍坚持最小修改、优先复用、业务语义稳定和安全边界。
- 不因套用规范批量迁移历史风格；references 中的默认推荐不能覆盖目标项目已有稳定契约。

---

## 工作流程

1. **明确任务。** 确认目标、涉及模块、行为变化和验收要求；优先从代码、契约和测试中解决可发现的问题，只对真正无法确定的关键业务意图询问用户。
2. **建立上下文。** 阅读相关代码、构建配置、类似实现、测试和可用近期 Git 历史。找不到类似实现、测试或历史时如实说明，不假定存在。
3. **选择规范。** 按下表加载当前实际涉及的 references；任务扩展到新领域时再补读，不递归读取所有链接。
4. **判断职责。** 新增文件前先判断类的职责和边界，再确定模型类型、Package 和技术实现。
5. **实现最小变更。** 复用已有组件，不编造业务状态、编码、默认值、兼容行为，不擅自改变 API、权限、删除、事务或数据库约束语义。
6. **自检。** 检查完整差异和新增文件，按 [backend-code-review](../backend-code-review/SKILL.md) 的审查流程自检当前变更，不自动扩大范围或创建独立代理。
7. **验证。** 运行目标项目已有的相关测试、静态检查和架构检查；优先使用构建文件与 CI 中已有命令，不假设一定存在 Maven Wrapper、Spotless、Checkstyle 或 ArchUnit。
8. **汇报。** 说明修改内容、关键行为、实际运行的验证、失败和未执行项；不得把未运行的检查描述为通过。

---

## 规范选择

### Java 实现

以下情况加载 [Java](references/coding/java.md)：

```text
命名
类设计
Lombok
class / record
方法
方法参数数量 / 参数对象
Null / Optional
集合 / 泛型
集合返回值与 Null 契约
无依据的空集合 / Null 兜底
BigDecimal / 时间
catch / throw
日志
格式 / 注释
```

普通 Java 实现不因此自动加载分层、Spring 或 API。

---

### 项目目录与业务模块

以下情况加载 [项目与业务模块目录](references/architecture/project-structure.md)：

```text
新建业务 module
调整项目目录
业务优先 / 技术优先的代码组织
module 内 controller / service / manager / mapper 等目录
common / util / constant / third 等公共目录归属
跨模块物理组织
新增 Package 的模块位置
```

目录规范决定“项目和模块如何物理组织”，不替代分层规范判断类的逻辑职责。

如果同时需要判断 Controller / Service / Manager / Mapper / Client 的职责，再加载分层；如果只是新增一个符合既有结构的普通 Java 类，不因为存在 `module` 目录就机械加载全部架构规范。

---

### 分层、模型与 Package

以下情况加载 [分层](references/architecture/layering.md)：

```text
新增或移动类
判断 Controller / Service / Manager / Mapper / Client 职责
Request / Query / DTO / BO / DO / VO 分类
Package 职责归属
跨模块调用
入站 / 出站适配
当前调用者上下文边界
Service 职责拆分
SOLID
是否过度抽象
```

涉及项目 / module 的物理目录组织时再加载 `project-structure.md`；涉及模型 Java 实现时再同时加载 Java。

---

### Spring 框架

以下情况加载 [Spring](references/coding/spring.md)：

```text
@RestController / @Controller
@RequestMapping / @GetMapping / @PostMapping
Spring MVC 路由注解位置
OpenAPI / Swagger 注解机制
Bean Validation / @Valid
重复结构校验 / Service 二次校验
必填字段默认兜底
依赖注入
Spring Bean 生命周期
@ConfigurationProperties
@Transactional / TransactionTemplate 的 Spring 实现机制
@Async / @Cacheable 的代理行为
@RestControllerAdvice / @ExceptionHandler
```

Service / Manager 的业务职责本身属于分层规范，不因为类带 `@Service` 就必须同时加载 Spring。

---

### HTTP API

以下情况加载 [API](references/api/api-design.md)：

```text
URL
HTTP Method
Path / Query / Body 契约
Request / VO 对外语义
ApiResponse<T> 或项目统一响应
分页
错误码 / HTTP Status
幂等
兼容性
对外敏感字段
```

如果只是调整 Spring 注解而不改变 HTTP 契约，可以只加载 Spring。

如果同时涉及模型职责或 Package，再加载分层；涉及 Java 实现细节再加载 Java。

---

### 异常与错误边界

以下情况加载 [异常处理](references/architecture/error-handling.md)：

```text
Mapper / Client 异常如何传播
Manager / Service 是否转换异常
哪里记录完整异常现场
重复 log + throw
异常 cause
Web / API 如何收口
内部异常是否泄漏到客户端
```

按实际需要组合：

```text
catch / throw / 日志 API
→ Java

@RestControllerAdvice / @ExceptionHandler
→ Spring

错误码 / HTTP 错误响应
→ API
```

不要把四份文档全部作为每个异常问题的默认组合。

---

### MyBatis

以下情况加载 [MyBatis](references/coding/mybatis.md)：

```text
Mapper 接口
Mapper XML
@Param
#{}/ ${}
Mapper List<T> 集合查询 Null 契约
ResultMap
TypeHandler
Interceptor / Plugin
动态 SQL
MyBatis 技术 Package
```

如果实际修改 SQL，再同时加载 SQL；仅调整 ResultMap、TypeHandler 或集合返回契约时不需要自动加载全部 SQL 规则。

---

### SQL / PostgreSQL

编写、修改或优化实际 SQL 时加载：

- [SQL](references/database/sql.md)

SQL 规范负责：

```text
SELECT / JOIN
WHERE / NULL / 时间范围
INSERT / UPDATE / DELETE
分页 / 排序
N+1 / Batch
PostgreSQL
索引使用与 EXPLAIN
```

性能结论没有执行计划或真实证据时必须明确为结构性建议，而不是已验证结论。

---

### 数据库设计

修改以下内容时加载：

- [数据库设计](references/database/database-design.md)

```text
表
字段
类型
Null / 默认值
主键 / 唯一约束
索引
Migration
数据库命名
Schema 兼容
```

如果同时修改 SQL，再加载 SQL；如果同时修改 Java 映射，再加载 MyBatis。

---

### 事务

涉及以下内容加载：

- [事务](references/architecture/transactions.md)

```text
@Transactional 是否需要
TransactionTemplate / 编程式事务
Service 事务外准备 / Manager 原子事务
事务范围
传播
隔离级别
查询后修改
条件更新
乐观锁 / 悲观锁
一致性快照
rollbackFor / 回滚
长事务
```

涉及 `@Transactional` Proxy 或 `TransactionTemplate` Spring API 细节时再加载 Spring。

不要因为存在 Mapper 写操作就自动加载并引入事务设计。

---

### 并发

涉及以下内容加载：

- [并发](references/architecture/concurrency.md)

```text
CompletableFuture
@Async
Executor / ThreadPoolTaskExecutor
线程池
ThreadLocal / MDC / SecurityContext
Lock / synchronized / volatile
异步异常
重试
并发资源容量
```

如果并发逻辑涉及事务，再同时加载事务。

---

### 测试

以下情况加载：

- [测试](references/coding/testing.md)

```text
Bug 修复
新增或修改业务行为
新增测试
测试失败
SQL / 事务 / 并发的验证设计
Mock / Testcontainers
```

测试规范决定“怎么验证”，不会替代对应领域规范决定“怎么实现”。

---

### 安全

认证、权限、数据范围、租户和敏感数据优先读取目标项目已有安全规范、契约和实现。

本 Skill Pack 当前没有独立 `security.md`。

安全底线始终适用：

* 不绕过认证、权限、数据权限或租户隔离；
* 不硬编码或记录密码、Token、私钥等凭证；
* 不削弱已有安全机制。

---

## 常见组合示例

```text
修改一个普通 Java 工具方法
→ Java

普通业务方法参数超过 5 个
→ Java

新增一个全新 place 业务 module
→ 项目结构 + 分层；具体 Java 实现再加载 Java

只调整 module 下的物理 Package 组织
→ 项目结构

新增 PlaceVO
→ 分层 + Java

修改 Controller URL / 返回字段
→ API + 必要的 Spring

只调整 @RequestMapping / @GetMapping 的注解位置
→ Spring

只给 Controller 增加 @Valid
→ Spring

发现 Request 已 @NotBlank，Service 又 hasText 二次校验
→ Spring

发现 Service 过大，需要判断是否按业务能力拆分
→ 分层 + 必要的 Java

发现普通 Java 集合已经有非 Null 契约，上层又 list == null 兜底
→ Java

发现 MyBatis List<T> 查询后 Service 又 list == null ? emptyList : list
→ MyBatis + Java

第三方 SDK items 可能为 null，需要统一转换为空集合
→ Java；如果同时要判断 Client / Adapter 归属，再加载分层

新增 Mapper ResultMap
→ MyBatis

修改 Mapper 中实际 SELECT
→ MyBatis + SQL

修改表字段和映射
→ 数据库设计 + MyBatis

修复事务回滚问题
→ 事务 + 必要的异常处理

Service 中只有一小段数据库操作需要事务，考虑 TransactionTemplate
→ 事务 + Spring

Service 准备数据后由 Manager 收口多个原子写操作
→ 事务 + 必要的分层

新增 CompletableFuture 数据库查询
→ 并发 + 必要的事务

修复 Bug 并补测试
→ 对应领域 + 测试
```

原则：

> 多领域取真正需要的并集，不以“保险”为理由加载全部规范。

---

## 新建文件前的职责判断

1. 这个类实际负责什么？
2. 属于哪个业务模块，目标项目是否已有同类目录？
3. 是入站适配、业务用例、应用能力、数据库访问、外部技术适配、模型还是通用基础设施？
4. 是否已经存在相同或类似能力？
5. 如果是模型，属于 Request / Query / DTO / BO / DO / VO 中哪一种？
6. Package 应由职责决定，而不是由当前任务目录决定。
7. 确定模块、职责和 Package 后，再读取对应实现规范并创建文件。

重要默认：

- 新项目或没有既有约定的新业务区域，优先按业务模块组织，再在模块内部按职责分层；目标项目已有稳定目录时不强制迁移到 `module`。
- `common`、`util`、`constant`、`third` 等目录不能成为无边界兜底，公共能力必须具有真实跨模块职责。
- 通用 MyBatis TypeHandler 属于 MyBatis 技术基础设施，不因业务 Mapper 使用就放入业务 `mapper`。
- 具体业务输出优先使用 `*VO`；统一 HTTP 响应默认推荐 `ApiResponse<T>`，但目标项目已有响应契约时以项目为准，不无授权迁移历史 API。
- 普通业务模型默认使用普通 `class` 和项目现有 Lombok 风格；不为了减少样板代码主动换成 `record`。
- 普通业务方法参数默认不超过 5 个；超过时先判断参数是否共同描述一次完整操作。来源、生命周期和信任边界一致时，优先封装为一个语义完整对象；职责或信任边界不同则保持分离，不为了凑参数数量机械拆成多个参数对象。普通查询条件超过 3 个优先评估 Query，不使用 `Map<String, Object>` 机械兜底。
- 数据库物理命名与 Java 英文业务语义通过 Mapper / ResultMap 隔离。
- Client / Adapter 负责第三方协议细节；Manager 只有在存在真实应用级复用、组合或原子能力时才引入。
- 请求绑定的当前用户 / 部门 / 租户等上下文优先在入站边界取得并按业务需要显式传递，不让 Service / Manager 无必要依赖 Web Request 或请求专用 ThreadLocal。
- Service 过大时按真实业务用例和变化原因评估拆分，不按文件行数机械创建 Helper / Common / Validator Service。
- 已由可信入站边界通过 Bean Validation 保证的结构约束，不在 Service / Manager 机械重复同义 `null` / blank / size 校验；多入口场景应补齐真正缺失的入口或公共契约。
- 已声明必填或非空的字段不得通过 `""`、`0`、默认编码、默认状态等无依据兜底掩盖非法输入；只有既有契约明确要求时才允许默认行为。
- 集合无结果优先使用空集合表达；已有明确非 Null 集合契约时，上层不再机械增加 Null 防御。外部或遗留来源确实允许 Null 时，在最靠近来源的边界归一化一次，再让上层依赖稳定契约。
- 标准 MyBatis `List<T>` 集合查询无匹配记录按空集合处理；不要在 Service / Manager 中无依据重复 `list == null ? emptyList : list`。
- 事务先判断真实一致性边界。能在 Service 完成事务外数据准备、由 Manager 收口原子数据库操作时优先缩短事务；跨多个 Manager 必须整体提交时事务提升到 Service。方法级边界清晰时优先 `@Transactional(rollbackFor = Exception.class)`，只有局部代码块需要精确事务控制时再评估 `TransactionTemplate`，目标项目已有更具体事务契约时以项目为准。
- SOLID 用于解决真实职责和依赖问题，不用于机械创建 `Interface + Impl`、Strategy、Factory、Repository 或额外层级。

---

## 交付要求

完成任务后给出：

* 变更目的；
* 关键行为；
* 必要文件定位；
* 实际执行的测试 / 静态检查 / 架构检查；
* 失败或未执行项及原因；
* 仍存在的业务待确认或真实风险。

已通过且未受后续修改影响的检查无需机械重复运行。

最终原则：

> Skill 负责流程和路由，references 各自维护唯一领域知识；只加载当前任务真正需要的规范。
