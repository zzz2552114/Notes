# SELECT 查询基础

---

# 第一部分：SELECT 核心结构

最核心的查询结构：

```sql
SELECT ...
FROM ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
```

不要要求自己一次记住所有子句。先把下面这个练熟：

```sql
SELECT student_name, age
FROM students;
```

理解成：

```text
SELECT      我要什么
student_name, age

FROM        从哪里拿
students
```

## 1. 所有列

```sql
SELECT *
FROM students;
```

`*` 在这里表示：

> 当前表的全部列。

## 2. 只看某几列

```sql
SELECT student_name, age
FROM students;
```

## 3. 起别名 AS

```sql
SELECT
    student_name AS name,
    age + 1 AS next_age
FROM students;
```

`AS` 可以理解成：

> "把这个结果叫做……"

例如：

```sql
age + 1 AS next_age
```

就是把"age+1 算出来的那一列"命名为 `next_age`，它会显示在结果表头里。

## 4. 表也可以起别名

```sql
SELECT s.student_name
FROM students AS s;
```

后面：

```sql
s.student_name
```

就表示 students 表里的 student_name。多表查询几乎一定会用到这个——两张表可能有同名列，不起别名就分不清是哪张表的。

---

# 第二部分：查询运行顺序（极其重要）

对于如下查询：

```sql
SELECT ...
FROM table_a
JOIN table_b ON ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
```

虽然你写的时候是先写 SELECT，但数据库**逻辑执行**的顺序是：

1. **FROM / JOIN / ON**
   先确定数据来源，做表连接。`ON` 属于连接条件，在 `WHERE` 之前处理。
2. **WHERE**
   对连接后的**行**做过滤。此时还没有分组，也没有计算 `SELECT` 中的别名。
3. **GROUP BY**
   对过滤后的行进行分组。
4. **聚合函数计算**
   例如 `COUNT()`、`SUM()`、`AVG()` 等。
5. **HAVING**
   对分组后的**组**做过滤。
6. **SELECT**
   计算最终输出的列，并在这里定义**列别名**。例如 `student_id AS SID` 中的 `SID` 就是在这里产生的。
7. **DISTINCT**
   如果有 `DISTINCT`，在这里去重。
8. **ORDER BY**
   对最终结果排序。
9. **LIMIT / OFFSET**
   最后截取行数。

把它浓缩成一句话：

```text
FROM → WHERE → GROUP BY → 聚合 → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

---

# 第三部分：别名在哪一步可见？

## 1. 表别名

表别名在 `FROM` / `JOIN` 中定义，定义之后，同一查询块的后续子句基本都可见：

```sql
FROM students s
JOIN courses c ON s.student_id = c.student_id
WHERE s.age > 18
GROUP BY s.student_id
HAVING COUNT(c.course_id) > 1
ORDER BY s.student_id;
```

这里 `s`、`c` 在 `SELECT`、`ON`、`WHERE`、`GROUP BY`、`HAVING`、`ORDER BY` 中都能用。因为表别名是在最开头的 FROM 阶段定义的，后面所有阶段都跑在它"之后"。

## 2. 列别名

列别名是在 `SELECT` 中定义的，例如：

```sql
SELECT student_id AS SID,
       student_name AS NAME
FROM students;
```

`SID`、`NAME` 是在 **SELECT 阶段**才产生的。因此标准 SQL 中：

| 子句     | 能否使用 SELECT 中定义的列别名     |
| :------- | :--------------------------------- |
| WHERE    | 不能                               |
| GROUP BY | 标准不能，部分数据库可以           |
| HAVING   | 标准不能，部分数据库可以           |
| SELECT   | 正在定义                           |
| ORDER BY | 通常可以                           |
| 外层查询 | 可以，如果这个 SELECT 是子查询/CTE |

举例：

```sql
SELECT student_id AS SID, COUNT(*) AS cnt
FROM students
WHERE SID > 10        -- 标准 SQL 错：WHERE 在 SELECT 之前
GROUP BY SID          -- 标准 SQL 错；MySQL 可以
HAVING cnt > 1        -- 标准 SQL 错；MySQL 可以
ORDER BY SID;         -- 可以，ORDER BY 在 SELECT 之后
```

原因是：`WHERE`、`GROUP BY`、`HAVING` 都在 `SELECT` 之前执行，所以它们看不到 `SELECT` 里刚起的列别名。而 `ORDER BY` 在 `SELECT` 之后执行，所以通常可以使用列别名。

- **PostgreSQL**：允许 `ORDER BY` 使用别名，`GROUP BY` 也可以引用输出列名，但 `HAVING` 通常不行。

## 3. 为什么 FROM 先于 SELECT？

```sql
SELECT age
FROM students;
```

可以理解为：

> 先把 students 这张表作为当前数据来源。

所以它**不是先"算 age"**，而是先确定"我要从 students 里拿数据"。这也是为什么 SQL 的逻辑执行顺序不会从 SELECT 开始——你得先有一张表，才能谈"从里面选哪几列"。

---

# 第四部分：进阶——SELECT 后到底能放什么

## 1. SELECT 后能放表达式、函数、CASE

SELECT 后面不只列名，任何"算出一个值"的表达式都能放：

```sql
SELECT
    student_name,
    age + 1 AS next_age,              -- 算术表达式
    UPPER(major) AS major_upper,      -- 函数
    score * 1.1 AS bonus,
    CASE WHEN score >= 60 THEN 'pass' ELSE 'fail' END AS result
