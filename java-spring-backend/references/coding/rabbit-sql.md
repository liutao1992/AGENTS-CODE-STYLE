# Rabbit-SQL 编码规范

本文档定义 Rabbit-SQL 及其 Spring Boot Starter 的项目使用规范。

本文负责回答：

> `@XQLMapper`、`.xql`、`XQLFileManager`、Baki、参数绑定、动态 SQL、分页、Stream、Batch、缓存和 Spring 事务应该如何组织和使用。

SQL 本身的正确性、安全范围、PostgreSQL 语义和性能统一读取：

- [SQL 与 PostgreSQL](../database/sql.md)

其他相关规范：

- [应用分层与模型边界](../architecture/layering.md)
- [项目与业务模块目录](../architecture/project-structure.md)
- [Java 编码](java.md)
- [Spring](spring.md)
- [事务](../architecture/transactions.md)
- [测试](testing.md)

MyBatis / MyBatis-Plus 是另一套持久层框架，具体规则读取：

- [MyBatis / MyBatis-Plus](mybatis.md)

核心原则：

> Rabbit-SQL Mapper 仍然是数据库出站适配边界；业务代码优先通过职责明确的 `@XQLMapper` 接口访问数据库，不让 Baki、SQL 名称和 XQL 技术细节扩散到 Service / Controller。

> 数据值使用 `:name` 命名参数进行预编译绑定；`${}` 只用于可信的 SQL 模板或经过白名单映射的 SQL 结构，不能承载未经信任的外部输入。

> 动态 XQL 只表达 SQL 结构和查询条件，不替代 Controller 的结构校验、Service / Manager 的业务规则、权限判断和事务设计。

> 目标项目已有 Rabbit-SQL 版本、配置方式和稳定约定时优先沿用，不为了本规范升级依赖、切换持久层框架或批量迁移已有代码。

---

## 1. 先确认当前项目确实使用 Rabbit-SQL

出现以下任一特征时，再加载本文：

```text
rabbit-sql
rabbit-sql-spring-boot-starter
@XQLMapper
@XQLMapperScan
@XQL
@Arg
Baki / BakiDao
XQLFileManager
xql-file-manager.yml
*.xql
PagedResource
@CountQuery
@PageableConfig
```

如果项目使用的是 MyBatis / MyBatis-Plus，不因为接口也叫 Mapper 就套用本文。

同样，Rabbit-SQL 的 `@XQLMapper` 接口不要机械套用 MyBatis-Plus 的：

```text
BaseMapper<T>
QueryWrapper
LambdaQueryWrapper
Mapper XML
```

规则。

一个项目同时存在 Rabbit-SQL、MyBatis、JPA 等框架时，应根据具体接口的依赖、注解、SQL 资源和调用链判断当前数据访问实现，不能仅凭 `Mapper` / `DAO` 名称猜测框架。

原则：

> 先识别持久层技术，再加载对应框架规范；逻辑上的 Mapper / DAO 职责可以相同，技术实现规则不能混用。

---

## 2. 业务代码优先使用 `@XQLMapper` 接口

Rabbit-SQL 同时提供 Baki API 和 XQL 接口映射。

在普通业务模块中，默认优先：

```text
Service / Manager
      ↓
@XQLMapper Mapper
      ↓
*.xql
      ↓
Database
```

例如：

```java
@XQLMapper("place")
public interface PlaceMapper {

    PlaceDO getById(@Arg("id") String id);

    List<PlaceDO> listByQuery(PlaceQuery query);
}
```

避免在普通 Service / Controller 中直接：

```java
baki.query("&place.listByQuery")
baki.execute("&place.updateStatus", args)
```

否则容易形成：

```text
业务层
→ Rabbit-SQL API
→ XQL alias / SQL name
→ 参数 Map
```

使数据库访问协议和 SQL 定位信息扩散到业务层。

Baki 更适合：

* 数据访问基础设施；
* 确实不适合接口映射的低层通用能力；
* 开发诊断、工具或受控技术场景；
* 目标项目已有稳定 Baki 数据访问封装。

如果业务场景确实需要 Baki，也应优先把它收口在职责明确的数据访问边界，而不是让上层到处直接拼 SQL 或 SQL 名称。

原则：

