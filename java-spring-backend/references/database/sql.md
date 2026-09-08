# SQL 与 PostgreSQL 规范

本文档定义 SQL 正确性、安全范围、可读性和 PostgreSQL 查询规范。

本文负责回答：

> SQL 本身是否正确、安全、清晰，并在有证据时是否需要优化。

MyBatis 的 `#{}` / `${}`、Mapper XML、ResultMap、TypeHandler 等框架用法读取：

- [MyBatis](../coding/mybatis.md)

数据库表、字段、约束、索引和 Migration 设计读取：

- [数据库设计](database-design.md)

核心原则：

> SQL 先正确、再清晰、最后优化；性能结论尽可能由 PostgreSQL 执行计划和真实数据证明。

---

## 1. 参数化与注入安全

普通数据值必须通过目标框架提供的参数化机制传入 SQL，不直接拼接用户输入。

SQL 结构无法参数化时，例如动态排序字段、列名或表名，必须先经过严格白名单映射。

原则：

```text
普通数据 → 参数绑定
SQL 标识符 / 结构 → 白名单映射后使用
```

MyBatis 具体绑定语法统一由 `mybatis.md` 维护，本文不重复 `#{}` / `${}` 规则。

---

## 2. SELECT 字段

原则上不要使用：

```sql
SELECT *
FROM ryxx;
```

应明确需要的字段：

```sql
SELECT
    id,
    xm,
    zjhm,
    rqsj,
    lqsj
FROM ryxx;
```

原因包括：

* 减少无关字段传输；
* 降低 Schema 变化影响；
* 映射关系更明确；
* 避免无意暴露新增字段。

---

## 3. COUNT 与 NULL

统计行数通常使用：

```sql
COUNT(*)
```

除非确实需要某列的 NULL 语义。

NULL 判断使用：

```sql
IS NULL
IS NOT NULL
```

禁止：

```sql
column = NULL
column <> NULL
```

编写包含 NULL 的条件和聚合时必须考虑 SQL 三值逻辑。

---

## 4. WHERE 条件与时间范围

WHERE 条件优先直接作用于原始字段。

例如时间范围优先：

```sql
WHERE rqsj >= #{startTime}
  AND rqsj <  #{endTime}
```

而不是为了按天查询机械：

```sql
WHERE DATE(rqsj) = #{date}
```

时间范围优先使用左闭右开：

```text
[start, end)
```

不要使用 `23:59:59.999999` 人工表示一天结束。

字段和参数类型应一致，避免无意依赖 PostgreSQL 隐式类型转换。

---

## 5. JOIN

JOIN 必须明确：

* 关联条件；
* 一对一 / 一对多关系；
* 是否可能放大结果集；
* 过滤条件应该属于 JOIN 还是最终 WHERE。

禁止缺少关联条件导致意外笛卡尔积。

使用 `LEFT JOIN` 时注意：

```sql
LEFT JOIN csxx c ON ...
WHERE c.zt = '1'
```

可能使结果语义接近 INNER JOIN。必须确认这是业务真正需要的行为。

---

## 6. DISTINCT 与 GROUP BY

不要看到重复数据就直接加：

```sql
DISTINCT
```

先分析重复是否来自错误 JOIN、一对多关系或查询模型设计。

只有业务确实需要去重时才使用 DISTINCT。

聚合查询必须明确分组维度，并遵守 PostgreSQL 的 GROUP BY 语义。

---

## 7. ORDER BY 与分页

需要稳定顺序时必须显式 `ORDER BY`，分页查询尤其如此。

推荐使用能够稳定确定顺序的组合，例如：

```sql
ORDER BY cjsj DESC, id DESC
```

普通数据量可以使用：

```sql
LIMIT #{pageSize}
OFFSET #{offset}
```

深分页时评估 Keyset Pagination，但必须基于实际业务排序字段设计，不机械改造。

客户端可控排序字段必须经过白名单映射。

---

## 8. EXISTS 与 IN

只判断存在性时优先表达存在性，而不是加载完整列表后在 Java 中判断。

PostgreSQL 可以使用：

```sql
SELECT EXISTS (
    SELECT 1
    FROM ryxx
    WHERE zjhm = #{identityNumber}
);
```

`IN` 适合有限值集合，但禁止无边界构造巨大列表。

数据量大时根据实际情况评估：

* 分批；
* JOIN；
* 临时表；
* PostgreSQL 数组；
* 其他项目已有方案。

---

## 9. INSERT

INSERT 应明确字段，不依赖数据库字段顺序。

例如：

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
);
```

数据库默认值只能在其职责和业务语义明确时依赖。

---

## 10. UPDATE

UPDATE 必须具有明确 WHERE 条件，并检查实际修改范围。

状态竞争场景可以评估条件更新：

```sql
UPDATE csxx
SET zt = #{targetStatus}
WHERE id = #{id}
  AND zt = #{expectedStatus};
