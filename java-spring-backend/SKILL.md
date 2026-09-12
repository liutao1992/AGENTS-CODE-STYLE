---
name: java-spring-backend
description: 按团队后端规范开发、修复和重构 Java、Spring Boot、MyBatis、MyBatis-Plus、Rabbit-SQL、PostgreSQL 代码。用于实现后端功能、判断职责、业务规则与模型、设计 API、数据库访问、事务、并发和测试边界；纯代码审查使用 backend-code-review。
---

# Java Spring Backend

将当前任务转成最小、符合目标项目现有设计的代码变更。

核心方式：

> Skill 负责流程和规范路由；references 维护唯一详细领域规则。只加载当前任务真正需要的规范，不在 Skill 中复制第二套编码规则。

## 开始前

- 阅读目标项目适用的 `AGENTS.md`，遵守用户当前要求和执行权限。
- 纯代码审查转到配套的 [backend-code-review](../backend-code-review/SKILL.md)。
- 两个 Skill 保持同级目录；reference 链接相对 Skill Pack 解析。
- 目标项目已有稳定契约、框架机制和目录结构时优先沿用，不因为本 Skill 默认推荐批量迁移历史代码。

---

## 工作流程

1. **明确任务。** 确认目标、涉及模块、行为变化和验收要求；只对真正无法从代码、契约和测试确认的关键业务意图询问用户。
2. **建立上下文。** 阅读相关代码、类似实现、构建配置、测试和可用近期 Git 历史；找不到时如实说明。
3. **选择规范。** 按下表加载实际涉及的 references；任务扩展到新领域时再补读，不递归加载全部文档。
4. **判断职责。** 新增或移动文件前先确定业务模块、逻辑职责、模型类型和依赖边界，再确定物理位置与 Package。
5. **实现最小变更。** 优先复用已有能力，不自行创造业务状态、编码、默认值、兼容行为或额外抽象。
6. **自检。** 检查完整 diff 和新增文件，并按 [backend-code-review](../backend-code-review/SKILL.md) 的审查方法自检当前任务范围。
7. **验证。** 运行项目已有的相关测试、静态检查和架构检查；不假设项目一定存在某个 Wrapper、Formatter 或 ArchUnit 配置。
8. **汇报。** 说明实际修改、关键行为、验证结果、失败和未执行项；不得把未运行的检查描述为通过。

---

## 规范路由

### Java 实现

加载 [Java](references/coding/java.md)：

```text
命名
类设计
Lombok
class / record
常量 / Enum / 魔法值
POJO 属性默认值
方法设计
方法参数 / 参数对象
Null / Optional
集合 / 泛型
集合返回契约
BigDecimal / 时间
Java catch / throw
日志
格式 / 注释
```

普通 Java 实现不因此自动加载 Spring、API 或分层。

### 项目目录与业务模块

加载 [项目与业务模块目录](references/architecture/project-structure.md)：

```text
新建业务 module
调整项目物理目录
业务优先 / 技术优先组织
module 内职责目录
common / util / constant / third 归属
跨模块物理组织
```

需要判断类逻辑职责时再加载 `layering.md`。

### 分层、模型与职责 Package

加载 [应用分层](references/architecture/layering.md)：

```text
Controller / Service / Manager / Mapper / Client / Adapter 职责
入站 / 出站适配器
依赖方向
Request / Query / DTO / BO / DO / VO 分类
职责 Package
跨模块调用
调用者上下文边界
Service 拆分
SOLID
过度抽象
```

涉及物理 module / 目录组织时再加载 `project-structure.md`；涉及模型 Java 写法时再加载 Java。

### 业务规则与用例

加载 [业务规则与用例边界](references/architecture/business-rules.md)：

```text
核心业务规则 / 稳定不变量
应用特定业务规则 / 用例流程
行为业务对象
Clean Architecture Entity 与持久化 DO 的区别
Service 作为 Use Case 职责
贫血模型是否真的构成问题
业务规则应该放对象还是 Service / Manager
Request / VO 与应用输入输出模型边界
核心业务规则的依赖方向
何时不应该新增 Entity / UseCase / Repository / Command / Result
```

本文借鉴 Clean Architecture 的职责判断，但不要求目标项目改造成完整 Clean Architecture。涉及具体层间依赖时同时加载 `layering.md`；涉及模型实现时再加载 Java。

### Spring Framework

加载 [Spring](references/coding/spring.md)：

