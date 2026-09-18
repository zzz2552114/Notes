**SQL 语法规定：括号只在需要“列表、表达式、子查询、函数参数”等地方出现**。  
`ALTER TABLE 表名` 后面跟的是“动作子句”，不是列表，所以通常不加括号。

---

## 1. 对比看就很清楚

正确的：

```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(50),
    CHECK (student_id > 0)
);

ALTER TABLE students ADD COLUMN age INT;

ALTER TABLE students
ADD CONSTRAINT chk_age CHECK (age >= 0);
```

错误的：

```sql
ALTER TABLE students (ADD COLUMN age INT);  -- 一般会报错
CREATE TABLE students student_id INT;      -- 错，缺少表定义括号
CHECK student_id > 0;                      -- 错，CHECK 后要括号
```

---

## 2. 为什么 `CREATE TABLE xxx` 后面有括号？

因为 `CREATE TABLE` 的语法是：

```sql
CREATE TABLE 表名 (
    列定义,
    约束定义
);
```

括号里是**表元素列表**：列、主键、外键、唯一约束、检查约束等。  
因为要一次定义多个东西，所以需要括号把它们包起来，用逗号分隔。

例如：

```sql
CREATE TABLE t (
    id INT,
    name VARCHAR(50),
    PRIMARY KEY (id),
    CHECK (id > 0)
);
```

这里的括号不是函数调用，而是“表定义列表”。

---

## 3. 为什么 `ALTER TABLE xxx` 后面不加括号？

因为 `ALTER TABLE` 的语法是：

```sql
ALTER TABLE 表名 动作;
```

表名后面直接跟动作关键字，比如：

- `ADD COLUMN`
- `DROP COLUMN`
- `ALTER COLUMN`
- `ADD CONSTRAINT`
- `DROP CONSTRAINT`
- `RENAME TO`

例如：

```sql
ALTER TABLE students ADD COLUMN age INT;
ALTER TABLE students DROP COLUMN age;
ALTER TABLE students ADD CONSTRAINT chk_age CHECK (age >= 0);
```

如果你写成：

```sql
ALTER TABLE students (ADD COLUMN age INT);
```

解析器会认为 `(` 不是合法动作，所以报语法错误。

但注意：**有些数据库在动作内部允许括号列表**，比如一次加多列：

```sql
-- Oracle / MySQL 常见写法
ALTER TABLE students ADD (age INT, email VARCHAR(100));
```

这里括号不是加在 `ALTER TABLE 表名` 后面，而是 `ADD` 这个动作接受一个列定义列表。  
PostgreSQL 通常写成：

```sql
ALTER TABLE students
ADD COLUMN age INT,
ADD COLUMN email VARCHAR(100);
```

所以不同数据库有差异。

---

## 4. 为什么 `CHECK` 后面要加括号？

因为 `CHECK` 的语法就是：

```sql
CHECK (条件表达式)
```

括号必须包住一个布尔条件。例如：

```sql
CHECK (age >= 0)
CHECK (status IN ('active', 'inactive'))
CHECK (end_date > start_date)
```

你不能写：

```sql
CHECK age >= 0
```

这是语法错误。

类似需要括号的还有：

```sql
IN (1, 2, 3)
VALUES (1, 'abc')
INSERT INTO t (a, b) VALUES (1, 2)
COUNT(*)
EXISTS (SELECT ...)
```

---

## 5. 什么情况加括号？什么情况不加？

简单判断：

| 情况          | 是否加括号       | 例子                                        |
| ------------- | ---------------- | ------------------------------------------- |
| 一组定义/列表 | 加               | `CREATE TABLE t (id INT, name VARCHAR(50))` |
| 列名列表      | 加               | `INSERT INTO t (a, b) VALUES (1, 2)`        |
| 值列表        | 加               | `VALUES (1, 2, 3)`                          |
| `IN` 列表     | 加               | `WHERE id IN (1, 2, 3)`                     |
| 子查询        | 加               | `WHERE id IN (SELECT id FROM t2)`           |
| 函数调用      | 加               | `COUNT(*)`、`SUM(price)`                    |
| `CHECK` 条件  | 加               | `CHECK (age >= 0)`                          |
| 动作子句      | 不加外层括号     | `ALTER TABLE t ADD COLUMN age INT`          |
| `WHERE` 条件  | 通常不加外层括号 | `WHERE age >= 0`                            |
| `FROM` 表名   | 不加             | `FROM students`                             |
| `ORDER BY` 列 | 不加             | `ORDER BY age`                              |

注意：`WHERE` 后面通常不强制加括号，但复杂表达式可以自己加：

```sql
WHERE (age >= 18 AND city = '上海') OR vip = 1
```

这不是语法必须，只是为了可读性。

---

## 6. 一句话总结

- `CREATE TABLE 表名 ( ... )`：括号里是表定义列表。
- `ALTER TABLE 表名 ADD/DROP/...`：表名后直接跟动作，不加大括号。
- `CHECK (条件)`：括号是语法要求，包住条件表达式。
- 加不加括号，看那个位置需要的是“列表/表达式/子查询”，还是“动作/子句”。  
- 不同数据库细节有差异，写的时候以对应数据库文档为准。