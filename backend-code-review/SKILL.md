---
name: backend-code-review
description: 审查 Java、Spring Boot、MyBatis、PostgreSQL 后端代码或变更，检查包与模型归属、分层、业务兼容性、安全、映射、SQL、事务、并发和必要测试。用于代码审查及开发后的自检，输出有定位与证据的发现；纯审查默认不修改代码。
---

# Backend Code Review

评估指定后端变更是否正确、符合团队规范，并指出可操作的问题。只报告有代码、契约、测试或适用规则支持的发现，不以个人偏好代替项目规则。

## 边界与配套资源

- 先阅读目标项目适用的 AGENTS.md。纯审查默认只读，不自动修复、重命名、格式化、更新快照或安装依赖，不自动发布评论或创建独立代理。
- 开发任务引用本流程自检时，修复权限来自原开发任务，仅限其已授权范围；本 Skill 本身不扩大权限。
- 与 java-spring-backend 同级放置，直接读取其 references，不加载开发 Skill 的实施流程，不维护第二套规范副本。
- 所有路径相对所在文档解析，不以被审查项目的工作目录为基准。配套 references 缺失时说明规范检查受限；仍可完成有证据支持的正确性审查，但不能宣称已完整验证团队规范。

## 审查流程

1. **确定范围。** 优先使用用户明确指定的文件、差异或提交范围。未指定时检查本地已暂存、未暂存变更及相关未跟踪新增文件；不要仅看 git diff 而遗漏暂存和新增文件。可只读使用 git status --short、git diff、git diff --cached 和 git ls-files --others --exclude-standard。没有可识别范围时询问，不默认审查全仓库，也不猜测比较分支。
2. **建立上下文。** 阅读完整相关差异、受影响方法与调用者、数据模型、相关契约、测试和可用近期历史；搜索类似实现。文件审查无历史时按现有代码评估，不假装已确认何时引入问题。
3. **选择规范。** 按下表加载实际涉及的领域；新增领域取并集，不递归读取所有参考文档。
4. **验证发现。** 沿调用和数据流确认触发条件与影响，区分本次引入的问题和既有问题，核对规范例外。若需要验证推断，执行已有的不改写源文件的相关测试或静态检查；不执行数据库迁移、格式化或影响共享环境的操作。
5. **形成结果。** 按严重程度排序并合并同根因发现。对差异审查，优先报告本次引入或加剧的问题；无关既有问题不混入本次发现。说明未验证项，不自动进入修复流程。

## 规范与检查重点

| 涉及领域 | 参考规范 | 检查重点 |
| --- | --- | --- |
| Java、模型、Package | [Java](../java-spring-backend/references/coding/java.md) | 职责与包归属，Request / Query / DTO / BO / DO / VO 分类，复用，模型风格及例外 |
| Spring、Service、Controller | [Spring](../java-spring-backend/references/coding/spring.md)、[分层](../java-spring-backend/references/architecture/layering.md) | Controller 越层、HTTP 语义下沉、反向依赖、跨模块访问、无必要 Manager 或抽象 |
| API 与业务行为 | [API](../java-spring-backend/references/api/api-design.md)、[Java](../java-spring-backend/references/coding/java.md) | 未授权的 API 变化，字段语义、状态、默认值、校验与兼容行为，敏感字段暴露 |
| MyBatis 与映射 | [MyBatis](../java-spring-backend/references/coding/mybatis.md)、[SQL](../java-spring-backend/references/database/sql.md) | Mapper 职责、TypeHandler 归属、显式映射、数据库拼音泄漏、参数绑定与动态 SQL 白名单 |
| 数据库结构与 SQL | [数据库设计](../java-spring-backend/references/database/database-design.md)、[SQL](../java-spring-backend/references/database/sql.md) | 拼音术语复用、约束、数据完整性、注入、SELECT *、N+1、分页稳定性、写入条件 |
| 事务、锁、一致性 | [事务](../java-spring-backend/references/architecture/transactions.md) | 不必要或缺失的事务边界、回滚、传播、自调用、查询后修改竞态 |
| 并发、异步、线程池 | [并发](../java-spring-backend/references/architecture/concurrency.md) | 实际收益、操作独立性、线程与连接池、上下文、异常；涉及事务同时加载事务规范 |
| Bug 修复、行为变化、测试 | [测试](../java-spring-backend/references/coding/testing.md) | 有效回归覆盖、重要边界、断言强度、可重复性及真实验证结果 |
| 权限、租户、数据范围 | 目标项目已有安全规范、契约和实现 | 是否绕过认证或数据隔离、扩大数据范围、硬编码或记录敏感凭证 |

只修改 SQL 时不因表中已有拼音字段就报告 Java 命名问题；仅查询多个 Mapper 不构成事务缺失证据。评估缺陷要结合具体调用和业务要求，不按关键词机械判定。

## 发现成立条件

- 指出具体代码位置与适用规则或可复现的触发条件，说明实际影响；纯规范违规也应给出原文规则依据。
- 核对明确例外：模块统一使用 record 或任务要求时不能按默认 class 规则报错；普通查询默认无显式事务，但一致性快照、锁或原子写入可能需要事务。
- 新增通用 TypeHandler 错放业务 mapper 包、新增视图模型误用 DTO、HTTP 语义进入 Service 等，按对应领域规则判断；不要扩展为对历史代码的全仓库改造。
- 若缺少业务契约而无法确认风险，将其列为待确认事项，不描述成已发生的数据损坏或既定业务缺陷。
- 同一问题报告一次，建议最小修复方向，不顺带设计无关架构。

## 输出格式

先给出按严重程度排序的发现，再列待确认事项及验证情况。每个发现包含：

- **严重程度与标题**：P0 为已确认会阻断运行或造成广泛严重损害的紧急问题；P1 为明确的重大功能、安全或数据完整性风险；P2 为局部缺陷或明确的规范违规；P3 为影响较低但有依据的改进项。按实际影响判断，不因规则使用“必须”就升级严重程度。
- **位置**：文件和精确行号，优先定位本次差异中的相关语句。
- **证据与影响**：具体触发条件或适用规则、调用关系及影响；引用规范时标明文档与章节。
- **最小修复建议**：说明应调整什么，不直接改写代码。

没有已确认发现时明确写“未发现已确认的问题”，并说明检查范围和局限，不等同于保证没有缺陷。验证情况区分实际运行通过、失败和未执行；已有结果仍适用时可以复用并说明来源，不重复运行。
