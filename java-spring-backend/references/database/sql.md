# SQL 与 PostgreSQL 规范

本文档定义项目 SQL 编写和 PostgreSQL 查询规范。

参考《阿里巴巴 Java 开发手册》SQL 相关规约，并结合 PostgreSQL 实际行为制定。

核心原则：

> 先保证 SQL 正确和可读，再根据真实执行计划进行优化。

> 不根据经验猜测性能，复杂 SQL 应通过 PostgreSQL 执行计划验证。

---

# 1. SELECT 字段

原则上禁止：

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

好处：

* 减少无用数据传输；
* 防止新增字段影响查询；
* 映射关系更清楚；
* 减少数据库模型向上层泄漏。

---

# 2. COUNT

统计行数使用：

```sql
COUNT(*)
```

例如：

```sql
SELECT COUNT(*)
FROM ryxx
WHERE fzxbh = #{centerCode};
```

不要为了统计行数机械使用：

```sql
COUNT(id)
COUNT(1)
```

除非确实需要不同的 NULL 语义。

---

# 3. NULL

NULL 判断必须使用：

```sql
IS NULL
IS NOT NULL
```

禁止：

```sql
column = NULL
column <> NULL
```

编写涉及 NULL 的逻辑时，应明确 SQL 三值逻辑。

例如：

```sql
WHERE lqsj IS NULL
```

可以表示尚未离区。

---

# 4. 参数绑定

MyBatis 中优先：

```xml
#{value}
```

禁止直接拼接用户输入：

```xml
${value}
```

`${}` 只能用于无法参数化的结构性 SQL 内容，并且必须来自严格白名单。

---

# 5. WHERE 条件

WHERE 条件应尽量直接作用于原始字段。

例如时间查询优先：

```sql
WHERE rqsj >= #{startTime}
  AND rqsj < #{endTime}
```

避免：

```sql
WHERE DATE(rqsj) = #{date}
```

前一种方式通常更有利于普通 B-Tree 索引使用。

---

# 6. 时间范围

时间范围优先采用：

```text
[start, end)
```

即左闭右开。

例如查询某天：

```sql
WHERE rqsj >= TIMESTAMP '2026-09-01 00:00:00'
  AND rqsj <  TIMESTAMP '2026-09-02 00:00:00'
```

不要构造：

```text
23:59:59.999999
```

作为一天结束时间。

---

# 7. 避免隐式类型转换

查询条件的数据类型应与字段类型一致。

例如字段：

```text
varchar
```

参数也应按照字符串处理。

不要依赖 PostgreSQL 隐式转换。

隐式转换可能导致：

* 查询失败；
* 执行计划异常；
* 索引无法按预期使用。

---

# 8. JOIN

JOIN 必须明确：

* 关联条件；
* 一对一 / 一对多关系；
* 是否可能放大结果集。

例如：

```sql
SELECT
    a.id,
    b.csbh
FROM ajxx a
JOIN csxx b
    ON b.id = a.csid;
```

禁止缺失关联条件产生意外笛卡尔积。

---

# 9. LEFT JOIN

使用 `LEFT JOIN` 时要特别注意 WHERE 条件。

例如：

```sql
LEFT JOIN csxx c
    ON c.id = a.csid
WHERE c.zt = '1'
```

可能实际上将 LEFT JOIN 变成类似 INNER JOIN 的效果。

必须明确：

```text
过滤条件属于 JOIN 条件
还是最终结果过滤
```

---

# 10. DISTINCT

不要看到重复数据就直接添加：

```sql
DISTINCT
```

应先分析重复原因。

可能原因：

* JOIN 关系错误；
* 一对多关系；
* 查询模型设计错误。

只有业务确实要求去重时才使用 DISTINCT。

---

# 11. GROUP BY

聚合查询必须明确分组维度。

例如：

```sql
SELECT
    fzxbh,
    COUNT(*) AS sl
FROM ryxx
GROUP BY fzxbh;
```

不要依赖其他数据库宽松的 GROUP BY 行为。

SQL 必须符合 PostgreSQL 语义。

---

# 12. ORDER BY

需要稳定结果顺序时必须明确：

```sql
ORDER BY
```

尤其分页查询必须具有确定性排序。

推荐：

```sql
ORDER BY cjsj DESC, id DESC
```

避免只按非唯一字段排序，导致分页过程中顺序不稳定。

---

# 13. 分页

普通数据量可以使用：

```sql
LIMIT #{pageSize}
OFFSET #{offset}
```

