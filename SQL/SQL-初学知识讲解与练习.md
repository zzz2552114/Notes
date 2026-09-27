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

### 注意！！！！

外键约束的规则是：

> 如果外键列的值不是 NULL，那么它必须能在被引用表的主键/唯一键中找到；
> 如果外键列的值是 NULL，就表示“没有引用任何行”，约束自动通过。

所以：

- 被引用表的主键或UNIQUE键：不能 NULL。
- 引用表的外键列：默认可以 NULL，除非你写 `NOT NULL`。

---

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



## SQL 不只有 JOIN。

还有集合运算（Set Operations），它们将多个查询的“结果集”当作集合进行合并或相减：

```text
UNION      并集（去重）
UNION ALL  并集（不去重）
INTERSECT  交集
EXCEPT     差集
```

它们的核心区别于 JOIN：
- **JOIN** 主要是**左右横向**拼接数据（增加列，扩展信息）。
- **集合运算** 主要是**上下纵向**堆叠数据（增加行，合并结果）。

使用集合运算的**严格前提条件**：
1. **列数必须完全一致**：两个 SELECT 返回的字段数量必须相同。
2. **对应列的数据类型必须兼容**：第一列对第一列，第二列对第二列，类型必须能隐式转换（最好是完全一样）。
3. **最终列名**：结果集的列名通常由第一个 SELECT 语句决定。

---

## 1. UNION 与 UNION ALL（并集）

**作用**：将两个查询的结果合并到一起。

- `UNION`：合并后会**自动去除重复行**。
- `UNION ALL`：合并后**保留所有重复行**。

**常用情境**：

- 需要从多个结构相似的表中汇总数据（比如今年的订单表 `orders_2023` 和去年的订单表 `orders_2022` 汇总）。
- 当一个非常复杂的 `OR` 逻辑导致索引失效或者难以阅读时，可以拆分成两个单独的 `SELECT` 再 `UNION` 起来。

**注意事项**：

- `UNION` 在后台需要进行一次全局去重（通常是通过排序或哈希），非常耗费性能。
- **最佳实践**：如果你在逻辑上能确定两个结果集肯定没有交集，或者你不在乎重复数据，**请永远优先使用 `UNION ALL`**。只有当你明确需要去重时，才使用 `UNION`。

**示例**：

```sql
SELECT student_name FROM students WHERE major = 'CS'
UNION
SELECT student_name FROM students WHERE age >= 20;
```
如果有一个学生既是 CS 专业，年龄又 >= 20，`UNION` 会保证名字只出现一次。如果你用 `UNION ALL`，这个名字就会出现两次。

---

## 2. INTERSECT（交集）

**作用**：提取两个查询结果中**都存在**的行。也就是“取共同点”。

**常用情境**：

- 寻找同时满足两个复杂条件的数据集合。
- 比较两个表的数据差异，找出共同拥有的人或记录（例如：购买了商品 A 且同时购买了商品 B 的用户）。

**注意事项**：

- 很多时候 `INTERSECT` 可以被 `INNER JOIN` 或 `EXISTS` 子查询替代。但是当你要比较整个结果集的多列内容时，`INTERSECT` 写起来更直观。
- `INTERSECT` 也会自动去重。

**示例**：

```sql
-- 既是 CS 专业，又是年龄 >= 20 的学生
SELECT student_name FROM students WHERE major = 'CS'
INTERSECT
SELECT student_name FROM students WHERE age >= 20;
```

---

## 3. EXCEPT（差集）

> *注：在 Oracle 等一些数据库中，EXCEPT 被称为 MINUS。在 PostgreSQL 和 SQL Server 中使用 EXCEPT。*

**作用**：从第一个查询结果中，**减去**第二个查询结果中也存在的行。即“只在左边，不在右边”。

**常用情境**：

- 找出某种“缺失”：比如所有学生减去已经选课的学生，剩下的就是没选课的学生。
- 排除特定条件的数据：查询某类群体，但要从中剔除掉另一个复杂查询算出来的群体。

**注意事项**：
- `EXCEPT` 是有**方向性**的。`A EXCEPT B` 和 `B EXCEPT A` 的结果完全不同。
- `EXCEPT` 也可以用 `LEFT JOIN ... WHERE right_table.id IS NULL` 或 `NOT EXISTS` 替代，但在表达“全表差异”时 `EXCEPT` 非常直观。

**示例**：

```sql
-- 所有 CS 学生，排除掉年龄大于 21 岁的，剩下的 CS 学生
SELECT student_name FROM students WHERE major = 'CS'
EXCEPT
SELECT student_name FROM students WHERE age > 21;
```

---

### 集合运算与 ORDER BY / LIMIT

如果要对集合运算后的最终结果进行排序或限制数量，`ORDER BY` 和 `LIMIT` 只能写在**整个语句的最后面**，并且排序的字段名必须是第一个 `SELECT` 中输出的字段名。

```sql
SELECT student_name, age FROM students WHERE major = 'CS'
UNION
SELECT student_name, age FROM students WHERE major = 'Math'
ORDER BY age DESC -- 对合并后的结果排序
LIMIT 5;          -- 从合并结果中取前5个
```
*(注意：不能在第一个 SELECT 后面单独加 ORDER BY，除非用括号包起来当子查询使用。)*

---

**随堂练习 45：用 `UNION` 得到“CS 专业学生”和“年龄 >= 21 学生”的姓名和年龄集合。**

---

**随堂练习 46：用 `UNION ALL` 改写上面的查询，并比较结果数量的差异。**

---

**随堂练习 47：用 `INTERSECT` 得到“选修了 Database Systems (course_id=1)”且“选修了 Algorithms (course_id=3)”的 `student_id`。**

---

**随堂练习 48：用 `EXCEPT` 找出所有学生中没有选课的人（即在 students 表里，但不在 enrollments 表里出现的 student_id）。**

---

**随堂练习 49：结合 `EXCEPT` 和 `ORDER BY`，找出提供了成绩的 enrollments (score IS NOT NULL)，排除掉成绩不及格 (score < 60) 的记录，最后按 score 降序排列。查询输出 `student_id, course_id, score`。**

---

## 把查询结果写回表：INSERT ... SELECT

在之前的学习中，我们通过 `INSERT INTO ... VALUES (...)` 手动一条条插入数据。但如果数据已经存在于其他表里（或者可以通过某个复杂的 SQL 算出来），我们可以直接将 `SELECT` 的结果集作为 `INSERT` 的数据源，一键批量插入。

**核心语法**：
```sql
INSERT INTO target_table (col1, col2, col3)
SELECT expr1, expr2, expr3
FROM source_table
WHERE ...;
```

### 1. 各种用法与常用情境

- **数据备份与归档（Archive）**：年底到了，我们需要把 2022 年之前的订单全部挪到一张叫 `orders_archive` 的表里。
  ```sql
  INSERT INTO orders_archive (order_id, user_id, amount, created_at)
  SELECT order_id, user_id, amount, created_at 
  FROM orders 
  WHERE created_at < '2023-01-01';
  ```
- **数据清洗与转换**：原本的表设计不好，或者需要把多张表的数据聚合后存入一张新表。你可以用复杂的 `JOIN`、`GROUP BY` 写一个查询，直接 `INSERT` 进新表。
- **批量复制数据进行测试**：在开发时，为了不污染线上数据，可以根据线上数据衍生出一批测试数据。我们可以在 `SELECT` 中利用字符串拼接、加减法来生成测试数据。

### 2. 注意事项与重点

- **列的数量与顺序严格对应**：`INSERT INTO` 后面括号里的列名顺序，必须与 `SELECT` 选出的列**顺序严格一致，数量相等**。
- **列名不重要，数据类型最重要**：`SELECT` 出来的列名是什么根本无所谓（因为数据是按顺序填坑的），但**数据类型必须兼容**。
- **自增主键（IDENTITY/SERIAL）**：通常不要在 `INSERT INTO` 的列里写自增主键，让数据库自动生成即可。但如果必须写（例如数据迁移），要确保目标表的主键不会发生冲突。

---

**随堂练习 50：用 `INSERT ... SELECT` 批量生成测试数据**

把所有 `CS` 专业学生临时复制成一组“Copy”测试学生：
- 姓名后加 ` Copy`，例如 `'Alice Copy'`
- 邮箱前加 `copy_`，例如 `'copy_alice@example.com'`
- 年龄保持不变
- 专业保持不变
- 不要插入 `student_id`（让其自动生成），`is_active` 和 `created_at` 依靠默认值即可。

*注意：做完后记得验证数据是否插入成功，为了不影响后续实验，可以用 `DELETE` 清理掉他们。*

---

## 把复杂查询分步：CTE（WITH）

### 1. 是什么？
CTE 的全称是 **Common Table Expressions（通用表表达式）**。
你可以把它理解为在一个复杂的 SQL 查询语句内部，**临时定义的一个或多个“虚拟结果表”**，并给它们起了名字。在这一次查询执行期间，你随时可以像引用普通表一样引用这些 CTE。

### 2. 为什么有这个东西？
在没有 CTE 之前，如果我们想基于一个复杂的统计结果进行下一步过滤，只能使用**嵌套子查询**。
例如：“查出所有学生，从中过滤出分数大于 80 的，再从中过滤出名字带 A 的……”
这就导致 SQL 变成了“洋葱代码”，一层套一层，极其难以阅读，你必须从最里面那一层往外读才能看懂逻辑。
有了 CTE 之后，你可以把逻辑**扁平化**：第一步算什么（起名叫 A），第二步算什么（起名叫 B），最后 `SELECT * FROM B`。它让 SQL 变得像写普通代码一样，有清晰的自上而下的逻辑流。

