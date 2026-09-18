# PostgreSQL / SQL 初学者通透版：从“看不懂 SQL”到能自己写

你现在觉得 SQL 难记，通常不是因为你记忆力不行，而是因为你在记“单词”，而不是在记“结构”。

SQL 真正需要先建立的是下面这些基础。

# 第一部分：先建立“骨架感”

## 1. SQL 是“固定骨架 + 填空”

最常见的查询其实就是：

```sql
SELECT   我要看什么
FROM     从哪里拿数据
WHERE    哪些行不要
GROUP BY 怎么分组
HAVING   哪些组不要
ORDER BY 怎么排
```

再把多张表接起来：

```sql
SELECT ...
FROM table_a
JOIN table_b ON ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
```

> `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ...`

==

> **我要看什么 → 从哪里找 → 先筛掉哪些行 → 要不要分组 → 分完组后再筛哪些组 → 最后怎么排。**

这才是 SQL 的“骨架记忆法”。

---

| 类别 | 作用 | 先掌握 |
|---|---|---|
| DDL | 定义结构 | `CREATE` `ALTER` `DROP` |
| DML | 改数据 | `INSERT` `UPDATE` `DELETE` |
| DQL | 查数据 | `SELECT` |
| TCL | 事务 | `BEGIN` `COMMIT` `ROLLBACK` |
| DCL | 权限 | `GRANT` `REVOKE` |

初学阶段最重要的是：

```text
CREATE TABLE
INSERT
SELECT
UPDATE
DELETE
```

然后再进入：

```text
WHERE
ORDER BY
GROUP BY
HAVING
JOIN
子查询
CTE
集合运算
事务
```

索引、视图、权限这些属于“知道用途 + 会基本语法”，不应该和 `SELECT/WHERE/JOIN` 一起死记。

---

# 第二部分：书写习惯——大小写、引号、分号、注释

## 1. 关键字大小写不影响含义

这些基本等价：

```sql
SELECT * FROM students;
select * from students;
SeLeCt * FrOm students;
```

建议统一：

```sql
SELECT student_name, age
FROM students
WHERE age >= 18;
```

也就是：

- SQL 关键字大写
- 表名、列名小写
- 多词名字用下划线

例如：

```text
student_name
course_id
enrolled_at
```

这样最好读。

---

## 2. 字符串用单引号

字符串字面量：

```sql
SELECT 'Alice';
SELECT 'hello';
SELECT 'CSAPP';
```

字符串里的大小写是内容的一部分：

```sql
SELECT 'Alice' = 'alice';
```

结果不是 TRUE。

所以：

```sql
'Alice'
'alice'
```

是两个不同的字符串。

---

## 3. 单引号里想写单引号怎么办？

标准 SQL 的写法是：**单引号写两个单引号**。

```sql
SELECT 'Bob''s book';
```

它表示：

```text
Bob's book
```

这一点非常值得记住。

---

## 4. 双引号不是字符串

双引号通常表示“标识符”。

```sql
SELECT "student_name"
FROM students;
```

这里 `"student_name"` 是列名。

而：

```sql
SELECT 'student_name';
```

这里 `'student_name'` 是一个字符串。

一个非常实用的记忆法：

```text
'...'  → 数据里的文字
"..."  → 表名、列名等名字
```

---

## 5. 为什么初学最好不要乱用双引号命名？

在 PostgreSQL 中，不加双引号的标识符会按规则折叠为小写。

```sql
CREATE TABLE Users (
    ID integer,
    UserName varchar(50)
);
```

实际使用时可以写：

```sql
SELECT id, username
FROM users;
```

因为未加引号的名字会按 PostgreSQL 的规则处理成小写。

但如果你刻意写：

```sql
CREATE TABLE "Users" (
    "ID" integer,
    "UserName" varchar(50)
);
```

那这些名字就是精确区分大小写的。

之后必须：

```sql
SELECT "ID", "UserName"
FROM "Users";
```

**初学建议：**

```text
全小写 + 下划线
```

最省脑子。

---

## 1. 分号

一条 SQL 语句通常以分号结束。

例如：

```sql
SELECT * FROM students;
```

注意：

- 在 DataGrip 的 SQL Console 里，通常一条语句以分号结尾最清楚。
- 在 `psql` 交互环境里，分号表示一条 SQL 语句结束并执行。

---

## 2. 注释

单行：

```sql
-- 这是注释
SELECT * FROM students;
```

多行：

```sql
/*
这是多行注释
可以写很多行
*/
SELECT * FROM students;
```

---

# 第三部分：值、列、表达式与运算符

## 1. 一个值

例如：

```text
18
'Alice'
TRUE
3.14
NULL
```

这些都是值。

---

## 2. 一列

```sql
age
student_name
score
```

表示从当前这一行取对应的列值。

---

## 3. 表达式

表达式就是“算出一个值的东西”。

例如：

```sql
age + 1
score * 0.9
first_name
age >= 18
```

因此下面完全正常：

```sql
SELECT
    student_name,
    age,
    age + 1 AS next_age
FROM students;
```

你可以把它想成：

```text
SELECT
    这一行的 student_name
    这一行的 age
    用 age 算出一个新值
FROM students;
```



---

## 1. 算术运算

```text
+   加
-   减
*   乘
/   除
```

例如：

```sql
SELECT
    score,
    score + 5 AS bonus_score,
    score * 1.1 AS scaled_score
FROM enrollments;
```

**所以不要把 SQL 里的所有 `*` 都理解成“通配符”。**

---

## 字符串拼接：||

数值可以相加，字符串可以拼接。标准 SQL 用 `||`：

```sql
SELECT
    student_name || ' Copy' AS copy_name,
    'copy_' || email AS copy_email
FROM students;
```

PostgreSQL 里还可以用函数写法：

```sql
CONCAT('copy_', email)
```

`||` 读作“接起来”，在 SQL 里就是字符串拼接（注意别和某些编程语言里的“逻辑或”混淆；SQL 的逻辑或是 `OR`）。第 48 题复制学生数据时会用到它。

---

## 2. 比较运算

```text
=
<>
!=
<
>
<=
>=
```

标准 SQL 最值得记的是：

```sql
<>
```

它表示“不等于”。

有些数据库也接受 `!=`，PostgreSQL 也接受，但写可迁移 SQL 时，`<>` 更值得认识。

---

## 3. 逻辑运算

```text
AND
OR
NOT
```

---

## 4. 运算优先级：不确定就加括号

例如：

```sql
WHERE age >= 18
  AND score >= 60
  OR is_repeat = TRUE
```

人脑很容易误读。

初学阶段直接养成习惯：

```sql
WHERE
    (age >= 18 AND score >= 60)
    OR is_repeat = TRUE
```

SQL 写得清楚，比省两个括号重要得多。

---

# 第四部分：NULL 与三值逻辑

`NULL` 不是：

```text
0
''
FALSE
```

## 1. 不要写：

```sql
WHERE age = NULL;
```

也不要写：

```sql
WHERE age <> NULL;
```

因为 NULL 参与普通比较时，会进入 SQL 的 UNKNOWN（未知）结果。

因此：

```sql
NULL = NULL
```

不是 TRUE。

---

## 2. 判断 NULL 用 IS NULL

```sql
WHERE age IS NULL
```

