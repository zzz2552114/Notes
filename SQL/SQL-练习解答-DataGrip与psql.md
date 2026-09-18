# PostgreSQL / SQL 贯穿练习解答

## 使用方式

这份解答与知识文件采用完全相同的学习方式：

```text
知识点
  ↓
随堂练习
  ↓
自己写
  ↓
执行
  ↓
最后对答案
```

所以这里不是把 56 道题集中成一堆答案，而是按知识阶段分组；每组里都有对应题目的完整解答。

## DataGrip 与 psql

### DataGrip

主要用：

- PostgreSQL Data Source
- Query Console
- Database 工具窗口

SQL 本身不因为 DataGrip 而改变。学习的核心仍然是自己写 SQL。

事务练习时注意 DataGrip 的 Auto-commit / Transaction 控制。

### psql

进入：

```bash
psql -U postgres
```

连接数据库：

```sql
\c sql_learning
```

查看表：

```sql
\dt
```

查看结构：

```sql
\d students
\d courses
\d enrollments
```

退出：

```sql
\q
```

这些 `\c / \dt / \d / \q` 是 psql 客户端命令，不属于标准 SQL。

---

# 随堂解答：建库 / 建表 / 插入：1~7

**题 1：创建数据库**

## 思路

创建数据库属于 PostgreSQL 的数据库管理层面，不是你后面每天都写的查询语法。

## DataGrip

可以：

- Database 工具窗口 → 连接 → 右键 → New → Database
- 名称：

```text
sql_learning
```

也可以直接打开一个 PostgreSQL console 使用 PostgreSQL 的：

```sql
CREATE DATABASE sql_learning;
```

然后把 Query Console 切到 `sql_learning`。

## psql

```sql
CREATE DATABASE sql_learning;
\c sql_learning
```

## 检查

```sql
SELECT current_database();
```

预期：

```text
sql_learning
```

---

**题 2：创建 students**

## 思路

先把“列名 + 类型 + 约束”逐列写出来。

## 标准核心 SQL

```sql
CREATE TABLE students (
    student_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    student_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    age INTEGER CHECK (age >= 0),
    major VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 这里真正需要理解的语法

```text
student_id
    列名

INTEGER
    类型

GENERATED ALWAYS AS IDENTITY
    自动生成编号

PRIMARY KEY
    主键

NOT NULL
    不允许 NULL

UNIQUE
    不允许重复

CHECK
    数据必须满足条件

DEFAULT
    不提供值时使用默认值
```

---

**题 3：创建 courses**

```sql
CREATE TABLE courses (
    course_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    course_name VARCHAR(100) NOT NULL,
    department VARCHAR(50) NOT NULL,
    credits INTEGER NOT NULL CHECK (credits > 0)
);
```

检查：

```sql
SELECT *
FROM courses;
```

刚创建时应该是 0 行。

---

**题 4：创建 enrollments**

```sql
CREATE TABLE enrollments (
    student_id INTEGER NOT NULL,
    course_id INTEGER NOT NULL,
    score NUMERIC(5,2),
    enrolled_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (student_id, course_id),

    FOREIGN KEY (student_id)
        REFERENCES students(student_id),

    FOREIGN KEY (course_id)
        REFERENCES courses(course_id),

    CHECK (
        score IS NULL
        OR (score >= 0 AND score <= 100)
    )
);
```

## 为什么主键写两个列？

因为：

```text
(student_id, course_id)
```

必须唯一。

例如：

```text
Alice + Database
```

只能出现一次。

但：

```text
Alice + Database
Alice + OS
```

可以存在。

---

**题 5：插入学生**

```sql
INSERT INTO students
    (student_name, email, age, major, is_active)
VALUES
    ('Alice', 'alice@example.com', 20, 'CS', TRUE),
    ('Bob', 'bob@example.com', 19, 'Math', TRUE),
    ('Carol', 'carol@example.com', 21, 'CS', TRUE),
    ('David', 'david@example.com', NULL, 'Physics', TRUE),
    ('Eve', 'eve@example.com', 22, 'CS', FALSE),
    ('Frank', 'frank@example.com', 20, 'Math', TRUE),
    ('Grace', 'grace@example.com', 23, 'Biology', TRUE);
```

这里刻意没有插：

```text
student_id
created_at
```

因为：

- `student_id` 自动生成；
- `created_at` 有默认值。

检查：

```sql
SELECT *
FROM students
ORDER BY student_id;
```

---

**题 6：插入课程**

```sql
INSERT INTO courses
    (course_name, department, credits)
