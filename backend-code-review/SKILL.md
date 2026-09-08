---
name: backend-code-review
description: 审查 Java、Spring Boot、MyBatis、PostgreSQL 后端代码或变更，检查包与模型归属、分层、SOLID、业务兼容性、安全、映射、SQL、事务、并发和必要测试。用于代码审查及开发后的自检，输出有定位与证据的发现；纯审查默认不修改代码。
---

# Backend Code Review

评估指定后端变更是否正确、符合团队规范，并指出可操作的问题。只报告有代码、契约、测试或适用规则支持的发现，不以个人偏好代替项目规则。

## 边界与配套资源

- 先阅读目标项目适用的 AGENTS.md。纯审查默认只读，不自动修复、重命名、格式化、更新快照或安装依赖，不自动发布评论或创建独立代理。
- 开发任务引用本流程自检时，修复权限来自原开发任务，仅限其已授权范围；本 Skill 本身不扩大权限。
- 与 `java-spring-backend` 同级放置，直接读取其 references，不加载开发 Skill 的实施流程，不维护第二套规范副本。
- 所有路径相对所在文档解析，不以被审查项目的工作目录为基准。配套 references 缺失时说明规范检查受限；仍可完成有证据支持的正确性审查，但不能宣称已完整验证团队规范。

## 审查流程

1. **确定范围。** 优先使用用户明确指定的文件、差异或提交范围。未指定时检查本地已暂存、未暂存变更及相关未跟踪新增文件；不要仅看 `git diff` 而遗漏暂存和新增文件。没有可识别范围时询问，不默认审查全仓库，也不猜测比较分支。
2. **建立上下文。** 阅读完整相关差异、受影响方法与调用者、数据模型、相关契约、测试和可用近期历史；搜索类似实现。文件审查无历史时按现有代码评估，不假装已确认何时引入问题。
3. **选择规范。** 按下表加载实际涉及的领域；新增领域取并集，不递归读取所有参考文档。
4. **验证发现。** 沿调用和数据流确认触发条件与影响，区分本次引入的问题和既有问题，核对规范例外。需要验证推断时，只运行已有且不会改写源文件或影响共享环境的相关测试、静态检查。
5. **形成结果。** 按严重程度排序并合并同根因发现。对差异审查，优先报告本次引入或加剧的问题；无关既有问题不混入本次发现。说明未验证项，不自动进入修复流程。

## 规范与检查重点

| 涉及领域 | 参考规范 | 检查重点 |
| --- | --- | --- |
| Java 实现 | [Java](../java-spring-backend/references/coding/java.md) | 命名、Lombok、`class` / `record`、方法、集合、异常、日志及模型的 Java 实现风格 |
| 模型、Package、分层与 SOLID | [分层](../java-spring-backend/references/architecture/layering.md) | 职责与包归属，Request / Query / DTO / BO / DO / VO 分类，技术基础设施归属，依赖方向，SRP / OCP / LSP / ISP / DIP，以及是否过度抽象 |
| Spring、Service、Controller | [Spring](../java-spring-backend/references/coding/spring.md)、[分层](../java-spring-backend/references/architecture/layering.md) | Controller 越层、HTTP 语义下沉、反向依赖、跨模块访问、无必要 Manager 或抽象 |
| API 与业务行为 | [API](../java-spring-backend/references/api/api-design.md)、[分层](../java-spring-backend/references/architecture/layering.md) | 未授权 API 变化，具体业务输出是否使用 VO，统一响应是否遵循项目契约，字段语义、状态、默认值、校验、兼容行为与敏感字段暴露 |
| MyBatis 与映射 | [MyBatis](../java-spring-backend/references/coding/mybatis.md)、[SQL](../java-spring-backend/references/database/sql.md)、[分层](../java-spring-backend/references/architecture/layering.md) | Mapper 职责、TypeHandler 等基础设施归属、显式映射、数据库拼音泄漏、参数绑定与动态 SQL 白名单 |
| 数据库结构与 SQL | [数据库设计](../java-spring-backend/references/database/database-design.md)、[SQL](../java-spring-backend/references/database/sql.md) | 拼音术语复用、约束、数据完整性、注入、`SELECT *`、N+1、分页稳定性、写入条件 |
| 事务、锁、一致性 | [事务](../java-spring-backend/references/architecture/transactions.md) | 不必要或缺失的事务边界、回滚、传播、自调用、查询后修改竞态 |
| 并发、异步、线程池 | [并发](../java-spring-backend/references/architecture/concurrency.md) | 实际收益、操作独立性、线程与连接池、上下文、异常；涉及事务同时加载事务规范 |
| Bug 修复、行为变化、测试 | [测试](../java-spring-backend/references/coding/testing.md) | 有效回归覆盖、重要边界、断言强度、可重复性及真实验证结果 |
| 权限、租户、数据范围 | 目标项目已有安全规范、契约和实现 | 是否绕过认证或数据隔离、扩大数据范围、硬编码或记录敏感凭证 |