### 3. 语法是什么？
```sql
WITH CTE_Name1 AS (
    -- 这里写第一个完整的 SELECT 查询
    SELECT column1, column2 FROM some_table
),
CTE_Name2 AS (
    -- 这里写第二个查询，并且可以直接使用上面的 CTE_Name1
    SELECT column1 FROM CTE_Name1 WHERE column2 > 10
)
-- 最后的终极查询，可以使用上面定义的所有 CTE
SELECT * 
FROM CTE_Name2;
```

### 4. 与题目无关的简单例子
假设我们有一张全国城市人口表 `cities`，你想找出“人口超过平均值的北方城市”。
如果不使用 CTE，代码可能是嵌套的。使用 CTE：
```sql
WITH AvgPop AS (
    -- 第一步：算出全国平均人口
    SELECT AVG(population) AS avg_p FROM cities
),
NorthCities AS (
    -- 第二步：挑出所有的北方城市
    SELECT city_name, population FROM cities WHERE region = 'North'
)
-- 第三步：联合查询
SELECT n.city_name, n.population
FROM NorthCities n
CROSS JOIN AvgPop a
WHERE n.population > a.avg_p;
```
逻辑非常清晰：先求平均值，再找北方城市，最后对比。

### 5. 常见用法是什么？
- **多步聚合分析**：比如第一步求出每门课的平均分，第二步求出整个学校的所有课程平均分的“平均分”。
- **复用子查询**：如果你的最终查询中，需要好几次用到同一段复杂的 `JOIN` 结果（比如又要做分子又要做分母），你可以把它定义成一个 CTE，然后多次引用。
- **递归查询（进阶）**：`WITH RECURSIVE` 甚至可以用来查询树形结构（如员工上下级关系，多级分类菜单等）。

### 6. 重要注意事项
- **生命周期极短**：CTE 仅仅在**当前这一条完整的 SQL 语句**执行期间存在！它绝不是真正的建表。这个查询执行完毕，CTE 就自动消失了。
- **性能评估**：在大多数现代数据库（包括较新的 PostgreSQL）中，CTE 相当于语法糖，优化器会自动将其展开并做全局优化。但在某些旧版本或特殊情况下，它可能成为性能瓶颈，不过作为初学者，现在请**无脑优先使用 CTE 以提升代码可读性**。

---

**随堂练习 51：用 CTE 把多步统计拆解清晰**

统计“平均成绩 `>= 80` 的课程”，显示 `课程名` 和 `平均分`。
- **步骤 1**：使用 `WITH` 建立一个 CTE 叫 `course_avg`，在这个 CTE 里仅仅对 `enrollments` 表进行分组汇总，求出每门课的 `course_id` 和 `avg_score`。
- **步骤 2**：在主查询中，将 `course_avg` 与 `courses` 表进行 `JOIN`，通过 `WHERE` 筛选平均分 `>= 80` 的课程，最后按平均分降序排列。

---

**随堂练习 52：多层 CTE 的递进调用**

我们要找出：**选修了 "Database Systems" (course_name) 并且成绩高于该门课平均分的学生姓名。**
这需要三步走，请用两个 CTE 来实现：
- **CTE 1 (`db_course`)**：从 `courses` 表中查出 "Database Systems" 对应的 `course_id`。
- **CTE 2 (`db_avg`)**：结合 `db_course` 和 `enrollments`，算出这门课的平均分。
- **主查询**：结合 `db_course`、`db_avg`、`enrollments`、`students`，找出选了这门课且分数高于 `db_avg` 的人，输出 `student_name`。

*(这道题考验你如何让后一个 CTE 引用前一个 CTE！)*

---

## 事务：BEGIN / COMMIT / ROLLBACK

### 1. 是什么？
事务（Transaction）是数据库操作的**最小不可分割工作单元**。它将多条相互关联的 SQL 语句捆绑在一起，这批语句要么**全部执行成功**，要么**全部不执行**，绝对不允许出现“只成功一半”的中间状态。

### 2. 为什么有这个东西？
核心是为了保证数据的**安全性、完整性和一致性 (ACID 理论)**。
最经典的例子就是“银行转账”：A 给 B 转账 100 元。
操作 1：A 账户余额减 100。
操作 2：B 账户余额加 100。
如果操作 1 成功后，突然停电了或者报错了，操作 2 没执行，那么这 100 元就凭空消失了！有了事务，如果操作 2 失败，数据库会自动将操作 1 的状态**回滚（撤销）**，就像一切都没发生过一样。

### 3. 语法是什么？
```sql
-- 开启事务（宣布进入保护伞模式）
BEGIN;  -- 也可以写 START TRANSACTION;

-- 执行各种增删改查
UPDATE accounts SET balance = balance - 100 WHERE name = 'A';
UPDATE accounts SET balance = balance + 100 WHERE name = 'B';

-- 如果确认一切无误，提交事务，永久保存！
COMMIT;

-- 如果中间发现不对劲，或者外部应用捕获了报错，回滚事务，撤销所有修改！
ROLLBACK;
```

### 4. 与题目无关的简单例子
假设电商系统有人下单：
```sql
BEGIN;
-- 1. 生成订单记录
INSERT INTO orders (user_id, total) VALUES (99, 500);
-- 2. 扣减商品库存
UPDATE inventory SET stock = stock - 1 WHERE product_id = 10;
-- 此时，如果发现库存不足报错了，我们执行：
ROLLBACK;
-- 如果完全没问题：
COMMIT;
```

### 5. 常见用法是什么？
- **资金/积分变动**：绝对需要事务。
- **多表级联写入**：例如注册用户时，不仅在 `users` 表里写入数据，还要在 `user_profiles` 里写入初始资料，还要在 `permissions` 表里写入默认权限。必须用事务包起来。
- **危险运维操作的“后悔药”**：你想手动 `UPDATE` 生产环境的一批数据，但不确定 `WHERE` 条件写得对不对。你可以先敲 `BEGIN;`，然后执行 `UPDATE`，接着写一个 `SELECT` 看看改了哪些行。如果发现连全表都改了（没加 WHERE），马上大喊一声 `ROLLBACK;` 救你一命。如果没问题，才 `COMMIT;`。

### 6. 重要注意事项
- **自动提交（Auto-Commit）坑**：很多客户端工具（如 DataGrip、DBeaver、JDBC 驱动）默认开启了自动提交模式。这意味着你没写 `BEGIN` 时，每执行一条 SQL，工具都会自动在背后偷偷给你加上 `BEGIN` 和 `COMMIT`。因此要测试事务，必须明确手写 `BEGIN;` 或在客户端设置里关闭 Auto-Commit。
- **不能把事务开太久**：事务执行期间，被你 `UPDATE` / `DELETE` 的行往往会被加锁（Lock）。如果你写了 `BEGIN`，改了数据，然后去吃午饭了没 `COMMIT`，其他想要修改这些数据的程序全都会被卡死！**事务要尽可能的短、快**。
- **PostgreSQL 里的神级特性**：在 PostgreSQL 中，你甚至可以在事务里撤销**改表结构（DDL）**！比如 `BEGIN; DROP TABLE students; ROLLBACK;`，表又会回来。而在 MySQL 中，DDL 语句一旦执行会强制立刻提交，无法回滚。

---

**随堂练习 53：体验事务的反悔机制（ROLLBACK）**

1. 执行 `BEGIN;`
2. 用 `UPDATE` 把 Alice 的专业改成 `'Math'`。
3. 执行 `SELECT` 确认 Alice 现在的专业确实是 Math。
4. 哎呀，改错了！执行 `ROLLBACK;`
5. 再次执行 `SELECT`，确认 Alice 的专业恢复成了原本的 `'CS'`。

---

**随堂练习 54：体验事务的确认机制（COMMIT）**

1. 执行 `BEGIN;`
2. 再次用 `UPDATE` 把 Alice 的专业改成 `'Math'`。
3. 执行 `COMMIT;`
4. 再次执行 `SELECT` 确认修改已经永久生效。

---

**随堂练习 55：复杂的联合更改回滚**

我们要开除一名学生，并清理他的选课记录，如果不一起做就容易出数据脏块。
1. `BEGIN;`
2. 删除选课表 (`enrollments`) 中 `student_id = 6` (Frank) 的所有记录。
3. 删除学生表 (`students`) 中 `student_id = 6` 的记录。
4. 假设现在领导说“算了，再给他一次机会”，请使用 `ROLLBACK;` 撤销上述删除。
5. 使用 `SELECT` 确认 Frank 依然在学生表和选课表中。

---

## 改结构：ALTER TABLE

### 1. 是什么？
在数据库运行了一年之后，业务发展了，原本设计的表结构不再满足需求（比如需要给用户增加一个“微信号”字段，或者把“姓名”列加长）。
`ALTER TABLE` 允许你在**不删除原表、不丢失数据**的情况下，动态修改表的结构。

### 2. 为什么有这个东西？
如果没有它，你需要：创建一个新表 -> 把旧表几百万条数据导出来 -> 转换格式 -> 塞进新表 -> 删掉旧表。不仅速度慢，还要停机维护。`ALTER TABLE` 让你可以像给飞行的飞机换引擎一样修改结构。

### 3. 语法是什么？
```sql
-- 增加一列
ALTER TABLE 表名 ADD COLUMN 列名 数据类型;

-- 删除一列
ALTER TABLE 表名 DROP COLUMN 列名;

-- 修改列的数据类型
ALTER TABLE 表名 ALTER COLUMN 列名 TYPE 新数据类型;

-- 增加一个约束 (例如唯一约束)
ALTER TABLE 表名 ADD CONSTRAINT 约束名 UNIQUE(列名);
```

### 4. 与题目无关的简单例子
```sql
-- 假设有个 orders 表，我想加个备注字段
ALTER TABLE orders ADD COLUMN remarks VARCHAR(200);

-- 把金额的数据类型从 INTEGER 扩大到 NUMERIC(10,2) 以支持小数
ALTER TABLE orders ALTER COLUMN amount TYPE NUMERIC(10,2);
```