VALUES
    ('Database Systems', 'CS', 4),
    ('Operating Systems', 'CS', 4),
    ('Algorithms', 'CS', 3),
    ('Calculus', 'Math', 4),
    ('Linear Algebra', 'Math', 3);
```

检查：

```sql
SELECT *
FROM courses
ORDER BY course_id;
```

---

**题 7：插入选课关系**

题目要求你不要猜 id。

先查询：

```sql
SELECT student_id, student_name
FROM students
ORDER BY student_id;

SELECT course_id, course_name
FROM courses
ORDER BY course_id;
```

如果本次数据库刚创建，通常会得到：

```text
students
1 Alice
2 Bob
3 Carol
4 David
5 Eve
6 Frank
```

以及：

```text
courses
1 Database Systems
2 Operating Systems
3 Algorithms
4 Calculus
5 Linear Algebra
```

然后：

```sql
INSERT INTO enrollments
    (student_id, course_id, score)
VALUES
    (1, 1, 91),
    (1, 2, 88),
    (1, 3, 95),
    (2, 4, 78),
    (2, 5, 85),
    (3, 1, 84),
    (3, 3, NULL),
    (4, 4, 59),
    (5, 2, 97),
    (6, 5, 72),
    (6, 4, 81);
```

检查：

```sql
SELECT *
FROM enrollments
ORDER BY student_id, course_id;
```

如果你之前已经插过数据，**不要盲目重新执行整段**，否则可能触发组合主键冲突。

---

# 随堂解答：查询基础：8~18

```sql
SELECT *
FROM students;
```

核心理解：

```text
SELECT * → 我要所有列
FROM students → 从 students 找
```

---

```sql
SELECT
    student_name,
    age,
    major
FROM students;
```

---

```sql
SELECT *
FROM students
WHERE age >= 20;
```

注意：

`David.age` 是 NULL。

NULL 不会满足：

```text
age >= 20
```

因此 David 不会出现在结果中。

---

```sql
SELECT *
FROM students
WHERE major = 'CS'
  AND age >= 20;
```

为什么使用：

```sql
AND
```

而不是把两个条件分成两个 WHERE？

因为一条 SELECT 只能有一个 WHERE 子句：

```sql
WHERE 条件1 AND 条件2
```

---

```sql
SELECT *
FROM students
WHERE age BETWEEN 19 AND 21;
```

等价思维：

```sql
WHERE age >= 19
  AND age <= 21
```

---

```sql
SELECT *
FROM students
WHERE age IN (19, 20, 22);
```

不要写成：

```sql
WHERE age = 19 OR 20 OR 22
```

那不是你想表达的 SQL 逻辑。

---

```sql
SELECT *
FROM students
WHERE student_name LIKE '%a%';
```

这里：

```text
% → 任意长度字符
```

所以：

```text
%a%
```

就是：

> a 前面可以有任何东西，a 后面也可以有任何东西。

---

```sql
SELECT *
FROM students
ORDER BY
    age DESC,
    student_name ASC;
```

理解：

1. 先比较 age；
2. age 相同再比较名字。

---

```sql
SELECT *
FROM students
ORDER BY age DESC
LIMIT 3;
```

如果要分页：

```sql
LIMIT 3 OFFSET 3
```

就是从排序结果跳过前 3 条，再取 3 条。

---

```sql
SELECT DISTINCT major
FROM students;
```

如果你特别想把：

```text
NULL
```

也当成一种“结果项”，DISTINCT 会保留一个 NULL 结果。

---

```sql
SELECT *
FROM students
WHERE age IS NULL;
```

不是：

```sql
WHERE age = NULL;
```

---

# 随堂解答：表达式 / CASE / COALESCE：19~21

```sql
SELECT
    student_name,
    age,
    age + 1 AS age_next_year
FROM students;
```

这里：

```text
age + 1
```

是表达式。

```text
AS age_next_year
```

是给这个表达式的结果起名字。

注意：如果 age 是 NULL，那么：

```text
NULL + 1
```

仍然是 NULL。

---

```sql
SELECT
    student_name,
    CASE
        WHEN age >= 18 THEN 'adult'
        ELSE 'minor'
    END AS age_group
FROM students;
```

David 的 age 为 NULL：

```text
age >= 18
```

不是 TRUE，所以进入 ELSE：

```text
minor
```

这正好让你观察 NULL 与 CASE 的关系。

---

```sql
SELECT
    student_name,
    COALESCE(major, 'Unknown') AS major_display
