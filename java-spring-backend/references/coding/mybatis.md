# MyBatis / MyBatis-Plus 编码规范

本文档定义 MyBatis 与 MyBatis-Plus 框架使用规范。

本文负责回答：

> Mapper、Mapper XML、MyBatis-Plus BaseMapper、参数绑定、ResultMap、TypeHandler、动态 SQL 和 MyBatis 技术组件应该怎么组织和实现。

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

> 使用 MyBatis-Plus 时，复用 `BaseMapper` 提供的基础 CRUD；自定义条件查询和更新保持显式 Mapper 方法与可定位 SQL，不使用 Wrapper 隐藏查询语义。

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

普通 MyBatis 示例：

```java
public interface PlaceMapper {

    PlaceDO getById(String id);

    List<PlaceDO> listByQuery(PlaceQuery query);

    int insert(PlaceDO place);

    int update(PlaceDO place);
}
```

使用 MyBatis-Plus 时的基础形式读取本文第 7 节。

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

## 5. Mapper / DAO 方法命名

Mapper / DAO 方法名应直接表达数据访问意图。在目标项目没有更具体稳定约定时，自定义数据访问方法默认使用以下前缀：

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
listPlaces
listByQuery
listByStatus
countByQuery
countByStatus
insert
insertBatch
update
updateStatus
deleteById
deleteByStatus
```

获取单个对象的方法使用 `get` 前缀；不要使用含义不清的：

```text
queryOne
findData
loadInfo
```

集合查询使用 `list` 前缀。直接表达实体集合时可以使用复数名词：

```text
listPlaces
listCases
```

如果方法重点在查询条件，则 `listByStatus`、`listByQuery` 等形式同样允许，不为了满足“复数结尾”牺牲条件语义。

统计数量使用 `count` 前缀，例如：

```text
countByQuery
countByStatus
```

写操作按持久化语义使用：

```text
insert...
delete...
update...
```

Service 层的新增 / 删除业务动作默认使用 `save` / `remove`；Mapper / DAO 使用 `insert` / `delete`，避免把业务动作和数据库操作术语混在同一层。Service 方法命名读取 `layering.md`。

MyBatis-Plus `BaseMapper` 已定义的框架方法保持其原始命名，例如：

```text
selectById
selectList
selectCount
insert
updateById
deleteById
```

不得为了本规范重新包装一层只做改名的方法。本文命名规则主要约束项目自定义 Mapper / DAO 方法。

避免：

```text
handle
process
doQuery
executeBusiness
```

原则：

> 自定义 Mapper / DAO 使用稳定的数据访问动词；名称表达“访问什么数据、按什么条件访问”，不表达完整业务流程，也不为了命名形式包装框架已有能力。

---

## 6. Mapper 参数

单个或少量参数可以直接传递。

多个简单参数需要在 XML 中明确引用时，可以使用：

```java
@Param
```

例如：

```java
int updateStatus(@Param("id") String id, @Param("status") String status);
```

查询条件较多时优先使用 Query，而不是不断扩展参数列表或使用：

```java
Map<String, Object>
```

Query 的职责和 Package 读取：

- [layering.md](../architecture/layering.md#92-query)

普通 Java 方法参数数量和 Query 使用阈值读取：

- [java.md](java.md#51-控制参数数量)

方法声明本身的空行和换行风格统一读取 Java 格式规范；能在项目行宽内清晰表达时优先保持单行，不因 `@Param` 机械换行。

本文不重复维护第二套阈值或格式规则。

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

## 7. MyBatis-Plus 使用规范

本节只在目标项目实际使用 MyBatis-Plus 时适用。项目仍使用普通 MyBatis 时，不为了本规则引入 MyBatis-Plus。

### 7.1 Mapper / DAO 统一继承 `BaseMapper`

采用 MyBatis-Plus 的项目中，与持久化实体对应的业务 Mapper / DAO 统一继承：

```java
BaseMapper<T>
```

例如：

```java
public interface PlaceMapper extends BaseMapper<PlaceDO> {

    List<PlaceDO> listByQuery(PlaceQuery query);