不为空：

```sql
WHERE age IS NOT NULL
```

记忆：

```text
NULL → IS
```

---

## 3. 为什么 SQL 多了 UNKNOWN？

普通逻辑里你会习惯：

```text
TRUE / FALSE
```

SQL 里还存在：

```text
UNKNOWN
```

例如：

```text
年龄 = NULL
```

数据库无法判断：

> 这个未知年龄是不是 18？

所以不是 TRUE，也不是 FALSE，而是 UNKNOWN。

`WHERE` 最终只保留条件为 TRUE 的行。

这就是为什么：

```sql
WHERE age = NULL
```

查不到“年龄未知”的人。

### 注意！！！

任何值和 `NULL` 做比较，结果通常都是 UNKNOWN：

```sql
NULL >= 0      -- UNKNOWN
NULL = 1       -- UNKNOWN
NULL <> 1      -- UNKNOWN
```

而 `CHECK` 约束的规则是：

> 只有结果为 **FALSE** 时才违反约束；
> 结果为 **TRUE** 或 **UNKNOWN** 都算通过。

所以你写：

```sql
CHECK (age >= 0)
```

实际效果是：

| age 值 | `age >= 0` 结果 | CHECK |
| :----- | :-------------- | :---- |
| 10     | TRUE            | 通过  |
| -5     | FALSE           | 拒绝  |
| NULL   | UNKNOWN         | 通过  |

也就是说，`CHECK (age >= 0)` 本身就已经允许 `age` 为 `NULL` 了，不需要额外写 `NULL OR`。

---

## 4. COALESCE：给 NULL 一个替代值

这是非常值得早学的标准 SQL 功能：

```sql
COALESCE(age, 0)
```

意思：

> 如果 `age` 不是 NULL，就用 age；否则用 0。

例：

```sql
SELECT
    student_name,
    COALESCE(age, 0) AS display_age
FROM students;
```

记忆：

```text
COALESCE = 从左往右找第一个不是 NULL 的值
```

例如：

```sql
COALESCE(NULL, NULL, 5, 10)
```

结果是：

```text
5
```

---

SQL 不只有数字和字符串。

本练习主线中会用：

```text
INTEGER
VARCHAR(50)
NUMERIC(5,2)
DATE
TIMESTAMP
BOOLEAN
```

---

# 第五部分：数据类型

## 1. INTEGER

```sql
age INTEGER -- 可以写作 int4 ，表示4字节，同理有 int2 和 int8
```

---

## 2. VARCHAR(n)

```sql
student_name VARCHAR(50)
```

表示最多 50 个字符的变长字符串。

---

## 3. NUMERIC(p, s)

例如：

```sql
score NUMERIC(5,2)
```

可以理解成：

```text
最多 5 位有效数字
其中 2 位在小数点后
```

对于需要精确小数的值，NUMERIC 很适合。

---

## 4. DATE

保存日期，例如：

```text
2026-09-17
```

---

## 5. TIMESTAMP

保存日期 + 时间。

```
2024-01-01 13:45:30.123
```

PostgreSQL 的 `timestamptz` 属于 PostgreSQL 自己很常用的时间类型，放到 PostgreSQL 专题再学。

---

## 6. BOOLEAN

```text
TRUE
FALSE
NULL
```

注意：

```text
NULL 不是 FALSE
```

---

## 7. 类型转换

标准 SQL 形式：

```sql
CAST('123' AS INTEGER)
```

PostgreSQL 还支持：

```sql
'123'::integer
```

后者非常方便，但属于 PostgreSQL 风格。

---

# 贯穿项目：同一个“大学选课数据库”

后面的练习不再“一学一个语法就换一张新表”。

从现在开始一直使用三张核心表：

```text
students
courses
enrollments
```

它们代表：

```text
students      学生
courses       课程
enrollments   学生与课程之间的选课关系
```

关系：

```text
students 1 ─── N enrollments N ─── 1 courses
```

你会不断在这三张表上增加操作能力：

```text
建结构
→ 插数据
→ 查数据
→ 改数据
→ 删数据
→ 连表
→ 分组统计
→ 子查询
→ 集合运算
→ 事务
→ 综合查询
```

---

**随堂练习 1：先把实验环境建起来**

先创建练习数据库：

```text
sql_learning
```

创建数据库本身就是一条 SQL：

```sql
CREATE DATABASE sql_learning;
```

注意 `DATABASE` 后面的名字不加引号。它是“名字”，不是字符串字面量——这一点和前面讲的单引号规则正好互相印证。

`CREATE DATABASE` 属于 DDL（定义结构），和后面每天写的 `SELECT` 不是一类东西：DDL 改的是“数据库里有哪些对象”，DML/DQL 改的是“对象里面的数据”。

创建完以后还要“进入”这个数据库。在 PostgreSQL 里，这通常不是靠一条 SQL 完成，而是靠连接时指定数据库，或使用客户端自己的命令（psql 里是 `\c sql_learning`）。这一步是客户端行为，不是 SQL 本身，所以别把它和 `CREATE DATABASE` 混成一件事。

这题只关注“数据库环境”，不要把 `CREATE DATABASE` 当成以后每天写的查询语法。

**练习目标：**

- 在 DataGrip 中建立/选择这个数据库；
- 或在 psql 中创建并连接它。

> psql 里常用几个客户端命令：`\c 数据库名`（切换数据库）、`\l`（列出所有数据库）、`\dt`（列出表）、`\d 表名`（查看表结构）、`\q`（退出）。它们以反斜杠开头，属于 psql 的“元命令”，不是标准 SQL。另外，`SELECT current_database();` 可以查看当前连的是哪个数据库。

---

## 建表：CREATE TABLE

建表就是把“这张表长什么样”写下来，交给数据库保存。骨架是：

```sql
CREATE TABLE 表名 (
    列名 类型 约束,
    列名 类型 约束,
    ...
    表级约束
);
```

例如：

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

拆开看，每一列都是“列名 + 类型 + 约束”：

```text
student_id   INTEGER   GENERATED ALWAYS AS IDENTITY   PRIMARY KEY
└─ 列名       └─ 类型    └─ 自动生成编号               └─ 主键
student_name VARCHAR(50) NOT NULL
└─ 列名       └─ 类型     └─ 不允许为空
```

`CREATE TABLE` 这个名字也说明了 SQL 的一种习惯：`CREATE` 负责“造出一个新对象”，后面跟对象的种类——`DATABASE` / `TABLE` / `INDEX` / `VIEW`。所以以后看到 `CREATE INDEX`、`CREATE VIEW`，不用重新记一套动词，只要看 `CREATE` 后面是谁。

### PRIMARY KEY（主键）

```sql
student_id INTEGER PRIMARY KEY
```

主键的核心是：

```text
唯一
非 NULL
用来标识一行
```

一个表可以有多个 `UNIQUE` 列，但通常只有一个主键，因为主键回答的是“这行到底是谁”。

### UNIQUE（唯一）

```sql
email VARCHAR(100) UNIQUE
```

不允许重复。

```text
PRIMARY KEY 是“这行是谁”
UNIQUE      是“这个值不能重复”
```

### NOT NULL（不能为空）

```sql
student_name VARCHAR(50) NOT NULL
```

不允许缺失。

---

### CHECK（检查条件）

