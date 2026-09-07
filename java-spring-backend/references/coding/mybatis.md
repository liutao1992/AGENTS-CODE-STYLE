# MyBatis 编码规范

本文档定义项目 MyBatis 使用规范。

数据库设计、命名、字段、索引及约束规则读取：

- [database-design.md](../database/database-design.md)

PostgreSQL 与 SQL 编写规则读取：

- [sql.md](../database/sql.md)

事务规则读取：

- [transactions.md](../architecture/transactions.md)

核心原则：

> Mapper 负责数据访问，不负责业务流程。

> 数据库使用拼音命名，Java 使用英文业务语义，通过 MyBatis 建立明确映射边界。

> MyBatis 通用技术基础设施统一放入 `common.mybatis`，不得因为被某个业务模块使用就放入该模块的 Mapper 包。

---

## 1. Mapper 职责

Mapper 只负责数据库访问。

主要包括：

* SELECT；
* INSERT；
* UPDATE；
* DELETE；
* ResultMap；
* 参数映射；
* 动态 SQL。

Mapper 不负责：

* HTTP 请求处理；
* 权限业务判断；
* 状态流转；
* 业务流程编排；
* 事务边界设计；
* 第三方服务调用；
* 与数据库访问无关的业务逻辑。

例如：

```java
public interface PlaceMapper {

    PlaceDO getById(String id);

    List<PlaceDO> listByQuery(PlaceQuery query);

    int insert(PlaceDO place);

    int update(PlaceDO place);

    int deleteById(String id);
}
```

不要在 Mapper 中实现：

```text
审核场所
注册案件
权限判断
业务状态判断
跨模块业务流程
```

这些逻辑应由 Service / Manager 负责。

---

## 2. Mapper 包职责

业务模块中的：

```text
<module>.mapper
```

只用于该模块的数据访问 Mapper。

例如：

```text
place.mapper.PlaceMapper
case.mapper.CaseMapper
equipment.mapper.EquipmentMapper
```

不得因为某个技术类被 Mapper 使用，就将其放入业务模块的 `mapper` 包。

例如：

```text
place.mapper.JsonStringListTypeHandler
```

不符合职责归属。

`JsonStringListTypeHandler` 表达的是：

```text
MyBatis TypeHandler
```

而不是：

```text
Place 数据访问 Mapper
```

因此应放入：

```text
common.mybatis.handler.JsonStringListTypeHandler
```

原则：

> Package 根据类本身的职责确定，而不是根据哪个 Mapper 当前使用它确定。

---

## 3. MyBatis 基础设施归属

MyBatis 通用技术基础设施统一归入：

```text
common.mybatis
```

典型目录：

```text
common.mybatis.handler
common.mybatis.interceptor
common.mybatis.plugin
common.mybatis.config
```

对应关系：

```text
TypeHandler
→ common.mybatis.handler

Interceptor
→ common.mybatis.interceptor

Plugin
→ common.mybatis.plugin

公共 MyBatis 配置
→ common.mybatis.config
```

以下类型不得放入：

```text
place.mapper
case.mapper
equipment.mapper
```

等业务 Mapper 包：

* TypeHandler；
* Interceptor；
* Plugin；
* 通用 ResultHandler；
* MyBatis 公共配置；
* 与具体业务无关的数据访问基础设施。

即使当前只有一个业务模块使用，也按类的实际技术职责确定归属。

---

## 4. 新增 MyBatis 组件前必须搜索

创建以下类型之前：

```text
Mapper
TypeHandler
Interceptor
Plugin
ResultHandler
MyBatis Configuration
```

必须先搜索仓库，确认：

1. 是否已经存在可复用实现；
2. 是否存在相同职责的组件；
3. 同类组件当前位于哪个 Package；
4. 是否已经存在对应的 `common.mybatis` 基础设施目录；
5. 是否真的需要新增。

不得因为当前正在修改：

```text
place
```

模块，就默认创建：

```text
place.mapper.*
```

创建新文件应遵循：

```text
判断职责
   ↓
搜索同类实现
   ↓
确定 Package
   ↓
创建文件
```

---

## 5. Mapper 方法命名

Mapper 方法名称应表达明确的数据访问目的。