```

是否需要条件更新、乐观锁、悲观锁或其他机制，由真实并发和事务需求决定。

详细一致性规则读取：

- [事务](../architecture/transactions.md)

---

## 11. DELETE

DELETE 必须具有明确范围。

禁止无意生成：

```sql
DELETE FROM ryxx;
```

执行删除前应确认目标项目已有：

* 物理 / 逻辑删除语义；
* 关联数据处理；
* 数据权限；
* 业务影响。

不得因为当前实现方便自行改变删除语义。

---

## 12. UPSERT

PostgreSQL 可以使用：

```sql
INSERT ... ON CONFLICT ...
```

但冲突语义必须建立在明确的 PRIMARY KEY 或 UNIQUE Constraint 上。

不要为了隐藏重复问题无依据使用：

```sql
ON CONFLICT DO NOTHING
```

---

## 13. 批量操作

大量数据写入应评估批处理，而不是无边界循环单条 SQL。

可根据项目和数据规模评估：

* JDBC / MyBatis Batch；
* PostgreSQL 多 Values INSERT；
* COPY；
* 分批提交。

批次大小必须结合数据量、SQL 长度、内存、事务范围和数据库承载能力，不照搬固定经验阈值。

---

## 14. N+1

必须关注调用方式，而不仅是单条 SQL。

典型问题：

```text
查询 100 条主数据
       ↓
循环执行 100 次关联查询
```

发现循环数据库访问时评估：

* 批量查询；
* JOIN；
* IN；
* 一次查询后内存分组；
* 改变查询模型。

不要为了消除 N+1 又制造无边界巨大 JOIN 或 IN。

---

## 15. PostgreSQL 函数、LIKE 与索引

WHERE 条件对索引列使用函数时，应评估索引是否仍能满足查询。

例如：

```sql
WHERE lower(xm) = lower(#{name})
```

高频模糊搜索：

```sql
LIKE '%keyword%'
ILIKE '%keyword%'
```

不能默认在大表上性能可接受，可根据真实需求评估 `pg_trgm`、GIN / GiST 或全文检索。

不凭经验直接创建索引，索引设计统一读取 `database-design.md`。

---

## 16. OR、UNION、子查询与 CTE

大量 OR 条件应结合可读性和执行计划评估，不为了 SQL 短而强行合并不同查询语义。

业务不需要去重时可以评估：

```sql
UNION ALL
```

但不机械替换 `UNION`。

子查询本身不是性能问题，不看到 subquery 就自动改 JOIN。

CTE：

```sql
WITH ...
```

适合分解复杂查询和阶段性计算，但不要为了“高级写法”把简单 SQL 全部改成 CTE。

---

## 17. 索引与真实查询配套

索引应服务真实：

```text
WHERE
JOIN
ORDER BY
GROUP BY
```

例如实际查询长期使用：

```sql
WHERE fzxbh = #{centerCode}
  AND rqsj >= #{startTime}
  AND rqsj <  #{endTime}
```

可以评估联合索引：

```text
(fzxbh, rqsj)
```

是否创建及列顺序必须结合真实数据量和执行计划。

Schema 索引规则读取：

- [database-design.md](database-design.md)

---

## 18. EXPLAIN 与性能结论

复杂或性能敏感 SQL 应使用：

```sql
EXPLAIN
```

分析扫描方式、Join Strategy、Sort、估算行数等。

需要真实执行信息时可以使用：

```sql
EXPLAIN ANALYZE
```

但它会实际执行 SQL，不得对生产环境危险写操作随意执行。

性能判断不能简化成：

```text
走索引 = 快
Seq Scan = 慢
```

应结合：

* 数据量；
* 返回行数；
* 执行时间；
* Buffer；
* 扫描行数；
* 估算误差。

如果没有真实执行计划，应明确说明只是基于 SQL 结构的建议，不得声称“已经更快”或“一定走索引”。

---

## 19. SQL 可读性

复杂 SQL 应保持统一格式和清晰结构。

推荐：

```sql
SELECT
    a.id,
    a.zjhm,
    a.rqsj
FROM ryxx a
WHERE a.fzxbh = #{centerCode}
  AND a.rqsj >= #{startTime}
  AND a.rqsj <  #{endTime}
ORDER BY
    a.rqsj DESC,
    a.id DESC;
```

核心业务规则原则上由业务层表达；SQL 负责数据查询、聚合和数据库擅长的数据运算。

---

## 20. Codex SQL 检查

编写或修改 SQL 后检查：

1. 普通数据是否通过参数化机制传入；结构性动态内容是否经过白名单。
2. 是否使用无必要 `SELECT *`。
3. NULL 和时间范围语义是否正确。
4. 是否发生无意隐式类型转换。
5. JOIN 是否缺少条件或放大结果集。
6. DISTINCT 是否只是掩盖错误 JOIN。
7. 分页是否具有稳定排序。
8. UPDATE / DELETE 范围是否安全。
9. 是否存在无意识 N+1 或无边界批量 / IN。
10. PostgreSQL 语法和类型是否正确。
11. 性能结论是否有数据、索引和 EXPLAIN 支撑。
12. 是否为了性能猜测牺牲 SQL 正确性和可读性。

最终原则：

> MyBatis 负责“如何绑定和映射”，SQL 规范负责“数据库语句本身是否正确、安全、清晰和高效”。
