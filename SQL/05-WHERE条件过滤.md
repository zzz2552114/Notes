# WHERE 条件过滤

---

# 第一部分：记住一个特别重要的区别

```text
WHERE → 过滤行
HAVING → 过滤组
```

WHERE 在分组之前一行一行地筛；HAVING 在分组之后一组一组地筛。这个区别贯穿整个 SQL 查询。

---

# 第二部分：比较 WHERE vs CHECK

## 相同处

**二者后面都需要跟 bool 表达式。**

## 不同处

|          | CHECK                         | WHERE                    |
| :------- | :---------------------------- | :----------------------- |
| 所属语言 | DDL                           | DML                      |
| 作用     | 定义数据完整性规则            | 查询/更新/删除时筛选行   |
| 执行时机 | INSERT / UPDATE 时自动检查    | 查询或修改时逐行判断     |
| 语法位置 | CREATE TABLE / ALTER TABLE 等 | SELECT / UPDATE / DELETE |
| 例子     | `CHECK (age >= 0)`            | `WHERE age >= 0`         |

**CHECK 后面要加括号，WHERE 后面不用。**

这里 `WHERE` 本身就在做"检查"，不需要再写 `CHECK`。

---

# 第三部分：WHERE 的各种用法

## 1. 比较

```sql
SELECT *
FROM students
WHERE age >= 18;
```

比较运算符：`=, <>, !=, <, >, <=, >=`。

## 2. AND

```sql
SELECT *
FROM students
WHERE age >= 18
  AND age <= 25;
```

## 3. BETWEEN-AND

```sql
WHERE age BETWEEN 18 AND 25
```

两端都包含。所以：

```text
18 ≤ age ≤ 25
```

## 4. IN

```sql
WHERE age IN (18, 20, 22)
```

相当于：

```sql
WHERE age = 18
   OR age = 20
   OR age = 22
```

所以记：

```text
IN = "属于这个列表"
```

## 5. NOT

```sql
WHERE NOT (age < 18)
```

## 6. EXISTS

先记住它的核心意思即可：

> "是否存在至少一行满足条件？"

例如：

```sql
SELECT s.student_name
FROM students AS s
WHERE EXISTS (
    SELECT 1
    FROM enrollments AS e
    WHERE e.student_id = s.student_id
);
```

意思：

> 只找至少选过一门课的学生。

`EXISTS` 更适合记成"存在性判断"，不要把它理解成普通的"取值子查询"。这里括号里的 `SELECT 1 ...` 是子查询，现在只要看懂"存在性判断"这个意思就够了。

---

# 第四部分：为什么 WHERE 不能直接用 SELECT 的别名？

例如：

```sql
SELECT
    age + 1 AS next_age
FROM students
WHERE next_age > 18;
```

在标准 SQL 的一般规则下，这里不成立。因为逻辑上：

```text
WHERE 先于 SELECT
```

此时 `next_age` 这个输出别名还没有形成。可以写：

```sql
SELECT
    age + 1 AS next_age
FROM students
WHERE age + 1 > 18;
```

这就是执行顺序在 WHERE 上的直接体现——WHERE 阶段数据库还不知道你给这列起了什么别名。

---

# 第五部分：进阶——WHERE 的高级用法

## 1. OR 的陷阱：OR 与 AND 的优先级

`AND` 优先级高于 `OR`。如果你写：

```sql
WHERE major = 'CS' AND age >= 18 OR is_active = FALSE
```

数据库实际解释为：

```sql
WHERE (major = 'CS' AND age >= 18) OR (is_active = FALSE)
```

很可能不是你想要的。想表达"CS 学生且（成年或未激活）"就要加括号：

```sql
WHERE major = 'CS'
  AND (age >= 18 OR is_active = FALSE)
```

**注意！！！** 只要 WHERE 里同时出现 AND 和 OR，就老老实实加括号，别赌自己记对了优先级。

## 2. ANY / SOME / ALL 量词

和子查询配合使用的高级比较符：

```sql
-- 成绩大于子查询里任意一个值（即大于最小值）
WHERE score > SOME (SELECT score FROM enrollments WHERE course_id = 1);

-- 成绩大于子查询里所有值（即大于最大值）
WHERE score > ALL (SELECT score FROM enrollments WHERE course_id = 1);
```

- `SOME` 和 `ANY` 完全等价。
- `> SOME` = 大于任意一个 = 大于最小的那个。
- `> ALL` = 大于所有 = 大于最大的那个。

## 3. IS DISTINCT FROM（NULL 安全比较）

普通的 `=` 遇到 NULL 会得到 UNKNOWN。但有时你就是想"稳妥地判断两个值是否不同，哪怕其中一个是 NULL"：

```sql
WHERE age IS DISTINCT FROM NULL
```

`IS DISTINCT FROM` 的规则：把两个 NULL 视为"相等"，只要两边不一样（包括一个 NULL 一个非 NULL）就返回 TRUE。对应的有 `IS NOT DISTINCT FROM`。这是 NULL 安全的等值比较。

## 4. 行比较（Row comparison）

可以把多列打包成"行"来比较：

```sql
WHERE (major, age) > ('CS', 20)
```

它按字典序比较：先比第一列 major，major 相同再比 age。这在和组合索引、组合主键配合时很有用。