```sql
CHECK (age >= 0)
```

告诉数据库：

> 不允许年龄为负数。

`CHECK` 的作用是把“业务上显然荒唐的值”挡在数据库门口，而不是让它散落在应用程序的各个 `if` 里。

### 什么样的约束应该放到check里，什么样的直接写？

没有“某类约束必须写 `CHECK`”这种绝对说法。更准确地说：

> 那些**没有专门约束语法**、又需要用布尔条件限制数据的约束，通常只能用 `CHECK` 表达。

括号里必须是一个**布尔表达式 / 条件**，比如：

```sql
CHECK (age >= 0)
CHECK (status IN ('active', 'inactive'))
CHECK (end_date > start_date)
CHECK (id IS NOT NULL)
```

---

### DEFAULT（默认值）

```sql
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
```

如果插入时没提供 `created_at`，数据库自己给默认值。`CURRENT_TIMESTAMP` 就是“现在”，所以插入数据时不用手动填时间。

### 自增主键：GENERATED ALWAYS AS IDENTITY

```sql
student_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

它表示：

> 这一列的值由数据库自己生成，不要你手填。

`GENERATED ... AS IDENTITY` 是标准 SQL 的写法。PostgreSQL 里你还可能看到更老的写法：

```sql
student_id SERIAL PRIMARY KEY
```

`SERIAL` 是 PostgreSQL 的历史习惯写法，能少打几个字，但它不是标准 SQL。初学阶段建议先记住 `GENERATED ... AS IDENTITY`，看到 `SERIAL` 知道“它是自增”即可。

### FOREIGN KEY（外键）与 REFERENCES

外键可以写在列定义里，也可以写在表末尾：

例如：

先建被引用的表 `students`：

```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(50) NOT NULL
);
```

然后建子表 `enrollments`，用**表级外键**：

```sql
CREATE TABLE enrollments (
    enrollment_id INT PRIMARY KEY,
    student_id INT NOT NULL,
    course_name VARCHAR(50),
    FOREIGN KEY (student_id)
        REFERENCES students(student_id)
);
```

这里：

- `enrollments` 是子表；
- `students` 是父表；
- `FOREIGN KEY (student_id)` 里的 `student_id` 是 `enrollments` 表的列；
- `REFERENCES students(student_id)` 表示引用 `students` 表的 `student_id` 列。

## 也可以写在列定义里

外键也可以直接跟在列后面，这叫**列级外键**：

sql

```
CREATE TABLE enrollments (
    enrollment_id INT PRIMARY KEY,
    student_id INT NOT NULL REFERENCES students(student_id),
    course_name VARCHAR(50)
);
```

这里:

```sql
student_id INT NOT NULL REFERENCES students(student_id)
```

就等价于在表末尾写：

```sql
FOREIGN KEY (student_id) REFERENCES students(student_id)
```

只不过列级写法更紧凑，适合单列外键。

意思：

> enrollments.student_id 必须引用 students 中真实存在的 student_id。

`REFERENCES` 这个词很形象：它不复制学生数据，只是“指向”students 里的某一行。这正是关系数据库“用键把表连起来”的体现——数据只存一份，其他表引用它。

外键默认要求被引用的值必须存在，所以你不能先往 enrollments 里插一个不存在的 student_id。这是数据库在替你维护三张表之间的一致性。

### 组合主键

有时单独一列不够充当主键，比如选课表：

```sql
PRIMARY KEY (student_id, course_id)
```

它表示：

```text
(student_id, course_id) 这个组合不能重复
```

也就是同一个学生不能把同一门课选两次，但一个学生可以选很多门课，一门课也可以被很多人选。

---

### 关于SQL什么时候加括号，见 [SQL括号哲学](./SQL-小括号.md)

---

**随堂练习 2：创建 students**

要求：

- `student_id`：整数、自增、主键
- `student_name`：最多 50 字符，不能为空
- `email`：最多 100 字符，不能为空且唯一
- `age`：整数，可以为 NULL；有值时必须 `>= 0`
- `major`：最多 50 字符，可以为 NULL
- `is_active`：BOOLEAN，不能为空，默认 TRUE
- `created_at`：TIMESTAMP，不能为空，默认当前时间

先自己写 `CREATE TABLE`，不要看答案。

把它拆成：

```text
CREATE TABLE
    建一张表

students
    表名

student_id
    列名

INTEGER
    数据类型

GENERATED ALWAYS AS IDENTITY
    数据库自动生成编号

PRIMARY KEY
    主键

NOT NULL
    不能为空

UNIQUE
    不能重复

CHECK
    数据必须满足条件

DEFAULT
    不提供值时使用默认值
```

这才是会写 `CREATE TABLE` 的入门：先能逐列说清楚“列名、类型、约束”，再组合成完整语句。

---

**随堂练习 3：创建 courses**

要求：

- `course_id`：整数、自增、主键
- `course_name`：最多 100 字符，不能为空
- `department`：最多 50 字符，不能为空
- `credits`：整数，不能为空，且 `> 0`

---

**随堂练习 4：创建 enrollments**

要求：

- `student_id`：不能为空，外键引用 `students(student_id)`
- `course_id`：不能为空，外键引用 `courses(course_id)`
- `score`：可以 NULL；有值时必须在 `0~100`
- `enrolled_at`：TIMESTAMP，不能为空，默认当前时间
- `(student_id, course_id)` 作为组合主键

这张表是两张业务表之间的“桥”。

---

## 写数据：INSERT

表建好以后是空的，往后所有查询都会没有结果，所以先把数据放进去。SQL 往表里放数据的语句是：

```sql
INSERT INTO students
    (student_name, email, age, major)
VALUES
    ('Alice', 'alice@example.com', 20, 'CS');
```

语法骨架：

```text
INSERT INTO 表
    (列1, 列2, 列3)
VALUES
    (值1, 值2, 值3);
```

拆开看每一块：

```text
INSERT      → 动词：插入
INTO        → 方向：插到哪里去
students    → 目标表
(...)       → 这次要给的列
VALUES      → 后面是“一行行具体的值”
```

`INTO` 不是凭空来的。SQL 里***凡是“把东西放进某个目标”的动作，都会用 `INTO` 标记目的地***。所以除了 `INSERT INTO`，你后面还会见到 `MERGE INTO`（合并进目标表）、`SELECT ... INTO`（把结果存进变量或新表）；在 psql / PL/pgSQL 里还有 `FETCH ... INTO`。反过来，`FROM` 永远表示“从哪里取”。一句话：

```text
INTO → 目标
FROM → 来源
```

记住这个方向感，就不用背“INSERT 后面为什么非要加 INTO”。

三件最值得养成的习惯：

1. **尽量显式写列名。** 不要养成：

   ```sql
   INSERT INTO students
   VALUES (...);
   ```

   这种完全依赖列顺序的写法。一旦以后表里加了列、改了顺序，这种语句就会悄悄插错位置。

2. **给了值的列，才写进去。** 没写的列如果定义了 `DEFAULT`，数据库会用默认值；如果允许 NULL，就填 NULL。`student_id` 有 `GENERATED ... AS IDENTITY`，`created_at` 有 `DEFAULT CURRENT_TIMESTAMP`，插入时都不需要手动填。

3. **默认值和自动生成，是数据库在替你补全“理所当然”的部分。** 能由数据库保证的规则，尽量别留给写代码的人去记。

## 一次插多行

```sql
INSERT INTO students
    (student_name, email, age, major)
