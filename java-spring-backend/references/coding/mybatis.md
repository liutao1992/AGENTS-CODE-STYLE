# MyBatis 编码规范

本文档定义 MyBatis 框架使用规范。

本文负责回答：

> Mapper、Mapper XML、参数绑定、ResultMap、TypeHandler、动态 SQL 和 MyBatis 技术组件应该怎么组织和实现。

SQL 本身的正确性、安全范围、PostgreSQL 语义和性能统一读取：

- [SQL 与 PostgreSQL](../database/sql.md)

其他相关规范：

- [应用分层与模型边界](../architecture/layering.md)
- [数据库设计](../database/database-design.md)
- [事务](../architecture/transactions.md)
- [Java 编码](java.md)

核心原则：

> Mapper 负责数据访问接口和 MyBatis 映射，不负责业务流程。

> 数据库物理命名与 Java 业务命名通过 MyBatis 显式隔离。

> MyBatis 技术规则由本文维护，SQL 规则不在本文重复维护。

---

## 1. Mapper 职责

Mapper 属于数据库出站适配边界。

主要负责：

* 定义数据访问方法；
* 参数映射；
* 结果映射；
* 关联 Mapper XML；
* 执行对应 SQL。

例如：

```java
public interface PlaceMapper {

    PlaceDO getById(String id);

    List<PlaceDO> listByQuery(PlaceQuery query);

    int insert(PlaceDO place);

    int update(PlaceDO place);
}
```

Mapper 不负责：

* HTTP / RPC；
* 权限业务判断；
* 状态流转；
* 完整业务流程；
* 业务事务编排；
* 第三方服务调用。

分层职责统一读取：

- [layering.md](../architecture/layering.md)

---

## 2. Mapper Package

业务模块中的：

```text
<module>.mapper
```

只用于该模块的数据访问 Mapper。

例如：

```text
place.mapper.PlaceMapper
case.mapper.CaseMapper
```

不得因为一个技术类被 Mapper 使用，就把它放入业务 `mapper` Package。

例如：

```text
place.mapper.JsonStringListTypeHandler
```

通常职责错误，因为 TypeHandler 是 MyBatis 技术基础设施，不是 Place Mapper。

原则：

> Package 由组件自身职责决定，不由当前使用者决定。

---

## 3. MyBatis 技术基础设施

通用 MyBatis 技术组件可以按目标项目已有结构归入：

```text
common.mybatis.handler
common.mybatis.interceptor
common.mybatis.plugin
common.mybatis.config
```

典型关系：

```text
TypeHandler   → common.mybatis.handler
Interceptor   → common.mybatis.interceptor
Plugin        → common.mybatis.plugin
Configuration → common.mybatis.config
```

如果目标项目已有不同但职责清晰的公共 Package，应沿用项目结构，不为了本文重新创建第二套目录。

---

## 4. 新增 MyBatis 组件前先搜索

新增以下类型前：

```text
Mapper
TypeHandler
Interceptor
Plugin
ResultHandler
MyBatis Configuration
```

必须先确认：

1. 是否已有相同或类似能力；
2. 同类组件当前放在哪里；
3. 是否可以直接复用；
4. 当前组件属于业务 Mapper 还是通用技术基础设施；
5. 是否真的需要新增。

流程：

```text
判断职责
→ 搜索同类实现
→ 确定 Package
→ 创建或修改
```

---

## 5. Mapper 方法命名

方法名应表达明确的数据访问意图。

例如：

```text
getById
getByCode
listByQuery
listByStatus
countByQuery
existsByCode
insert
update
deleteById
```

避免：

```text
handle
process
doQuery
executeBusiness
```

Mapper 方法表达“访问什么数据、按什么条件访问”，不表达完整业务流程。

---

## 6. Mapper 参数

单个或少量参数可以直接传递。

多个简单参数需要在 XML 中明确引用时，可以使用：

```java
@Param
```

例如：

```java
int updateStatus(
        @Param("id") String id,
        @Param("status") String status);
```

查询条件较多时优先使用 Query，而不是不断扩展参数列表或使用：

```java
Map<String, Object>
```

Query 的职责和 Package 读取：