### 5. 常见用法是什么？
- **业务迭代扩充**：最常见的就是 `ADD COLUMN`。
- **数据清洗善后**：早期的 `phone` 字段随便大家填，后来清洗完脏数据后，通过 `ALTER TABLE ... ADD CONSTRAINT` 强制加上 `CHECK` 规则。

### 6. 重要注意事项
- **带默认值的新增列可能很慢**：在较老的数据库版本中，如果你新增一列且指定了 `DEFAULT 'xxx'`，数据库会强行把全表百万行数据全部重写一遍塞入这个默认值，这会导致整张表被长时间锁死。不过 PostgreSQL 11 之后优化了这点，带常量默认值的添加几乎是瞬间完成的。
- **DROP COLUMN 的不可逆转**：删除列会导致该列数据永久丢失。
- **修改类型可能失败**：如果你要把 `VARCHAR` 转成 `INTEGER`，但这一列里已经有一些存了 `'abc'` 的脏数据，那么 `ALTER COLUMN TYPE` 会直接报错阻止你。你需要结合 `USING` 子句来指明转换规则。

---

**随堂练习 56：给学生表增加新字段**

随着学校系统的升级，我们需要记录学生的手机号码。
请使用 `ALTER TABLE` 给 `students` 表增加一列 `phone`，数据类型为 `VARCHAR(20)`。
*验证：新增后，用 `SELECT * FROM students` 观察，所有现有学生的手机号默认都是 `NULL`。*

---

**随堂练习 57：为新增的列添加约束**

学校要求，录入的手机号长度至少要大于等于 8 位。
请使用 `ALTER TABLE ... ADD CONSTRAINT ... CHECK (...)`，为刚添加的 `phone` 列增加一个校验规则（名字可以叫 `check_phone_length`）。
*(提示：在 PostgreSQL 中测量字符串长度可以用 `LENGTH(phone) >= 8`)*

---

## 视图（VIEW）：保存查询的捷径

### 1. 是什么？

视图是**一段被保存下来的 SQL 查询语句，并在数据库里被当做一张“虚拟表”来对待**。

一旦建好，你就可以对这个视图使用 `SELECT * FROM 视图名`，就像查询真表一样。

拆开看这个定义里有两个关键词：

```text
“被保存下来的 SQL 查询语句”  →  它本质上是一段 SELECT，不是数据
“被当做一张虚拟表来对待”      →  你可以像查表一样查它，但它本身不存数据
```

所以视图和真表的区别是：

| | 真表（TABLE） | 视图（VIEW） |
|---|---|---|
| 存不存数据 | 存，数据在硬盘上 | 不存，只存一段查询语句 |
| 数据从哪来 | 直接存在表里 | 每次查询时当场去底层表现算 |
| 能不能 INSERT | 能 | 简单视图能，复杂视图不能 |
| 占不占硬盘空间 | 占（数据本身） | 几乎不占（只占语句那点空间） |

一个非常实用的记忆法：

```text
视图 = 给一段复杂的 SELECT 起了个名字，以后用名字代替整段 SQL
```

就像你在 Excel 里给一个复杂公式定义了一个名称，以后直接用名称引用它，而不用每次重写公式。

---

### 2. 为什么有这个东西？

#### 用途一：隐藏复杂逻辑（化繁为简）

某个月度财务报表需要 8 个 `JOIN` 加一堆复杂的 `GROUP BY`。如果你每次查都重写一遍，容易写错又浪费时间。

把它包装成一个视图：

```sql
CREATE VIEW finance_report_view AS
SELECT ... -- 8个JOIN + 复杂GROUP BY
FROM ...;
```

所有看报表的人只需：

```sql
SELECT * FROM finance_report_view;
```

不用关心背后到底 JOIN 了几张表、过滤了什么条件。

#### 用途二：安全与权限隔离

假设员工表 `employees` 里有薪水列、身份证号列。你不想把这个表暴露给实习生。

你可以创建一个不含敏感列的视图：

```sql
CREATE VIEW public_emps AS
SELECT id, name, department
FROM employees;
```

然后只给实习生查这个视图的权限：

```sql
GRANT SELECT ON public_emps TO intern_role;
```

实习生能看到姓名和部门，但永远看不到薪水和身份证号——因为视图里根本没有这两列。

#### 用途三：兼容性隔离（底层表改了，上层代码不动）

底层表结构大改了（字段重命名了等），为了不让上层的几十个应用代码跟着改，可以创建一个视图，名字和老表一样，里面的字段名 alias 回老名字，骗过应用代码。

例如原来的表叫 `users`，有列 `username`。后来重构把表拆了，`username` 列移到了 `user_profiles` 表里。但老应用还在写 `SELECT username FROM users`。

你可以建一个视图：

```sql
CREATE VIEW users AS
SELECT u.id, p.username
FROM user_accounts u
JOIN user_profiles p ON u.id = p.user_id;
```

老应用完全不用改，继续 `SELECT username FROM users` 就能正常工作。

---

### 3. 语法是什么？

#### 最基本的创建语法

```sql
CREATE VIEW 视图名称 AS
SELECT 字段1, 字段2...
FROM 表名
WHERE 条件...;
```

#### 创建或替换（如果已存在则覆盖）

```sql
CREATE OR REPLACE VIEW 视图名称 AS
SELECT ...;
```

> `OR REPLACE` 的意思是：如果这个名字的视图已经存在，就用新的定义替换掉它，而不是报错。这在迭代开发中非常常用——你改了视图逻辑，直接重跑这条语句就行。

但要注意：`CREATE OR REPLACE VIEW` 对已有列有严格限制——已有列的**名字、顺序和数据类型**必须保持不变。不过，它**允许在列的末尾追加新列**。如果你需要删除列、在中间插入列、改变已有列的顺序或类型，就需要先 `DROP VIEW` 再 `CREATE VIEW`。

#### 删除视图

```sql
DROP VIEW 视图名称;
```

如果视图不存在会报错，可以加 `IF EXISTS` 避免：

```sql
DROP VIEW IF EXISTS 视图名称;
```

#### 给视图的列起别名

你可以在视图名后面直接指定列名：

```sql
CREATE VIEW student_grades (姓名, 课程, 分数) AS
SELECT s.student_name, c.course_name, e.score
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id;
```

这样查出来的列名就是 `姓名`、`课程`、`分数`，而不是底层表的 `student_name` 等。

> 注意：PostgreSQL 中用中文列名是可以的，但实际项目中建议还是用英文+下划线，避免编码和工具兼容性问题。

---

### 4. 与题目无关的简单例子

#### 例子一：查看哪些课程没人选

```sql
CREATE VIEW empty_courses AS
SELECT c.course_name
FROM courses c
LEFT JOIN enrollments e ON c.course_id = e.course_id
WHERE e.student_id IS NULL;

-- 以后直接查：
SELECT * FROM empty_courses;
```

#### 例子二：只看活跃学生

```sql
CREATE VIEW active_students AS
SELECT student_id, student_name, major, created_at
FROM students
WHERE is_active = TRUE;
```

以后任何查询只要想排除已退学的学生，直接 `FROM active_students` 就行，不用每次都写 `WHERE is_active = TRUE`。

#### 例子三：把三表 JOIN 封装成成绩单宽表

```sql
CREATE VIEW transcript AS
SELECT
    s.student_name,
    s.major,
    c.course_name,
    e.score,
    e.enrolled_at
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
WHERE e.score IS NOT NULL;
```

老师查 90 分以上的成绩单：

```sql
SELECT * FROM transcript WHERE score >= 90;
```

查某个学生的全部成绩：

```sql
SELECT * FROM transcript WHERE student_name = 'Alice';
```

都不用再写 JOIN 了。

---

### 5. 常见用法是什么？

- **把基础过滤条件封在视图里**：比如所有表都有 `is_deleted = false`、`status = 'active'` 这种软删除/状态过滤，写一个视图统一处理，避免每个查询都漏写。
- **把核心几张表 JOIN 好做宽表**：数据分析师经常需要学生+选课+课程的关联数据，建一个视图一次性 JOIN 好，分析师直接查视图。
- **给不同角色提供不同粒度的数据**：管理层看汇总视图，普通员工看明细视图，通过视图控制每个人能看到什么。
- **兼容老代码**：表结构重构后，用视图模拟旧表结构，让上层应用零改动过渡。

---

### 6. 重要注意事项（最容易犯错的地方！）

#### 注意一：普通视图绝对不存数据！！

这是初学者最容易误解的一点。

当你执行 `SELECT * FROM some_view` 时，数据库底层做的事情是：

```text
1. 把视图背后保存的那段长 SQL 拿出来
2. 跟你当前写的查询拼接到一起
3. 当场去底层真实表里现查
```

所以，**视图不能提升哪怕一丁点查询性能**。它纯粹只是为了可维护性和安全性。

> 打个比方：视图就像你把一个复杂的搜索条件存成了浏览器书签。你点书签打开搜索，浏览器还是要当场去网上搜一遍，并不会因为你存了书签就搜得更快。

如果你视图背后的 SQL 本身很慢（比如 8 个 JOIN 大表），那查视图一样慢，甚至可能因为拼接了额外条件导致优化器更难优化。

#### 注意二：视图能更新吗？

非常简单的单表视图（没有聚合、没有 JOIN、没有 DISTINCT、没有 UNION）是支持 `UPDATE / INSERT / DELETE` 的，更改会自动透传到原表。

例如：

```sql
CREATE VIEW active_students AS
SELECT student_id, student_name, major
FROM students
WHERE is_active = TRUE;

-- 这个 UPDATE 会直接改 students 表
UPDATE active_students
SET major = 'Math'
WHERE student_id = 1;
```

但如果视图包含 `GROUP BY`、`JOIN`、聚合函数、`DISTINCT` 等，通常就是**只读**的，你不能对它执行 `INSERT/UPDATE/DELETE`。