只修改 SQL 时不因表中已有拼音字段就报告 Java 命名问题；仅查询多个 Mapper 不构成事务缺失证据。评估缺陷要结合具体调用和业务要求，不按关键词机械判定。

## 模型与 Package 审查原则

模型职责与 Package 归属以 `layering.md` 为唯一详细事实来源，`java.md` 只负责 Java 实现方式。

新增或调整以下模型时检查：

```text
Request → 接口输入 → <module>.request
Query   → 查询条件 → <module>.query
DTO     → 内部传输 → <module>.dto
BO      → 业务处理 → <module>.bo
DO      → 持久化   → <module>.domain
VO      → 视图输出 → <module>.vo
```

重点检查：

- 是否把所有数据对象机械放入 `dto`；
- `Query` 是否因为 Mapper 使用而放入 `mapper`；
- 具体业务输出是否错误新增为 `*Response` / `response` 包，而不是 VO；
- DO 是否直接暴露为 HTTP 输出；
- 通用技术基础设施是否因为被某业务模块使用就放入该业务 Package；
- 是否机械创建无实际职责的 DTO / BO / Converter / Assembler。

统一 HTTP 响应默认可以使用 `ApiResponse<T>`，但这只是 Skill 默认推荐。如果目标项目已有其他统一响应类型、历史 API 或固定序列化契约，以项目为主，不得为了改成 `ApiResponse<T>` 报错或要求无授权迁移。

## SOLID 审查原则

SOLID 是代码设计审查维度，不是要求所有代码套用接口、设计模式或额外分层的模板。

只有当本次变更出现真实的职责混乱、扩展困难、契约破坏、接口污染或对易变技术实现的强耦合时，才形成 SOLID 相关发现。

不得仅因为存在另一种“更优雅”的设计就报告问题。

### S — 单一职责原则（SRP）

检查类、接口或模块是否同时承担多个明显不同且独立变化的职责。

重点检查：

- Service 是否同时承担 HTTP、SQL、第三方协议、文件解析、数据库字段转换等无关职责；
- Controller 是否混入业务流程；
- Mapper 是否混入业务决策；
- 通用技术组件是否混入具体业务逻辑；
- 一个类是否因为职责混杂导致修改一个需求时需要同时触碰多个不相关领域。

例如：

```text
PlaceService
  ├─ 场所审核业务流程
  ├─ HTTP 状态码构造
  ├─ 第三方 SDK 调用细节
  └─ JSON 字段解析
```

可能存在 SRP 问题。

但不要机械认为：

```text
一个类方法较多
→ 一定违反 SRP
```

也不要仅为满足 SRP 建议“一方法一类”或大量无价值的 Manager / Converter。

---

### O — 开闭原则（OCP）

检查本次变更是否在一个已经存在明确、多实现、稳定变化方向的核心流程中继续增加大量分支，而项目已有或确实需要稳定扩展点。

例如：

```text
if (type == A) ...
else if (type == B) ...
else if (type == C) ...
```

以下情况更值得报告：

- 同类分支持续增加；
- 每增加一种类型都必须修改核心流程；
- 已有明确扩展机制却被绕过；
- 不同分支存在明显独立且稳定的策略职责。

不要因为当前只有两个简单分支，就机械要求：

```text
Strategy + Factory
```

也不要为了“未来可能扩展”提出没有现实依据的抽象。

---

### L — 里氏替换原则（LSP）

检查新增或修改的实现类能否保持父类或接口已有契约。

重点检查：

- 实现类是否缩小可接受输入范围；
- 是否改变既有返回语义；
- 是否新增调用方无法合理预期的副作用；
- 是否改变 Null、异常或状态变化约定；
- 是否通过 `UnsupportedOperationException` 等方式拒绝父类型要求的核心能力。

如果父类型核心方法只能这样实现：

```java
@Override
public void audit(...) {
    throw new UnsupportedOperationException();
}
```

通常应检查抽象关系是否合理。

不要仅因为不同实现内部代码不同就认为违反 LSP。

---

### I — 接口隔离原则（ISP）

检查接口是否迫使调用方或实现方依赖大量与自身无关的能力。

重点检查：

- 一个公共 Facade / SPI 是否承担多个不相关能力；
- 某些实现是否被迫提供大量无意义方法；
- 调用方是否为了一个很小的能力依赖一个非常宽泛且易变化的接口。

只有存在真实调用边界或实现负担时才建议拆分。

不要根据接口方法数量直接判定违反 ISP。