VALUES
    ('Alice', 'alice@example.com', 20, 'CS'),
    ('Bob', 'bob@example.com', 19, 'Math'),
    ('Carol', 'carol@example.com', 21, 'Physics');
```

`VALUES` 后面其实是一张“临时的小表”，每一对括号就是一行。一次性插入多行，本质上只是把这张小表写长一点，而不是重复执行很多次 `INSERT`。

---

**随堂练习 5：插入 students 数据**

插入：

| name | email | age | major | active |
|---|---|---:|---|---|
| Alice | alice@example.com | 20 | CS | TRUE |
| Bob | bob@example.com | 19 | Math | TRUE |
| Carol | carol@example.com | 21 | CS | TRUE |
| David | david@example.com | NULL | Physics | TRUE |
| Eve | eve@example.com | 22 | CS | FALSE |
| Frank | frank@example.com | 20 | Math | TRUE |
| Grace | grace@example.com | 23 | Biology | TRUE |

不要手动填写 `student_id`，也不要手动填写 `created_at`。

---

**随堂练习 6：插入 courses 数据**

插入：

| course_name | department | credits |
|---|---|---:|
| Database Systems | CS | 4 |
| Operating Systems | CS | 4 |
| Algorithms | CS | 3 |
| Calculus | Math | 4 |
| Linear Algebra | Math | 3 |

---

**随堂练习 7：插入 enrollments 数据**

要求：

| student | course | score |
|---|---|---:|
| Alice | Database Systems | 91 |
| Alice | Operating Systems | 88 |
| Alice | Algorithms | 95 |
| Bob | Calculus | 78 |
| Bob | Linear Algebra | 85 |
| Carol | Database Systems | 84 |
| Carol | Algorithms | NULL |
| David | Calculus | 59 |
| Eve | Operating Systems | 97 |
| Frank | Linear Algebra | 72 |
| Frank | Calculus | 81 |

先用查询找出对应的 `student_id / course_id`，不要凭感觉猜编号。

---

## 读数据：SELECT

最核心的查询结构：

```sql
SELECT ...
FROM ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
```

不要要求自己一次记住所有子句。

先把下面这个练熟：

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

---

## 1. 所有列

```sql
SELECT *
FROM students;
```

`*` 在这里表示：

> 当前表的全部列。

---

## 2. 只看某几列

```sql
SELECT student_name, age
FROM students;
```

---

## 3. 起别名 AS

```sql
SELECT
    student_name AS name,
    age + 1 AS next_age
FROM students;
```

`AS` 可以理解成：

> “把这个结果叫做……”

例如：

```sql
age + 1 AS next_age
```

---

## 4. 表也可以起别名

```sql
SELECT s.student_name
FROM students AS s;
```

后面：

```sql
s.student_name
```

就表示 students 表里的 student_name。

多表查询几乎一定会用到这个。

### 注意！！！

## 1. 查询运行顺序

对于如下查询

```sql
SELECT ...
FROM table_a
JOIN table_b ON ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
```

逻辑顺序是：

1. **FROM / JOIN / ON**
   先确定数据来源，做表连接。
   `ON` 属于连接条件，在 `WHERE` 之前处理。
2. **WHERE**
   对连接后的**行**做过滤。
   此时还没有分组，也没有计算 `SELECT` 中的别名。
3. **GROUP BY**
   对过滤后的行进行分组。
4. **聚合函数计算**
   例如 `COUNT()`、`SUM()`、`AVG()` 等。
5. **HAVING**
   对分组后的**组**做过滤。
6. **SELECT**
   计算最终输出的列，并在这里定义 **列别名**。
   例如 `student_id AS SID` 中的 `SID` 就是在这里产生的。
7. **DISTINCT**
   如果有 `DISTINCT`，在这里去重。
8. **ORDER BY**
   对最终结果排序。
9. **LIMIT / OFFSET**
   最后截取行数。

------

## 2. 别名在哪一步可见？

### 表别名

表别名在 `FROM` / `JOIN` 中定义，定义之后，同一查询块的后续子句基本都可见：

```sql
FROM students s
JOIN courses c ON s.student_id = c.student_id
WHERE s.age > 18
GROUP BY s.student_id
HAVING COUNT(c.course_id) > 1
ORDER BY s.student_id;
```

这里 `s`、`c` 在 `SELECT`、`ON`、`WHERE`、`GROUP BY`、`HAVING`、`ORDER BY` 中都能用。

------

### 列别名

列别名是在 `SELECT` 中定义的，例如：

```sql
SELECT student_id AS SID,
       student_name AS NAME
FROM students;
```

`SID`、`NAME` 是在 **SELECT 阶段**才产生的。

因此标准 SQL 中：

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

原因是：
`WHERE`、`GROUP BY`、`HAVING` 都在 `SELECT` 之前执行，所以它们看不到 `SELECT` 里刚起的列别名。
而 `ORDER BY` 在 `SELECT` 之后执行，所以通常可以使用列别名。

- **PostgreSQL**：允许 `ORDER BY` 使用别名，`GROUP BY` 也可以引用输出列名，但 `HAVING` 通常不行。

---

解释：

```sql
SELECT ...
FROM students;
```

可以理解为：

> 先把 students 这张表作为当前数据来源。

所以：

```sql
SELECT age
FROM students
```

不是先“算 age”，而是先确定：

> 我要从 students 里拿数据。

这也是为什么 SQL 的逻辑执行顺序不会从 SELECT 开始。

---

**随堂练习 8：查询所有学生**

先自己写：`SELECT * FROM students;`

---

**随堂练习 9：只查询 `student_name / age / major`**

不允许直接 `SELECT *`。

---

## 记住一个特别重要的区别：

```text
WHERE → 过滤行
HAVING → 过滤组
```

## 比较：WHERE vs CHECK

### 相同处

**二者后面都需要跟 bool 表达式**

---

### 不同处

|          | CHECK                         | WHERE                    |
| :------- | :---------------------------- | :----------------------- |
| 所属语言 | DDL                           | DML                      |
| 作用     | 定义数据完整性规则            | 查询/更新/删除时筛选行   |
| 执行时机 | INSERT / UPDATE 时自动检查    | 查询或修改时逐行判断     |
| 语法位置 | CREATE TABLE / ALTER TABLE 等 | SELECT / UPDATE / DELETE |
| 例子     | `CHECK (age >= 0)`            | `WHERE age >= 0`         |

**CHECK 后面要加括号，WHERE 后面不用**

这里 `WHERE` 本身就在做“检查”，不需要再写 `CHECK`。

---

### 下面是 WHERE 的一些用法

## 1. 比较

```sql
SELECT *
FROM students
WHERE age >= 18;
```

---

## 2. AND

```sql
SELECT *
FROM students
WHERE age >= 18
  AND age <= 25;
```

---

## 3. BETWEEN-AND

```sql
WHERE age BETWEEN 18 AND 25
```

两端都包含。

所以：

```text
18 ≤ age ≤ 25
```

---

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
IN = “属于这个列表”
```

---

## 5. NOT

```sql
WHERE NOT (age < 18)
```