FROM students;
```

David：

```text
major = Physics
```

不会替换。

如果某个学生 major 为 NULL，显示：

```text
Unknown
```

---

# 随堂解答：UPDATE / DELETE：22~25

```sql
UPDATE students
SET major = 'CS'
WHERE student_name = 'Bob';
```

强烈建议：

先执行：

```sql
SELECT *
FROM students
WHERE student_name = 'Bob';
```

确认对象以后再 UPDATE。

---

先看范围：

```sql
SELECT student_id, student_name, age
FROM students
WHERE major = 'CS';
```

再：

```sql
UPDATE students
SET age = age + 1
WHERE major = 'CS';
```

这个题故意让你体会：

```text
SELECT 可以先验证 WHERE
UPDATE 再真正修改
```

这会成为你以后的好习惯。

---

```sql
UPDATE students
SET is_active = TRUE
WHERE student_name = 'Eve';
```

---

这一题先用“学到哪儿，就用哪儿”的方法做：分别查出两个 id。

```sql
SELECT student_id
FROM students
WHERE student_name = 'Frank';

SELECT course_id
FROM courses
WHERE course_name = 'Calculus';
```

假设查到 `student_id = 6`、`course_id = 4`，先用它确认目标行：

```sql
SELECT *
FROM enrollments
WHERE student_id = 6
  AND course_id = 4;
```

确认只有目标行后，再删除：

```sql
DELETE FROM enrollments
WHERE student_id = 6
  AND course_id = 4;
```

这样只用到了 `SELECT / WHERE / DELETE`，完全在已学范围内。

> 等以后学了 JOIN 和子查询，还可以一步到位：
>
> ```sql
> DELETE FROM enrollments
> WHERE student_id = (SELECT student_id FROM students WHERE student_name = 'Frank')
>   AND course_id  = (SELECT course_id  FROM courses   WHERE course_name = 'Calculus');
> ```
>
> 但名字如果不唯一，这种写法就不够安全；当前练习数据里它们唯一。初学阶段更推荐上面的“先查 id、再删”两步法。

---

# 随堂解答：约束错误实验：26~27

### 1. UNIQUE（题 26）

```sql
INSERT INTO students
    (student_name, email, age, major)
VALUES
    ('Alice2', 'alice@example.com', 20, 'CS');
```

预期：

> 失败。

原因：

```text
email 有 UNIQUE 约束
```

这正是在观察：

> 数据库不是只“存数据”，还会执行规则。

### 2. CHECK（题 27.1）

```sql
UPDATE enrollments
SET score = 120
WHERE student_id = 1
  AND course_id = 1;
```

预期：

> 失败。

因为：

```sql
CHECK (
    score IS NULL
    OR (score >= 0 AND score <= 100)
)
```

120 不满足条件。

### 3. FOREIGN KEY（题 27.2）

```sql
INSERT INTO enrollments
    (student_id, course_id, score)
VALUES
    (9999, 1, 80);
```

预期：

> 失败。

因为：

```sql
FOREIGN KEY (student_id)
REFERENCES students(student_id)
```

`9999` 这个学生在 `students` 里并不存在，外键不允许引用一个不存在的行。

---

# 随堂解答：JOIN：28~33

```sql
SELECT
    s.student_name,
    c.course_name,
    e.score
FROM enrollments AS e
JOIN students AS s
    ON s.student_id = e.student_id
JOIN courses AS c
    ON c.course_id = e.course_id;
```

这里可以看出：

```text
enrollments
    是“连接桥”
```

它把：

```text
students
courses
```

连接起来。

---

```sql
SELECT
    s.student_name,
    c.course_name,
    c.department,
    e.score
FROM enrollments AS e
JOIN students AS s
    ON s.student_id = e.student_id
JOIN courses AS c
    ON c.course_id = e.course_id;
```

---

```sql
SELECT
    s.student_name,
    c.course_name,
    e.score
FROM students AS s
LEFT JOIN enrollments AS e
    ON e.student_id = s.student_id
LEFT JOIN courses AS c
    ON c.course_id = e.course_id
ORDER BY s.student_id, c.course_id;
```

关键：

```sql
FROM students
LEFT JOIN ...
```

因为：

> students 是你想“全保留”的左表。

---

```sql
SELECT
    s.student_id,
    s.student_name
FROM students AS s
LEFT JOIN enrollments AS e
    ON e.student_id = s.student_id
