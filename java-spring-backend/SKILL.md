---
name: java-spring-backend
description: 按团队后端规范开发、修复和重构 Java、Spring Boot、MyBatis、PostgreSQL 代码。用于需要判断模型与包归属、应用分层、数据库映射、事务或并发边界的后端任务；纯代码审查使用 backend-code-review。
---

# Java Spring Backend

将当前任务转成最小、符合现有项目设计的代码变更。先判断任务，再加载相关规范，不一次性读取全部 references。

## 开始前

- 阅读目标项目适用的 AGENTS.md，遵守当前用户要求及执行权限。纯审查转到配套的 [backend-code-review](../backend-code-review/SKILL.md)，不执行下面的修改步骤。
- 两个 Skill 必须同级放置；本文件所有链接均相对本文件解析，参考文档内的链接相对该参考文档解析，不以目标项目工作目录为基准。
- 独立用于其他项目时仍坚持最小修改、优先复用、业务语义稳定和安全边界。不得因套用规范而批量改造历史代码；保留领域规范明确允许的现有风格例外。

## 工作流程

1. 明确任务目标、涉及模块、行为变化及验收要求；通过代码、契约和测试解决可发现的问题，仅对无法确定的关键业务意图询问用户。
2. 检查构建配置、适用项目规则、相关代码和可用的近期 Git 历史；搜索至少一个类似实现，阅读相关测试。无历史、无类似实现或无测试时记录事实，不假定它们存在。
3. 根据下表加载相关规范，再确定实现方案。多个领域取并集；开始涉及新领域时补读。读到其他规范链接不意味着必须递归读取，仍按当前任务判断。
4. 新建文件前执行下方职责判断。涉及数据库时检查已有 Schema、Migration、Mapper 和术语；涉及事务时先确定一致性要求；涉及并发时先确认实际收益及操作独立性。
5. 实施最小正确变更，复用已有组件，不编造业务规则、兼容行为或默认值，不擅自改变 API、权限、删除与事务语义。发现无关问题单独说明。
6. 检查当前任务的完整差异及新增文件；使用 [审查流程](../backend-code-review/SKILL.md#审查流程) 自检当前变更，不自动委派代理或扩大审查范围。已获授权的开发任务可修复本次自检发现的问题；纯审查保持只读。
7. 运行目标项目已有的相关测试、静态检查和架构检查。优先采用构建文件与 CI 定义的命令，不假设一定存在 Maven Wrapper、Spotless、Checkstyle 或 ArchUnit；不因检查缺失而自动安装依赖或新增流水线。
8. 汇报修改内容、关键行为和实际验证结果。已通过且未受后续变更影响的检查无需重复执行；无法执行的检查说明原因。

## 规范选择

| 任务或触发条件 | 加载规范 |
| --- | --- |
| 普通 Java 实现、Lombok、class / record、集合、异常、日志 | [Java](references/coding/java.md) |
| 新增或调整模型、Package、职责边界 | [分层](references/architecture/layering.md)；涉及 Java 实现方式同时加载 [Java](references/coding/java.md) |
| Controller / HTTP API | [分层](references/architecture/layering.md)、[Java](references/coding/java.md)、[Spring](references/coding/spring.md)、[API](references/api/api-design.md) |
| Service / Manager 业务流程、Spring 配置与组件 | [分层](references/architecture/layering.md)、[Java](references/coding/java.md)、[Spring](references/coding/spring.md) |
| 分层、职责调整、跨模块调用、SOLID | [分层](references/architecture/layering.md) |
| Mapper、ResultMap、TypeHandler、MyBatis 基础设施 | [分层](references/architecture/layering.md)、[MyBatis](references/coding/mybatis.md)、[SQL](references/database/sql.md)；涉及 Java 类型同时加载 Java |
| SQL 编写或优化 | [SQL](references/database/sql.md) |
| 表、字段、索引、约束、数据库模型 | [数据库设计](references/database/database-design.md)、[SQL](references/database/sql.md)；涉及 Java 映射同时加载 MyBatis 和分层 |
| @Transactional、传播、隔离级别、锁、查询后修改、一致性快照 | [事务](references/architecture/transactions.md) |
| CompletableFuture、@Async、Executor、线程池、跨线程上下文 | [并发](references/architecture/concurrency.md)；涉及事务时同时加载事务 |
| Bug 修复、行为变化、测试修改 | 对应领域规范与[测试](references/coding/testing.md) |
| 认证、权限、数据范围、租户或敏感数据 | 目标项目已有安全规范及相关实现；本包无独立安全文档，不引用不存在的文件 |

安全底线始终适用：不绕过认证、权限、数据权限或租户隔离，不硬编码或记录敏感凭证，不削弱现有安全机制。

## 新建文件前的职责判断

1. 先说明该类负责什么，搜索已有等价能力及同类组件所在位置。
2. 按职责识别业务组件、Web 输入、查询条件、内部传输、业务中间模型、持久化模型、视图输出或框架基础设施。
3. 新增模型先读取分层规范，判断 Request / Query / DTO / BO / DO / VO 及 Package；再读取 Java 规范确定 Lombok、`class` / `record` 等实现方式。不要统一塞进 dto，也不要为每一层机械创建模型或 Converter。
4. 新增技术组件先按分层规范判断技术职责，再读取其领域规范确定具体 Package 和实现方式，不按当前使用者归属。
5. 确认正确包名、命名和复用方式后再创建文件。

使用规范中的具体判断及例外：

- 通用 MyBatis TypeHandler 属于 `common.mybatis.handler`，不因某个业务 Mapper 使用而放进业务 mapper 包。
- 具体业务返回模型优先使用模块 `vo` 包中的 `*VO`；统一 HTTP 响应包装默认使用 `ApiResponse<T>`。如果目标项目已有其他统一响应类型、历史 API 或序列化契约，以项目现有约定为准，不得为了本 Skill 强制替换。也不因此无授权重命名已有公共 API。
- 业务模型默认普通 class，使用 Lombok 生成无业务逻辑的访问器；模块明确统一使用 record 或任务明确要求时允许 record。不机械给所有字段添加 Setter。
- 数据库拼音通过显式映射转换为 Java 英文属性；示例词汇需结合目标项目已有数据字典，不自行创造第二套术语。
- 普通快照读不因多个 Mapper 就加事务；需要原子性或一致性快照时按事务规范判断。异步不得用于规避事务问题。
- SOLID 用于识别真实职责、扩展、契约、接口和依赖问题，不用于机械创建 `Interface + Impl`、Strategy、Factory、Repository 包装或额外层级。

## 交付要求

给出变更目的、实际行为、必要的文件定位及检查结果。区分已通过、失败和未执行的验证；报告仍存在的风险或业务待确认项，不将未执行的检查描述为通过。