FROM enrollments;
```

## 2. 常量列（字面量列）

你也可以在 SELECT 里放一个写死的值，它会在结果的每一行上重复出现：

```sql
SELECT student_name, '2026 学年' AS school_year
FROM students;
```

结果里每一行都有一个 `school_year = '2026 学年'` 的列。这在拼接数据、补常量字段时很常用。

## 3. 行构造器 ROW

```sql
SELECT ROW(student_name, age) AS student_info
FROM students;
```

它把多列打包成一个"行值"，进阶用法见行比较。

## 4. SELECT DISTINCT ON（PostgreSQL 特有）

这是 PG 非常实用但新手不知道的语法："每组只保留第一行"。

```sql
-- 每个学生只保留成绩最高的那条选课记录
SELECT DISTINCT ON (student_id)
    student_id, course_id, score
FROM enrollments
ORDER BY student_id, score DESC;
```

意思：按 `student_id` 分组，每组只取排序后的第一行（这里因为按 score DESC 排，所以取的是最高分那条）。**注意：`DISTINCT ON` 的列必须和 `ORDER BY` 的最左边列一致。**

## 5. 集合返回函数（generate_series、unnest）

PostgreSQL 的 SELECT 还能"凭空生成多行"：

```sql
-- 生成 1 到 5 的一列
SELECT generate_series(1, 5) AS n;

-- 把数组拆成多行
SELECT unnest(ARRAY['a', 'b', 'c']) AS item;
```

这在造测试数据、展开数组列时非常好用。

---

# 第五部分：进阶——别名、通配符与限定名的细节

## 1. AS 关键字可以省略

列别名的 `AS` 可以不写，效果一样：

```sql
SELECT student_name name, age + 1 next_age
FROM students;
```

表别名的 `AS` 也可以省略：

```sql
SELECT s.student_name
FROM students s;     -- 不用写 AS
```

**注意！！！** 省略 AS 虽然合法，但初学者建议写出来。因为不写 AS 时，如果漏打一个逗号，数据库会把下一个列名当成上一个列的别名，产生诡异的结果。写 AS 更安全、更好读。

## 2. SELECT * 的利弊

```sql
SELECT * FROM students;
```

- **好处**：调试、快速看表结构时方便。
- **坏处（生产环境为什么不建议用）**：
  1. 多拉了不需要的列，浪费网络和内存；
  2. 表结构变了（加列、删列），`SELECT *` 的结果列顺序也跟着变，依赖列位置的应用会崩；
  3. 配合"只取需要的列"原则，显式列出列名是更稳妥的工程实践。

## 3. 限定列名（schema.table.column）

可以用点号逐级限定列名，在多表、跨 schema 时非常清楚：

```sql
SELECT students.student_name
FROM students;

-- 跨 schema
SELECT public.student_name
FROM public.students;
```

多表查询时，凡是可能混淆的列都要用 `表别名.列名` 写清楚。

---

# 第六部分：进阶——理解执行顺序的实际意义

## 1. 为什么理解执行顺序能避免 90% 的 SQL 错误

绝大多数"我的 SQL 为什么报错/查不出数"，根源都是忘了执行顺序：

- 想在 WHERE 里用 SELECT 的别名 → 报错（WHERE 比 SELECT 早）；
- 想在 HAVING 里引用普通列而不是聚合结果 → 语义错误（HAVING 是过滤组的）；
- 搞混 WHERE 和 HAVING → 过滤时机不对。

记住"FROM 最先、SELECT 在中间、ORDER BY 靠后"，这些问题就迎刃而解。

## 2. 逻辑执行顺序 vs 物理执行顺序

上面讲的是**逻辑执行顺序**——它描述的是"结果应该长什么样"的语义顺序。

但真实数据库里，优化器（planner/optimizer）为了性能，**物理上**可能会调换执行顺序：比如它可能先做 JOIN 再做 WHERE，或者提前把投影（只取需要的列）下推。这些物理重排**不改变逻辑结果**，只改变"怎么算更快"。

**注意！！！** 你学习时按逻辑顺序理解（保证写出来的语义正确），性能调优时才去关心物理执行计划（`EXPLAIN`）。别把两者混为一谈。

---

# 本文件覆盖的原文档知识点清单

1. SELECT 核心查询结构（SELECT/FROM/WHERE/GROUP BY/HAVING/ORDER BY）
2. SELECT * 所有列
3. 只看某几列（指定列）
4. 列别名 AS（"把这个结果叫做……"）
5. 表别名（FROM students AS s，多表查询必用）
6. 查询运行顺序：FROM/JOIN/ON → WHERE → GROUP BY → 聚合 → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET
7. 表别名在 FROM/JOIN 定义后后续子句都可见
8. 列别名在 SELECT 阶段才产生
9. 各子句能否用 SELECT 列别名的完整表格（WHERE不能/GROUP BY标准不能/HAVING标准不能/ORDER BY通常可以/外层查询可以）
10. PostgreSQL 特殊行为（GROUP BY 可引用输出列名，HAVING 通常不行）
11. FROM 先于 SELECT 的原理解释
12. 【补充】SELECT 中表达式计算（算术、函数、CASE 作为列）
13. 【补充】常量列（字面量列）的使用
14. 【补充】行构造器 ROW 的使用
15. 【补充】SELECT DISTINCT ON（PostgreSQL 特有）语法与示例
16. 【补充】集合返回函数 generate_series、unnest
17. 【补充】SELECT * 的利弊（生产环境为什么不建议）
18. 【补充】列别名 AS、表别名 AS 可省略的情况及风险
19. 【补充】限定列名（schema.table.column）的使用
20. 【补充】理解执行顺序能避免 90% SQL 错误
21. 【补充】逻辑执行顺序 vs 物理执行顺序的区别