WHERE e.student_id IS NULL;
```

逻辑：

1. 先把所有学生保留；
2. 没有匹配到 enrollments 的人，右边字段为 NULL；
3. 找 NULL。

这是 `LEFT JOIN + IS NULL` 的经典组合。

---

直接 JOIN：

```sql
SELECT DISTINCT
    s.student_name
FROM students AS s
JOIN enrollments AS e
    ON e.student_id = s.student_id
JOIN courses AS c
    ON c.course_id = e.course_id
WHERE c.course_name = 'Database Systems';
```

这里加：

```sql
DISTINCT
```

是一种保险写法，表示结果只需要名字集合。

---

```sql
SELECT
    s.student_name,
    c.course_name,
    e.score
FROM enrollments AS e
JOIN students AS s
    ON s.student_id = e.student_id
JOIN courses AS c
    ON c.course_id = e.course_id
WHERE e.score >= 90
ORDER BY e.score DESC;
```

---

# 随堂解答：聚合 / GROUP BY / HAVING：34~40

```sql
SELECT COUNT(*) AS student_count
FROM students;
```

当前原始数据为 7 人；如果你之前做了题目 22~24，人数不会变化。

---

```sql
SELECT
    c.course_name,
    COUNT(e.student_id) AS student_count
FROM courses AS c
LEFT JOIN enrollments AS e
    ON e.course_id = c.course_id
GROUP BY
    c.course_id,
    c.course_name
ORDER BY c.course_id;
```

为什么用：

```text
LEFT JOIN
```

而不是 INNER JOIN？

因为题目说：

> 每门课程。

即使某门课程没有任何学生，也应该能显示出来。

为什么：

```text
COUNT(e.student_id)
```

而不是：

```text
COUNT(*)
```

因为 LEFT JOIN 后，没有匹配选课的课程仍然会有一行，但 `e.student_id` 是 NULL。

---

```sql
SELECT
    major,
    COUNT(*) AS student_count
FROM students
GROUP BY major
ORDER BY student_count DESC;
```

---

```sql
SELECT
    c.course_name,
    AVG(e.score) AS avg_score
FROM courses AS c
JOIN enrollments AS e
    ON e.course_id = c.course_id
GROUP BY
    c.course_id,
    c.course_name
ORDER BY avg_score DESC;
```

`AVG` 不会把 NULL 成绩当成 0。

例如：

```text
84
NULL
```

平均值是：

```text
84
```

而不是：

```text
42
```

---

```sql
SELECT
    c.course_name,
    AVG(e.score) AS avg_score
FROM courses AS c
JOIN enrollments AS e
    ON e.course_id = c.course_id
GROUP BY
    c.course_id,
    c.course_name
HAVING AVG(e.score) >= 80
ORDER BY avg_score DESC;
```

注意：

这里必须用：

```sql
HAVING
```

而不是：

```sql
WHERE
```

因为：

```text
AVG(e.score)
```

是在 GROUP BY 之后形成的组级结果。

---

```sql
SELECT
    s.student_name,
    COUNT(e.course_id) AS course_count
FROM students AS s
LEFT JOIN enrollments AS e
    ON e.student_id = s.student_id
GROUP BY
    s.student_id,
    s.student_name
ORDER BY course_count DESC;
```

依然用 LEFT JOIN，是为了保留：

> 0 门课的学生。

---

```sql
SELECT
    s.student_name,
    COUNT(e.course_id) AS course_count
FROM students AS s
JOIN enrollments AS e
    ON e.student_id = s.student_id
GROUP BY
    s.student_id,
    s.student_name
HAVING COUNT(e.course_id) >= 2
ORDER BY course_count DESC;
```

---

# 随堂解答：子查询 / EXISTS：41~44

```sql
SELECT
    e.student_id,
    e.course_id,
    e.score
FROM enrollments AS e
WHERE e.score > (
    SELECT AVG(score)
    FROM enrollments
);
```

这就是：

```text
外层：查每条成绩
内层：算一个整体平均值
外层拿每条成绩和这个值比较
```

注意：

子查询返回的是**一个值**。

---

```sql
SELECT
    s.student_name
FROM students AS s
WHERE EXISTS (
    SELECT 1
    FROM enrollments AS e
    WHERE e.student_id = s.student_id
)
ORDER BY s.student_id;
```

逻辑：

> 对每个学生，数据库检查 enrollments 中是否存在至少一行属于他。

---

```sql
SELECT
    s.student_name
