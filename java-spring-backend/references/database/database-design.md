# 数据库设计规范

本文档定义数据库表、字段、约束、索引及数据库模型设计规范。

参考《阿里巴巴 Java 开发手册》数据库相关规约，并结合本项目 PostgreSQL 使用方式制定。

核心原则：

> 数据库结构应明确表达数据约束，而不是只依赖 Java 代码保证正确性。

> 数据库统一使用拼音命名，Java 统一使用英文业务语义，两者通过 Mapper / ResultMap 建立防腐层。

---

# 1. 数据库命名

数据库对象统一采用：

```text
小写汉语拼音 + 下划线
```

适用于：

* 表名；
* 字段名；
* 索引名；
* 约束名；
* 序列名；
* 视图名。

例如：

```text
ryxx
ajxx
csxx

zjhm
rqsj
lqsj
cjsj
gxsj
```

禁止同一个 Schema 中混合：

```text
ryxx
person_info
人员信息
```

---

# 2. 统一业务词汇

新增表或字段之前，必须先搜索已有：

* Schema；
* Migration；
* SQL；
* Mapper XML；
* 数据字典；

确认项目是否已经存在相同业务概念的拼音表达。

例如项目已经使用：

```text
zjhm    证件号码
rqsj    入区时间
lqsj    离区时间
csbh    场所编号
fzxbh   分中心编号
```

后续必须继续沿用。

禁止针对同一个概念重新创造：

```text
zjhm
zhengjianhao
zjh
```

等不同名称。

原则：

> 一个业务概念在数据库中只保留一套标准表达。

---

# 3. 数据库与 Java 防腐层

数据库和 Java 属于不同命名边界。

数据库：

```text
ryxx
zjhm
rqsj
lqsj
```

Java：

```text
PersonDO
identityNumber
entryTime
exitTime
```

推荐：

```text
Database
   拼音
     ↓
Mapper / ResultMap
     ↓
Java Persistence Model
   英文
```

例如：

```xml
<resultMap id="PersonResultMap" type="PersonDO">
    <id property="id" column="id"/>
    <result property="identityNumber" column="zjhm"/>
    <result property="entryTime" column="rqsj"/>
    <result property="exitTime" column="lqsj"/>
</resultMap>
```

禁止为了减少映射代码：

```java
private String zjhm;
private LocalDateTime rqsj;
```

数据库拼音不得向 Java 业务模型扩散。

---

# 4. 建表方式

数据库结构必须通过正式 SQL / Migration 脚本管理。

禁止：

* 只在数据库客户端手工建表；
* 修改数据库后不保留 Migration；
* 依赖 ORM 自动创建生产表；
* 使用 `ddl-auto=create` 维护生产 Schema。

如果项目已经使用：

```text
Flyway
Liquibase
项目自定义 Migration
```

必须遵循已有机制。

数据库 Schema 必须能够通过版本库中的脚本重新构建。

---

# 5. 建表必须明确约束

创建表时应明确考虑：

* 主键；
* 数据类型；
* Null；
* 默认值；
* 唯一约束；
* 索引；
* 表注释；
* 字段注释。

例如：

CREATE TABLE ryxx (
id      varchar(64)  NOT NULL,
xm      varchar(100) NOT NULL,
zjhm    varchar(32),
rqsj    timestamp,
lqsj    timestamp,
cjsj    timestamp    NOT NULL,
gxsj    timestamp    NOT NULL,

```
CONSTRAINT pk_ryxx PRIMARY KEY (id)
```

);

COMMENT ON TABLE ryxx IS '人员信息表';

COMMENT ON COLUMN ryxx.id IS '主键ID';
COMMENT ON COLUMN ryxx.xm IS '姓名';
COMMENT ON COLUMN ryxx.zjhm IS '证件号码';
COMMENT ON COLUMN ryxx.rqsj IS '入区时间';
COMMENT ON COLUMN ryxx.lqsj IS '离区时间';
COMMENT ON COLUMN ryxx.cjsj IS '创建时间';
COMMENT ON COLUMN ryxx.gxsj IS '更新时间';

不要只创建字段而忽略数据约束。

---

# 6. 主键

每张业务表原则上必须具有明确主键。

主键应：

* 稳定；
* 唯一；
* 不承载容易变化的业务语义。

不要使用可能发生变化的业务字段直接作为主键。

例如：

```text
身份证号码
手机号
场所名称
```

通常不适合作为系统主键。

具体主键生成方式遵循项目现有约定。

---

# 7. 字段类型

字段类型应表达真实数据语义。

不要为了方便统一使用：

```text
varchar
```

存储所有内容。

例如：

时间：

```text
timestamp
date
```

数字：

```text
integer
bigint
numeric
```

布尔：

```text
boolean
```

JSON 数据在确有动态结构需求时使用：

```text
jsonb
```

不要将本应结构化的数据随意序列化成字符串保存。

---

# 8. 字符串字段

`varchar` 长度应结合真实业务约束设计。

不要无依据统一：

```sql
varchar(255)
```

例如：

```text
编号
名称
身份证件号码
备注
URL
```

具有不同长度特征，应分别设计。

超长文本根据实际情况考虑：

```text
text
```

---

# 9. NULL

字段是否允许 NULL 必须有明确语义。

不要无意识让所有字段：

```text
NULL
```

也不要为了方便全部：

```text
NOT NULL DEFAULT ''
```

需要明确区分：

```text
NULL
```

和：

```text
''
```

的业务含义。

例如：

```text
离区时间 = NULL
```

可以明确表示：

> 当前尚未离区。