推荐：

```java
getById(...)
listByQuery(...)
countByQuery(...)
insert(...)
update(...)
deleteById(...)
```

可以根据具体查询目的使用：

```java
getByCode(...)
listByStatus(...)
countByCenterCode(...)
existsByCode(...)
```

避免：

```java
handle(...)
process(...)
doQuery(...)
executeBusiness(...)
```

Mapper 方法名称表达：

> 查什么数据、按什么条件查。

不要表达完整业务流程。

---

## 6. 查询参数

参数较少时可以直接传递。

例如：

```java
PlaceDO getById(String id);
```

多个简单参数需要明确名称时，可以使用：

```java
@Param
```

例如：

```java
int updateStatus(
        @Param("id") String id,
        @Param("status") String status);
```

查询条件较多时，应封装为 Query 对象。

例如：

```java
List<PlaceDO> listByQuery(PlaceQuery query);
```

当普通业务查询条件超过 3 个时，原则上应优先使用 Query 对象，避免方法参数持续膨胀。

避免：

```java
List<PlaceDO> list(
        String name,
        String status,
        String centerCode,
        String type,
        LocalDateTime startTime,
        LocalDateTime endTime);
```

也避免使用：

```java
Map<String, Object>
```

传递普通业务查询条件。

Query 对象字段必须使用英文业务语义。

---

## 7. 参数绑定

MyBatis 参数默认使用：

```xml
#{placeId}
```

例如：

```xml
WHERE id = #{placeId}
```

禁止将用户可控数据直接通过：

```xml
${placeId}
```

拼接进入 SQL。

`${}` 只允许用于无法使用 PreparedStatement 参数化的 SQL 结构，例如：

* 表名；
* 字段名；
* 排序字段；
* 特殊 SQL 片段。

并且必须满足：

```text
用户输入
   ↓
白名单映射
   ↓
SQL 结构
```

禁止：

```text
用户输入
   ↓
${}
   ↓
SQL
```

排序字段等场景优先由 Java 枚举或白名单转换为固定 SQL 字段。

---

## 8. 数据库与 Java 映射

本项目数据库与 Java 使用不同命名体系。

数据库：

```text
拼音
```

Java：

```text
英文业务语义
```

例如：

```text
数据库              Java

zjhm       →        identityNumber
xm         →        name
rqsj       →        entryTime
lqsj       →        exitTime
csbh       →        placeCode
fzxbh      →        centerCode
```

MyBatis 是数据库与 Java 之间的重要防腐边界：

```text
Database（拼音）
        ↓
Mapper / ResultMap
        ↓
Java Model（英文）
```

禁止为了减少映射代码，让数据库拼音进入 Java 模型。

---

## 9. ResultMap

数据库字段与 Java 属性名称不一致时，应优先使用显式 `resultMap`。

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

推荐显式定义：

* 主键；
* 普通字段；
* 特殊类型；
* TypeHandler。

例如：

```xml
<result
    property="tags"
    column="bq"
    typeHandler="com.example.common.mybatis.handler.JsonStringListTypeHandler"/>
```

不要因为可以使用自动映射，就让数据库命名影响 Java 属性设计。

---

## 10. DO

DO 表达数据库持久化数据。

数据库：

```text
ryxx
zjhm
rqsj
```

Java：

```text
PersonDO
identityNumber
entryTime
```

DO 不要求机械复制数据库表名或字段名。

例如：

```text
数据库表：ryxx

Java：
PersonDO
```

而不是为了与表名一致创建：

```text
RyxxDO
```

原则：

> DO 表达持久化数据职责，但仍属于 Java 命名体系。

---

## 11. TypeHandler

TypeHandler 用于数据库类型与 Java 类型之间的技术转换。

例如：

```text
JSON / JSONB
    ↓
List<String>

数据库编码
    ↓
Java Enum

特殊数据库类型
    ↓
Java 类型
```

TypeHandler 不负责业务判断。

禁止在 TypeHandler 中：

* 查询业务数据；
* 调用 Service；
* 调用业务 Manager；
* 执行业务状态判断；
* 实现权限逻辑。

TypeHandler 应保持：