FROM students AS s
WHERE NOT EXISTS (
    SELECT 1
    FROM enrollments AS e
    WHERE e.student_id = s.student_id
);
```

逻辑：

> 不存在任何他的选课记录。

这比硬记“NOT EXISTS 是高级语法”更重要。

---

```sql
SELECT
    student_name
FROM students
WHERE student_id IN (
    SELECT e.student_id
    FROM enrollments AS e
    JOIN courses AS c
        ON c.course_id = e.course_id
    WHERE c.course_name = 'Database Systems'
);
```

这里的子查询返回：

```text
一列 student_id
```

所以外层可以：

```text
student_id IN (...)
```

---

# 随堂解答：集合运算：45~47

```sql
SELECT student_name
FROM students
WHERE major = 'CS'

UNION

SELECT student_name
FROM students
WHERE age >= 21;
```

`UNION` 自动去重。

如果你写：

```sql
UNION ALL
```

就不去重。

---

```sql
SELECT student_name
FROM students
WHERE major = 'CS'

INTERSECT

SELECT student_name
FROM students
WHERE age >= 20;
```

含义：

> 两个结果集的交集。

---

```sql
SELECT student_name
FROM students

EXCEPT

SELECT s.student_name
FROM students AS s
JOIN enrollments AS e
    ON e.student_id = s.student_id;
```

意思：

> 所有学生 - 选过课的学生

得到没选课的人。

---

# 随堂解答：INSERT ... SELECT：48

题目要把 CS 学生复制一份，姓名加 ` Copy`、邮箱加前缀 `copy_`。这里正好用上前面的字符串拼接 `||`。

```sql
INSERT INTO students
    (student_name, email, age, major)
SELECT
    student_name || ' Copy',
    'copy_' || email,
    age,
    major
FROM students
WHERE major = 'CS';
```

注意：

- `SELECT` 出来的列顺序，要和 `INSERT INTO students (...)` 里列的顺序一一对应，类型也要兼容。
- 没有写 `is_active`，所以用它的 `DEFAULT TRUE`；没有写 `created_at`，所以用 `DEFAULT CURRENT_TIMESTAMP`；`student_id` 是 `GENERATED ... AS IDENTITY`，自动生成。
- `WHERE major = 'CS'` 决定复制哪些行。

执行完先看一眼：

```sql
SELECT *
FROM students
WHERE student_name LIKE '% Copy'
ORDER BY student_id;
```

观察完后删掉这些临时数据：

```sql
DELETE FROM students
WHERE student_name LIKE '% Copy';
```

结构上要记住的是：`VALUES` 和 `SELECT` 都可以作为 `INSERT` 的数据来源。

```sql
INSERT INTO table (...)
VALUES (...);
```

是“直接给值”；

```sql
INSERT INTO table (...)
SELECT ...
FROM ...;
```

是“把查询结果塞进去”。

---

# 随堂解答：CTE：49

```sql
WITH course_avg AS (
    SELECT
        course_id,
        AVG(score) AS avg_score
    FROM enrollments
    GROUP BY course_id
)
SELECT
    c.course_name,
    ca.avg_score
FROM course_avg AS ca
JOIN courses AS c
    ON c.course_id = ca.course_id
WHERE ca.avg_score >= 80
ORDER BY ca.avg_score DESC;
```

理解：

```text
course_avg
    先算出“每门课的平均分”，并给它起个名字

外层 SELECT
    直接把 course_avg 当成一张表来查
```

这和不用 CTE 的写法结果一样：

```sql
SELECT
    c.course_name,
    AVG(e.score) AS avg_score
FROM courses AS c
JOIN enrollments AS e
    ON e.course_id = c.course_id
GROUP BY c.course_id, c.course_name
HAVING AVG(e.score) >= 80
ORDER BY avg_score DESC;
```

区别只是“分步骤写”还是“一口气写”。查询越长，`WITH` 越能让每一步说得清楚。

---

# 随堂解答：事务：50~51

## DataGrip

打开 Transaction 控制方式后，执行：

```sql
BEGIN;

UPDATE students
SET major = 'Math'
WHERE student_name = 'Alice';

SELECT *
FROM students
WHERE student_name = 'Alice';

ROLLBACK;
```

再：

```sql
SELECT *
FROM students
WHERE student_name = 'Alice';
```

应该恢复到修改前。

### 如果 DataGrip 看起来没有回滚

检查当前 Data Source / Console 的事务模式。

有些 DataGrip 配置可能使用自动提交（auto-commit）。为了学习事务，确保当前操作确实在同一事务中，并观察 Transaction 控件。

---

```sql
BEGIN;