> 业务层依赖数据访问契约，不依赖 Rabbit-SQL 执行 API；`@XQLMapper` 是普通业务数据访问的默认边界，Baki 是底层能力，不是 Service 的快捷 SQL 工具。

---

## 3. Mapper 与 XQL 资源组织

Rabbit-SQL Mapper 继续遵循通用分层规范，默认位于：

```text
<module>.mapper
```

XQL 资源建议集中在：

```text
src/main/resources/
├── xql-file-manager.yml
└── xqls/
    ├── place/
    │   └── place.xql
    ├── casecenter/
    │   └── case.xql
    └── equipment/
        └── equipment.xql
```

如果目标项目已经有稳定资源目录，例如：

```text
sql/
xql/
rabbit-sql/
```

则沿用已有结构，不为了本文迁移。

`xql-file-manager.yml` 中的 alias 使用稳定、清晰的业务语义，例如：

```yaml
files:
  place: xqls/place/place.xql
  case: xqls/casecenter/case.xql
```

避免：

```text
a
sql1
common
misc
all
```

这种无法表达职责的别名。

不要把整个系统所有 SQL 都放进一个巨大 XQL 文件。优先按业务模块、聚合或稳定数据访问职责拆分，使：

```text
Mapper
↔ XQL alias
↔ XQL file
↔ SQL object
```

能够快速对应。

默认优先使用 classpath 中随应用版本管理的 XQL 文件。Rabbit-SQL 支持 `file://`、FTP、HTTP(S) 等远程资源，但普通业务系统不要无明确需求引入运行时远程 SQL：它会增加配置漂移、可用性、权限和部署一致性风险。

确实使用远程 XQL 时至少确认：

* 来源可信且访问受控；
* 不在仓库中硬编码 Token / Secret；
* 配置和发布过程能够追踪版本；
* 远程不可用时的启动 / 运行行为已有明确设计；
* 不允许调用者通过输入决定任意远程 SQL 地址。

---

## 4. SQL 对象名与 Mapper 方法保持明确映射

XQL SQL 对象使用：

```sql
/*[listByQuery]*/
select ...;
```

进行命名。

默认优先让 SQL 对象名与 Mapper 方法名一致：

```text
PlaceMapper.listByQuery
↕
/*[listByQuery]*/
```

这样代码搜索、插件跳转、慢 SQL 排查和人工阅读都更直接。

只有真实需要以下情况时再使用 `@XQL` 指定 SQL 名或行为：

* 方法名与 SQL 对象名确实不同；
* 同一 SQL 需要映射到不同返回类型的方法；
* 方法名前缀无法可靠推断 SQL 类型；
* 需要显式覆盖默认执行类型。

不要给所有方法机械添加冗余 `@XQL`。

### 4.1 方法命名继续遵循团队 Mapper 规约

项目自定义 Rabbit-SQL Mapper 默认仍使用：

```text
获取单个对象 → get
获取多个对象 → list
获取统计数量 → count
插入         → insert
删除         → delete
修改         → update
```

例如：

```text
getById
getByCode
listByQuery
listByStatus
countByQuery
insert
updateStatus
deleteById
```

Rabbit-SQL 官方默认可根据方法名前缀推断 SQL 类型，其中查询前缀包含 `select / query / find / get / fetch / search / list`，写操作包含 `insert / save / add / append / create`、`update / modify / change`、`delete / remove` 等。

当前团队规范中的 `get`、`list`、`insert`、`update`、`delete` 可以直接与框架推断规则对齐。

但 `count` 不属于框架默认查询前缀，因此 `count...` 方法不要依赖隐式推断，应显式声明查询类型：

```java
@XQL(type = SqlStatementType.query)
long countByQuery(PlaceQuery query);
```

如果 SQL 对象名也不同，同时指定 `value`：

```java
@XQL(value = "countEnabled", type = SqlStatementType.query)
long countByStatus(@Arg("status") String status);
```

原则：

> 团队命名规约优先保持稳定；框架无法从命名可靠推断行为时用注解补足，不反过来为了框架推断改变清晰的团队命名。

---

## 5. Mapper 参数优先使用类型明确的对象或 `@Arg`

Rabbit-SQL 映射接口支持：