---

## 6. EXISTS

先记住它的核心意思即可：

> “是否存在至少一行满足条件？”

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

`EXISTS` 更适合记成“存在性判断”，不要把它理解成普通的“取值子查询”。

这里括号里的 `SELECT 1 ...` 是子查询，现在只要看懂“存在性判断”这个意思就够了；后面“子查询”一节会专门解释它。

---

这里一定要和 `SELECT *` 区分。

## 1. `%`

在 `LIKE` 模式里：

```text
% = 任意长度（可以是 0 个字符）
```

例如：

```sql
WHERE student_name LIKE 'A%'
```

表示：

```text
以 A 开头
```

---

## 2. `_`

```text
_ = 任意一个字符
```

例如：

```sql
WHERE student_name LIKE '_a%'
```

表示：

> 第二个字符是 a。



---

## 3. 转义

如果数据本身真的包含 `%`，需要使用转义。

```sql
WHERE code LIKE '100\%' ESCAPE '\'
```

---

## 4. DISTINCT

例如：

```sql
SELECT age
FROM students;
```

可能出现：

```text
18
18
20
20
21
```

加：

```sql
SELECT DISTINCT age
FROM students;
```

得到：

```text
18
20
21
```

***注意 1 ！！！***

**`distinct` 写在 `select` 后面而不是语句后**

***注意2 ！！！***

```sql
SELECT DISTINCT major, age
```

去掉的是：

> `(major, age)` 这个组合重复的行。

不是分别对 major 和 age 各自去重。

---

## 5. ORDER BY

```sql
SELECT student_name, age
FROM students
ORDER BY age;
```

默认升序。

显式写：

```sql
ORDER BY age ASC;
```

降序：

```sql
ORDER BY age DESC;
```

多个排序条件：

```sql
ORDER BY age DESC, student_name ASC;
```

意思：

1. 先按 age 从大到小；
2. age 一样的，再按名字升序。

记住：

```text
ORDER BY 是“先排第一关键字，再用第二关键字打破平局”
```

还有一个容易被忽略的点：**NULL 参与排序时排在哪里？** 标准 SQL 没有规定死，PostgreSQL 的默认行为是：

```text
ASC  → NULL 排在最后
DESC → NULL 排在最前
```

如果你希望明确控制，可以写：

```sql
ORDER BY age DESC NULLS LAST
```

或者：

```sql
ORDER BY age ASC NULLS FIRST
```

`NULLS FIRST / NULLS LAST` 在很多数据库里都支持，是显式表达意图的好习惯。后面的综合统计题里，平均成绩可能是 NULL，就需要注意这一点。

---

## 6. LIMIT 和 OFFSET

PostgreSQL 中常用：

```sql
LIMIT 10
OFFSET 20
```

注意：`LIMIT` 是 PostgreSQL 等数据库里很常见的写法，但它不是标准 SQL 最核心的可移植写法。

意思：

> 先 OFFSET 跳过前 20 行，再 LIMIT 取最多 10 行。

它和分页有关。

注意：

如果没有 `ORDER BY`，你不能把分页结果理解成稳定的“第 1 页、第 2 页”。

所以通常：

```sql
ORDER BY student_id
LIMIT 10 OFFSET 20;
```

比单独：

```sql
LIMIT 10 OFFSET 20
```

更可靠。

---

## 为什么 WHERE 不能直接用 SELECT 的别名？

例如：

```sql
SELECT
    age + 1 AS next_age
FROM students
WHERE next_age > 18;
```

在标准 SQL 的一般规则下，这里不成立。

因为逻辑上：

```text
WHERE
```

先于：

```text
SELECT
```

此时 `next_age` 这个输出别名还没有形成。

可以写：

```sql
SELECT
    age + 1 AS next_age
FROM students
WHERE age + 1 > 18;
```

---

## 为什么 ORDER BY 经常可以用 SELECT 别名？

因为逻辑上：

```text
SELECT
```

先于：

```text
ORDER BY
```

所以：

```sql
SELECT
    age + 1 AS next_age
FROM students
ORDER BY next_age;
```

是自然的。

---

**随堂练习 10：查询年龄 `>= 20` 的学生。**

---

**随堂练习 11：查询 `major = 'CS'` 且年龄 `>= 20` 的学生。**

---

**随堂练习 12：查询年龄在 19 到 21 岁之间的学生。**

注意 `BETWEEN` 是否包含两端。

---

**随堂练习 13：查询年龄属于 `19 / 20 / 22` 的学生。**

提示：`IN`。

---

**随堂练习 14：查询名字中包含 `a` 的学生。**

v1：标准 SQL `LIKE`。

v2：~ 正则表达式

---

**随堂练习 15：按年龄降序，年龄相同按姓名升序。**

---

**随堂练习 16：查询年龄最大的 3 名学生。**

---

**随堂练习 17：查询所有不同的专业。**

---

**随堂练习 18：查询年龄为 NULL 的学生。**

不要写 `age = NULL`。

---

## 条件表达式：CASE-END

```sql
SELECT
    student_name,
    CASE
        WHEN age >= 18 THEN 'adult'
        ELSE 'minor'
    END AS age_group
FROM students;
```

逻辑就是：

```text
如果 age >= 18
    → adult
否则
    → minor


记忆结构
CASE
    WHEN 条件 THEN 结果
    WHEN 条件 THEN 结果
    ELSE 结果
END
```

注意：***只有 when 结果为 true 才不进入 else，而unknown 会进入 else***

以后做统计题时：

```sql
SUM(
    CASE WHEN score >= 60 THEN 1 ELSE 0 END
)
```

会非常常见。

---

**随堂练习 19：查询 `student_name / age / age + 1`，第三列命名为 `age_next_year`。**

---

**随堂练习 20：用 `CASE` 显示 `adult / minor`。**

规则：`age >= 18 → adult`，其他 → `minor`。NULL 年龄也归 `minor`。

---

**随堂练习 21：查询学生姓名，并把 NULL 的 `major` 显示为 `Unknown`。**

提示：`COALESCE`。

---

## 改数据：UPDATE 与删数据：DELETE

```sql
UPDATE students
SET age = age + 1
WHERE student_id = 1;
```

理解：

```text
UPDATE → 改哪张表
SET    → 改成什么
WHERE  → 改哪些行
```

---

## 最大的危险

```sql
UPDATE students
SET age = age + 1;
```

因为没有 WHERE：

> 所有人都改。

所以初学阶段：

```text
UPDATE / DELETE 写完后，第一反应检查 WHERE。
```

---

```sql
DELETE FROM students
WHERE student_id = 1;
```

注意这里写作 `DELETE FROM`，而 `UPDATE` 是直接 `UPDATE students SET ...`。看起来不对称，但其实是故意的：

```text
UPDATE 表 SET ...     → 改的是这张表自己
DELETE FROM 表        → 从这张表里把行拿走
INSERT INTO 表        → 往这张表里放行
```

`FROM` 表示“来源”，`INTO` 表示“去向”。`DELETE` 是“从表里取走”，所以配 `FROM`；`UPDATE` 改的是表本身，不需要方向介词。把这三个介词放在一起看，比单独背三条语法更省力。

如果没有 WHERE：

```sql
DELETE FROM students;
```

就是删除整张表里的所有行。

这和：