```text
输入数据库值
      ↓
类型转换
      ↓
Java 值
```

以及反方向转换。

通用 TypeHandler 统一放入：

```text
common.mybatis.handler
```

例如：

```text
common.mybatis.handler.JsonStringListTypeHandler
```

---

## 12. SELECT

原则上不要使用：

```sql
SELECT *
```

应明确查询字段。

例如：

```sql
SELECT
    id,
    xm,
    zjhm,
    rqsj,
    lqsj
FROM ryxx
WHERE id = #{id}
```

原因：

* 减少无关字段读取；
* 防止新增字段影响现有映射；
* ResultMap 更明确；
* 数据库与 Java 边界更稳定；
* 更容易检查实际返回内容。

详细 SQL 规则读取：

- [sql.md](../database/sql.md)

---

## 13. 动态 SQL

复杂查询条件可以使用 MyBatis：

```xml
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

动态 SQL 应保持：

* 结构清晰；
* 条件明确；
* 可直接阅读；
* 不隐藏业务流程。

不要把大量业务判断塞入 Mapper XML。

---

## 14. SQL 片段复用

可以使用：

```xml
<sql>
<include>
```

复用稳定、明确的 SQL 片段。

例如：

```xml
<sql id="BaseColumns">
    id,
    xm,
    zjhm,
    rqsj,
    lqsj
</sql>
```

但是不要为了减少几行代码创建层层嵌套、难以追踪的 SQL 片段。

原则：

> SQL 可读性优先于形式上的复用。

---

## 15. N+1 查询

禁止无意识地在循环中不断执行 Mapper 查询：

```java
for (PlaceDO place : places) {
    equipmentMapper.listByPlaceId(place.getId());
}
```

发现 N+1 时，应先评估是否可以：

* 批量查询；
* JOIN；
* `IN`；
* 一次性查询后在内存分组；
* 改变查询模型。

例如：

```text
100 条 Place
    ↓
100 次 Equipment 查询
```

应优先评估：

```text
100 个 placeId
    ↓
一次批量查询
    ↓
Java 分组
```

但也不要为了避免 N+1 构造无限增长的巨大 `IN`。

---

## 16. INSERT

INSERT 应明确字段。

推荐：

```sql
INSERT INTO ryxx (
    id,
    xm,
    zjhm,
    rqsj
)
VALUES (
    #{id},
    #{name},
    #{identityNumber},
    #{entryTime}
)
```

不要依赖数据库字段顺序。

需要数据库默认值时，应明确确认该字段是否应该由数据库生成。

---

## 17. UPDATE

UPDATE 必须具有明确的 `WHERE` 条件。

例如：

```sql
UPDATE csxx
SET
    zt = #{targetStatus},
    gxsj = #{updateTime}
WHERE id = #{id}
```

对于状态流转，优先考虑条件更新：

```sql
UPDATE csxx
SET zt = #{targetStatus}
WHERE id = #{id}
  AND zt = #{expectedStatus}