```text
单个 Map / DataRow / JavaBean
多个 @Arg 参数
批量 Iterable
```

普通业务查询条件较多时，优先使用已有 Query：

```java
List<PlaceDO> listByQuery(PlaceQuery query);
```

对应 XQL 可以直接按属性名使用命名参数：

```sql
/*[listByQuery]*/
select id, csbh, zt
from csxx
where 1 = 1
-- #if :placeCode != blank
  and csbh = :placeCode
-- #fi
-- #if :status != null
  and zt = :status
-- #fi
;
```

只有少量独立参数时可以使用：

```java
int updateStatus(@Arg("id") String id, @Arg("status") String status);
```

参数名必须与 XQL 中命名参数一致。

不要为了省建模在稳定业务接口中默认使用：

```java
Map<String, Object>
DataRow
Object[]
```

承载本来具有明确业务语义的输入。

这些动态结构只在数据结构本身确实动态、框架基础设施或目标项目已有明确契约时使用。

普通方法参数数量、Query 和参数对象设计继续读取 `java.md`；Query 默认位于 `<module>.request`。

---

## 6. 数据值统一使用 `:name` 预编译参数

Rabbit-SQL 的：

```sql
:id
:name
:status
```

属于命名参数，最终通过 PreparedStatement 方式绑定。

所有来自：

```text
HTTP Request
RPC / Message
Service 参数
数据库查询结果
外部系统返回
用户输入
```

的数据值，默认都通过 `:name` 传入。

例如：

```sql
where id = :id
  and zt = :status
```

不要因为字符串拼接更方便改成模板替换。

原则：

> 能作为数据值绑定的内容全部使用 `:name`；PreparedStatement 参数是默认路径，不把数据值变成 SQL 文本。

---

## 7. 严格限制 `${}` 字符串模板

Rabbit-SQL 的：

```text
${name}
${!name}
```

是字符串模板替换，不属于 PreparedStatement 数据参数。

即使 `${!name}` 对部分字符串集合做安全处理，也不能把它理解成与 `:name` 等价的预编译参数，更不能因此直接接收任意外部输入。

`${}` 只允许用于两类场景：

### 7.1 XQL 内部可信模板复用

例如：

```sql
/*{whereCondition}*/
where id = :id;

/*[getById]*/
select id, csbh
from csxx
${whereCondition};
```

模板内容由版本受控的 XQL 文件定义，不来自外部请求。

### 7.2 无法预编译的 SQL 结构

例如确实需要动态：

```text
列名
表名
ORDER BY 字段
排序方向
受控 SQL 片段
```

必须先在 Java / 稳定配置边界中完成 Enum 或白名单映射：

```text
外部输入
→ Enum / allow-list
→ 固定 SQL 标识符
→ ${}
```

禁止：

```text
用户输入
→ ${}
→ SQL
```

对于 `IN` 值集合，优先通过 `#for` + `:item` 保持逐值预编译，而不是为了方便把用户集合直接做字符串模板替换。

SQL 注入和动态排序等安全规则同时读取 `sql.md`。

---

## 8. XQL 中禁止硬编码业务常量

`.xql` 中不要直接写死业务状态、业务类型、来源编码、固定业务标识等常量。

避免：

```sql
where zt = '1'
  and lx = 'FORMAL'
```

应由 Mapper 参数显式传入：

```java
List<PlaceDO> listByStatusAndType(@Arg("status") String status, @Arg("type") String type);
```

```sql
where zt = :status
  and lx = :type
```

如果值域固定，Java 侧优先根据 `java.md` 使用职责明确的 Enum，并在数据访问边界传入已有稳定编码。

这条规则针对业务常量，不意味着 SQL 中不能出现任何字面量。`COUNT(*)`、`IS NULL`、数据库函数参数以及真正属于 SQL 结构的值按 SQL 语义判断。

原则：

> XQL 负责 SQL 表达，业务编码由 Java 业务契约拥有；不要让同一业务状态同时散落在 Java 和多个 XQL 文件中。

---

## 9. 动态 SQL 只表达 SQL 结构

Rabbit-SQL 通过 SQL 注释提供：

```text
#check
#var
#if / #else / #fi
#guard / #throw
#switch / #case / #end
#choose / #when / #end
#for / #done
```

