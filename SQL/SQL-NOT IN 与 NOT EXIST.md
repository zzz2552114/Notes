这段话的核心是：**SQL 里的 `NULL` 会让 `NOT IN` 变成三值逻辑，从而破坏“不在集合里”的直觉；而找“没有对应行”这种需求，本质上应写成 `NOT EXISTS` 的存在性否定。**

下面拆开讲。

---

## 1. 你原本的直觉是什么？

你想找“没有选课的学生”，直觉写法是：

```sql
SELECT *
FROM students s
WHERE s.student_id NOT IN (
    SELECT e.student_id
    FROM enrollments e
);
```

直觉上这表示：

> 从 `enrollments` 里取出所有选过课的学生 id，然后找那些 student_id 不在这个列表里的学生。

如果 `enrollments` 里的 `student_id` 都是正常值，比如 `(1, 2, 3)`，那这个写法通常没问题。

问题在于：**如果子查询结果里出现 `NULL`，结果可能完全不是你想的那样。**

---

## 2. SQL 不是二值逻辑，而是三值逻辑

普通编程里布尔值通常只有：

- `TRUE`
- `FALSE`

但 SQL 里有：

- `TRUE`
- `FALSE`
- `UNKNOWN`

`NULL` 不是普通值，它表示“未知”。所以很多比较结果是 `UNKNOWN`：

```sql
1 = 1        -- TRUE
1 = 2        -- FALSE
1 = NULL     -- UNKNOWN
NULL = NULL  -- UNKNOWN
1 <> NULL    -- UNKNOWN
NULL <> NULL -- UNKNOWN
```

`WHERE` 子句有一个关键规则：

> 只保留条件结果为 `TRUE` 的行。  
> `FALSE` 和 `UNKNOWN` 都会被过滤掉。

也就是说，`UNKNOWN` 在 `WHERE` 里和 `FALSE` 一样，不通过。

---

## 3. `NOT IN` 实际展开后是什么？

```sql
x NOT IN (a, b, c)
```

大致等价于：

```sql
NOT (x IN (a, b, c))
```

而：

```sql
x IN (a, b, c)
```

等价于：

```sql
x = a OR x = b OR x = c
```

所以：

```sql
x NOT IN (a, b, c)
```

等价于：

```sql
NOT (x = a OR x = b OR x = c)
```

也就是：

```sql
x <> a AND x <> b AND x <> c
```

关键来了：如果集合里有 `NULL`：

```sql
x NOT IN (1, NULL)
```

展开成：

```sql
x <> 1 AND x <> NULL
```

假设 `x = 2`：

```sql
2 <> 1     -- TRUE
2 <> NULL  -- UNKNOWN
```

于是：

```sql
TRUE AND UNKNOWN
```

结果是：

```sql
UNKNOWN
```

所以对 `x = 2`：

```sql
2 NOT IN (1, NULL)
```

结果不是 `TRUE`，而是 `UNKNOWN`。

而 `WHERE` 不保留 `UNKNOWN`。

---

## 4. 具体例子：为什么结果会反直觉？

假设学生表：

```text
students
student_id | name
1          | 小明
2          | 小红
3          | 小刚
```

选课表：

```text
enrollments
student_id | course
1          | 数学
NULL       | 英语
```

注意：`enrollments.student_id` 里有一行是 `NULL`。

执行：

```sql
SELECT s.*
FROM students s
WHERE s.student_id NOT IN (
    SELECT e.student_id
    FROM enrollments e
);
```

子查询返回：

```text
(1, NULL)
```

然后逐行判断：

- 小明 `student_id = 1`：

```sql
1 NOT IN (1, NULL)
```

因为 `1 IN (1, NULL)` 是 `TRUE`，所以 `NOT IN` 是 `FALSE`。不选。

- 小红 `student_id = 2`：

```sql
2 NOT IN (1, NULL)
```

展开：

```sql
NOT (2 = 1 OR 2 = NULL)
= NOT (FALSE OR UNKNOWN)
= NOT UNKNOWN
= UNKNOWN
```

`WHERE` 不保留 `UNKNOWN`。不选。

- 小刚 `student_id = 3`：

同理，结果是 `UNKNOWN`。不选。

最终结果：**空集**。

但你的直觉可能是：小红和小刚没选课，应该被选出来。  
实际却是：因为子查询里有一个 `NULL`，整个 `NOT IN` 条件被污染，导致没有任何行能让条件为 `TRUE`。

更准确地说：

> 在标准 SQL 三值逻辑下，如果子查询结果包含 `NULL`，那么单独使用 `WHERE x NOT IN (subquery)` 通常不会返回任何行。  
> 因为匹配非 NULL 的行得到 `FALSE`，不匹配的行得到 `UNKNOWN`，没有 `TRUE`。

---

## 5. `NOT EXISTS` 为什么更稳妥？

你给出的稳妥写法是：

```sql
SELECT s.*
FROM students s
WHERE NOT EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
);
```

它的逻辑是：