    long countByQuery(PlaceQuery query);
}
```

基础 CRUD 优先复用 `BaseMapper` 已提供的明确方法，例如：

```text
insert
selectById
updateById
deleteById
```

不要在每个 Mapper 中重复声明完全相同的基础 CRUD 方法。

如果某个数据访问接口本身不是单表持久化实体 Mapper，例如纯聚合查询、跨表只读投影或项目已有特殊 DAO 抽象，不得为了满足形式伪造一个无意义的 `BaseMapper<...>` 泛型实体；以目标项目真实数据访问边界为准。

原则：

> 有明确持久化实体的 MyBatis-Plus Mapper 继承 `BaseMapper` 复用基础 CRUD，不重复造轮子，也不为了继承而伪造实体。

### 7.2 禁止使用 MyBatis-Plus Wrapper 条件构建器

业务代码禁止使用 MyBatis-Plus 的 Wrapper 条件构建器表达查询或更新条件，包括但不限于：

```text
QueryWrapper
LambdaQueryWrapper
UpdateWrapper
LambdaUpdateWrapper
Wrappers.query(...)
Wrappers.lambdaQuery(...)
Wrappers.update(...)
Wrappers.lambdaUpdate(...)
```

也不要把复杂条件隐藏在 Service / Manager 中通过 Wrapper 链式拼装。

主要原因：

1. SQL 逻辑分散在 Java 条件构建代码中，不利于稳定复用和集中维护；
2. 排查慢 SQL、线上 SQL 或数据库日志时，无法直接通过 SQL 关键片段快速搜索定位到对应 XML / Mapper 实现；
3. 复杂 Wrapper 容易把数据访问细节扩散到 Service / Manager，削弱 Mapper 与 SQL 边界；
4. 条件逐步链式拼装后，最终 SQL 语义通常不如显式 XML 易读、易审查。

自定义条件查询、更新、统计等优先定义明确 Mapper 方法并在 XML 中维护 SQL：

```java
public interface PlaceMapper extends BaseMapper<PlaceDO> {

    List<PlaceDO> listByQuery(PlaceQuery query);

    int updateStatus(
            @Param("id") String id,
            @Param("expectedStatus") String expectedStatus,
            @Param("targetStatus") String targetStatus);
}
```

```xml
<select id="listByQuery" resultMap="PlaceResultMap">
    SELECT
        id,
        csbh,
        zt
    FROM csxx
    <where>
        <if test="placeCode != null and placeCode != ''">
            AND csbh = #{placeCode}
        </if>
        <if test="status != null">
            AND zt = #{status}
        </if>
    </where>
</select>
```

`BaseMapper` 的直接主键 CRUD 能力不属于 Wrapper 条件构建器，可以正常使用。

原则：

> MyBatis-Plus 用于复用稳定基础 CRUD，不用 Wrapper 把业务查询重新搬回 Java；需要条件 SQL 时保持 Mapper 方法与 SQL 显式、可搜索、可复用。

### 7.3 Mapper XML 禁止写死业务常量

Mapper XML 中禁止直接硬编码业务状态、业务类型、来源编码、固定业务标识等常量。

避免：

```xml
SELECT
    id,
    csbh,
    zt
FROM csxx
WHERE zt = '1'
  AND lx = 'FORMAL'
```

应由 Java 调用边界通过 Mapper / DAO 参数显式传入：

```java
List<PlaceDO> listByStatusAndType(@Param("status") String status, @Param("type") String type);
```

```xml
SELECT
    id,
    csbh,
    zt
FROM csxx
WHERE zt = #{status}
  AND lx = #{type}
```

这样业务常量由 Java 业务语义统一维护，XML 只负责 SQL 表达，避免同一编码散落在多个 SQL 中且难以修改、搜索和复用。

这条规则针对的是**业务常量**，不是禁止 SQL 中出现任何字面量。以下属于 SQL 本身的正常结构或数据库表达时可以保留：

```text
SELECT 1
COUNT(*)
IS NULL / IS NOT NULL
固定 LIMIT（确有明确技术语义时）
CASE / COALESCE 等 SQL 结构中的必要字面量
```

但如果某个 `'1'`、`'0'`、`'PENDING'`、`'FORMAL'` 实际代表业务状态或业务类型，就不能因为写起来方便留在 XML 中。

原则：

> SQL 结构常量可以属于 SQL；业务常量属于 Java 业务契约，通过 Mapper 参数进入 XML，不在 XML 中复制业务编码。

---

## 8. `#{}` 与 `${}`

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