等动态能力。

这些能力用于：

* 可选查询条件；
* SQL 分支；
* `IN` 参数展开；
* 数据库方言差异；
* SQL 局部变量和结构拼装。

不要在 XQL 中实现：

```text
用户权限业务判断
审核状态机
完整业务流程
跨数据源业务校验
Controller 已负责的结构校验
Service / Manager 应负责的业务规则
```

### 9.1 `#if` 用于可选 SQL 条件

例如：

```sql
-- #if :status != null
  and zt = :status
-- #fi
```

不要嵌套大量 `#if` 最终形成难以理解的 SQL 程序。如果分支已经代表不同的数据访问语义，优先拆成不同 SQL 对象 / Mapper 方法。

### 9.2 `#for` 优先用于参数化集合

构造 `IN` 时保持值参数化：

```sql
and id in (
-- #for item of :ids; last as isLast
  :item
  -- #if !:isLast
  ,
  -- #fi
-- #done
)
```

重点不是“能拼出 SQL”，而是确保集合元素仍然通过命名参数进入预编译流程。

### 9.3 `#check` 不替代业务校验

`#check` 可以用于与当前 SQL 执行直接相关的必要前置条件，但不要重复已经由可信入站 Bean Validation 保证的同义结构约束，也不要把业务状态、权限、跨表规则下沉到 XQL。

判断顺序仍然是：

```text
结构约束
→ 入站边界

业务规则
→ Service / Manager

SQL 执行条件
→ XQL / Database
```

### 9.4 `#var` 不承载业务决策

`#var` 适合轻量 SQL 局部计算和动态脚本辅助变量；如果变量本身代表业务状态转换、权限结果或复杂业务算法，应在 Java 业务层产生后作为明确参数传入。

---

## 10. SQL 片段复用以可读性为前提

Rabbit-SQL 支持独立模板和内联模板。

只有多个 SQL 确实需要保持同一逻辑时才提取模板，不为了减少几行 SQL 创建层层递归引用。

特别适合复用的是分页列表和 count 查询的共同条件：

```text
list/page SQL condition
          ↕
shared inline condition
          ↕
count SQL condition
```

这样可以减少：

```text
列表查询增加条件
但 count 忘记同步
```

导致的分页总数错误。

优先使用作用域明确的内联模板处理只服务于当前 SQL 组的片段，避免把局部条件污染成全局模板。

原则：

> SQL 复用只提取稳定、同职责的片段；能够减少语义漂移才复用，不能为了 DRY 牺牲可定位性。

---

## 11. 返回类型不要泄漏无必要的框架动态结构

Rabbit-SQL 支持：

```text
List / Set / Stream
Optional
Map
DataRow
JavaBean
PagedResource / IPageable
基础标量
BatchResult
```

普通业务 Mapper 优先返回职责明确的 Java 类型，例如：

```text
PlaceDO
List<PlaceDO>
PlaceStatsDO
```

不要无必要让：

```text
DataRow
Map<String, Object>
```

一路泄漏到 Service、Controller 和对外 API。

如果 SQL 是动态列、通用报表或基础设施能力，确实无法稳定建模时可以保留 DataRow / Map，但调用边界必须清楚。

数据库 DO 不直接作为外部 API 输出，具体模型边界读取 `layering.md`。

`PagedResource<T>` / `IPageable` 是 Rabbit-SQL 技术分页类型。目标项目已有统一分页输出契约时，应在合适边界转换，不把框架分页类型直接变成公共 HTTP 契约。

单对象“不存在”和 Optional 的使用遵循目标项目现有契约，不为了 Rabbit-SQL 支持 Optional 就批量改变历史方法签名。

---

## 12. 分页查询明确 count 语义

Rabbit-SQL 可以自动构建分页和 count，也支持：

```text
@CountQuery
@PageableConfig
```

对于简单查询可以复用框架默认分页机制。

以下情况应重点检查 count 是否与数据查询语义一致：

* `GROUP BY`；
* `DISTINCT`；
* 多层 CTE；
* 复杂 JOIN；
* 自定义分页 SQL；
* 列表和 count 使用不同动态条件；
* 默认 count 改写无法可靠表达实际总数。

