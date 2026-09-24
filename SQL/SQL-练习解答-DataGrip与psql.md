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

# 随堂解答：建库 / 建表 / 插入

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

# 随堂解答：查询基础

**题 8：查询所有学生**
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

**题 9：只查询姓名、年龄、专业**
```sql
SELECT
    student_name,
    age,
    major
FROM students;
```

---

**题 10：查询年龄 >= 20 的学生**
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

**题 11：查询 CS 专业且年龄 >= 20 的学生**
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

**题 12：查询年龄在 19 到 21 之间的学生**
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

**题 13：查询年龄为 19, 20, 22 的学生**
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

**题 14：查询名字包含 a 的学生**
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

**题 15：按年龄降序、姓名升序排列**
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

**题 16：查询年龄最大的 3 个学生**
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

**题 17：查询有哪些不同的专业**
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

**题 18：查询年龄未知 (NULL) 的学生**
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

# 随堂解答：表达式 / CASE / COALESCE

**题 19：查询明年年龄**
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

**题 20：使用 CASE 判断成年**
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

**题 21：处理 NULL 缺省值**
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

# 随堂解答：UPDATE / DELETE

**题 22：将 Bob 改为 CS 专业**
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

**题 23：给所有 CS 学生长一岁**
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

**题 24：激活 Eve 的账号**
```sql
UPDATE students
SET is_active = TRUE
WHERE student_name = 'Eve';
```

---

**题 25：删除 Frank 的 Calculus 选课**

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

# 随堂解答：约束错误实验

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

# 随堂解答：JOIN

**题 28：INNER JOIN 三表查询**
**题 33：查找高分成绩单**
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

**题 29：查询附加 department**
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

**题 30：LEFT JOIN 包含未选课学生**
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

# 随堂解答：聚合 / GROUP BY / HAVING

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

# 随堂解答：子查询 / EXISTS

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

# 随堂解答：集合运算

**随堂练习 45：用 UNION 得到集合**
```sql
SELECT student_name, age
FROM students
WHERE major = 'CS'

UNION

SELECT student_name, age
FROM students
WHERE age >= 21;
```
`UNION` 会自动去重。

---

**随堂练习 46：用 UNION ALL**
```sql
SELECT student_name, age
FROM students
WHERE major = 'CS'

UNION ALL

SELECT student_name, age
FROM students
WHERE age >= 21;
```
观察区别：如果某人（如 Carol，年龄 21，专业 CS）同时满足两个条件，`UNION ALL` 中会看到她的名字出现两次，而 `UNION` 只有一次。

---

**随堂练习 47：用 INTERSECT**
```sql
SELECT student_id
FROM enrollments
WHERE course_id = 1

INTERSECT

SELECT student_id
FROM enrollments
WHERE course_id = 3;
```
含义：找既存在于“选了课 1 的集合”中，又存在于“选了课 3 的集合”中的 `student_id`。

---

**随堂练习 48：用 EXCEPT 找没选课的人**
```sql
SELECT student_id
FROM students

EXCEPT

SELECT student_id
FROM enrollments;
```
意思：所有学生的 id，减去在选课表里出现过的学生 id，剩下的就是没选过课的人。

---

**随堂练习 49：EXCEPT 结合 ORDER BY**
```sql
SELECT student_id, course_id, score
FROM enrollments
WHERE score IS NOT NULL

EXCEPT

SELECT student_id, course_id, score
FROM enrollments
WHERE score < 60

ORDER BY score DESC;
```
注意：`ORDER BY` 必须写在整个语句的最末尾，作用于最终相减之后的结果。

---

# 随堂解答：INSERT ... SELECT：48

**题 50：批量生成测试数据**

```sql
INSERT INTO students (student_name, email, age, major)
SELECT
    student_name || ' Copy',
    'copy_' || email,
    age,
    major
FROM students
WHERE major = 'CS';
```

**解析**：
- `SELECT` 选出的列顺序严格对应 `INSERT INTO` 括号中的 `(student_name, email, age, major)`。
- 我们利用 `||` 拼接符，在源数据的基础上衍生出了不冲突的新名字和新邮箱。
- 自动避开了唯一约束（email）的冲突，自增 id 由数据库自动接管。

做完后，可以将其清理掉：
```sql
DELETE FROM students WHERE email LIKE 'copy_%';
```

---

# 随堂解答：CTE（WITH）：49 及其衍生练习

**题 51：用 CTE 拆解查询（单层）**

```sql
WITH course_avg AS (
    -- 第一步：专门负责计算每门课的平均分
    SELECT course_id, AVG(score) AS avg_score
    FROM enrollments
    GROUP BY course_id
)
-- 第二步：主查询直接使用上面的虚拟表 course_avg
SELECT c.course_name, ca.avg_score
FROM course_avg ca
JOIN courses c ON ca.course_id = c.course_id
WHERE ca.avg_score >= 80
ORDER BY ca.avg_score DESC;
```
**解析**：
相较于把聚合逻辑全部揉进 `HAVING` 中，使用 `WITH` 可以让你像搭积木一样，先把一个子模块组装好（`course_avg`），然后通过简单的 `JOIN` 和 `WHERE` 与别的表交互，极大地提升了复杂 SQL 的可维护性。