这种业务语义不要使用：

```text
1970-01-01
''
0
```

等特殊值代替。

---

# 10. 默认值

默认值必须具有真实业务语义。

不要为了避免 NULL 随意设置：

```text
0
''
1970-01-01
```

如果默认值代表业务状态，应明确记录其含义。

不要让：

```text
Java 默认值
数据库默认值
```

表达不同语义。

---

# 11. 状态字段

状态字段应使用稳定编码。

例如：

```text
zt
shzt
```

不要直接将页面展示文字作为数据库状态值。

推荐：

```text
数据库稳定编码
        ↓
Java Enum
        ↓
展示名称
```

Java 中应使用枚举等明确类型表达状态，不要在业务代码中散落：

```java
"01"
"02"
"03"
```

状态含义必须统一维护。

---

# 12. 时间字段

时间字段必须明确表达语义。

常见：

```text
lssj    录入时间
gxsj    更新时间
rqsj    入区时间
lqsj    离区时间
```

不要使用含义模糊的：

```text
sj
time
date
```

同一个业务概念统一使用同一个字段名称。

涉及跨时区业务时，应明确数据库和应用的时区策略。

---

# 13. 金额和精确数字

金额、比例以及需要精确计算的数据，应使用精确数值类型。

例如：

```sql
numeric(...)
```

不要使用：

```text
real
double precision
```

表示需要精确计算的金额。

Java 对应优先：

```java
BigDecimal
```

---

# 14. 唯一约束

业务上必须唯一的数据，应优先通过数据库唯一约束保证。

例如：

```text
某业务编号在某范围内唯一
```

不要只依赖：

```text
先 SELECT
↓
Java 判断不存在
↓
INSERT
```

这种逻辑存在并发竞争。

数据库约束是最终数据一致性防线。

---

# 15. 外键

是否使用数据库 Foreign Key 根据项目已有架构统一决定。

如果项目统一不使用物理外键：

* 仍必须明确关联关系；
* 应由应用逻辑和必要的数据检查维护完整性。

不要在单个模块中随意改变项目既有外键策略。

---

# 16. 索引

索引用于支持真实查询场景。

创建索引前应分析：

* WHERE；
* JOIN；
* ORDER BY；
* 唯一性要求；
* 查询频率；
* 数据量。

不要看到一个字段就创建一个索引。

索引会增加：

* INSERT 成本；
* UPDATE 成本；
* DELETE 成本；
* 存储成本。

---

# 17. 联合索引

联合索引应根据实际查询条件设计。

例如常见查询：

```sql
WHERE fzxbh = ?
  AND rqsj >= ?
  AND rqsj < ?
```

可评估：

```text
(fzxbh, rqsj)
```

而不是机械分别创建：

```text
fzxbh
rqsj
```

两个单列索引。

索引顺序必须结合真实 SQL 和 PostgreSQL 执行计划决定。

---

# 18. 不重复创建索引

新增索引前必须检查：

* 现有索引；
* UNIQUE 约束产生的索引；
* PRIMARY KEY 索引；
* 联合索引是否已经覆盖需求。

禁止重复创建作用基本相同的索引。

---

# 19. 表和字段注释

新增业务表和重要业务字段应提供清晰注释。

注释用于说明：

* 业务含义；
* 状态含义；
* 特殊约束。

例如：

```sql
COMMENT ON TABLE ryxx IS '人员信息';

COMMENT ON COLUMN ryxx.zjhm IS '证件号码';
COMMENT ON COLUMN ryxx.rqsj IS '入区时间';
COMMENT ON COLUMN ryxx.lqsj IS '离区时间';
```

不要把复杂业务规则全部塞进字段注释。

---

# 20. 删除设计

设计删除功能时必须明确：

```text
物理删除
还是
逻辑删除
```

如果使用逻辑删除，应保持项目统一字段和语义。

不要由 Codex 自行决定将：

```text
DELETE
```

改成逻辑删除，或反过来。

删除语义属于业务契约。

---

# 21. 冗余字段

不要为了避免 JOIN 就随意增加大量冗余字段。

允许冗余时，应明确：

* 为什么需要；
* 数据来源；
* 谁负责更新；
* 一致性如何保证。

如果无法明确维护策略，不应增加冗余字段。

---

# 22. 大字段

大文本、JSON、二进制等字段应谨慎放在高频主表。

设计时考虑：

* 查询是否经常需要；
* 是否影响普通列表查询；
* 是否应该拆表；
* 是否应该使用对象存储。

不要把文件本身无脑保存到普通业务表。

---

# 23. Schema 变更

修改已有 Schema 时必须考虑：

* 历史数据；
* 线上兼容性；
* Java 旧版本兼容；
* 默认值；
* NULL；
* 索引创建成本；
* 回滚方案。

禁止未经明确要求：

* 删除已有字段；
* 修改已有字段业务含义；
* 改变主键；
* 放宽或收紧关键约束。

---

# 24. Codex 数据库设计检查

新增或修改数据库结构前检查：

1. 是否已经存在相同业务概念；
2. 拼音命名是否沿用已有词汇；
3. 是否出现英文/拼音混用；
4. Java 层是否保持英文命名；
5. Null 是否具有明确语义；
6. 数据类型是否合理；
7. 是否需要唯一约束；
8. 是否真正需要索引；
9. 是否存在重复索引；
10. 是否提供必要注释；
11. 是否提供正式 Migration；
12. 是否考虑已有数据和兼容性。

原则：

> 数据库负责保证数据结构和数据底线，Java 负责表达业务语义，两者通过明确映射隔离。