```text
@RestController / @Controller
@RequestMapping / @GetMapping / @PostMapping
Spring MVC 注解机制
Bean Validation / @Valid / @Validated
依赖注入
Spring Bean 生命周期
@ConfigurationProperties
@Transactional Proxy
TransactionTemplate Spring API
@Async / @Cacheable Proxy
@RestControllerAdvice / @ExceptionHandler
```

业务职责本身由 `layering.md` 判断；事务是否需要由 `transactions.md` 判断。

### HTTP API

加载 [API](references/api/api-design.md)：

```text
URL
HTTP Method
Path / Query / Body
Request / VO 对外语义
ApiResponse<T> 或项目统一响应
分页 / 排序
错误码 / HTTP Status
幂等
兼容性
对外敏感字段
```

只调整 Spring 注解且不改变 HTTP 契约时，可以只加载 Spring。

### 异常与错误边界

加载 [异常处理](references/architecture/error-handling.md)：

```text
Mapper / Client 异常传播
Manager / Service 是否转换异常
异常 cause
重复 log + throw
错误在哪个边界收口
内部异常是否泄漏到客户端
```

Java `catch` / `throw` / 日志 API 同时需要时加载 Java；Web Advice 加载 Spring；HTTP 错误契约加载 API。

### MyBatis / MyBatis-Plus

加载 [MyBatis / MyBatis-Plus](references/coding/mybatis.md)：

```text
MyBatis Mapper / DAO 接口
Mapper XML
MyBatis-Plus
BaseMapper<T>
QueryWrapper / LambdaQueryWrapper
UpdateWrapper / LambdaUpdateWrapper
XML 业务常量
@Param
#{}/ ${}
ResultMap
TypeHandler
Interceptor / Plugin
MyBatis 动态 SQL
Mapper List<T> 返回契约
MyBatis 技术 Package
```

看到普通 `Mapper` / `DAO` 名称但无法确认持久层框架时，先检查依赖、注解和 SQL 资源；不要仅凭名称套用 MyBatis-Plus 的 `BaseMapper` / Wrapper 规则。

实际修改 SQL 时再加载 SQL。

### Rabbit-SQL

加载 [Rabbit-SQL](references/coding/rabbit-sql.md)：

```text
rabbit-sql / rabbit-sql-spring-boot-starter
@XQLMapper / @XQLMapperScan
@XQL / @Arg
Baki / BakiDao
XQLFileManager
xql-file-manager.yml
*.xql
:name 命名参数
${} XQL 字符串模板
#if / #for / #choose 动态 SQL
@CountQuery / @PageableConfig
PagedResource / IPageable
Stream 查询
Batch
QueryCacheManager / executionWatcher
Rabbit-SQL Spring 事务
```

Rabbit-SQL Mapper 与 MyBatis Mapper 都属于数据库出站适配器，但框架规则不同；`@XQLMapper` 不要求继承 MyBatis-Plus `BaseMapper`，`.xql` 也不是 MyBatis Mapper XML。

实际修改 XQL 中的 SQL 时同时加载 SQL；涉及 Spring 事务时再加载事务和必要的 Spring。

### SQL / PostgreSQL

加载 [SQL](references/database/sql.md)：

```text
SELECT / JOIN
WHERE / NULL / 时间范围
INSERT / UPDATE / DELETE
分页 / 排序
N+1 / Batch
PostgreSQL
索引使用
EXPLAIN
```

没有真实执行计划时，性能判断只能作为结构性建议。

### 数据库设计

加载 [数据库设计](references/database/database-design.md)：

```text
表 / 字段
类型
Null / 默认值
主键 / 唯一约束
索引
Migration
数据库物理命名
Schema 兼容
```

同时修改 SQL 或 Java 映射时再加载对应 SQL / MyBatis / Rabbit-SQL 规范。

### 事务

加载 [事务](references/architecture/transactions.md)：

```text
是否需要事务
事务一致性范围
Service / Manager 事务边界
@Transactional
TransactionTemplate / 编程式事务
传播 / 隔离
查询后修改
条件更新
乐观锁 / 悲观锁
一致性快照
rollbackFor / 回滚语义
长事务
事务与线程切换
```

涉及 Spring Proxy 或 `TransactionTemplate` API 细节时再加载 Spring。

### 并发

加载 [并发](references/architecture/concurrency.md)：

```text
CompletableFuture
@Async
Executor / ThreadPoolTaskExecutor
线程池
ThreadLocal / MDC / SecurityContext
Lock / synchronized / volatile
异步异常
exceptionally / fallback
重试
并发资源容量
```