```sql
DROP TABLE students;
```

不是一回事。

---

这是非常容易混淆的一组。

```text
DELETE
    删除行，表还在

TRUNCATE
    快速清空表，表还在

DROP TABLE
    表本身没了
```

记忆：

```text
DELETE → 扔数据
TRUNCATE → 清空
DROP → 拆掉整个表
```

---

**随堂练习 22：把 Bob 的 `major` 改成 `CS`。**

先 SELECT 确认目标，再 UPDATE。

---

**随堂练习 23：把所有 CS 学生的年龄加 1。**

先查询范围，再执行 UPDATE。

---

**随堂练习 24：把 Eve 的 `is_active` 改成 TRUE。**

---

**随堂练习 25：删除 Frank 的 Calculus 选课记录。**

先分别查出 Frank 的 `student_id` 和 Calculus 的 `course_id`，确认目标行以后再 `DELETE`。这一步先不要求写 JOIN 或子查询——`DELETE FROM ... WHERE ...` 配合已经学过的 `SELECT` 就够用了。

---

## 附加练习

前面建表时讲过的那些约束（`PRIMARY KEY` / `UNIQUE` / `NOT NULL` / `CHECK` / `DEFAULT` / `FOREIGN KEY`）不是写在文档里给人看的，而是数据库真的要执行的规则。下面用两个实验亲眼看一下。

---

**随堂练习 26：尝试插入一个 email 与 Alice 相同的学生。**

观察 `UNIQUE` 如何阻止重复数据。

---

**随堂练习 27：观察约束如何拦下非法数据。**

1. 把某条选课成绩改成 `120`，观察 `CHECK` 如何阻止非法值。
2. 尝试插入一条 `student_id = 9999` 的选课记录（这个学生并不存在），观察 `FOREIGN KEY` 如何阻止它。

---

## 连接表：JOIN

到现在为止，我们一次只处理一张表。真实问题往往要同时用多张表，还是这三张：

```text
students
courses
enrollments
```

三张表分别保存：

```text
学生是谁
课程是什么
谁选了哪门课、成绩多少
```

它们之间通过键连接起来。

---

## 1. INNER JOIN

```sql
SELECT
    s.student_name,
    e.course_id,
    e.score
FROM students AS s
JOIN enrollments AS e
    ON e.student_id = s.student_id;
```

意思：

> 把 students 和 enrollments 中能够对应上的行拼起来。

`ON` 是“按什么条件把两边对上”。`JOIN ... ON ...` 这个形状很像自然语言：“把 A 和 B 接起来，条件是……”——`ON` 后面永远写“两边怎么对应”。

记：

```text
INNER JOIN = 只看匹配上的
```

---

## 2. LEFT JOIN

```sql
SELECT
    s.student_name,
    e.course_id
FROM students AS s
LEFT JOIN enrollments AS e
    ON e.student_id = s.student_id;
```

意思：

> 左边 students 全保留。

某个学生没选课时：

```text
e.course_id = NULL
```

记：

```text
LEFT = 左边一个都不能丢
```

---

## 3. RIGHT JOIN / FULL JOIN

知道含义即可：

```text
RIGHT JOIN → 右表全保留
FULL JOIN  → 两边都尽量保留
```

实际写业务 SQL 时，LEFT JOIN 通常更常见。

---

## 4. CROSS JOIN

```sql
SELECT *
FROM students
CROSS JOIN courses;
```

表示每个学生和每门课全部配对。

如果：

```text
3 个学生
4 门课
```

结果有：

```text
3 × 4 = 12 行
```

这就是笛卡尔积。

---

例如：

```sql
SELECT ...
FROM students s
LEFT JOIN enrollments e
    ON e.student_id = s.student_id
WHERE e.score >= 60;
```

很多人会以为：

> 我只是想筛选成绩。

但由于：

```text
WHERE
```

是在连接结果形成之后再过滤行的，所以 `e.score IS NULL` 的学生会被 WHERE 筛掉。

于是这个 LEFT JOIN 在效果上可能变得很像 INNER JOIN。

如果你的目的是真正：

> 左表学生全部保留，只把“匹配的成绩”限制在 60 分以上

常见写法是：

```sql
SELECT ...
FROM students s
LEFT JOIN enrollments e
    ON e.student_id = s.student_id
   AND e.score >= 60;
```

这个区别非常值得理解。

---

**随堂练习 28：查询 `学生姓名 / 课程名称 / 成绩`，一行代表一条选课记录。**

---

**随堂练习 29：查询 `学生姓名 / 课程名称 / 院系 / 成绩`。**

---

**随堂练习 30：查询所有学生，即使没有任何选课也必须出现。**

提示：`LEFT JOIN`。

---

**随堂练习 31：查询从未选过课程的学生。**

提示：`LEFT JOIN + IS NULL`。

---

**随堂练习 32：查询选了 Database Systems 的学生。**

---

**随堂练习 33：查询成绩 `>= 90` 的学生姓名和课程名。**

---

## 聚合与分组：GROUP BY / HAVING

先认识这些：

```text
COUNT
SUM
AVG
MIN
MAX
```

---

## 1. COUNT(*)

```sql
SELECT COUNT(*)
FROM students;
```

表示：

> 当前结果里有多少行。

---

## 2. COUNT(column)

```sql
SELECT COUNT(age)
FROM students;
```

表示：

> age 不为 NULL 的行有多少。

这是初学者特别容易搞错的点。

---

## 3. AVG / SUM / MIN / MAX

```sql
SELECT AVG(score)
FROM enrollments;
```

---

## 4. GROUP BY

例如：

```sql
SELECT course_id, COUNT(*) AS cnt
FROM enrollments
GROUP BY course_id;
```

理解成：

> 按 course_id 把行分成一组一组，然后每组算 COUNT。

---

## 5. GROUP BY 为什么会“压缩结果”？

原始：

```text
数学
数学
英语
英语
英语
```

按课程分组以后，从“五行”变成了：

```text
数学   2
英语   3
```

所以：

```text
GROUP BY = 把多行折成组
```

---

## 6. HAVING

```sql
SELECT course_id, COUNT(*) AS cnt
FROM enrollments
GROUP BY course_id
HAVING COUNT(*) >= 2;
```

顺序：

```text
WHERE → 先删掉某些行
GROUP BY → 把剩余行分组
HAVING → 再删掉某些组
```

最重要的一句话：

```text
WHERE 管行，HAVING 管组。
```

---

例如：

```sql
SELECT
    course_id,
    student_id,
    COUNT(*)
FROM enrollments
GROUP BY course_id;
```

你会问：

> 一个 course_id 组里可能有多个 student_id，到底该显示哪个？

数据库当然不知道。

所以一般来说，SELECT 中：

- 要么是分组列；
- 要么是聚合结果；
- 要么是能够从分组列唯一确定的表达式（这里先不要靠这个绕规则）。

初学阶段直接记：

```text
SELECT 里的普通列，通常必须出现在 GROUP BY。
```

---

**随堂练习 34：统计学生总数。**

---

**随堂练习 35：统计每门课程有多少人选，显示 `课程名 / 选课人数`。**

注意：没有人选的课程也应该出现。

---

**随堂练习 36：统计每个专业的学生数量。**

---

**随堂练习 37：统计每门课的平均成绩。**