## 9. 数据库与 Java 映射边界

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

## 10. ResultMap

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

## 11. TypeHandler

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

## 12. 动态 SQL

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

## 13. SQL 片段复用

可以使用：

```xml
<sql>
<include>
```

复用稳定且明确的 SQL 片段。

不要为了减少几行代码创建层层嵌套、难以追踪的 `<sql>` 片段。

> SQL 可读性优先于形式上的复用。

---

## 14. Mapper XML

Mapper XML 应保持：

* namespace 清晰；
* SQL 与 Mapper 方法容易对应；
* 参数名称清楚；
* ResultMap 明确；
* 动态 SQL 可读；
* TypeHandler 使用明确；
* 业务常量由 Mapper 参数传入，不在 XML 中散落硬编码。

XML 不应包含：

* 完整业务流程；
* 权限业务编排；
* 与数据库访问无关的大量判断；
* 无依据硬编码的业务状态、类型和业务编码。

具体 SQL 格式、SELECT、JOIN、写入范围、分页和性能统一读取：

- [sql.md](../database/sql.md)

---

## 15. Mapper 与事务

Mapper 执行数据库操作，但不负责定义完整业务事务边界。

不要因为 Mapper 中存在 INSERT / UPDATE / DELETE 就机械在 Mapper 层增加业务事务。

事务统一读取：

- [transactions.md](../architecture/transactions.md)

本文只要求 Mapper 不自行编排跨业务操作的事务语义。

---

## 16. SQL 规则不在 MyBatis 规范重复维护

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

MyBatis 任务如果修改了 SQL，应同时加载 `sql.md`；如果只调整 BaseMapper、ResultMap、TypeHandler 或参数映射，不需要为了形式加载全部 SQL 规范。

---

## 17. Codex MyBatis 修改流程

修改 MyBatis / MyBatis-Plus 代码时：

1. 判断是 Mapper 接口、Mapper XML、BaseMapper、ResultMap、TypeHandler 还是其他 MyBatis 基础设施。
2. 搜索当前项目已有类似实现。
3. 新增组件前按职责确定 Package。
4. 检查自定义 Mapper / DAO 方法是否使用清晰的 `get / list / count / insert / delete / update` 数据访问语义；不要为了改名包装 MyBatis-Plus 已有方法。
5. 项目使用 MyBatis-Plus 时，检查实体 Mapper / DAO 是否继承 `BaseMapper<DO>`，是否重复声明基础 CRUD。
6. 检查是否使用 `QueryWrapper`、`LambdaQueryWrapper`、`UpdateWrapper` 等 Wrapper；业务条件 SQL 应改为明确 Mapper 方法 + XML。
7. 检查 XML 是否硬编码业务状态、类型、来源等业务常量；应由 Mapper 参数传入。
8. 检查参数是否应使用 `#{}`，`${}` 是否确实属于结构并经过白名单。
9. 检查集合 Mapper 的 Null 契约；标准集合查询无结果不在上层机械增加 Null 兜底。
10. 检查数据库列与 Java 属性的映射是否明确。
11. 检查 TypeHandler 是否只承担技术转换。
12. 动态 SQL 是否清晰且没有隐藏业务流程。
13. 修改 SQL 时同时读取 `sql.md`。
14. 涉及事务时读取 `transactions.md`。
15. 执行目标项目已有相关测试。

检查重点：

* Mapper 是否只负责数据访问；
* 自定义 Mapper / DAO 方法命名是否准确表达单对象、集合、统计、新增、删除和修改语义；
* MyBatis-Plus 实体 Mapper 是否正确复用 `BaseMapper`；
* 是否为了统一命名给 `BaseMapper` 已有方法增加无意义转发包装；
* 是否使用 Wrapper 把条件 SQL 隐藏在 Java 业务代码中；
* XML 是否写死本应由 Java 业务契约维护的常量；
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

> MyBatis / MyBatis-Plus 规范负责“Java 与 SQL 如何连接和映射”；MyBatis-Plus 只复用稳定基础 CRUD，自定义条件保持显式 SQL；标准集合查询无结果使用空集合；SQL 规范负责“SQL 本身是否正确、安全、清晰和高效”。