PostgreSQL 会明确告诉你：

```text
ERROR: cannot update a view because it contains aggregates
```

#### 注意三：WITH CHECK OPTION——防止通过视图插入“看不见”的数据

考虑这个视图：

```sql
CREATE VIEW active_students AS
SELECT * FROM students WHERE is_active = TRUE;
```

你可以通过这个视图插入一个 `is_active = FALSE` 的学生吗？

```sql
INSERT INTO active_students (student_name, is_active)
VALUES ('Bob', FALSE);
```

答案是：**默认可以**。插入成功了，但这个学生不会出现在 `active_students` 视图里（因为他不活跃），你“插进去就找不到了”。

为了防止这种情况，PostgreSQL 提供了 `WITH CHECK OPTION`：

```sql
CREATE VIEW active_students AS
SELECT * FROM students WHERE is_active = TRUE
WITH CHECK OPTION;
```

现在如果你尝试插入 `is_active = FALSE` 的学生，数据库会直接报错：

```text
ERROR: new row violates check option for view "active_students"
```

> 记忆法：`WITH CHECK OPTION` = “通过这个视图插入或修改的数据，必须能被这个视图自己看到”。

#### 注意四：视图可以嵌套，但别套太深

视图可以基于另一个视图创建：

```sql
CREATE VIEW v1 AS SELECT ... FROM students;
CREATE VIEW v2 AS SELECT ... FROM v1;
CREATE VIEW v3 AS SELECT ... FROM v2;
```

这在语法上完全合法，但嵌套太深会带来两个问题：
1. **调试困难**——出了问题要一层层剥开看底层到底查了什么。
2. **优化器可能无法高效优化**——多层视图拼接后的 SQL 可能非常复杂，导致执行计划变差。

经验法则：视图嵌套尽量不超过 2-3 层。

#### 注意五：视图里的 ORDER BY 不可靠

你可能想写：

```sql
CREATE VIEW sorted_students AS
SELECT * FROM students ORDER BY student_name;
```

然后以为查这个视图出来的数据一定是按名字排好序的。

**错！** SQL 标准规定，只有最外层查询的 `ORDER BY` 才保证排序。视图里的 `ORDER BY` 在很多数据库中会被忽略或不保证生效（PostgreSQL 允许但不推荐，因为当你对视图再做过滤或 JOIN 时，顺序就乱了）。

正确做法是：查视图的时候再排序。

```sql
SELECT * FROM sorted_students ORDER BY student_name;
```

#### 注意六：物化视图（Materialized View）——真的存数据的视图

这是 PostgreSQL 的高级功能。它和普通视图不同，它是**真的把查询结果算出来存在硬盘上**，查询极快！

```sql
CREATE MATERIALIZED VIEW sales_summary AS
SELECT date, SUM(amount) AS total
FROM orders
GROUP BY date;
```

但原表数据更新时，它不会自动同步，需要人工执行：

```sql
REFRESH MATERIALIZED VIEW sales_summary;
```

所以物化视图适合：
- 数据不经常变（比如每日统计、历史报表）
- 查询本身非常慢，且需要反复查
- 能容忍数据有一定延迟

普通视图适合：
- 数据实时性要求高
- 主要为了简化查询和权限控制

两者对比：

| | 普通视图（VIEW） | 物化视图（MATERIALIZED VIEW） |
|---|---|---|
| 存不存数据 | 不存 | 存，结果集在硬盘上 |
| 查询速度 | 取决于底层 SQL | 极快，直接读结果 |
| 数据实时性 | 实时，每次现查 | 有延迟，需手动 REFRESH |
| 适用场景 | 简化查询、权限控制 | 慢查询加速、报表预计算 |

---

**随堂练习 58：创建基础数据视图**

创建一个名为 `active_students_view` 的视图，该视图只包含 `is_active = TRUE` 的学生信息。
创建完成后，执行 `SELECT * FROM active_students_view;` 验证。

---

**随堂练习 59：创建多表关联大宽表视图**

每次查选课明细都要写 `JOIN` 太累了。
请创建一个名为 `student_course_details` 的视图，该视图需要整合三张表，输出如下列：
- `student_name`
- `major`
- `course_name`
- `score`
- （过滤条件：成绩必须非 NULL 才出现）。

以后老师想查成绩单，直接 `SELECT * FROM student_course_details WHERE score >= 90;` 就可以了。请创建完毕后试跑一下这句查询验证。

---

**随堂练习 60：WITH CHECK OPTION 安全门实验**

创建一个名为 `active_students_check` 的视图，只包含 `is_active = TRUE` 的学生，选取 `student_id, student_name, major, is_active` 四列。要求：**通过这个视图插入或修改的数据，必须也能被这个视图自己看到**（提示：在视图定义末尾加上 `WITH CHECK OPTION`）。

创建完成后，请依次完成以下操作并记录结果：
1. 尝试通过这个视图插入一个 `is_active = FALSE` 的学生（姓名、专业自定），记录数据库返回了什么。
2. 尝试通过这个视图把 Alice 的 `is_active` 改成 `FALSE`，记录数据库返回了什么。
3. 去掉 `WITH CHECK OPTION` 重新创建该视图，再次执行上面两条插入/更新操作，这次能成功吗？成功后，这条数据还能在 `active_students_check` 视图里查询到吗？请自己实验验证并解释原因。

---

**随堂练习 61：修改视图定义与视图嵌套**

视图不是建完就不能改了。这道题练习修改视图的正确姿势，以及视图嵌套。

假设题 58 中你创建的 `active_students_view` 选取了 `student_id, student_name, major` 三列，过滤条件是 `is_active = TRUE`。（如果你当时选了其他列，请相应调整。）

1. **用 `CREATE OR REPLACE VIEW` 修改过滤条件（列不变）**：学校希望这个视图只显示成年学生。请在保持原有三列不变的前提下，使用 `CREATE OR REPLACE VIEW` 给视图增加 `age >= 18` 的过滤条件。修改后用 `SELECT * FROM active_students_view;` 验证。

2. **用 `CREATE OR REPLACE VIEW` 在末尾追加列（合法操作）**：学校又希望这个视图能额外看到学生的 `email` 和 `created_at`。请使用 `CREATE OR REPLACE VIEW`，在原有三列**末尾**追加这两列（已有列的名字和顺序保持不变）。修改后用 `SELECT` 验证新列出现了。

3. **尝试用 `CREATE OR REPLACE VIEW` 改变已有列的顺序（会报错）**：请尝试用 `CREATE OR REPLACE VIEW` 把 `major` 列移到 `student_name` 前面（改变已有列的顺序），观察 PostgreSQL 报什么错，并结合上面的知识点解释为什么。

4. **用 `DROP VIEW` + `CREATE VIEW` 重新定义视图（改变列顺序）**：上面的操作失败了，因为 `CREATE OR REPLACE VIEW` 不能改变已有列的顺序。请先用 `DROP VIEW` 删除该视图，再用 `CREATE VIEW` 重新创建，这次把列顺序改成 `student_id, major, student_name, email, created_at`，过滤条件保持 `is_active = TRUE AND age >= 18`。创建后用 `SELECT` 验证。

5. **视图嵌套**：基于修改后的 `active_students_view` 再创建一个名为 `cs_active_students` 的视图，只保留 `major = 'CS'` 的学生。创建后用 `SELECT * FROM cs_active_students;` 验证。

6. 查询 `cs_active_students` 时，数据库底层实际查了几张表？请结合"视图不存数据"的知识思考答案。
---

## 索引（INDEX）与 EXPLAIN 分析

### 1. 是什么？

**索引（Index）** 就像是书本背后的目录，或者字典里的部首检字表。

在数据库里，最常用的索引是一种称为 **B-Tree（平衡树）** 的数据结构，它独立存储在硬盘上，和表数据是分开存的。

`EXPLAIN` 则是数据库提供的一个“透视镜”，它告诉你，数据库在执行你的 SQL 时，到底打不打算使用这本目录，以及打算怎么执行。

#### B-Tree 到底长什么样？

你不用去手写 B-Tree，但要理解它的核心特点：

```text
B-Tree 是一棵“排好序的平衡树”
├── 所有叶子节点在同一层（所以叫“平衡”）
├── 每个节点里存的是“索引列的值 + 指向表中对应行的指针”
├── 查找时从根节点开始，每次比较后走左或右子树
└── 因为是平衡的，不管找哪个值，比较次数都差不多（树的高度）
```

打个比方：你在字典里找“张”字。
- 没有索引（全表扫描）：从字典第一页开始一页页翻，直到找到“张”。
- 有 B-Tree 索引（部首检字）：先找部首“弓”（3画），翻到对应页码，再在里面找“张”，几步就到了。

对于一张 1000 万行的表，B-Tree 的高度通常只有 3-4 层，意味着找任何一行最多只需要 3-4 次硬盘读取。而全表扫描可能要读几百万行。

#### PostgreSQL 还有哪些索引类型？

初学阶段你只需要知道 B-Tree 就够了，但了解一下其他类型有助于以后遇到时不慌：

| 索引类型 | 适用场景 | 初学需要掌握吗 |
|---|---|---|
| B-Tree | 等值查询、范围查询、排序，最通用 | 必须掌握 |
| Hash | 只支持等值查询（`=`），不支持范围 | 了解即可 |
| GIN | 数组、全文搜索、JSONB 键值查询 | 以后用到再学 |
| GiST | 地理空间数据、范围类型 | 以后用到再学 |
| BRIN | 超大表、数据物理上有序（如时间序列） | 以后用到再学 |

> 99% 的初学者场景，`CREATE INDEX` 默认创建的就是 B-Tree，完全够用。

---

### 2. 为什么有这个东西？

如果表里有 1000 万个学生，你要找 `email = 'bob@example.com'` 的人。