涉及事务时同时加载事务。

### 测试

加载 [测试](references/coding/testing.md)：

```text
Bug 修复
新增 / 修改业务行为
新增测试
测试失败
API / SQL / 事务 / 并发验证
Mock / 集成测试 / Testcontainers
```

测试规范决定“怎么验证”，不替代对应领域 reference 决定“怎么实现”。

### 安全

认证、权限、数据范围、租户和敏感信息优先读取目标项目已有安全规范、契约和实现。

本 Skill Pack 当前没有独立 `security.md`。始终不得绕过认证/权限/租户隔离，不得硬编码或记录密码、Token、Secret、私钥等凭证。

---

## 常见组合

```text
普通业务方法参数过多
→ Java

魔法值 / 常量类 / 固定值域 / POJO 默认值
→ Java

新增业务 module
→ 项目结构 + 分层

新增 VO / Query / DO
→ 分层 + Java

同一业务状态规则在多个 Service / 入口重复 if + set
→ 业务规则 + 分层 + Java

判断一条规则应该进入行为业务对象还是留在 Service / Manager
→ 业务规则 + 分层

Controller Request 是否需要转换成 Command / DTO 才能调用 Service
→ 业务规则 + 分层；改变 HTTP 契约时再加 API

把传统 CRUD 改造成 Entity / UseCase / Repository 结构
→ 业务规则 + 分层；先证明稳定不变量、职责收益和目标项目兼容性，不机械迁移

Controller URL 或返回契约变化
→ API + 必要的 Spring

只调整 MVC Mapping 注解
→ Spring

Bean Validation 与 Service 重复结构校验
→ Spring

标准 MyBatis List<T> 查询后的 Null 防御
→ MyBatis + Java

项目使用 MyBatis-Plus，新增 Mapper / DAO
→ MyBatis

发现 QueryWrapper / LambdaQueryWrapper / UpdateWrapper
→ MyBatis

Mapper XML 中出现业务状态或类型硬编码
→ MyBatis；如果同时判断 SQL 正确性再加 SQL

新增 @XQLMapper / .xql / xql-file-manager.yml
→ Rabbit-SQL；修改实际 SQL 再加 SQL

Rabbit-SQL 中出现 Baki 直接进入 Service / Controller、${} 外部输入、Stream 生命周期问题
→ Rabbit-SQL + 必要的分层 / SQL / Java

Rabbit-SQL 与 Spring 事务共同修改
→ Rabbit-SQL + 事务 + 必要的 Spring

第三方 SDK nullable 集合归一化
→ Java；需要判断 Client / Adapter 时再加分层

新增 ResultMap / TypeHandler
→ MyBatis

修改 Mapper 实际 SQL
→ 先识别 MyBatis / Rabbit-SQL，再加载对应框架规范 + SQL

修改表字段和映射
→ 数据库设计 + 对应持久层框架规范

判断事务是否需要 / 边界放在哪里
→ 事务 + 必要的分层

TransactionTemplate 局部事务代码块
→ 事务 + Spring

@Transactional 自调用是否生效
→ 事务 + Spring

CompletableFuture 数据库操作
→ 并发 + 必要的事务

Bug 修复并补回归测试
→ 对应领域 + 测试
```

原则：

> 多领域任务取真正需要的并集，不以“保险”为理由加载全部 references。

---

## 新建文件前的职责判断

新增文件前依次回答：

```text
属于哪个业务模块？
        ↓
它是什么逻辑职责？
        ↓
已有同类实现能否复用？
        ↓
如果是模型，属于哪种模型职责？
        ↓
应该放在哪个职责 Package？
        ↓
目标项目的物理目录在哪里？
        ↓
是否真的需要新建？
```

“业务模块位置”由 `project-structure.md` 负责；“Controller / Service / Manager / Mapper / Client / 模型是什么职责”由 `layering.md` 负责；“核心业务规则与应用用例流程如何分层”由 `business-rules.md` 负责。

---

## 交付要求

完成任务后说明：

* 变更目的；
* 关键行为；
* 必要文件定位；
* 实际执行的测试、静态检查和架构检查；
* 失败或未执行项及原因；
* 仍存在的真实风险或待确认业务问题。

已通过且未受后续修改影响的检查无需机械重复运行。

最终原则：

> Skill 负责工作流和路由；references 负责唯一领域知识；目标项目真实契约优先于 Skill 默认示例。