---

**题 52：多层 CTE 的递进调用**

我们要找出选修了 "Database Systems" 并且成绩高于该门课平均分的学生姓名。

```sql
WITH db_course AS (
    -- 1. 找出这门课的 id
    SELECT course_id 
    FROM courses 
    WHERE course_name = 'Database Systems'
),
db_avg AS (
    -- 2. 利用上一个 CTE 找出这门课的平均分
    SELECT AVG(score) AS avg_score
    FROM enrollments
    WHERE course_id = (SELECT course_id FROM db_course)
)
-- 3. 主查询：找出在这门课中得分比 db_avg 高的学生
SELECT s.student_name
FROM enrollments e
JOIN students s ON e.student_id = s.student_id
WHERE e.course_id = (SELECT course_id FROM db_course)
  AND e.score > (SELECT avg_score FROM db_avg);
```
**解析**：
通过两个 CTE `db_course` 和 `db_avg`，我们将复杂的参数计算完全抽离出来了，最后的主查询只需要用非常直白的大于号，就能明白我们要过滤什么。这展示了多个 CTE 是如何配合降低心智负担的。

---

# 随堂解答：事务（BEGIN / COMMIT / ROLLBACK）

**题 50 / 51：体验 ROLLBACK 和 COMMIT 的反悔与确认**

```sql
BEGIN; 
-- 事务开启，进入保护伞模式。接下来所有的增删改操作都不对外公开。

UPDATE students SET major = 'Math' WHERE student_name = 'Alice';

-- 此时当前窗口能查到已修改，但如果别的同事另开一个窗口查，Alice 依然是 CS
SELECT * FROM students WHERE student_name = 'Alice';

ROLLBACK; 
-- 发现改错了，大喊撤销所有修改！此时 Alice 恢复为 CS。
```
如果你把最后一句换成 `COMMIT;`，则修改正式落盘存入硬盘，变为不可撤销。
*(在 DataGrip/DBeaver 中练习时，请确保**关闭了界面上的 Auto-Commit 按钮**，或者明确执行 `BEGIN;`)*

---

**题 55：复杂的联合更改与撤销**

```sql
BEGIN;

-- 1. 删除 Frank (student_id = 6) 的选课记录
DELETE FROM enrollments WHERE student_id = 6;

-- 2. 删除 Frank 的学籍档案
DELETE FROM students WHERE student_id = 6;

-- 假如此时确认没问题，执行 COMMIT; 会永远删掉他。
-- 但题目说领导后悔了，要求恢复：
ROLLBACK;

-- 验证恢复成功：
SELECT * FROM students WHERE student_id = 6;
SELECT * FROM enrollments WHERE student_id = 6;
```
**解析**：这是一个典型的父子表级联删除操作，必须由事务包装。有了 `ROLLBACK`，即使删除了两条极其危险的数据，也可以毫发无损地回退。

---

# 随堂解答：ALTER TABLE 与 VIEW

**题 56：增加新列**
```sql
ALTER TABLE students ADD COLUMN phone VARCHAR(20);
```

**题 57：增加数据校验约束**
```sql
ALTER TABLE students 
ADD CONSTRAINT check_phone_length 
CHECK (LENGTH(phone) >= 8);
```
**解析**：此后任何人尝试 `UPDATE students SET phone = '123';` 都会直接报错，因为不满足长度 `>= 8` 的约束，保证了后续流入数据的纯洁性。

---

**题 58：创建基础数据视图**
```sql
CREATE VIEW active_students_view AS
SELECT student_id, student_name, email, major
FROM students
WHERE is_active = TRUE;
```
随后测试：`SELECT * FROM active_students_view;`

**题 59：创建多表关联大宽表视图**
```sql
CREATE VIEW student_course_details AS
SELECT 
    s.student_name, 
    s.major, 
    c.course_name, 
    e.score
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
WHERE e.score IS NOT NULL;
```
随后的测试查询就变得极其简单：
```sql
-- 以后别人想看高分榜单，只需要写这一句简单的查询，内部庞大的三表 JOIN 被完美隐藏！
SELECT * FROM student_course_details WHERE score >= 90;
```

---

# 随堂解答：索引与 EXPLAIN

**题 54 / 54.1：索引的创建与优化器的博弈**

```sql
-- 1. 创建索引
CREATE INDEX idx_students_major ON students(major);

-- 2. 观察计划
EXPLAIN SELECT * FROM students WHERE major = 'CS';
```
**深度解析题 54**：你在用 `EXPLAIN` 观察时，**极大概率依然看到的是 `Seq Scan` (全表扫描)**，并没有看到期望的 `Index Scan`。
为什么？因为优化器极其聪明。它发现 `students` 表总共才十来条数据，把这十来条数据全读出来筛选，开销远远小于“先去读一次硬盘上的索引树，再根据索引里的指针跳回原表读取”的开销。索引是为百万级数据准备的，数据太少时，优化器会主动弃用它！