UPDATE students
SET major = 'Math'
WHERE student_name = 'Alice';

COMMIT;
```

然后：

```sql
SELECT *
FROM students
WHERE student_name = 'Alice';
```

修改仍然存在。

---

# 随堂解答：ALTER / VIEW / INDEX：52~54

```sql
ALTER TABLE students
ADD COLUMN phone VARCHAR(20);
```

检查：

```sql
\d students
```

psql 中非常直观。

DataGrip 中可以从表结构面板看到新增列，也可以：

```sql
SELECT *
FROM students;
```

确认。

---

```sql
CREATE VIEW active_students AS
SELECT
    student_id,
    student_name,
    email,
    age,
    major
FROM students
WHERE is_active = TRUE;
```

查询：

```sql
SELECT *
FROM active_students;
```

理解成：

```text
视图 = 被命名的查询
```

初学先不要把它当作“复制出来的一张独立表”。

---

```sql
CREATE INDEX idx_students_major
ON students(major);
```

检查：

```sql
```

在 PostgreSQL 中可以结合：

```sql
EXPLAIN
SELECT *
FROM students
WHERE major = 'CS';
```

但注意：

> 小表上，即使你建了索引，也不代表 PostgreSQL 一定选择索引扫描。

优化器会根据数据量、选择性、成本估算等决定访问路径。

所以：

```text
有索引
≠
一定使用索引
```

---

# 随堂解答：窗口函数：55

要求：

> 每条选课记录继续保留，同时显示这门课的平均分。

答案：

```sql
SELECT
    e.student_id,
    e.course_id,
    e.score,
    AVG(e.score) OVER (
        PARTITION BY e.course_id
    ) AS course_avg
FROM enrollments AS e
ORDER BY e.course_id, e.student_id;
```

理解：

```text
GROUP BY
    每门课变成一行

窗口函数
    每条选课记录都保留
    只是额外算一个“这门课平均分”
```

这就是窗口函数最核心的感觉。

---

# 随堂解答：终极综合：56

要求：

```text
学生姓名
专业
选课数量
平均成绩
```

且：

```text
没选课的学生也要出现
NULL 成绩不能当成 0
平均分从高到低
平均分相同按名字升序
0 门课的人平均分显示 NULL
```

答案：

```sql
SELECT
    s.student_name,
    s.major,
    COUNT(e.course_id) AS course_count,
    AVG(e.score) AS avg_score
FROM students AS s
LEFT JOIN enrollments AS e
    ON e.student_id = s.student_id
GROUP BY
    s.student_id,
    s.student_name,
    s.major
ORDER BY
    avg_score DESC NULLS LAST,
    s.student_name ASC;
```

如果你希望主线尽量不依赖 PostgreSQL 对 NULL 排序位置的扩展写法，可以先写：

```sql
ORDER BY
    avg_score DESC,
    s.student_name ASC;
```

然后再单独学习：

```sql
NULLS LAST
```

这是 PostgreSQL 很方便的排序控制功能。

---

# 附：题 50 / 51 的 psql 写法

```sql
BEGIN;

UPDATE students
SET major = 'Math'
WHERE student_name = 'Alice';

SELECT *
FROM students
WHERE student_name = 'Alice';

ROLLBACK;
```

或者：

```sql
BEGIN;

UPDATE students
SET major = 'Math'
WHERE student_name = 'Alice';

COMMIT;
```

在 psql 中，事务完全由 SQL 控制。

---

# DataGrip / psql 小抄

## DataGrip

```text
Data Source
   ↓
Database
   ↓
Query Console
   ↓
执行 SQL
   ↓
观察结果
```

你可以把每阶段 SQL 保存成：

```text
01_create.sql
02_insert.sql
03_select.sql
04_update_delete.sql
05_join.sql
06_group.sql
07_subquery.sql
08_set_operation.sql
09_transaction.sql
10_view_index_window.sql
```

## psql

执行一个脚本：

```sql
\i 01_create.sql
```

或者在 shell：

```bash
psql -U postgres -d sql_learning -f 01_create.sql
```

普通 SQL：

```sql
SELECT ...
INSERT ...
UPDATE ...
DELETE ...
BEGIN;
COMMIT;
ROLLBACK;
```

在 DataGrip 与 psql 中都是同一套 SQL。