需要独立 count SQL 时显式使用 `@CountQuery` 关联，并优先复用稳定的共同条件模板，避免列表数据和 total 漂移。

不要仅因为框架能够自动 count 就假设复杂 SQL 的 total 一定正确；实际 SQL 语义仍由 `sql.md` 判断。

---

## 13. Stream 查询必须明确连接生命周期

Rabbit-SQL 的 Stream 查询是惰性查询，只有终端操作时才真正执行，并持有底层数据库资源。

使用 Stream 时必须在明确边界关闭：

```java
try (Stream<PlaceDO> stream = placeQuery.stream()) {
    return stream.map(...).toList();
}
```

不要：

* 忘记 close；
* 把未关闭 Stream 存入成员变量；
* 把数据库 Stream 直接作为 Controller HTTP 返回值；
* 在 Stream 生命周期不明确时跨线程消费；
* 为了所谓“少一次循环”机械把所有 List 查询改成 Stream。

只有数据量、处理方式和资源生命周期确实适合惰性消费时再选择 Stream。

原则：

> Stream 是数据库资源生命周期，不只是 Java 集合 API；谁创建惰性查询，谁必须明确关闭责任。

---

## 14. 批量写入使用框架批处理，不循环单条执行

批量插入 / 更新时，优先使用 Rabbit-SQL 的 batch 能力或目标项目已有批处理封装，避免：

```text
for each item
→ 单条 SQL
→ 单次数据库往返
```

接口映射可以使用 batch 类型，Baki 也提供集合批量执行能力。

批大小遵循目标项目和数据库容量，不自行发明固定数字。Rabbit-SQL 的 batchSize、数据库连接池、单条数据大小、事务范围和 PostgreSQL 参数限制需要一起考虑。

是否应该把整个批次放在同一事务中由 `transactions.md` 的一致性需求决定，不因为“batch”自动扩大事务范围。

---

## 15. Spring Boot 项目统一使用 Spring 事务体系

使用 `rabbit-sql-spring-boot-starter` 时，Rabbit-SQL 可以参与 Spring 管理的数据库事务。

事务边界仍然先读取 `transactions.md`：

```text
先确定哪些数据库操作必须共同提交 / 回滚
→ 再确定 Service / Manager 谁拥有边界
→ 再选择 Spring 事务实现方式
```

项目已有 `@Transactional` / `TransactionTemplate` 时优先保持统一，不为了 Rabbit-SQL 再引入第二套事务习惯。

Spring Boot Starter 虽然提供自己的 Spring `Tx` 简易封装，但如果目标项目没有使用它，不因为框架支持就主动引入。

特别禁止在 Spring Boot Starter 项目中混用 Rabbit-SQL Core 自带的旧事务实现：

```text
com.github.chengyuxing.sql.transaction.Tx
```

Starter 已由 Spring 全局事务接管。

如果 Rabbit-SQL 与 MyBatis / JPA 在同一业务事务中共同访问数据库，必须确认：

* 使用兼容的 Spring TransactionManager；
* 数据源确实属于同一事务资源；
* 多数据源场景没有误用默认事务管理器；
* 不假设跨线程事务传播。

Spring Proxy 和事务 API 细节读取 `spring.md`。

---

## 16. 缓存不是默认优化

Rabbit-SQL 提供 `QueryCacheManager` 扩展点，但不要仅因为框架支持就开启查询缓存。

引入前必须明确：

```text
重复查询是否真实存在
缓存 Key 是否完整
TTL 的业务依据
写入后如何失效
事务提交前后如何处理
多实例是否一致
允许多长时间陈旧
```

没有监控、压测或明确业务收益时，不为推测性能收益增加缓存。

缓存不能用于掩盖明显慢 SQL、错误索引或 N+1。

---

## 17. SQL 可观测性与日志

Rabbit-SQL 提供 SQL interceptor、execution watcher 等扩展能力，可用于 SQL 观察、耗时统计和诊断。

如果项目已有统一 SQL 监控 / tracing / metrics，应优先接入现有体系，不创建第二套。

日志和监控应能够帮助定位：

```text
XQL alias / SQL object
执行耗时
失败类型
必要的调用上下文
```

但不得为了排查方便记录：

```text
密码
Token / Secret
完整证件信息
敏感业务正文
未脱敏的大量 SQL 参数
```