```sql
-- 3. 强行干预（题 54.1）
SET enable_seqscan = OFF;
EXPLAIN SELECT * FROM students WHERE major = 'CS';
SET enable_seqscan = ON;
```
**解析**：关闭全表扫描开关后，数据库被迫走弯路，这时候你才能看到那句久违的 `Index Scan using idx_students_major on students`。这完美证明了：有索引，并不意味着它会被使用，我们要相信优化器的成本估算。

---

# 随堂解答：窗口函数进阶

**题 62：保留明细并附加整体平均分**
```sql
SELECT 
    student_id, 
    course_id, 
    score,
    AVG(score) OVER (PARTITION BY course_id) AS course_avg
FROM enrollments;
```
**解析**：
如果没有 `OVER`，单独写 `AVG(score)` 数据库会逼着你写 `GROUP BY course_id`，导致每门课最后被压成一行。用了窗口函数（并使用 `PARTITION BY course_id` 作为“隔板”），原来的个人选课明细一行没少，只是多出了一列该课的全局平均分。

**题 63：实战分组内排名**
```sql
SELECT 
    student_id, 
    course_id, 
    score,
    ROW_NUMBER() OVER (PARTITION BY course_id ORDER BY score DESC) AS rank_in_course
FROM enrollments
WHERE score IS NOT NULL;
```
**解析**：
在每门课内部（`PARTITION BY course_id`），按照成绩从高到底（`ORDER BY score DESC`）排定 1,2,3 的名次（`ROW_NUMBER()`）。这是最常用的报表语法。

**题 64：CTE 组合技 - 破解组内第一名提取难题**
```sql
-- 第 1 步：用 CTE 把带有排名的数据集准备好
WITH ranked_scores AS (
    SELECT 
        student_id, 
        course_id, 
        score,
        ROW_NUMBER() OVER (PARTITION BY course_id ORDER BY score DESC) AS rank_in_course
    FROM enrollments
    WHERE score IS NOT NULL
)
-- 第 2 步：在外层安全地使用 WHERE 进行筛选，挑出每个组的第 1 名
SELECT r.course_id, c.course_name, s.student_name, r.score
FROM ranked_scores r
JOIN students s ON r.student_id = s.student_id
JOIN courses c ON r.course_id = c.course_id
WHERE r.rank_in_course = 1;
```
**终极解析**：
千万记住那个大坑：你**绝对不能写** `WHERE ROW_NUMBER() OVER (...) = 1`。因为在 SQL 底层执行顺序里，`WHERE` 比 `OVER` 先执行，它不认识也不允许出现窗口函数。唯一的正解就是包裹在一层 CTE（或者子查询）里，把它变成固定的列后，再从外层剔除！这是面试和实战的必考题。

---

# 随堂解答：终极综合：56

**题 65：全方位实战终极报表**

这是检验你是否通透的试金石。不要放过解析里的任何一个细节！

```sql
SELECT
    s.student_name,
    s.major,
    COUNT(e.course_id) AS course_count,
    AVG(e.score) AS avg_score
FROM students s
LEFT JOIN enrollments e 
    ON s.student_id = e.student_id
GROUP BY
    s.student_id,
    s.student_name,
    s.major
ORDER BY
    avg_score DESC NULLS LAST,
    s.student_name ASC;
```

**终极深度解析 5 大连环坑**：
1. **为什么要用 `LEFT JOIN`？** 题目要求没选课的人（如 David）也要出现。如果你用默认的 `JOIN` (INNER JOIN)，连不上 `enrollments` 的学生会被瞬间丢弃抛弃。
2. **为什么写 `COUNT(e.course_id)` 而不是 `COUNT(*)`？** 对于没选课的 David，他连上宽表后，右边补充的都是 NULL。如果你用 `COUNT(*)`，数据库会认为 David 所在的这一行“客观存在”，无脑返回 1 门课；而写 `COUNT(列名)`，聚合函数会自动过滤 NULL 列，正确地为他返回 0 门课！
3. **`AVG()` 会不会把未出分（成绩是 NULL）算作 0 分拉低平均值？** 放心，不会。标准 SQL 的 `AVG` 原生就会自动跳过 NULL 进行计算。
4. **`GROUP BY` 里的安全守则**：虽然业务上是按“人”分组，但严谨起见，不仅要放 `student_id`，也要把 `SELECT` 里所有**没被包裹在聚合函数里的字段**（如 `student_name`, `major`）统统放在 `GROUP BY` 里，否则在大多数数据库下都会报错。
5. **神来之笔 `NULLS LAST`**：在 PostgreSQL 中，你对含 NULL 的列进行排序时，可以指定 `NULLS FIRST` 还是 `NULLS LAST`。如果不写，当你 `ORDER BY avg_score DESC` 降序时，那些 0 门课、平均分为 NULL 的人会诡异地排在满分 100 分的大神头上！加上 `NULLS LAST` 完美压制了乱跳的空值。

如果你能**完全不看答案、独立理清上述逻辑**，自己一口气写出了这段代码并运行成功……那么热烈恭喜你！你的 SQL 基础已经比很多开发初学者还要扎实透彻，完全具备应对日常工作报表的强悍战斗力！