#### 没有索引：全表扫描（Sequential Scan）

数据库只能使用最笨的办法：从第 1 行读到第 1000 万行，逐行比较 email 是不是 `'bob@example.com'`。

```text
读第1行 → 不是 → 读第2行 → 不是 → ... → 读第500万行 → 找到了！
```

这可能耗时好几秒，甚至几十秒。而且不管数据在第几行，平均都要扫一半。

#### 有索引：索引扫描（Index Scan）

数据库会去 B-Tree 里进行查找：

```text
根节点比较 → 走左子树 → 再比较 → 走右子树 → 叶子节点找到值 → 拿到行指针 → 去表里取数据
```

只需要几次比对（几毫秒）就能精准拿到数据在硬盘上的物理位置。

#### 对比一下

| | 全表扫描（Seq Scan） | 索引扫描（Index Scan） |
|---|---|---|
| 做法 | 一行一行读全表 | 先查 B-Tree 定位，再取数据 |
| 小表（几十行） | 快，甚至更快 | 反而慢（翻目录的开销 > 直接读） |
| 大表（百万行以上） | 慢 | 快 |
| 取大部分数据时 | 合适 | 不合适（每条都要回表取数据） |
| 取少量数据时 | 不合适 | 非常合适 |

> 关键直觉：索引不是“让所有查询都变快”，而是“让从大量数据中找少量数据变得极快”。如果你要取表里 80% 的行，用索引反而更慢——因为每条都要先查树再回表，不如直接顺序读全表。

---

### 3. 语法是什么？

#### 创建普通索引

```sql
CREATE INDEX 索引名称 ON 表名 (列名);
```

例如：

```sql
CREATE INDEX idx_students_email ON students(email);
```

#### 创建唯一索引

唯一索引不但加速查询，还强制该列的值不能重复：

```sql
CREATE UNIQUE INDEX 索引名称 ON 表名 (列名);
```

> 注意：当你给列加 `PRIMARY KEY` 或 `UNIQUE` 约束时，PostgreSQL 会**自动**在后台帮你创建一个唯一索引。所以你不需要手动再建。

#### 创建复合索引（多列索引）

```sql
CREATE INDEX 索引名称 ON 表名 (列1, 列2, ...);
```

例如：

```sql
CREATE INDEX idx_enrollments_student_course
ON enrollments(student_id, course_id);
```

复合索引有一个非常重要的规则——**最左前缀原则**，后面注意事项会详细讲。

#### 创建部分索引（只索引满足条件的行）

```sql
CREATE INDEX 索引名称 ON 表名 (列名)
WHERE 条件;
```

例如，你只想给活跃学生的 email 建索引（因为已退学的学生几乎不会被查）：

```sql
CREATE INDEX idx_active_students_email
ON students(email)
WHERE is_active = TRUE;
```

部分索引更小、更快、维护成本更低，非常适合“只查某一类数据”的场景。

#### 创建表达式索引（对函数/表达式的结果建索引）

```sql
CREATE INDEX 索引名称 ON 表名 (表达式);
```

例如，你经常按邮箱的小写形式查询（避免大小写问题）：

```sql
CREATE INDEX idx_students_lower_email
ON students(LOWER(email));
```

这样 `WHERE LOWER(email) = 'bob@example.com'` 就能用上索引了。

> 这正好对应后面“索引失效”里说的“不要在索引列上做函数”——如果你确实需要用函数查询，那就给函数表达式建索引，而不是给原列建索引。

#### 删除索引

```sql
DROP INDEX 索引名称;
```

避免报错可以加 `IF EXISTS`：

```sql
DROP INDEX IF EXISTS idx_students_email;
```

#### 查看表上有哪些索引

在 PostgreSQL 中：

```sql
-- 方法一：用系统视图查
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'students';

-- 方法二：在 psql 里用 \d 表名
\d students
```

---

### 4. EXPLAIN：怎么看执行计划？

`EXPLAIN` 是你和数据库优化器对话的工具。它不会真的执行查询，只是告诉你“如果执行，我打算这么干”。

#### 基本用法

```sql
EXPLAIN SELECT * FROM students WHERE email = 'bob@example.com';
```

输出大概长这样：

```text
Index Scan using idx_students_email on students
   Index Cond: (email = 'bob@example.com'::text)
```

#### EXPLAIN ANALYZE：真的跑一遍，给你真实数据

```sql
EXPLAIN ANALYZE SELECT * FROM students WHERE email = 'bob@example.com';
```

`EXPLAIN ANALYZE` 会**实际执行**这条 SQL，然后告诉你每一步花了多少时间、实际返回了多少行。这是排查慢查询最有用的工具。

> 注意：`EXPLAIN ANALYZE` 对 `SELECT` 是安全的，但如果是 `UPDATE/DELETE`，它会真的修改数据！对写操作要用 `EXPLAIN ANALYZE` 时要格外小心，或者包在事务里执行完再 ROLLBACK。

#### 执行计划输出怎么读？

看一个典型输出：

```text
Hash Join  (cost=100.00..500.00 rows=1000 width=100)
  Hash Cond: (e.student_id = s.student_id)
  ->  Seq Scan on enrollments e  (cost=0.00..200.00 rows=5000 width=20)
  ->  Hash  (cost=80.00..80.00 rows=1000 width=80)
        ->  Seq Scan on students s  (cost=0.00..80.00 rows=1000 width=80)
              Filter: (is_active = true)
```

读执行计划的要点：

1. **从最内层、最缩进的地方开始读**，那是最先执行的步骤。上面的例子中，先扫 `students` 表（过滤活跃学生），把结果放进 Hash 表，同时扫 `enrollments` 表，然后做 Hash Join。

2. **扫描类型是关键**：
   - `Seq Scan` = 全表扫描，没有用索引
   - `Index Scan` = 用了索引，先查树再回表取数据
   - `Index Only Scan` = 只用索引就够了，不用回表（最快！说明查询的列都在索引里）
   - `Bitmap Index Scan` + `Bitmap Heap Scan` = 先用索引建位图，再批量回表取数据（适合取中等量数据）

3. **`cost` 是估算成本**，不是真实时间。第一个数字是启动成本，第二个是总成本。数字越小越好，但主要用来对比不同执行计划，不用纠结绝对值。

4. **`rows` 是估算返回行数**。如果估算行数和实际行数差很多（比如估算 10 行实际 10000 行），说明统计信息过期了，优化器可能选错执行计划。这时候可以跑 `ANALYZE 表名;` 更新统计信息。

5. **`Filter` vs `Index Cond`**：
   - `Index Cond` = 用索引直接过滤的条件（好，说明索引起作用了）
   - `Filter` = 取出行后再过滤的条件（如果过滤掉大量行，说明索引没覆盖到这个条件）

---

### 5. 与题目无关的简单例子

#### 例子一：电商网站查找订单

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 用户在前端点击“我的订单”时，数据库能瞬间找出他买过的东西
EXPLAIN SELECT * FROM orders WHERE user_id = 12345;
```

#### 例子二：按时间范围查询

```sql
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- B-Tree 天然支持范围查询
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at < '2024-02-01';
```

#### 例子三：复合索引加速多条件查询

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- 两个条件都能用上复合索引
EXPLAIN SELECT * FROM orders
WHERE user_id = 12345 AND status = 'shipped';
```

---

### 6. 常见用法是什么？

- **为经常放在 `WHERE` 后面做等值或范围查询的列建索引**：如 `email = '...'`、`age > 20`、`created_at BETWEEN ...`。
- **为经常用于 `JOIN` 连接条件的列建索引**：如外键 `student_id`、`course_id`。多表 JOIN 时，如果连接列没有索引，数据库只能做嵌套循环全表扫描，极慢。
- **为经常需要排序的列建索引**：`ORDER BY created_at`。因为 B-Tree 索引本身就是排好序的，数据库可以直接顺着索引读，免去临时排序的巨大开销。
- **`PRIMARY KEY` 和 `UNIQUE` 约束会自动建索引**：不用你手动建，建了也是重复。

---

### 7. 重要注意事项（最容易犯错的地方！）

#### 注意一：索引的代价——天下没有白吃的午餐

每一条插入、修改、删除操作（`INSERT/UPDATE/DELETE`），数据库不但要改表里的数据，还得顺带修改所有相关的索引（维护每一棵 B-Tree）。

所以：

```text
索引建得越多
├── 写入速度越慢（每次写都要维护多棵树）
├── 占用的硬盘空间越庞大（索引本身也要存）
└── 优化器选择执行计划时越纠结（选项太多反而可能选错）
```

**切忌给每个列都建索引！** 一个表有 3-5 个精心设计的索引通常就够了。

> 经验法则：先写查询，再根据查询模式建索引。而不是先建一堆索引再写查询。

#### 注意二：有索引 ≠ 一定会用索引

这是无数新手的误区。

如果你有一张仅有 10 个人的学生表，你为 `major` 建了索引。当你执行：

```sql
SELECT * FROM students WHERE major = 'CS';
```

优化器会掐指一算：

```text
去翻目录（读B-Tree）→ 拿到行指针 → 按指针去表里取数据
vs
直接把这10行扫一遍
```

对于只有 10 行的表，全表扫描可能只需要一次硬盘读取（一页就装下了），而索引扫描要先读索引页再读数据页，反而更慢。

于是优化器会主动放弃索引，强行走 `Seq Scan`。

> 这不是 bug，这是优化器在帮你选最快的方案。索引是给大表用的，小表全表扫描就是最优解。

优化器做决策依赖的是**统计信息**。如果统计信息不准（比如表刚导入大量数据但没跑 `ANALYZE`），优化器也可能选错。这时候可以手动执行：

```sql
ANALYZE 表名;
```

来更新统计信息。

#### 注意三：索引失效的常见场景（重点中的重点！）