但必须有稳定：

```sql
ORDER BY
```

深分页场景：

```text
OFFSET 很大
```

时应评估 Keyset Pagination。

例如：

```sql
WHERE cjsj < #{lastCreateTime}
ORDER BY cjsj DESC
LIMIT #{pageSize}
```

实际方案必须结合业务排序字段设计。

---

# 14. EXISTS

只判断是否存在数据时，优先表达“存在性”，而不是加载所有数据。

例如可以考虑：

```sql
SELECT EXISTS (
    SELECT 1
    FROM ryxx
    WHERE zjhm = #{identityNumber}
);
```

或使用项目已有 Mapper 风格。

不要为了判断存在：

```text
加载完整对象列表
↓
Java 判断 size > 0
```

---

# 15. IN

少量确定值可以使用：

```sql
WHERE id IN (...)
```

MyBatis 使用：

```xml
<foreach>
```

批量生成。

禁止构造无限增长的巨大 `IN` 列表。

数据量较大时应考虑：

* 分批；
* JOIN；
* 临时表；
* PostgreSQL 数组；
* 其他适合当前场景的方案。

---

# 16. UPDATE

UPDATE 必须具有明确 WHERE 条件。

例如：

```sql
UPDATE ryxx
SET lqsj = #{exitTime}
WHERE id = #{id};
```

更新前必须判断修改范围。

对于状态流转，优先考虑条件更新：

```sql
UPDATE ...
SET zt = #{targetStatus}
WHERE id = #{id}
  AND zt = #{expectedStatus};
```

并检查受影响行数。

---

# 17. DELETE

DELETE 必须有明确范围。

禁止生成：

```sql
DELETE FROM ryxx;
```

除非任务明确要求清空整个表。

执行删除前应确认：

* 是否逻辑删除；
* 是否物理删除；
* 关联数据；
* 数据权限；
* 业务影响。

---

# 18. 批量写入

禁止大量数据：

```text
循环
 ↓
单条 INSERT
```

而不评估批处理。

可以根据场景考虑：

* Batch；
* 多 Values INSERT；
* PostgreSQL COPY；
* 分批提交。

批次大小必须结合：

* 数据量；
* SQL 长度；
* 内存；
* 事务大小；

进行控制。

---

# 19. UPSERT

PostgreSQL 需要插入或更新时，可以使用：

```sql
INSERT ...
ON CONFLICT ...
```

但必须依赖明确的：

* Primary Key；
* UNIQUE Constraint；

定义冲突语义。

不要为了避免重复异常而随意使用：

```text
ON CONFLICT DO NOTHING
```

这可能掩盖真正的数据问题。

---

# 20. 函数与索引

在 WHERE 条件中对索引字段使用函数时，应评估索引使用情况。

例如：

```sql
WHERE lower(xm) = lower(#{name})
```

普通索引未必能够满足查询。

如果这种查询确实高频，应评估：

* 表达式索引；
* 数据标准化；
* 其他查询策略。

不要凭经验直接假设索引会生效。

---

# 21. LIKE / ILIKE

PostgreSQL 支持：

```sql
LIKE
ILIKE
```

前缀查询：

```sql
LIKE 'abc%'
```

和：

```sql
LIKE '%abc%'
```

具有完全不同的索引使用特征。

大量模糊搜索需求应结合：

* 数据量；
* `pg_trgm`；
* GIN / GiST；
* 全文检索；

进行专门设计。

不要默认 `%keyword%` 在大表上性能可接受。

---

# 22. OR 条件

大量：

```sql
WHERE a = ?
   OR b = ?
   OR c = ?
```

应结合执行计划评估。

不要为了“SQL 看起来短”将不同查询语义强行合并成大量 OR。

有时拆分：

```text
UNION ALL
```

可能更清晰，但必须依据真实执行计划决定。

---

# 23. UNION 与 UNION ALL

如果业务不需要去重，优先考虑：

```sql
UNION ALL
```

`UNION` 会额外执行去重。

不要机械替换，应先确认业务是否允许重复。

---

# 24. 子查询

子查询本身不是性能问题。

不要看到：

```text
subquery
```

就自动改 JOIN。

PostgreSQL 优化器能够处理很多子查询。

应根据：

```text
EXPLAIN
```

结果决定是否需要重写。

---

# 25. CTE

PostgreSQL 可以使用：

```sql
WITH ...
```

提高复杂 SQL 的表达清晰度。