## 5. WHERE 中的子查询

- **标量子查询**（返回单个值）：

  ```sql
  WHERE score > (SELECT AVG(score) FROM enrollments)
  ```

- **IN 子查询**：

  ```sql
  WHERE student_id IN (SELECT student_id FROM enrollments WHERE score > 90)
  ```

- **EXISTS 子查询**：见前面第 6 节。

## 6. WHERE 中 NULL 的处理（三值逻辑的应用）

回想三值逻辑：WHERE 只保留条件为 TRUE 的行，UNKNOWN 的行被过滤掉。所以：

```sql
WHERE age >= 18
```

会自动把 `age IS NULL` 的行排除掉（因为 `NULL >= 18` 是 UNKNOWN）。如果你想把 NULL 的人也捞出来，要显式写：

```sql
WHERE age >= 18 OR age IS NULL
```

## 7. BETWEEN 的边界问题

`BETWEEN a AND b` 等价于 `>= a AND <= b`，两端都包含。**注意它和"区间"的直觉相反**——很多人以为 between 是开区间，其实是闭区间。另外如果写 `BETWEEN 25 AND 18`（大小写反），结果永远是空。

## 8. IN 列表与 NULL 的交互陷阱

```sql
WHERE age IN (18, 20, NULL)
```

等价于：

```sql
WHERE age = 18 OR age = 20 OR age = NULL
```

最后那个 `age = NULL` 永远是 UNKNOWN，不会帮你匹配到 NULL 的行。想找 age 为 NULL 的，必须写 `OR age IS NULL`。

## 9. EXISTS 与 IN 的性能对比

两者都能做"半连接"判断，但：

- **IN**：把外层和内层结果逐个比对。
- **EXISTS**：找到第一行匹配就短路返回（"存在即可"）。

一般来说，内层结果集大时 EXISTS 更优（短路省时间），内层小时差别不大。初学知道两者都能用，后面子查询章节会展开。

---

# 第六部分：进阶——WHERE 的工程实践

## 1. 条件下推（pushdown）优化

优化器会尽量把过滤条件"下推"到离数据最近的地方。比如一个视图里包了一堆 JOIN，你在外层 `WHERE age > 18`，优化器会尝试把这个条件推进视图内部，在 JOIN 之前就把无关行过滤掉，减少后续连接的数据量。这叫条件下推。**它的限制**：某些嵌套子查询、函数包裹的列会让优化器无法下推。

## 2. WHERE 中使用函数对索引的影响

如果你写：

```sql
WHERE UPPER(name) = 'ALICE'
```

即使 name 列上有索引，这个索引也用不上——因为你在列外面套了函数，数据库得把每一行的 name 都算一遍 UPPER 才能比较。**规律：对索引列做函数/运算，索引往往失效。** 这是写 WHERE 时要注意的性能点。

## 3. 复杂 WHERE 的可读性

条件一多，务必用括号、缩进、换行让它可读：

```sql
WHERE
    (major = 'CS' AND age >= 18)
    OR (major = 'Math' AND score >= 90)
```

## 4. WHERE 与 HAVING 的本质区别

| 对比 | WHERE | HAVING |
|---|---|---|
| 过滤对象 | 行 | 分组后的组 |
| 执行时机 | 分组前 | 分组+聚合后 |
| 能否用聚合函数 | 不能 | 能 |
| 例子 | `WHERE age >= 18` | `HAVING COUNT(*) > 1` |

---

# 本文件覆盖的原文档知识点清单

1. WHERE 过滤行 vs HAVING 过滤组的核心区别
2. WHERE vs CHECK 相同处（都跟布尔表达式）
3. WHERE vs CHECK 完整对比表（所属语言/作用/执行时机/语法位置/例子）
4. CHECK 要加括号、WHERE 不用
5. WHERE 比较运算符（=, <>, !=, <, >, <=, >=）
6. AND 的使用
7. BETWEEN-AND（两端都包含）
8. IN（属于列表，相当于多个 OR）
9. NOT 的使用
10. EXISTS（存在性判断，"是否存在至少一行"，SELECT 1 的用法）
11. WHERE 不能用 SELECT 别名的执行顺序原理
12. 【补充】OR 详解与 OR/AND 优先级陷阱（AND 高于 OR，加括号）
13. 【补充】ANY / SOME / ALL 量词用法
14. 【补充】IS DISTINCT FROM（NULL 安全比较）与 IS NOT DISTINCT FROM
15. 【补充】行比较 (a, b) > (c, d) 的用法
16. 【补充】WHERE 中的子查询（标量子查询、IN 子查询、EXISTS 子查询）
17. 【补充】WHERE 中 NULL 的三值逻辑处理（UNKNOWN 行被过滤）
18. 【补充】BETWEEN 边界问题与等价 >= AND <=
19. 【补充】IN 列表与 NULL 的交互陷阱
20. 【补充】EXISTS 与 IN 的性能对比和选择原则
21. 【补充】条件下推（pushdown）优化原理与限制
22. 【补充】WHERE 中使用函数对索引的影响
23. 【补充】复杂 WHERE 条件可读性最佳实践
24. 【补充】WHERE 与 HAVING 本质区别对比表