建了索引但查询没用到，90% 的情况是以下几种：

##### 场景 1：在索引列上用函数或运算

```sql
-- 索引失效！YEAR(created_at) 是函数，不是裸列
WHERE YEAR(created_at) = 2023

-- 正确写法：把函数挪到值那边，保持列是裸的
WHERE created_at >= '2023-01-01'
  AND created_at < '2024-01-01'
```

```sql
-- 索引失效！age + 1 是运算
WHERE age + 1 = 20

-- 正确写法
WHERE age = 19
```

> 记忆法：**索引列必须是“裸”的，不能被函数包着，不能参与运算。** 如果你确实需要按函数查，就建表达式索引（前面讲过）。

##### 场景 2：隐式类型转换

```sql
-- 如果 student_id 是 INTEGER 类型，你传了字符串
WHERE student_id = '123'
```

PostgreSQL 通常会自动转换，但某些数据库（如 MySQL）中这会导致索引失效。养成习惯：**查询参数的类型和列的类型保持一致**。

##### 场景 3：LIKE 左模糊

```sql
-- 索引失效！通配符在开头，B-Tree 没法定位
WHERE email LIKE '%@example.com'

-- 索引有效！通配符在结尾，可以用前缀匹配
WHERE email LIKE 'bob%'
```

> B-Tree 是按值排序的，`'bob%'` 可以定位到以 'bob' 开头的区间，但 `'%@example.com'` 没有固定前缀，只能全表扫。

##### 场景 4：OR 连接的条件中有一列没索引

```sql
WHERE email = 'bob@example.com' OR phone = '12345678'
```

如果 `email` 有索引但 `phone` 没有，数据库可能干脆全表扫描，因为要满足 OR 就得把所有行都检查一遍。

解决办法：给 `phone` 也建索引，或者用 `UNION` 改写。

##### 场景 5：NOT / != / <>

```sql
WHERE status != 'active'
WHERE NOT status = 'active'
```

这种“不等于”查询通常无法有效使用 B-Tree 索引，因为结果集可能很大。优化器往往选择全表扫描。

##### 场景 6：复合索引不满足最左前缀

复合索引 `(a, b, c)` 相当于建了三个索引：`(a)`、`(a, b)`、`(a, b, c)`。

```sql
-- 能用索引
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3

-- 不能用索引（跳过了最左列 a）
WHERE b = 2
WHERE b = 2 AND c = 3
WHERE c = 3
```

> 这就是“最左前缀原则”：复合索引只有从最左边的列开始连续使用时才生效。建复合索引时，把最常用的查询条件列放在最左边。

#### 注意四：索引选择性（Selectivity）

选择性 = 不重复值的数量 / 总行数。

- `email` 列：几乎每个人都不一样，选择性接近 1 → **非常适合建索引**
- `gender` 列：只有男/女两个值，选择性约 0.0000002 → **不适合建索引**

为什么？因为如果一个条件能匹配表里 50% 的行（比如 `gender = '女'`），用索引逐条回表取数据反而比全表扫描慢。优化器会直接放弃索引。

> 经验法则：**区分度高的列才值得建索引。** 布尔列、性别列、状态列（只有两三个值）通常不值得单独建索引，但可以和其他列组成复合索引。

#### 注意五：索引和 ORDER BY

如果你的查询既有 `WHERE` 又有 `ORDER BY`，一个好的索引可以同时满足两者：

```sql
CREATE INDEX idx_students_major_created ON students(major, created_at DESC);

-- 这个查询可以直接用索引，既过滤又排序，完全不需要额外排序
SELECT * FROM students
WHERE major = 'CS'
ORDER BY created_at DESC
LIMIT 10;
```

因为 B-Tree 里 `(major, created_at)` 本身就是先按 major 排、major 相同再按 created_at 排的，数据库顺着索引读就是排好序的。

#### 注意六：索引和 JOIN

多表 JOIN 时，**被驱动表的连接列一定要有索引**。

```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.created_at >= '2024-01-01';
```

假设数据库先扫 `orders` 表（用 `created_at` 索引过滤出今年的订单），然后对每个订单去 `customers` 表里找对应的客户。这时候 `customers.customer_id` 有主键索引（自动建的），查找极快。

但如果连接列没有索引，数据库对每个订单都要全表扫一遍 `customers`，这叫“嵌套循环连接”，慢到无法接受。

> 所以：**外键列几乎总是应该建索引。** PostgreSQL 不会自动给外键建索引（主键会自动建，但外键不会），需要你手动建。

---

**随堂练习 62：索引与 EXPLAIN 观察实验**

我们来做个实验体会优化器的聪明程度：
1. 尚未建索引时，使用 `EXPLAIN SELECT * FROM students WHERE major = 'CS';` 观察查询计划，记录下扫描类型（应该是 Seq Scan）。
2. 为 `students` 表的 `major` 列创建索引：`CREATE INDEX idx_students_major ON students(major);`。
3. 再次执行 `EXPLAIN SELECT ...`。
*观察：是不是依然变成了 Seq Scan？为什么？结合上文的知识思考原因。*

---

**随堂练习 63：如何骗过优化器强制看 Index Scan？**

我们刚才的数据太少了，优化器不屑于用索引。
要强行看到索引扫描，我们可以关闭全表扫描的倾向。在 PostgreSQL 中（可以在 DataGrip 控制台或 psql 里执行）：
1. 先执行 `SET enable_seqscan = OFF;` （这是告诉优化器：求你了，只要有索引就尽量用）。
2. 再次执行 `EXPLAIN SELECT * FROM students WHERE major = 'CS';`。
3. 这次你应该能看到赫然醒目的 `Index Scan using idx_students_major` 了！
4. 实验完记得恢复：`SET enable_seqscan = ON;`

---

**随堂练习 64：索引失效实验——函数是索引的杀手**

企业中最常见的慢查询原因之一就是"在索引列上用了函数"。请亲手实验验证：

1. 为 `students` 表的 `student_name` 列创建一个普通索引。
2. 写一条查询：找出名字叫 `Alice` 的学生。要求 `student_name` 列保持"裸列"状态（不被函数包裹），直接等值匹配。用 `EXPLAIN` 查看执行计划，记录扫描类型。
3. 再写一条查询：同样找出名字叫 `alice`（全小写）的学生，但这次在 `WHERE` 中对 `student_name` 使用 `LOWER()` 函数后再比较。用 `EXPLAIN` 查看执行计划，记录扫描类型。
4. 对比两者：为什么第二种写法用不上第 1 步创建的索引？
5. 进阶：如果你确实需要经常按小写名字查询，除了改写条件，还可以用什么索引来加速？请创建它并验证效果。

> 数据太少时优化器可能依然走 Seq Scan，可以用 `SET enable_seqscan = OFF;` 强制看索引效果，实验完记得 `SET enable_seqscan = ON;`

**随堂练习 65：复合索引与最左前缀原则**

复合索引的列顺序至关重要。请亲手验证"最左前缀原则"：

1. 为 `students` 表创建一个复合索引 `idx_students_major_age`，包含列 `(major, age)`。
2. 分别写以下三条查询，并用 `EXPLAIN` 查看执行计划，**重点记录每条查询的 `Index Cond` 里包含哪些列**：
   - 查询 A：只按 `major = 'CS'` 筛选
   - 查询 B：按 `major = 'CS' AND age >= 20` 筛选
   - 查询 C：只按 `age >= 20` 筛选（跳过第一列）
3. 思考：
   - 查询 A 和 B 的 `Index Cond` 里包含 `major` 吗？这说明什么？
   - 查询 C 的 `Index Cond` 里包含 `major` 吗？如果它依然显示了 `Index Scan`，请仔细看：它是利用 `major` 定位到了索引的某个范围，还是**从头到尾扫描了整个索引**再用 `age` 过滤？（提示：小表数据量太少时，优化器可能觉得"扫整个小索引"比"扫全表"还便宜，于是选择了全索引扫描。这并不代表最左前缀失效了——判断是否真正利用了最左前缀，关键看 `Index Cond` 里有没有最左列的定位条件。）
   - 如果业务上经常需要只按 `age` 筛选，应该怎么办？
   - 建复合索引时，列的顺序应该根据什么决定？
4. 实验完用 `DROP INDEX` 清理测试索引。
---

## 进阶巅峰：窗口函数（Window Functions）

### 1. 是什么？

窗口函数是 SQL 标准中极其强大的一部分，它可以**在不“压缩/折叠”原有行的前提下，对一系列与当前行相关的行集合（称为“窗口”）执行聚合计算或排名计算。**

拆开看这个定义里的两个关键词：

```text
“不压缩/折叠原有行”  →  原来有多少行，结果还是多少行，不会像 GROUP BY 那样把多行揉成一行
“与当前行相关的行集合” →  对每一行来说，都有一个属于它的“窗口”，窗口里是和它相关的其他行
```

所以窗口函数的执行逻辑是：

```text
对结果集中的每一行：
  1. 找到属于这一行的“窗口”（一组相关的行）
  2. 在这个窗口上执行计算（求和、排名、取上一行等）
  3. 把计算结果作为这一行的一个新列
```

> 打个比方：`GROUP BY` 就像把全班同学按班级分成几组，每组只报一个平均分（每组只剩一行）。窗口函数则是让每个同学都站在原地不动，然后在每个人旁边贴一张纸条，上面写着“你们班的平均分是多少”——人一个没少，每人多了一条信息。

---

### 2. 为什么有这个东西？

在没有窗口函数时，如果我们用 `GROUP BY`，它会把多行“揉碎”压缩成一行。

例如你想算：“全班所有人的明细成绩单，并且要在每人成绩旁边附上全班的平均分。”

#### 没有窗口函数的写法（麻烦！）

如果你用 `GROUP BY` 求了平均分，人的名字就看不到了：