例如：

```sql
WITH dates AS (...),
in_stats AS (...),
out_stats AS (...)
SELECT ...
```

CTE 应用于：

* 分解复杂查询；
* 提高可读性；
* 表达阶段性计算。

不要为了“高级”而把简单 SQL 全部改写成 CTE。

---

# 26. 索引设计与 SQL 必须配套

不要独立设计索引。

索引应从真实查询出发：

```text
WHERE
JOIN
ORDER BY
GROUP BY
```

例如：

```sql
WHERE fzxbh = #{centerCode}
  AND rqsj >= #{startTime}
  AND rqsj < #{endTime}
```

可以评估：

```text
(fzxbh, rqsj)
```

联合索引。

最终是否创建必须依据真实数据量和执行计划判断。

---

# 27. EXPLAIN

复杂或性能敏感 SQL 应使用：

```sql
EXPLAIN
```

分析：

* Seq Scan；
* Index Scan；
* Bitmap Scan；
* Join Strategy；
* Sort；
* Estimated Rows。

需要真实执行数据时可以使用：

```sql
EXPLAIN ANALYZE
```

但必须注意：

> `EXPLAIN ANALYZE` 会实际执行 SQL。

禁止对生产环境的危险：

```text
UPDATE
DELETE
INSERT
```

随意执行 `EXPLAIN ANALYZE`。

---

# 28. 不要只看是否走索引

SQL 性能判断不能简化成：

```text
走索引 = 快
Seq Scan = 慢
```

对于：

* 小表；
* 返回大部分数据；
* 低选择性字段；

PostgreSQL 选择 Seq Scan 可能完全合理。

应关注：

* 数据量；
* 返回行数；
* 实际执行时间；
* Buffer；
* 扫描行数；
* 估算误差。

---

# 29. 避免 N+1

SQL 性能问题不仅来自单条 SQL。

必须关注调用方式：

```text
查询 100 条数据
      ↓
循环执行 100 次 SQL
```

即使每条 SQL 都很快，整体也可能很慢。

发现 Mapper 位于循环中时，应优先检查：

* 批量查询；
* JOIN；
* IN；
* 数据预加载。

---

# 30. 数据库函数

可以合理使用 PostgreSQL 内置函数。

但不要把大量核心业务规则隐藏在复杂 SQL 中。

SQL 负责：

* 数据查询；
* 聚合；
* 数据库擅长的计算。

核心业务规则原则上仍应由业务层表达。

---

# 31. SQL 可读性

复杂 SQL 应保持统一格式。

推荐：

```sql
SELECT
    a.id,
    a.zjhm,
    a.rqsj
FROM ryxx a
WHERE a.fzxbh = #{centerCode}
  AND a.rqsj >= #{startTime}
  AND a.rqsj < #{endTime}
ORDER BY
    a.rqsj DESC,
    a.id DESC;
```

避免：

```sql
SELECT a.id,a.zjhm,a.rqsj FROM ryxx a WHERE a.fzxbh=#{centerCode} AND ...
```

可读性优先于少写几行。

---

# 32. Codex SQL 优化原则

Codex 不得仅凭经验声称某 SQL：

```text
性能更好
一定走索引
一定更快
```

性能相关修改前应优先检查：

1. SQL 当前结构；
2. 表数据量；
3. 现有索引；
4. 查询选择性；
5. `EXPLAIN`；
6. 必要时 `EXPLAIN ANALYZE`。

如果无法获得真实执行计划，应明确：

> 这是基于 SQL 结构的优化建议，而不是已经验证的性能结论。

---

# 33. Codex SQL 检查

编写或修改 SQL 后检查：

* 是否使用 `SELECT *`；
* WHERE 是否遗漏；
* UPDATE / DELETE 范围是否安全；
* NULL 语义是否正确；
* 时间范围是否使用合理的左闭右开区间；
* 是否发生隐式类型转换；
* JOIN 是否可能导致重复；
* DISTINCT 是否只是为了掩盖 JOIN 问题；
* 分页是否有稳定排序；
* 是否存在深分页；
* 是否存在 N+1；
* 是否构造巨大 IN；
* 索引是否与实际查询条件匹配；
* 数据库拼音命名是否保持一致；
* 是否使用了 MySQL 特有 SQL；
* 是否对性能做出了没有执行计划支撑的结论。

最终原则：

> SQL 先正确、再清晰、最后优化；性能结论尽可能由 PostgreSQL 执行计划证明。