注意：NULL 成绩不能当成 0。

---

**随堂练习 38：找出平均成绩 `>= 80` 的课程。**

提示：这是组过滤。

---

**随堂练习 39：统计每位学生选了多少门课。**

0 门课的学生也应该出现。

---

**随堂练习 40：找出至少选了 2 门课的学生。**

---

## 子查询：查询里面的查询

先看一个简单例子：

```sql
SELECT student_name
FROM students
WHERE student_id IN (
    SELECT student_id
    FROM enrollments
    WHERE score >= 90
);
```

外层在问：

> 哪些学生？

内层在产生：

> 分数至少 90 的学生 id。

---

## 子查询三类最常见用法

### A. 当一个值

```sql
WHERE score > (
    SELECT AVG(score)
    FROM enrollments
)
```

---

### B. 当一个集合

```sql
WHERE student_id IN (
    SELECT student_id
    FROM enrollments
)
```

---

### C. 当存在性判断

```sql
WHERE EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
)
```

你以后看到子查询，先问：

```text
这个子查询最后是在产生一个值？
一列值？
还是“存在/不存在”？
```

这样就不会乱。

---

例如：

```sql
WHERE student_id NOT IN (
    SELECT student_id
    FROM enrollments
)
```

如果子查询结果里可能出现 NULL，`NOT IN` 会受到三值逻辑影响，结果可能和直觉完全不一样。

所以在“找不存在的对应行”时，常见而稳妥的思路是：

```sql
WHERE NOT EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
)
```

这不是要求你现在就熟练写，而是先把“存在性查询”和“集合包含”区分开。

---

**随堂练习 41：查询成绩高于“所有选课成绩平均值”的选课记录。**

内层查询应该产生一个值。

---

**随堂练习 42：用 `EXISTS` 查询至少选过一门课程的学生。**

---

**随堂练习 43：用 `NOT EXISTS` 查询从未选过课程的学生。**

---

**随堂练习 44：用 `IN` 子查询查询选过 Database Systems 的学生。**

SQL 不只有 JOIN。

还有：

```text
UNION
UNION ALL
INTERSECT
EXCEPT
```

例如：

```sql
SELECT student_name
FROM students
WHERE major = 'CS'

UNION

SELECT student_name
FROM students
WHERE age >= 20;
```

它们是把“查询结果”当集合处理。

---

## UNION vs UNION ALL

```text
UNION
    合并 + 去重

UNION ALL
    合并 + 不去重
```

如果你确定不需要去重，`UNION ALL` 往往更直接。

两个查询要能对应起来，最基本的要求是：

```text
列数一致
对应列的数据类型可兼容
```

---

**随堂练习 45：用 `UNION` 得到“CS 专业学生”和“年龄 >= 21 学生”的姓名集合。**

---

**随堂练习 46：用 `INTERSECT` 得到“CS 且年龄 >= 20”的学生名字。**

---

**随堂练习 47：用 `EXCEPT` 找出所有学生中没有选课的人。**

---

## 把查询结果写回表：INSERT ... SELECT

`INSERT` 的数据来源不一定是手写的 `VALUES`，也可以是一段查询：

```sql
INSERT INTO target_table (col1, col2)
SELECT col1, col2
FROM source_table
WHERE ...;
```

结构上它其实是两件事拼起来的：

```text
INSERT INTO 目标表 (列...)
    → 往哪里写

SELECT ...
    → 写什么（一个结果集）
```

`VALUES` 和 `SELECT` 在这里是同一位置的两种选择：一个直接给一张常量小表，一个用查询算出结果集。SQL 的很多地方都体现这个思路——语句可以被组合，而不是每种情况发明一个新关键字。

---

**随堂练习 48：`INSERT ... SELECT`**

不新建业务表，直接练“查询结果可以成为 INSERT 的输入”。

把所有 CS 学生临时复制成一组“Copy”学生：

- 姓名后加 ` Copy`
- 邮箱前加 `copy_`
- 其他字段尽量沿用

为了不污染后面的练习，观察完后把这些“Copy 学生”删掉即可（`DELETE` 前面已经学过；可以按邮箱前缀或姓名后缀来定位）。

---

## 把复杂查询分步：CTE（WITH）

当一个查询越来越复杂时，可以写：

```sql
WITH high_scores AS (
    SELECT *
    FROM enrollments
    WHERE score >= 90
)
SELECT *
FROM high_scores;
```

可以把它理解成：

> 先给一个中间结果起名字，再继续查询。

它非常适合“分步骤思考”。以后遇到复杂统计题（比如最后那道综合题），你可以把一部分统计先写进 `WITH`，再在外层继续查。

---

**随堂练习 49：用 CTE 把统计拆成两步**

用 `WITH` 统计“平均成绩 `>= 80` 的课程”，显示 `课程名 / 平均分`。

做法提示：

1. 第一个 CTE 先算出每门课的 `course_id` 和平均分；
2. 外层再连 `courses` 取课程名并筛选。

体会一下：`WITH` 不是新功能，只是把“先算一部分”写得更像一句人话。

---

## 事务：BEGIN / COMMIT / ROLLBACK

最基础：

```sql
BEGIN;

UPDATE students
SET major = 'CS'
WHERE student_id = 1;

UPDATE enrollments
SET score = 95
WHERE student_id = 1
  AND course_id = 1;

COMMIT;
```

如果中途发现改错：

```sql
ROLLBACK;
```

记忆：

```text
BEGIN    开始
COMMIT   确认
ROLLBACK 反悔
```

`BEGIN` 和 `COMMIT` 分开，是 SQL 的一个关键设计：数据库不会一边执行一边把改动永久写死，而是让你在一个事务里做完一串操作后，由你决定是 `COMMIT`（确认）还是 `ROLLBACK`（反悔）。这保证了“要么全成功，要么全不发生”。

> 在 DataGrip 里练习事务时要注意：如果当前 console 处于 auto-commit（自动提交）模式，每条语句执行完就自动确认了，`ROLLBACK` 就看不到效果。练习前先确认事务控制方式；在 psql 里则完全由你写的 `BEGIN` 决定。

ACID 先知道名字：

```text
Atomicity
Consistency
Isolation
Durability
```

15-445 以后会把 Isolation 等内容讲得非常深入，SQL 入门阶段不需要在这里展开。

---

**随堂练习 50：事务 + ROLLBACK**

1. `BEGIN`
2. 把 Alice 的专业改成 Math。
3. 查询确认。
4. `ROLLBACK`
5. 再查询确认恢复。

重点是观察事务中的修改为什么可以反悔。

---

**随堂练习 51：事务 + COMMIT**

1. `BEGIN`
2. 把 Alice 的专业改成 Math。
3. `COMMIT`
4. 再查询确认修改仍然存在。

---

## 改结构：ALTER TABLE

前面 `CREATE TABLE` 是把表“造出来”。表造好以后，结构也可能要变，改结构的动词是 `ALTER`（改变）：

```sql
ALTER TABLE students
ADD COLUMN phone VARCHAR(20);
```

`ALTER TABLE` 后面跟着“改哪张表”，然后再说“怎么改”：`ADD COLUMN` 是加列。改列类型、重命名、删列等，也都属于 `ALTER TABLE` 这件事，初学先掌握 `ADD COLUMN` 就够。