- [layering.md](../architecture/layering.md#92-query)

普通 Java 方法参数数量和 Query 使用阈值读取：

- [java.md](java.md#51-控制参数数量)

本文不重复维护第二套阈值。

### 6.1 集合查询的 Null 契约

普通 MyBatis 集合查询使用：

```java
List<PlaceDO> listByQuery(PlaceQuery query);
```

表达“零到多条结果”。对于标准 MyBatis 集合查询，无匹配记录时按空集合处理，不使用 `null` 表示“没有记录”。

因此调用方不应机械增加：

```java
List<PlaceDO> places = placeMapper.listByQuery(query);
List<PlaceDO> safePlaces = places == null
        ? new ArrayList<>()
        : places;
```

也不需要：

```java
Optional.ofNullable(places)
        .orElseGet(Collections::emptyList);
```

已经具有非 Null 集合契约时，应直接按契约使用：

```java
List<PlaceDO> places = placeMapper.listByQuery(query);
return places.stream()
        .map(...)
        .toList();
```

如果目标项目存在自定义 Mapper 实现、插件、代理或其他数据访问封装，明确改变了标准集合返回契约，应以实际项目契约为准；不要仅根据方法名猜测。

单对象查询不适用本规则。例如：

```java
PlaceDO getById(String id);
```

不存在时究竟返回 `null`、`Optional` 还是抛出异常，应遵循项目已有契约。

如果 `null` 来自第三方 SDK、外部 Client 或其他确实允许 Null 的来源，应在最靠近来源的 Client / Adapter 等边界统一归一化，再向上提供稳定集合契约；不要让 Service / Manager 层层重复兜底。

原则：

> 数据来源的 Null 差异在边界归一化一次，上层依赖稳定契约；标准集合查询无结果使用空集合，不使用 Null。

---

## 7. `#{}` 与 `${}`

普通数据参数使用：

```xml
#{placeId}
```

例如：

```xml
WHERE id = #{placeId}
```

`${}` 表示文本替换，不是普通数据参数绑定。

只有 SQL 结构无法通过 PreparedStatement 参数化时才可以评估 `${}`，例如：

* 表名；
* 字段名；
* 排序字段；
* 固定 SQL 结构片段。

这类值必须先经过严格白名单映射：

```text
用户输入
   ↓
白名单 / Enum 映射
   ↓
固定 SQL 标识符
   ↓
${}
```

禁止：

```text
用户输入 → ${} → SQL
```

SQL 注入和排序等具体安全规则读取 `sql.md`。

---

## 8. 数据库与 Java 映射边界

数据库物理字段和 Java 属性可以使用不同命名体系。

当前规范中：

```text
Database
→ 项目统一数据库命名

Java
→ 英文业务语义
```

例如项目数据库已有：

```text
zjhm
rqsj
lqsj
csbh
```

Java 可以表达为：

```text
identityNumber
entryTime
exitTime
placeCode
```

MyBatis 通过 ResultMap、列别名、TypeHandler 等建立映射边界。

不要为了减少映射代码让数据库物理命名直接扩散到 Java 业务模型。

数据库命名详细规则读取：

- [database-design.md](../database/database-design.md)

DO 的模型职责读取：

- [layering.md](../architecture/layering.md#95-do)

---

## 9. ResultMap

数据库列与 Java 属性名称或类型不一致时，优先使用清晰的显式映射。

例如：

```xml
<resultMap id="PersonResultMap" type="PersonDO">
    <id property="id" column="id"/>
    <result property="identityNumber" column="zjhm"/>
    <result property="name" column="xm"/>
    <result property="entryTime" column="rqsj"/>
    <result property="exitTime" column="lqsj"/>
</resultMap>
```

对于特殊类型可以显式指定 TypeHandler：

```xml
<result
    property="tags"
    column="bq"
    typeHandler="com.example.common.mybatis.handler.JsonStringListTypeHandler"/>
```

映射应让读者能够判断：

```text
数据库列
→ Java 属性
→ 特殊类型转换
```

不要为了使用自动映射反向修改 Java 业务命名。

---

## 10. TypeHandler

TypeHandler 负责数据库类型与 Java 类型之间的**技术转换**。

常见：

```text
JSON / JSONB ↔ Java Collection / Object
数据库编码 ↔ Java Enum
数据库特殊类型 ↔ Java 类型
```

TypeHandler 不负责：

* 查询业务数据；
* 调用 Service / Manager；
* 权限判断；
* 业务状态判断；
* 完整业务异常处理。

原则：

> TypeHandler 做类型转换，不做业务流程。

新增通用 TypeHandler 前必须先搜索项目是否已有等价实现。

---

## 11. 动态 SQL

MyBatis 动态 SQL 可以合理使用：

```text
<if>
<choose>
<when>
<otherwise>
<foreach>
<where>
<set>
<trim>
```

例如：

```xml
<where>
    <if test="placeName != null and placeName != ''">
        AND csmc LIKE CONCAT('%', #{placeName}, '%')
    </if>
    <if test="status != null">
        AND zt = #{status}
    </if>
</where>
```

动态 SQL 负责 SQL 结构选择，不应承载完整业务流程或复杂业务状态机。

SQL 条件本身是否正确、安全、高效由 `sql.md` 判断。

---

## 12. SQL 片段复用

可以使用：

```xml
<sql>
<include>
```

复用稳定且明确的 SQL 片段。

不要为了减少几行代码创建层层嵌套、难以追踪的 `<sql>` 片段。

> SQL 可读性优先于形式上的复用。

---

## 13. Mapper XML

Mapper XML 应保持：

* namespace 清晰；
* SQL 与 Mapper 方法容易对应；
* 参数名称清楚；
* ResultMap 明确；
* 动态 SQL 可读；
* TypeHandler 使用明确。

XML 不应包含：

* 完整业务流程；
* 权限业务编排；
* 与数据库访问无关的大量判断。

具体 SQL 格式、SELECT、JOIN、写入范围、分页和性能统一读取：

- [sql.md](../database/sql.md)

---

## 14. Mapper 与事务

Mapper 执行数据库操作，但不负责定义完整业务事务边界。

不要因为 Mapper 中存在 INSERT / UPDATE / DELETE 就机械在 Mapper 层增加业务事务。

事务统一读取：

- [transactions.md](../architecture/transactions.md)

本文只要求 Mapper 不自行编排跨业务操作的事务语义。

---

## 15. SQL 规则不在 MyBatis 规范重复维护

以下内容统一由 `sql.md` 维护：

```text
SELECT *
COUNT / NULL
JOIN / LEFT JOIN
WHERE / 时间范围
INSERT / UPDATE / DELETE
分页 / 排序
EXISTS / IN
N+1
Batch
PostgreSQL 语法
索引与 EXPLAIN
SQL 性能
```

MyBatis 任务如果修改了 SQL，应同时加载 `sql.md`；如果只调整 ResultMap、TypeHandler 或参数映射，不需要为了形式加载全部 SQL 规范。

---

## 16. Codex MyBatis 修改流程

修改 MyBatis 代码时：

1. 判断是 Mapper 接口、Mapper XML、ResultMap、TypeHandler 还是其他 MyBatis 基础设施。
2. 搜索当前项目已有类似实现。
3. 新增组件前按职责确定 Package。
4. 检查参数是否应使用 `#{}`，`${}` 是否确实属于结构并经过白名单。
5. 检查集合 Mapper 的 Null 契约；标准集合查询无结果不在上层机械增加 Null 兜底。
6. 检查数据库列与 Java 属性的映射是否明确。
7. 检查 TypeHandler 是否只承担技术转换。
8. 动态 SQL 是否清晰且没有隐藏业务流程。
9. 修改 SQL 时同时读取 `sql.md`。
10. 涉及事务时读取 `transactions.md`。
11. 执行目标项目已有相关测试。

检查重点：

* Mapper 是否只负责数据访问；
* MyBatis 技术组件是否错误放入业务 Mapper Package；
* 是否重复创建已有 TypeHandler / Interceptor / Plugin；
* Query / DO 的职责和 Package 是否符合 `layering.md`；
* 标准 `List<T>` 查询的调用方是否无依据增加 `list == null ? emptyList : list` 等防御；
* 如果数据源确实允许 Null，是否在最靠近来源的边界归一化，而不是 Service / Manager 层层兜底；
* 是否把数据库物理命名无必要泄漏到 Java；
* ResultMap 是否清楚表达列与属性映射；
* `${}` 是否直接接收用户输入；
* TypeHandler 是否混入业务逻辑；
* Mapper XML 是否隐藏复杂业务流程；
* 是否在本文范围内重复发明 SQL 或事务规则。

最终原则：

> MyBatis 规范负责“Java 与 SQL 如何连接和映射”；标准集合查询无结果使用空集合；SQL 规范负责“SQL 本身是否正确、安全、清晰和高效”。