> 对每一个学生 `s`，去 `enrollments` 里找是否存在一行 `e`，使得 `e.student_id = s.student_id`。  
> 如果不存在这样的行，就选出来。

继续用上面的例子：

- 小明 `student_id = 1`：

子查询：

```sql
SELECT 1
FROM enrollments e
WHERE e.student_id = 1;
```

能找到 `(1, 数学)`，所以 `EXISTS` 为 `TRUE`，`NOT EXISTS` 为 `FALSE`。不选。

- 小红 `student_id = 2`：

子查询：

```sql
SELECT 1
FROM enrollments e
WHERE e.student_id = 2;
```

`enrollments` 里有 `(NULL, 英语)`，但：

```sql
NULL = 2
```

结果是 `UNKNOWN`，不是 `TRUE`。  
`WHERE e.student_id = 2` 不会把这一行当作满足条件。

所以子查询没有返回任何行，`EXISTS` 为 `FALSE`，`NOT EXISTS` 为 `TRUE`。选小红。

- 小刚同理，选小刚。

最终结果：小红和小刚。这才符合“没有选课的学生”的直觉。

关键区别：

- `NOT IN` 会去比较“不等于集合中的每个值”，遇到 `NULL` 产生 `UNKNOWN`。
- `NOT EXISTS` 只关心“是否存在一行让条件为 `TRUE`”。`NULL = 2` 是 `UNKNOWN`，不算存在。
- `EXISTS` / `NOT EXISTS` 本身返回 `TRUE` 或 `FALSE`，不会返回 `UNKNOWN`。

所以 `NOT EXISTS` 不会被 `NULL` 以同样的方式污染。

---

## 6. “存在性查询”和“集合包含”是两件事

你这段话最后强调：

> 先把“存在性查询”和“集合包含”区分开。

这很重要。

### 集合包含：`IN` / `NOT IN`

它问的是：

> 某个值是否属于子查询返回的值集合？

例如：

```sql
s.student_id IN (SELECT e.student_id FROM enrollments e)
```

这是“集合成员测试”。

问题是：SQL 中的集合可能含 `NULL`，而 `NULL` 不是普通值，比较会变成 `UNKNOWN`。

### 存在性查询：`EXISTS` / `NOT EXISTS`

它问的是：

> 是否存在至少一行满足某个条件？

例如：

```sql
EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
)
```

这是“存在量词”：

> 存在一行 `e`，使得 `e.student_id = s.student_id`。

`NOT EXISTS` 就是：

> 不存在这样一行。

找“没有对应行”时，业务语义其实是：

> 在关联表里不存在匹配行。

所以 `NOT EXISTS` 更贴近本意。

在**没有 NULL** 的情况下，下面两者逻辑上等价：

```sql
s.student_id NOT IN (
    SELECT e.student_id
    FROM enrollments e
)
```

和：

```sql
NOT EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
)
```

但一旦有 `NULL`，它们就不等价了。

---

## 7. 实践建议

1. **找“没有对应行”时，优先用 `NOT EXISTS`。**

```sql
WHERE NOT EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
)
```

2. **也可以用 `LEFT JOIN ... IS NULL`**，但要注意选择非空列判断：

```sql
SELECT s.*
FROM students s
LEFT JOIN enrollments e
    ON e.student_id = s.student_id
WHERE e.student_id IS NULL;
```

3. **如果非要用 `NOT IN`，必须确保子查询列不会出现 `NULL`。**

例如：

```sql
WHERE s.student_id NOT IN (
    SELECT e.student_id
    FROM enrollments e
    WHERE e.student_id IS NOT NULL
)
```

如果外层 `s.student_id` 也可能为 `NULL`，还要处理外层：

```sql
WHERE s.student_id IS NOT NULL
  AND s.student_id NOT IN (
      SELECT e.student_id
      FROM enrollments e
      WHERE e.student_id IS NOT NULL
  )
```

4. **不要依赖“当前数据没有 NULL”。**  
   只要列没有 `NOT NULL` 约束，未来就可能出现 `NULL`，或者通过外连接、`UNION`、`CASE` 等引入 `NULL`。

---

## 总结

这段话在说：

- `NOT IN` 看起来像“不在集合里”，但 SQL 会把它展开成一系列 `<>` 比较。
- 如果子查询结果含 `NULL`，`x <> NULL` 不是 `TRUE`，而是 `UNKNOWN`。
- `WHERE` 只保留 `TRUE`，所以 `UNKNOWN` 会让本该返回的行不返回。
- 单独使用 `WHERE x NOT IN (subquery)` 且子查询含 `NULL` 时，通常整个查询结果为空。
- `NOT EXISTS` 表达的是“不存在满足关联条件的行”，`EXISTS` 只关心有没有行满足条件，`NULL` 比较为 `UNKNOWN` 的行不算满足。
- 因此，找“没有对应行”时，`NOT EXISTS` 是更稳妥、更符合语义的写法。
- 核心区别：`NOT IN` 是“集合包含/成员测试”，`NOT EXISTS` 是“存在性否定”。在 SQL 的 `NULL` 三值逻辑下，两者不能随便互换。