```sql
-- 这样只能看到每门课的平均分，看不到每个人的明细
SELECT course_id, AVG(score) AS course_avg
FROM enrollments
GROUP BY course_id;
```

如果你不想丢掉名字，就只能写个子查询查平均分，然后再和原表 JOIN 起来：

```sql
SELECT e.student_id, e.course_id, e.score, t.course_avg
FROM enrollments e
JOIN (
    SELECT course_id, AVG(score) AS course_avg
    FROM enrollments
    GROUP BY course_id
) t ON e.course_id = t.course_id;
```

极为麻烦，而且要扫两遍表。

#### 有了窗口函数的写法（简洁！）

```sql
SELECT
    student_id, course_id, score,
    AVG(score) OVER (PARTITION BY course_id) AS course_avg
FROM enrollments;
```

一行搞定，表只扫一遍。

窗口函数让你“鱼与熊掌兼得”：**既保留了明细，又计算了宏观聚合指标。**

---

### 3. 语法是什么？

```sql
函数名(...) OVER (
    [PARTITION BY 分组列]
    [ORDER BY 排序列 [ASC|DESC]]
    [ROWS/RANGE BETWEEN ... AND ...]
)
```

拆开看每个部分：

#### 函数名

可以是三类函数：

| 类别 | 代表函数 | 作用 |
|---|---|---|
| 聚合函数 | `SUM`, `AVG`, `COUNT`, `MIN`, `MAX` | 在窗口内做聚合 |
| 排名函数 | `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` | 在窗口内排名 |
| 位移函数 | `LAG()`, `LEAD()` | 取窗口内前/后第 N 行的值 |

#### PARTITION BY（分区）

相当于窗口函数里的 `GROUP BY`，把数据划分成几个独立的小隔间分别计算。

```text
PARTITION BY course_id
  → 把所有选课记录按课程分成几个隔间
  → 每个隔间内独立计算平均分、排名等
  → 隔间之间互不影响
```

如果不写 `PARTITION BY`，就把整个结果集当做一个大窗口。

#### ORDER BY（排序）

规定在每个分区内，数据按什么顺序排列。对于排名、累计求和特别重要。

```text
ORDER BY score DESC
  → 在每个分区内按成绩从高到低排
  → ROW_NUMBER() 就按这个顺序给 1,2,3...
  → SUM() OVER (ORDER BY ...) 就按这个顺序逐行累加
```

#### ROWS/RANGE BETWEEN（窗口帧）

这是高级用法，定义“窗口到底包含哪些行”。默认情况下：

- 写了 `ORDER BY`：窗口帧是“从分区第一行到当前行”（适合累计求和）
- 没写 `ORDER BY`：窗口帧是“整个分区”（适合整体聚合）

你可以手动指定，比如移动平均：

```sql
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
-- 包含：当前行 + 前面2行，共3行
```

这个后面注意事项会详细讲。

---

### 4. 三大类窗口函数详解

#### 第一类：聚合窗口函数

把普通的聚合函数加上 `OVER()`，就变成了窗口函数：

```sql
SELECT
    student_id, course_id, score,
    AVG(score) OVER (PARTITION BY course_id) AS course_avg,
    COUNT(*) OVER (PARTITION BY course_id) AS course_student_count,
    MAX(score) OVER (PARTITION BY course_id) AS course_max_score
FROM enrollments
WHERE score IS NOT NULL;
```

每一行都附带了自己所在课程的平均分、人数、最高分，但原始明细一行没少。

#### 第二类：排名函数三兄弟（重点！）

这是最容易混淆的地方，必须搞清楚三者的区别。

假设有如下成绩（同一门课）：

```text
Alice: 95
Bob:   95
Carol: 90
Dave:  85
```

| 函数 | 结果 | 特点 |
|---|---|---|
| `ROW_NUMBER()` | 1, 2, 3, 4 | 强制连续编号，并列也强行分出先后 |
| `RANK()` | 1, 1, 3, 4 | 并列占名次，下一个名次跳号 |
| `DENSE_RANK()` | 1, 1, 2, 3 | 并列占名次，下一个名次不跳号 |

记忆法：

```text
ROW_NUMBER()  →  运动员编号，绝不重复
RANK()        →  考试排名，并列第一后下一个是第三名
DENSE_RANK()  →  密集排名，并列第一后下一个还是第二名
```

代码示例：

```sql
SELECT
    student_name, score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS rn,
    RANK()       OVER (ORDER BY score DESC) AS rk,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dr
FROM scores;
```

#### 第三类：位移函数 LAG / LEAD

`LAG(col, n)` 取当前行**前面**第 n 行的值，`LEAD(col, n)` 取当前行**后面**第 n 行的值。

最经典的用途是算环比增长率：

```sql
SELECT
    month,
    sales,
    LAG(sales, 1) OVER (ORDER BY month) AS prev_month_sales,
    ROUND(
        (sales - LAG(sales, 1) OVER (ORDER BY month))
        / LAG(sales, 1) OVER (ORDER BY month) * 100,
        2
    ) AS growth_rate_pct
FROM monthly_sales;
```

> 第一行没有“上一行”，所以 `LAG` 返回 `NULL`，增长率也是 `NULL`，这是正常的。

---

### 5. 与题目无关的简单例子

#### 例子一：累计求和

假设有个每日销售额表，我想看每一天的当日营业额，以及“从第一天截至当日的累计总营业额”：

```sql
SELECT
    date,
    daily_sales,
    SUM(daily_sales) OVER (ORDER BY date) AS cumulative_sales
FROM sales;
```

由于写了 `ORDER BY date`，`SUM` 会变成一行行往下滚雪球式累加。

#### 例子二：移动平均（3 日滑动平均）

```sql
SELECT
    date,
    daily_sales,
    AVG(daily_sales) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3d
FROM sales;
```

每一天的移动平均 = 当天 + 前两天 的平均值。前两天不足 2 行的，有几行算几行。

#### 例子三：排名三兄弟对比

```sql
SELECT
    student_name,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS rn,
    RANK()       OVER (ORDER BY score DESC) AS rk,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dr
FROM exam_scores;
```

跑一下就能直观看到三者在并列成绩时的不同编号方式。

### 例子四：累计加权平均 / 移动加权平均

如果要做“从第一行累计到当前行”的加权平均：

```sql
SELECT
    course_id,
    created_at,
    score,
    weight,
    SUM(score * weight) OVER (
        PARTITION BY course_id
        ORDER BY created_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )
    /
    NULLIF(
        SUM(weight) OVER (
            PARTITION BY course_id
            ORDER BY created_at
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ),
        0
    ) AS running_weighted_avg
FROM enrollments;
```

移动加权平均也类似，只改窗口帧：

```sql
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
```

表示当前行 + 前两行，共 3 行。

注意：**分子和分母的窗口定义必须完全一致**，否则一个是累计、一个是整体，结果就没意义。

---

### 6. 常见用法是什么？

- **携带整体统计数据**：不折叠行附带平均分、最高分、人数等。
- **分组内排名（Top N 问题）**：查询“每个班级里考试排名前 3 的学生”，用 `ROW_NUMBER()` + CTE 过滤。
- **同比/环比分析**：用 `LAG(sale, 1) OVER (ORDER BY month)` 取上个月销售额，计算环比增长率。
- **累计求和/移动平均**：用 `SUM OVER (ORDER BY ...)` 做累计，用 `ROWS BETWEEN` 做滑动窗口。
- **去重取最新**：用 `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC)` 给每个用户的记录编号，取 rn=1 就是最新一条。

---

### 7. 重要注意事项（最容易犯错的地方！）

#### 注意一：执行顺序处于最末端

窗口函数的计算发生在 `WHERE`, `GROUP BY`, `HAVING` 全部执行完毕之后，仅仅在最终返回给 `SELECT` 显示之前！

```text
SQL 执行顺序：
FROM → WHERE → GROUP BY → HAVING → 窗口函数 → SELECT → ORDER BY → LIMIT
```

所以窗口函数看到的数据，是已经被 `WHERE` 过滤、被 `GROUP BY` 聚合之后的数据。

#### 注意二：巨大的坑——绝不能把窗口函数写在 WHERE 里！！

因为上面这一条规则，**你绝不能把窗口函数写在 `WHERE` 子句里过滤数据！！**

```sql
-- 错误！！WHERE 比窗口函数先执行，不认识 ROW_NUMBER()
SELECT *
FROM enrollments
WHERE ROW_NUMBER() OVER (PARTITION BY course_id ORDER BY score DESC) = 1;
```

数据库会直接报错。

正确的做法是：**必须把窗口函数包在一个 CTE 或子查询里算好并起了别名，然后在外层通过 WHERE 去过滤。**

```sql
-- 正确：先在 CTE 里算好排名，再在外层过滤
WITH ranked AS (
    SELECT
        student_id, course_id, score,
        ROW_NUMBER() OVER (PARTITION BY course_id ORDER BY score DESC) AS rn
    FROM enrollments
    WHERE score IS NOT NULL
)
SELECT * FROM ranked WHERE rn = 1;
```

> 这是面试和实战的必考题，务必刻在脑子里。

#### 注意三：PARTITION BY 和 GROUP BY 的区别

| | GROUP BY | PARTITION BY（窗口函数） |
|---|---|---|
| 结果行数 | 折叠，每组一行 | 不折叠，原行数不变 |
| 用途 | 聚合汇总 | 给每行附加聚合/排名信息 |
| 能否同时显示明细 | 不能 | 能 |

简单记：

```text
GROUP BY     →  把数据“压缩”成汇总行
PARTITION BY →  把数据“分区”，但每行都保留，各自在区内计算
```

#### 注意四：排名函数三兄弟别用错

- 需要**强制去重编号**（比如取每个用户最新一条记录）→ 用 `ROW_NUMBER()`
- 需要**标准考试排名**（并列第一后下一个是第三）→ 用 `RANK()`
- 需要**密集排名**（并列第一后下一个还是第二）→ 用 `DENSE_RANK()`