改列定义、重命名、删除列等，都属于：

```sql
ALTER TABLE
```

初学先掌握：

```text
ADD COLUMN
```

然后慢慢认识其他形式。

---

## 索引

```sql
CREATE INDEX idx_students_major
ON students(major);
```

可以先把它理解成：

> 为某种查询建立更快的查找结构。

它不是：

> “加了索引以后任何查询都自动变快”。

索引也有空间和写入维护成本。

---

## EXPLAIN：看看数据库打算怎么查

建了索引，怎么知道数据库有没有真用上？用 `EXPLAIN`：

```sql
EXPLAIN
SELECT *
FROM students
WHERE major = 'CS';
```

它不会把查询结果给你，而是给你“查询计划”（query plan）：是整表顺序扫描（Seq Scan），还是走索引（Index Scan）。

注意：

> 小表上，即使建了索引，PostgreSQL 也未必选择索引扫描。

优化器会根据数据量、列的选择性、成本估算等自己决定访问路径。所以：

```text
有索引
≠
一定使用索引
```

---

## 视图

```sql
CREATE VIEW active_students AS
SELECT *
FROM students
WHERE is_active = TRUE;
```

以后：

```sql
SELECT *
FROM active_students;
```

可以把视图理解成：

> 给一个查询结果起名字。

---

**随堂练习 52：给 `students` 增加 `phone VARCHAR(20)`。**

---

**随堂练习 53：创建 `active_students` 视图，只显示 `is_active = TRUE` 的学生。**

---

**随堂练习 54：为 `students.major` 建立索引，然后用 `EXPLAIN` 观察查询计划。**

思考：为什么“有索引”不等于“必然使用索引”？

聚合：

```sql
SELECT course_id, AVG(score)
FROM enrollments
GROUP BY course_id;
```

会把多行压缩成每门课一行。

窗口函数：

```sql
SELECT
    student_id,
    course_id,
    score,
    AVG(score) OVER (PARTITION BY course_id) AS course_avg
FROM enrollments;
```

不会把原来的行压掉。

所以可以记：

```text
GROUP BY → 汇总、压缩

窗口函数 → 保留原行，同时给每行附加统计结果
```

这块先做到“看得懂”，不要打断前面的主线。

---

**随堂练习 55：窗口函数**

查询每条选课记录，同时显示该课程平均分；原来的选课行不能被折叠。提示：`AVG(...) OVER (PARTITION BY ...)`。

主线先不靠这些功能也能学好 SQL。

知道它们存在即可。

## 1. RETURNING

PostgreSQL 很方便：

```sql
INSERT INTO students (...)
VALUES (...)
RETURNING student_id;
```

可以直接拿刚插入的结果。

---

## 2. ON CONFLICT

PostgreSQL 提供：

```sql
INSERT INTO students (...)
VALUES (...)
ON CONFLICT (...) DO NOTHING;
```

或：

```sql
... DO UPDATE ...
```

很实用，但不是 SQL 入门第一优先级。

---

## 3. ILIKE

PostgreSQL 的：

```sql
WHERE student_name ILIKE 'a%'
```

进行不区分大小写的模式匹配。

标准 SQL 主线先掌握：

```sql
LIKE
```

---

## 4. `::` 类型转换

PostgreSQL：

```sql
SELECT '123'::INTEGER;
```

标准写法：

```sql
SELECT CAST('123' AS INTEGER);
```

入门先会 `CAST`，再认识 `::`。

---

## 5. JSON / JSONB、数组、COPY、EXPLAIN

这些都值得学 PostgreSQL，但不应该在你还没把：

```text
SELECT
WHERE
GROUP BY
JOIN
INSERT
UPDATE
DELETE
```

练熟之前喧宾夺主。

---

为了不让“先做题、后学语法”的事发生，知识顺序按“用到什么就先教什么”排成：

```text
0. SQL 的骨架和关键词
1. 大小写 / 引号 / 分号 / 注释
2. 值 / 列 / 表达式 / 运算符（含字符串拼接 ||）
3. 数据类型
4. NULL 和三值逻辑
5. CREATE DATABASE / CREATE TABLE / 约束
6. INSERT
7. SELECT / FROM
8. WHERE + 运算符（含 LIKE / IN / BETWEEN，EXISTS 先预览）
9. DISTINCT
10. ORDER BY / LIMIT / OFFSET
11. CASE / COALESCE
12. UPDATE / DELETE
13. JOIN
14. 聚合 / GROUP BY / HAVING
15. 子查询 / EXISTS
16. 集合运算 UNION / INTERSECT / EXCEPT
17. INSERT ... SELECT
18. CTE
19. 事务
20. ALTER TABLE
21. 视图 / 索引 / EXPLAIN
22. 窗口函数（先理解）
23. PostgreSQL 特性
```

这样每个阶段的习题，都只用已经讲过的语法；遇到新语法再往后加，而不是让你先猜，更不是一开始就把：

```text
CTE → 窗口函数 → JSONB → 数组 → COPY → 权限
```

全部塞进来。

---

## 图 1：查询主骨架

```text
SELECT   我要看什么
FROM     数据从哪里来
JOIN     怎么把表拼起来
ON       两张表怎么对应
WHERE    过滤行
GROUP BY 怎么分组
HAVING   过滤组
ORDER BY 怎么排序
LIMIT    截取多少
```

---

## 图 2：数据修改骨架

```text
INSERT INTO 表 (列...)
VALUES (值...)

UPDATE 表
SET 列 = ...
WHERE ...

DELETE FROM 表
WHERE ...
```

---

## 图 3：NULL

```text
NULL 不是 0
NULL 不是 ''
NULL 不是 FALSE

判断：
IS NULL
IS NOT NULL

处理：
COALESCE(...)
```

---

## 图 4：JOIN

```text
INNER → 匹配的
LEFT  → 左边全留
RIGHT → 右边全留
FULL  → 两边都留
CROSS → 全组合
```

---

**随堂练习 56：终极综合题**

查询：

```text
学生姓名
专业
选课数量
平均成绩
```

要求：

1. 没选课的学生也出现；
2. NULL 成绩不当成 0；
3. 平均成绩从高到低；
4. 平均成绩相同按姓名升序；
5. 0 门课的学生平均成绩显示 NULL。

这道题必须自己决定从哪张表开始，以及为什么应该使用 `LEFT JOIN`。

---

# 最后的学习节奏

以后每学一个新语法，都遵循：

```text
讲懂一个小知识点
        ↓
马上在同一个数据库上做 1~几个小题
        ↓
看结果
        ↓
自己解释“为什么”
        ↓
再继续下一块
```

不要等整章结束才做题。

真正需要练熟的第一批骨架：

```text
CREATE TABLE / 约束
INSERT
SELECT
FROM
WHERE
ORDER BY
UPDATE
DELETE
JOIN
GROUP BY
HAVING
```

然后逐步补：

```text
NULL
CASE
COALESCE
子查询
EXISTS
UNION / INTERSECT / EXCEPT
CTE
事务
窗口函数
```

PostgreSQL 自己的：

```text
RETURNING
ON CONFLICT
ILIKE
:: 类型转换
JSONB
数组
COPY
EXPLAIN
```

放在基础 SQL 已经顺手以后再深入。