慢 SQL 判断需要结合实际执行计划、数据规模和数据库指标；没有证据时不要仅凭 SQL 长度下结论。

---

## 18. IDEA Rabbit SQL 插件作为开发辅助，不替代规范和测试

官方插件可以提供：

* XQL 文件创建和注册；
* SQL 名自动完成；
* Java ↔ XQL 跳转；
* 动态 SQL 解析 / 执行测试；
* Mapper 接口生成；
* SQL 描述和元数据查看。

推荐使用插件降低手写 alias、SQL 名和动态脚本错误。

但代码生成结果仍需遵守目标项目：

```text
Package
命名
方法换行
模型职责
异常
事务
```

等规范。不要因为代码由插件生成就保留与项目风格冲突的模板格式。

插件中的动态 SQL 手工测试不能替代自动化测试。

---

## 19. Rabbit-SQL 测试重点

新增或修改 XQL 时至少检查：

1. XQL alias 是否正确注册。
2. `@XQLMapper` value 是否与 alias 对应。
3. Mapper 方法与 SQL 对象名是否一致；不一致时 `@XQL` 是否明确。
4. 方法执行类型能否正确推断；`count...` 等非默认前缀是否显式指定 query。
5. Java 参数名 / `@Arg` 是否与 `:name` 一致。
6. 所有外部数据值是否保持预编译绑定。
7. `${}` 是否仅来自可信模板或白名单 SQL 结构。
8. 动态 SQL 每个主要分支是否都能生成合法 SQL。
9. 列表与 count 的过滤条件是否一致。
10. Stream 是否在所有退出路径关闭。
11. Batch 行为、失败语义和事务范围是否正确。
12. PostgreSQL 特有 SQL 是否在真实兼容环境验证。
13. 变更是否破坏现有 Mapper 返回模型和上层契约。

涉及真实 SQL 行为时优先使用项目已有集成测试；数据库语义重要时可按 `testing.md` 评估 Testcontainers 或测试数据库。

---

## 20. Codex Rabbit-SQL 修改流程

修改 Rabbit-SQL 代码时：

1. **识别框架。** 从依赖、`@XQLMapper`、Baki、`.xql`、配置确认是 Rabbit-SQL，不套用 MyBatis-Plus 规则。
2. **定位边界。** 找到业务 Mapper、XQL alias、XQL 文件和 SQL 对象的完整映射关系。
3. **优先接口映射。** 普通业务查询优先通过 `@XQLMapper`；Baki 不无依据扩散到 Service / Controller。
4. **保持命名一致。** SQL 对象名默认与 Mapper 方法一致，方法继续遵循 `get / list / count / insert / update / delete` 规约。
5. **确认执行类型。** `count...` 或其他无法由框架前缀可靠推断的方法显式使用 `@XQL(type = ...)`。
6. **设计参数。** 多条件使用 Query / 明确 JavaBean，少量参数使用 `@Arg`；不默认使用 Map / DataRow 参数袋。
7. **检查注入边界。** 数据值使用 `:name`；`${}` 只用于可信模板或白名单 SQL 结构。
8. **检查业务常量。** 状态、类型、来源编码等不硬编码在 XQL，由 Java 参数传入。
9. **控制动态 SQL。** `#if / #for / #choose` 只表达 SQL 结构，不承载完整业务规则。
10. **检查返回模型。** DataRow / Map / PagedResource 等框架类型不无必要泄漏到业务和 HTTP 边界。
11. **检查资源。** Stream 明确关闭，Batch 避免逐条数据库往返。
12. **检查事务。** Spring Boot 项目统一读取 `transactions.md` 和 `spring.md`，不混用 Core Tx。
13. **检查性能。** 缓存、并行和批大小必须有真实依据；慢 SQL 读取 `sql.md` 并结合执行计划。
14. **验证。** 测试 XQL 注册、映射、参数、动态分支、分页 count、事务和数据库兼容性。

最终原则：

> Rabbit-SQL 的价值是让 SQL 保持原生、显式、可定位，同时提供接口映射和动态能力；最佳实践不是把更多逻辑塞进 XQL，而是让 Mapper 契约清楚、SQL 可搜索、参数安全、资源和事务边界明确。