用错了结果会差很多，尤其是有并列数据的时候。

#### 注意五：窗口帧（ROWS BETWEEN）与累计求和的关系

很多初学者疑惑：为什么 `SUM(score) OVER (ORDER BY date)` 会自动累计？

因为当你写了 `ORDER BY` 但没写窗口帧时，默认的窗口帧是：

```text
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

也就是“从分区第一行到当前行”，所以 SUM 自然就是累计的。

如果你想要的是“整个分区的总和”而不是累计，那就不要写 `ORDER BY`：

```sql
-- 整个分区的总和（每行都显示同一个总数）
SUM(score) OVER (PARTITION BY course_id)

-- 从第一行累计到当前行
SUM(score) OVER (PARTITION BY course_id ORDER BY enrolled_at)
```

两者结果完全不同，别搞混。

#### 注意六：LAG/LEAD 的第一行/最后一行为 NULL

`LAG()` 对分区第一行返回 `NULL`（因为前面没有行了），`LEAD()` 对分区最后一行返回 `NULL`。

这不是错误，是正常行为。算环比时第一行增长率为 `NULL` 是合理的，可以用 `COALESCE` 处理成 0 或保留 NULL。

---

**随堂练习 66：实战 - 保留明细的同时算平均分**

查询 `enrollments` 表中的所有选课记录，输出 `student_id`, `course_id`, `score`，同时新增一列 `course_avg`，显示**该门课的整体平均分**（即同一门课的所有学生共享同一个平均分），以便对比知道这个学生有没有拖后腿。

注意：不能用 `GROUP BY`，因为那样会把多行压缩成一行，丢失学生明细。你需要一种"既保留原始明细行，又能计算分组聚合值"的语法。

---

**随堂练习 67：实战 - 组内排名**

查询 `enrollments` 表，输出 `student_id`, `course_id`, `score`，并新增一列 `rank_in_course`，表示**该学生在自己这门课里的成绩名次**（按成绩从高到低，同分也强行分出先后）。排除没有成绩（`score IS NULL`）的记录。

（提示：需要在每个 `course_id` 分组内部按 `score` 排序编号，这正是排名类窗口函数的典型场景。）

---

**随堂练习 68：进阶综合挑战（组内 Top 1 筛选）**

领导要求：找出”所有课程中，每门课成绩排名第 1 的选课记录”，输出 `student_name`, `course_name`, `score`。

注意：排名计算不能直接写在 `WHERE` 里过滤，需要先把排名算好并起别名，然后在外层查询中过滤。（这是企业中大名鼎鼎的”求组内 Top N 数据”的标准解法！）

---

**随堂练习 69：排名三兄弟对比实验**

要体会三者的区别，需要有同分且至少 3 条记录。请先准备数据：
1. 给 Frank 增加一条选课记录：选 Database Systems（`course_id = 1`），成绩 84。（先确认 Frank 的 `student_id`，注意不要和已有选课记录冲突。）
2. 把 Carol 的 Database Systems 成绩（目前是 84）更新成 91，这样这门课就有 Alice(91)、Carol(91)、Frank(84) 三条记录，其中两人并列第一。

然后查询 `enrollments` 表中 `course_id = 1` 这门课的成绩，输出 `student_id`, `score`，并额外产生三列排名，三列都按 `score` 从高到低排名：
- 第一列 `rn`：强制连续编号，同分也强行分出先后
- 第二列 `rk`：标准考试排名，并列第一后下一个名次跳号
- 第三列 `dr`：密集排名，并列第一后下一个名次不跳号

排除 `score IS NULL` 的记录。

观察：Alice 和 Carol 同为 91 分时，三种排名给 Frank 的编号分别是多少？思考各自的适用场景。

> 实验完可以把新增的选课记录删掉、Carol 的成绩改回 84，或者保留数据供后续练习使用。

**随堂练习 70：计算成绩环比变化**

学校想观察每个学生的”成绩进步趋势”。查询 `enrollments` 表，输出：
- `student_id`
- `course_id`
- `score`
- `prev_score`：该学生**上一门课**的成绩（按 `course_id` 升序排列）
- `score_diff`：本次成绩减去上一次成绩，正数表示进步，负数表示退步

注意：要按学生分开比较（同一个学生的不同课程之间比较），每个学生的第一门课没有”上一门”，`prev_score` 为 `NULL` 是正常的。排除 `score IS NULL` 的记录。

---

**随堂练习 71：累计求和与移动平均**

假设你需要分析每日销售趋势。请先创建一张 `daily_sales` 表，包含 `sale_date`（日期类型）和 `amount`（数值类型，当日销售额）两列，并插入 5-7 行连续日期的测试数据。

然后写一条查询，输出：
- `sale_date`
- `amount`（当日销售额）
- `cumulative_amount`：从第一天到当天的**累计销售额**
- `moving_avg_3d`：**近 3 日移动平均**（当天 + 前两天的平均，不足 3 天的有几天算几天）

思考：为什么累计求和不用额外指定窗口范围也能自动累计？

---

## PostgreSQL 专属特性（快速预览）

至此，标准 SQL 的主线已经走通。在真实开发中，PostgreSQL 提供了一些极其舒爽的方言和独有特性，它们能极大提高开发效率。你不需要现在就精通，但务必混个眼熟：

1. **RETURNING 子句**
   - **痛点**：以往 `INSERT` 进去一条新数据，往往它的 ID 是自增生成的。如果后端代码紧接着要用到这个新 ID，不得不再发一条 `SELECT max(id)` 或者调一个 `LAST_INSERT_ID`，麻烦且并发下不安全。
   - **PG 方案**：直接写 `INSERT INTO students (name) VALUES ('Tom') RETURNING student_id, created_at;`，插入完毕后，立刻直接把这行新数据的这两个字段返回给你！

2. **ON CONFLICT (UPSERT 功能)**
   - **痛点**：我想插入一条数据，但万一这人已经存在了（触发了唯一主键冲突），我该怎么办？以往要先 `SELECT` 判断存不存在，存在就发 `UPDATE`，不存在发 `INSERT`，并发时容易死锁。
   - **PG 方案**：
     ```sql
     INSERT INTO students (student_id, age) VALUES (1, 18)
     ON CONFLICT (student_id) DO UPDATE SET age = 18;
     ```
     俗称“不存在则插入，存在则更新”，一条搞定，绝对并发安全。

3. **强大的类型转换符 `::`**
   - **痛点**：标准 SQL 的类型转换很繁琐：`CAST('123' AS INTEGER)`。
   - **PG 方案**：直接写 `'123'::INTEGER` 或者 `date_string::TIMESTAMP`，极其简练，还能嵌套使用。

4. **ILIKE 不区分大小写匹配**
   - `LIKE 'abc%'` 是严格区分大小写的。如果在 MySQL 里想忽略，常常要改排序规则。在 PG 里，直接用 `student_name ILIKE '%alice%'` 就能找出 'Alice', 'ALICE', 'alice'，非常实用。

5. **无敌的 JSONB 支持**
   - PG 允许你把一个巨大的、没有固定结构的 JSON 数据直接塞进名为 JSONB 的列里。不仅能存，你还能直接在 SQL 里用专属箭头运算符提取内部节点：`SELECT config->>'theme' FROM users;`。甚至还能为 JSON 内部的某个深度属性建立索引，查询飞快。PG 凭借此特性完美兼顾了关系型（SQL）和非关系型（NoSQL/MongoDB）的优势。

---

## 终极综合挑战：一锤定音

**随堂练习 72：全方位实战终极报表**

考验你是否真正通透的时候到了！这是一道包含 80% SQL 核心思想的报表统计题。

我们需要给教务处出具一份“学生整体学业概况”报告，要求查询出如下 4 个列：
1. `student_name` （学生姓名）
2. `major` （专业）
3. `course_count` （选课数量，如果一门都没选，必须显示为 0，不能不出现）
4. `avg_score` （该生所有选课的平均成绩）

**极其严苛的业务要求：**
- **必须覆盖所有人**：即便像 David 这种 1 门课都没有选的学生，也**必须**出现在最终列表里！
- **NULL 值的严谨处理**：计算平均成绩时，如果有某门课成绩是 NULL（刚选课还没考试出分），那么它不参与平均分计算（`AVG` 原本就具备此特性，你只需保证不破坏它）。
- **0 门课的呈现**：如果该生 1 门课都没选，他的平均分显示为 `NULL` 即可。
- **魔鬼级排序控制**：
  - 首要原则：优先按照平均成绩 `avg_score` **从高到低 (DESC)** 排序。
  - 由于包含了没选课的人，有的人平均分是 NULL。你需要保证**有真实成绩的人排在前面，NULL 值统统沉淀到列表最后面！**
  - 次要原则：如果两个人的平均成绩完全一样（或者同为 NULL），则继续按照学生姓名首字母**升序 (ASC)** 排列。

*(提示：慎重思考该从哪张表起手，用 INNER 还是 LEFT JOIN？如何确保 COUNT() 对于没选课的人不会虚报为 1？平均分是 NULL 的人如何让他们排在列表最后？分组字段该填全哪些列？请自己构思完整 SQL！)*

---

# 最后的学习节奏

以后每学一个新语法，都遵循：

```text
1. 讲懂一个小知识点
        ↓
2. 马上在同一个数据库上做 1~几个小题
        ↓
3. 看结果
        ↓
4. 自己解释“为什么是这样”
        ↓
5. 再继续下一块
```

不要等整章结束才做题。
当你把这 71 道题（含附加题）所涉及的核心骨架彻底刻进 DNA，以后再去学复杂的企业级性能优化或者特定的数据库方言，就会像顺水推舟一样简单。祝你早日攻克 SQL！