```

并根据业务需要检查受影响行数：

```java
int affectedRows;
```

如果：

```text
affectedRows == 0
```

应由上层判断是：

* 数据不存在；
* 状态已变化；
* 并发冲突；

而不是由 Mapper 编排完整业务逻辑。

---

## 18. DELETE

执行 DELETE 前必须明确：

* 删除范围；
* 是否物理删除；
* 是否逻辑删除；
* 是否存在关联数据；
* 是否符合项目已有删除语义。

不得因为当前实现方便，自行将：

```text
物理删除
```

改为：

```text
逻辑删除
```

或者反过来。

DELETE 必须具有明确条件。

---

## 19. 批量操作

大量数据操作应优先考虑合理批处理。

避免：

```java
for (...) {
    mapper.insert(...);
}
```

无边界逐条访问数据库。

可以根据实际情况使用：

* MyBatis Batch；
* PostgreSQL 多 Values；
* 分批处理；
* 其他项目已有批处理方式。

同时避免一次生成过大的：

```text
IN (...)
VALUES (...)
```

批量大小应结合：

* 数据量；
* SQL 长度；
* 内存；
* 事务范围；
* PostgreSQL 承载能力。

不得凭经验随意设置极大批次。

---

## 20. 分页

分页查询必须具有稳定排序。

例如：

```sql
ORDER BY cjsj DESC, id DESC
LIMIT #{pageSize}
OFFSET #{offset}
```

禁止分页但没有：

```sql
ORDER BY
```

对于大数据量深分页，应评估 Keyset Pagination。

详细分页和 PostgreSQL SQL 规则读取：

- [sql.md](../database/sql.md)

---

## 21. Mapper XML

Mapper XML 应保持：

* SQL 清晰；
* 缩进一致；
* 字段明确；
* 条件明确；
* ResultMap 明确；
* 动态 SQL 易读。

XML 中不要包含：

* 复杂业务流程；
* 权限业务编排；
* 与 SQL 无关的大量判断。

Mapper XML 应让开发者能够较快判断：

```text
查什么
从哪里查
按什么条件查
返回什么
```

---

## 22. Mapper 与事务

Mapper 负责执行数据库操作，不负责定义完整业务事务边界。

不要因为 Mapper 方法执行：

```text
INSERT
UPDATE
DELETE
```

就机械在 Mapper 层增加业务事务。

事务应根据数据一致性范围由 Manager / Service 等合适边界管理。

普通快照查询也不因为经过 Mapper 就自动需要显式事务。

涉及事务时读取：

- [transactions.md](../architecture/transactions.md)

---

## 23. SQL 性能

不要仅凭代码结构判断：

```text
这个 SQL 一定更快
这个写法一定走索引
这个 JOIN 一定比子查询快
```

复杂或性能敏感 SQL 应结合：

```text
EXPLAIN
EXPLAIN ANALYZE
```

以及：

* 数据量；
* 索引；
* 查询选择性；
* 返回行数；

进行判断。

SQL 性能规范统一读取：

- [sql.md](../database/sql.md)

MyBatis 规范不重复维护 PostgreSQL 的具体优化细节。

---

## 24. Codex 修改流程

修改 MyBatis 相关代码时：

1. 判断当前任务涉及 Mapper、XML 还是 MyBatis 基础设施。
2. 按需读取 `mybatis.md` 和 `sql.md`。
3. 搜索当前模块已有 Mapper 和 XML。
4. 找到至少一个类似实现。
5. 新增技术组件时先判断其 Package 职责。
6. 检查数据库拼音与 Java 英文之间的映射。
7. 检查参数绑定方式。
8. 检查 SQL 范围和安全性。
9. 检查是否存在 N+1 或明显重复数据库访问。
10. 修改完成后检查完整 Mapper / XML 调用链。
11. 执行相关测试。

---

## 25. Codex 检查

修改 MyBatis 代码后检查：

### 职责

* Mapper 是否只负责数据访问；
* Mapper 是否出现业务流程；
* MyBatis 技术基础设施是否错误放入业务 Mapper 包；
* TypeHandler 是否位于 `common.mybatis.handler`；
* 新增文件是否根据职责确定 Package。

### 映射

* 数据库拼音是否泄漏到 Java；
* Java DO 是否保持英文业务语义；
* 是否应该使用显式 ResultMap；
* 特殊类型是否使用合适 TypeHandler。

### 参数与安全

* 是否直接使用 `${}` 接收用户输入；
* `${}` 是否经过严格白名单；
* 查询条件较多时是否应该封装 Query；
* 是否无意义使用 `Map<String, Object>`。

### SQL

* 是否存在 `SELECT *`；
* INSERT 是否明确字段；
* UPDATE / DELETE 是否具有明确条件；
* 状态更新是否需要条件 UPDATE；
* 分页是否具有稳定排序；
* 是否存在无意识 N+1；
* 是否存在过大的 `IN` / Batch；
* SQL 是否符合 PostgreSQL 规范。

### 架构

* Mapper 是否承担事务编排；
* 是否因为当前模块使用某个技术类，就错误将其放入当前模块；
* 是否重复创建已有 MyBatis 公共能力。

最终原则：

> Mapper 管数据访问，ResultMap 管数据库与 Java 的映射，TypeHandler 管类型转换；业务逻辑留在业务层，通用 MyBatis 基础设施归入 `common.mybatis`。