---

### D — 依赖倒置原则（DIP）

检查高层业务逻辑是否直接耦合易变化的底层技术实现。

重点关注：

- 第三方 SDK；
- 外部 HTTP 服务；
- 对象存储；
- 消息系统；
- 可替换算法；
- 多供应商实现；
- 需要隔离测试的外部技术组件。

例如：

```text
CaseService
    ↓
VendorFaceSdk
```

如果供应商实现属于易变技术细节，且业务层需要直接理解其协议、异常和对象类型，应检查是否需要稳定适配边界。

但禁止机械报告：

```text
Service 没有接口
→ 违反 DIP
```

也不要要求所有代码改成：

```text
XxxService
    ↓
XxxServiceImpl
```

Mapper 接口本身通常已经构成数据访问边界，不应仅为了 DIP 再包装无实际价值的 Repository / RepositoryImpl。

---

### SOLID 与过度设计

代码审查必须同时检查“违反 SOLID”和“错误套用 SOLID”两种方向。

以下新增内容如果没有真实职责、替换、扩展或隔离需求，应检查是否属于过度设计：

```text
ServiceInterface + ServiceImpl
Strategy
Factory
AbstractFactory
Adapter
Repository + RepositoryImpl
大量单方法接口
只有一个实现且无替换需求的抽象层
```

正确审查方式：

```text
职责是否真实混乱？
        ↓ 是
考虑 SRP 问题

是否存在真实且稳定的变化方向？
        ↓ 是
考虑 OCP

实现是否破坏已有抽象契约？
        ↓ 是
考虑 LSP

调用方 / 实现方是否被迫依赖无关能力？
        ↓ 是
考虑 ISP

高层业务是否直接耦合易变技术细节？
        ↓ 是
考虑 DIP
```

核心原则：

> SOLID 用来识别真实设计风险，也用来防止错误抽象；不以“更理论化”代替“更适合当前代码”。

## 发现成立条件

- 指出具体代码位置与适用规则或可复现的触发条件，说明实际影响；纯规范违规也应给出原文规则依据。
- 核对明确例外：模块统一使用 record 或任务要求时不能按默认 class 规则报错；普通查询默认无显式事务，但一致性快照、锁或原子写入可能需要事务。
- 新增通用 TypeHandler 错放业务 mapper 包、新增视图模型误用 DTO、HTTP 语义进入 Service 等，按对应领域规则判断；不要扩展为对历史代码的全仓库改造。
- 新增具体业务输出模型时，如果无兼容性或项目既有风格例外，使用 `*Response` / `response` 包而不是 `*VO` / `vo` 包，可按分层与 API 规范形成发现；统一 HTTP 包装类型不属于具体业务模型。
- `ApiResponse<T>` 是默认推荐而非绝对要求；目标项目已有统一响应契约时必须以项目为主。
- 不得为了符合 VO 命名而要求无授权地重命名已经发布的历史 API 模型；兼容性优先。
- SOLID 发现必须指出具体职责冲突、变化点、契约破坏、接口负担或技术耦合；不能只写“建议遵循 SOLID”“建议抽接口”之类泛化意见。
- 不得因为某个类没有接口、没有 Strategy / Factory、没有 Repository 包装就认定违反 SOLID。
- 如果新增接口、Strategy、Factory、Adapter、Repository 等抽象没有实际多实现、扩展、隔离或复用需求，也可以按“过度设计”形成有证据的维护性发现。
- 若缺少业务契约而无法确认风险，将其列为待确认事项，不描述成已发生的数据损坏或既定业务缺陷。
- 同一问题报告一次，建议最小修复方向，不顺带设计无关架构。

## 输出格式

先给出按严重程度排序的发现，再列待确认事项及验证情况。每个发现包含：

- **严重程度与标题**：P0 为已确认会阻断运行或造成广泛严重损害的紧急问题；P1 为明确的重大功能、安全或数据完整性风险；P2 为局部缺陷或明确的规范违规；P3 为影响较低但有依据的改进项。按实际影响判断，不因规则使用“必须”就升级严重程度。
- **位置**：文件和精确行号，优先定位本次差异中的相关语句。
- **证据与影响**：具体触发条件或适用规则、调用关系及影响；引用规范时标明文档与章节。SOLID 类发现应指出对应的 S / O / L / I / D 原则及具体违反原因。
- **最小修复建议**：说明应调整什么，不直接改写代码；SOLID 问题优先建议最小职责或依赖调整，不自动建议引入新的设计模式。

没有已确认发现时明确写“未发现已确认的问题”，并说明检查范围和局限，不等同于保证没有缺陷。验证情况区分实际运行通过、失败和未执行；已有结果仍适用时可以复用并说明来源，不重复运行。
