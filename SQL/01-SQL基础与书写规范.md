# SQL 基础与书写规范

你觉得 SQL 难记，通常不是因为记忆力不行，而是因为你在记"单词"，而不是在记"结构"。SQL 真正需要先建立的，是下面这些基础：固定骨架、书写习惯、值与表达式、NULL 的三值逻辑、数据类型。

---

# 第一部分：先建立"骨架感"

## 1. SQL 是"固定骨架 + 填空"

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
>
> 等于
>
> **我要看什么 → 从哪里找 → 先筛掉哪些行 → 要不要分组 → 分完组后再筛哪些组 → 最后怎么排。**

这才是 SQL 的"骨架记忆法"。不要一上来就背某个函数怎么拼，先把这个六段式骨架刻进脑子里，后面所有语法都是往骨架里填空。

## 2. SQL 的五大类别（DDL / DML / DQL / TCL / DCL）

SQL 不是只有 `SELECT`。按"它在对数据库做什么"划分，五大类：

| 类别 | 全称 | 作用 | 先掌握的关键字 |
|---|---|---|---|
| DDL | Data Definition Language（数据定义语言） | 定义/修改数据库里有哪些对象（表、库、视图） | `CREATE` `ALTER` `DROP` |
| DML | Data Manipulation Language（数据操纵语言） | 改对象里面的数据 | `INSERT` `UPDATE` `DELETE` |
| DQL | Data Query Language（数据查询语言） | 查数据 | `SELECT` |
| TCL | Transaction Control Language（事务控制语言） | 控制事务提交/回滚 | `BEGIN` `COMMIT` `ROLLBACK` |
| DCL | Data Control Language（数据控制语言） | 控制谁有权限 | `GRANT` `REVOKE` |

**注意！！！** 这里有个极易混的点：`SELECT` 严格说是 DQL，但很多教材和人嘴里把 `INSERT/UPDATE/DELETE/SELECT` 一锅烩叫 DML。当有人说"DML 操作"时，你要根据上下文判断他到底指"只改数据的三兄弟"还是"所有数据相关操作"。

DDL 改的是"数据库里有哪些对象"（结构），DML/DQL 改的是"对象里面的数据"（内容）。这个区分非常重要——例如 PostgreSQL 里 DDL 可以被事务回滚，而 MySQL 里 DDL 一执行就强制提交。

## 3. 学习路径

初学阶段最重要的是这五句：

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

索引、视图、权限这些属于"知道用途 + 会基本语法"，不应该和 `SELECT/WHERE/JOIN` 一起死记。

---

# 第二部分：书写习惯——大小写、引号、分号、注释

## 1. 关键字大小写不影响含义

这些基本等价：

```sql
SELECT * FROM students;
select * from students;
SeLeCt * FrOm students;
```

数据库在词法分析阶段会把关键字归一化，所以大小写不影响含义。但"能跑"不等于"好读"。建议统一：

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

这样最好读。为什么关键字要大写、标识符要小写？因为这样一眼就能从视觉上区分"哪些是 SQL 的指令骨架，哪些是你业务里的名字"。

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

结果不是 TRUE（是 FALSE）。所以：

```text
'Alice'
'alice'
```

是两个不同的字符串。**字符串的大小写属于"数据内容"，数据库老老实实按字节比较。**

## 3. 单引号里想写单引号怎么办？

标准 SQL 的写法是：**单引号写两个单引号**。

```sql
SELECT 'Bob''s book';
```

它表示：

```text
Bob's book
```

**注意！！！** 是两个单引号 `''`，不是一个双引号 `"`。这是初学者最常写错的地方之一。如果你想表示"书名里有个撇号"，就连续敲两个单引号。

## 4. 双引号不是字符串

双引号通常表示"标识符"（也就是表名、列名等名字）。

```sql
SELECT "student_name"
FROM students;
```

这里 `"student_name"` 是列名。而：

```sql
SELECT 'student_name';
```

这里 `'student_name'` 是一个字符串（返回的就是这串字符本身）。

一个非常实用的记忆法：

```text
'...'  → 数据里的文字（字符串字面量）
"..."  → 表名、列名等名字（标识符）
```

## 5. 为什么初学最好不要乱用双引号命名？

在 PostgreSQL 中，不加双引号的标识符会按规则**折叠为小写**。

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

因为未加引号的名字会按 PostgreSQL 的规则统一处理成小写（`Users` → `users`，`UserName` → `username`），所以你怎么打大小写都能查到。

但如果你刻意写：

```sql
CREATE TABLE "Users" (
    "ID" integer,
    "UserName" varchar(50)
);
```

那这些名字就是**精确区分大小写**的，数据库会把它们原样保存下来。之后你必须：

```sql
SELECT "ID", "UserName"
FROM "Users";
```

一个字母大小写错了都报"表/列不存在"。

**初学建议：全小写 + 下划线，最省脑子。** 一旦你用双引号建了一个带大写或空格的名字（比如 `"User Info"`），以后每一处引用都得带着这对双引号和一模一样的大小写，非常痛苦。

> 补充：MySQL 里双引号默认也能当字符串用（取决于 `sql_mode` 的 `ANSI_QUOTES`），SQL Server 里方括号 `[ ]` 也能包标识符。但标准 SQL 和 PostgreSQL 里，`"..."` 就是标识符。这也是"写可迁移 SQL"时不要乱加双引号的原因。

## 6. 分号

一条 SQL 语句通常以分号结束。例如：

```sql
SELECT * FROM students;
```

注意：

- 在 DataGrip 等 GUI 工具的 SQL Console 里，通常一条语句以分号结尾最清楚；
- 在 `psql` 交互环境里，分号表示一条 SQL 语句结束并真正执行——只换行不打分号，psql 会一直等着你继续输入。

## 7. 注释

单行注释用两个短横：

```sql
-- 这是注释
SELECT * FROM students;
```

多行注释用 `/* */`：

```sql
/*
这是多行注释
可以写很多行
*/
SELECT * FROM students;
```

**注意！！！** 单行注释是 `--`（两个减号，后面通常跟一个空格），不是 `//`。`//` 在 SQL 里不是注释。

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

这些都是"值"（字面量）。值是死的、写死在语句里的常量。

## 2. 一列

```sql
age
student_name
score
```

表示从当前这一行取对应的列值。列名不是字面量，它代表"每一行在这个列上的内容"，所以查询多行时它会逐行取出不同的值。

## 3. 表达式

表达式就是"算出一个值的东西"。例如：

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

**SELECT 后面不只能放列名，任何"能算出一个值"的东西都能放。** 这是理解 SQL 的关键：SELECT 子句本质上是一个"逐行计算列值"的列表。

## 4. 算术运算

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

**注意！！！** 不要把 SQL 里所有的 `*` 都理解成"通配符"。`SELECT *` 里的 `*` 是"所有列"的意思；但 `score * 1.1` 里的 `*` 是"乘号"。同一个符号在不同上下文里含义完全不同。

补充两个细节：

- **整数除法**：在很多数据库里，两个整数相除会得到整数（截断小数）。例如 `7 / 2` 在某些数据库里是 `3` 而不是 `3.5`。想要小数，至少让一边是小数：`7.0 / 2` 或 `CAST(7 AS NUMERIC) / 2`。
- **取余**：`MOD(a, b)` 或 `a % b`（PostgreSQL 支持 `%`），例如 `MOD(10, 3) = 1`。

## 5. 字符串拼接：`||`

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

`||` 读作"接起来"，在 SQL 里就是字符串拼接（注意别和某些编程语言里的"逻辑或"混淆；SQL 的逻辑或是 `OR`）。

**注意！！！** `||` 和 `CONCAT` 对 NULL 的处理不同：

- `'a' || NULL` 的结果是 `NULL`（任何一边是 NULL，整个拼接就变 NULL）。
- `CONCAT('a', NULL)` 的结果是 `'a'`（CONCAT 会自动把 NULL 当空串处理）。

这是一个非常实用但容易踩坑的区别。

## 6. 比较运算

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

它表示"不等于"。有些数据库也接受 `!=`，PostgreSQL 也接受，但写可迁移 SQL 时，`<>` 更值得认识（它是 SQL 标准里的写法，`!=` 是 C 语言风格的民间写法）。

**注意！！！** SQL 里"等于"是一个等号 `=`，不是两个等号 `==`。`=` 在 SQL 里既是比较符，在 `SET age = 18` 里又是赋值符，全靠上下文区分。

## 7. 逻辑运算

```text
AND
OR
NOT
```

- `AND`：两边都为 TRUE 才 TRUE。
- `OR`：任意一边为 TRUE 就 TRUE。
- `NOT`：取反。

## 8. 运算优先级：不确定就加括号

例如：

```sql
WHERE age >= 18
  AND score >= 60
  OR is_repeat = TRUE
```

人脑很容易误读到底是"(A 且 B) 或 C"还是"A 且 (B 或 C)"。在 SQL 里，**`AND` 的优先级高于 `OR`**，所以上面这句实际是 `(age >= 18 AND score >= 60) OR is_repeat = TRUE`。

初学阶段直接养成习惯：

```sql
WHERE
    (age >= 18 AND score >= 60)
    OR is_repeat = TRUE
```

SQL 写得清楚，比省两个括号重要得多。**任何时候你不确定优先级，就加括号。** 括号不花钱，但能救命。

---

# 第四部分：NULL 与三值逻辑

`NULL` 不是：

```text
0        -- 零是一个确定的数字
''       -- 空串是一个确定长度为0的字符串
FALSE    -- FALSE 是一个确定的布尔值
```

`NULL` 的真正含义是"未知 / 缺失 / 不适用"。它是一个标记，代表"这里没有值"，而不是任何具体的值。

## 1. 不要写 `= NULL` 或 `<> NULL`

不要写：

```sql
WHERE age = NULL;
```

也不要写：

```sql
WHERE age <> NULL;
```

因为 NULL 参与普通比较时，会进入 SQL 的 UNKNOWN（未知）结果。因此：

```sql
NULL = NULL
```

不是 TRUE（两个"未知"无法判定是否相等，结果还是 UNKNOWN）。

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

**注意！！！** 这是 SQL 里极少数"运算符后面不能用普通符号"的地方。你必须写 `IS NULL`，不能写 `= NULL`。这是新手最高频的 bug 来源之一——写了 `WHERE age = NULL`，结果一行都查不出来，还以为是数据错了。

## 3. 为什么 SQL 多了一个 UNKNOWN？

普通逻辑里你会习惯只有两种：

```text
TRUE / FALSE
```

SQL 里还存在第三种：

```text
UNKNOWN
```

例如：

```text
年龄 = NULL
```

数据库无法判断：

> 这个未知年龄是不是 18？

所以它既不是 TRUE，也不是 FALSE，而是 UNKNOWN。

`WHERE` 最终**只保留条件为 TRUE 的行**。这就是为什么：

```sql
WHERE age = NULL
```

查不到"年龄未知"的人——因为 `age = NULL` 对每一行都是 UNKNOWN，而 WHERE 只留 TRUE，UNKNOWN 的行全部被过滤掉了。

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

也就是说，`CHECK (age >= 0)` 本身就已经允许 `age` 为 `NULL` 了，不需要额外写 `NULL OR`。这是三值逻辑在"约束"和"过滤"两个场景下的关键差异：

- **WHERE**：只留 TRUE（UNKNOWN 被丢弃）。
- **CHECK**：只拒绝 FALSE（UNKNOWN 被放过）。

同一个 UNKNOWN，在两个地方待遇相反。

## 4. 三值逻辑完整真值表

SQL 的逻辑运算不是二值的，而是三值的（TRUE / FALSE / UNKNOWN）。下面是完整的真值表。

### AND（两边都 TRUE 才 TRUE）

| A | B | A AND B |
|---|---|---|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE |
| TRUE | UNKNOWN | UNKNOWN |
| FALSE | TRUE | FALSE |
| FALSE | FALSE | FALSE |
| FALSE | UNKNOWN | **FALSE** |
| UNKNOWN | TRUE | UNKNOWN |
| UNKNOWN | FALSE | **FALSE** |
| UNKNOWN | UNKNOWN | UNKNOWN |

记忆：AND 里只要有一边是 FALSE，结果就是 FALSE（短路）；否则如果有一边 UNKNOWN，结果 UNKNOWN。

### OR（任意一边 TRUE 就 TRUE）

| A | B | A OR B |
|---|---|---|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | TRUE |
| TRUE | UNKNOWN | **TRUE** |
| FALSE | TRUE | TRUE |
| FALSE | FALSE | FALSE |
| FALSE | UNKNOWN | UNKNOWN |
| UNKNOWN | TRUE | **TRUE** |
| UNKNOWN | FALSE | UNKNOWN |
| UNKNOWN | UNKNOWN | UNKNOWN |

记忆：OR 里只要有一边是 TRUE，结果就是 TRUE（短路）；否则如果有一边 UNKNOWN，结果 UNKNOWN。

### NOT（取反）

| A | NOT A |
|---|---|
| TRUE | FALSE |
| FALSE | TRUE |
| UNKNOWN | **UNKNOWN** |

**注意！！！** `NOT UNKNOWN` 还是 UNKNOWN——你对"未知"取反，得到的仍然是"未知"。

## 5. COALESCE：给 NULL 一个替代值

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

`COALESCE` 可以接任意多个参数，它返回第一个非 NULL 的；如果全是 NULL，返回 NULL。它常用来：

- 把 NULL 的成绩显示为 0；
- 优先显示昵称，没有昵称就显示真名：`COALESCE(nickname, real_name, '匿名')`。

## 6. NULLIF：两个值相等时返回 NULL

`NULLIF(a, b)` 是标准 SQL 函数，逻辑是：

> 如果 `a = b`，返回 NULL；否则返回 `a`。

```sql
NULLIF(score, 0)
```

意思：成绩为 0 时当成 NULL 处理。它最经典的用途是**防止除零错误**：

```sql
SELECT total / NULLIF(count, 0) FROM stats;
```

如果 `count` 是 0，`NULLIF(count, 0)` 返回 NULL，那么整个除法结果是 NULL（而不是报错），避免了"除以零"异常。

`NULLIF(a, b)` 等价于这个 CASE：

```sql
CASE WHEN a = b THEN NULL ELSE a END
```

## 7. GREATEST / LEAST：取最大/最小

PostgreSQL 提供两个很实用的函数：

```sql
GREATEST(90, 80, 95)   -- 95，取最大
LEAST(90, 80, 95)       -- 80，取最小
```

**注意！！！** 它们对 NULL 的处理：只要有一个参数是 NULL，`GREATEST`/`LEAST` 的结果就是 NULL（因为"未知参与比较，结果未知"）。如果你想忽略 NULL，要先用 COALESCE 包一层。

---

# 第五部分：数据类型

SQL 不只有数字和字符串。本练习主线中会用：

```text
INTEGER
VARCHAR(50)
NUMERIC(5,2)
DATE
TIMESTAMP
BOOLEAN
```

## 1. INTEGER

```sql
age INTEGER  -- 可以写作 int4，表示 4 字节；同理有 int2 和 int8
```

PostgreSQL 里整数类型按字节宽度分三档：

| 写法 | 别名 | 字节 | 范围 |
|---|---|---|---|
| `SMALLINT` | `int2` | 2 字节 | -32768 ~ 32767 |
| `INTEGER` | `int4` | 4 字节 | 约 ±21 亿 |
| `BIGINT` | `int8` | 8 字节 | 约 ±9.2×10^18 |

**注意！！！** 初学别乱用 `BIGINT`。绝大多数业务用 `INTEGER`（int4）就够了，它是性能和空间的最佳平衡点。只有主键可能涨到 21 亿以上（比如日志表、流水表）才需要 `BIGINT`。

## 2. VARCHAR(n)

```sql
student_name VARCHAR(50)
```

表示最多 50 个字符的变长字符串。`VARCHAR` 是 "character varying" 的缩写——长度可变，但有上限。

## 3. CHAR(n) 与 VARCHAR(n)、TEXT 的区别

这是一个非常容易被忽略但面试常问的点：

| 类型 | 存储方式 | 定长/变长 | 适用场景 |
|---|---|---|---|
| `CHAR(n)` | 定长，不足补空格 | 固定 n 个字符 | 长度固定的编码，如性别 `CHAR(1)`、邮编 `CHAR(6)` |
| `VARCHAR(n)` | 变长，按实际长度存 | 最多 n 个字符 | 一般的名称、描述 |
| `TEXT` | 变长，无长度上限 | 不限 | 长文本、备注、文章正文 |

**注意！！！** 在 PostgreSQL 里，`VARCHAR(n)` 和 `TEXT` 的性能几乎完全一样，`TEXT` 反而更灵活（不用纠结 n 填多大）。很多 PostgreSQL 老手建表时干脆全用 `TEXT`，不写 `VARCHAR(n)`。但 `VARCHAR(n)` 带长度上限，能在数据库层面挡住"超长"的脏数据，这是它的价值。

## 4. NUMERIC(p, s) 精确小数

例如：

```sql
score NUMERIC(5,2)
```

可以理解成：

```text
最多 5 位有效数字
其中 2 位在小数点后
```

所以 `NUMERIC(5,2)` 能存 `-999.99` 到 `999.99` 之间的数（整数部分最多 3 位，小数部分 2 位）。

对于需要精确小数的值（金额、成绩），`NUMERIC` 很适合。它是"精确十进制"存储，不会有浮点误差。

**注意！！！** 钱必须用 `NUMERIC`，不能用浮点数。因为浮点数有二进制精度问题：`0.1 + 0.2` 在浮点数里不等于 `0.3`，算账会出错。

## 5. FLOAT / REAL / DOUBLE PRECISION（近似小数）

和 `NUMERIC` 相对的是"浮点近似类型"：

| 类型 | 精度 | 用途 |
|---|---|---|
| `REAL` | 4 字节单精度 | 科学计算、对精度不敏感的测量值 |
| `DOUBLE PRECISION` | 8 字节双精度 | 大范围科学计算 |

浮点类型算得快，但不精确。**凡是涉及钱、成绩、需要对账的，一律用 `NUMERIC`；浮点只用于图表、统计采样等容忍误差的场景。**

## 6. DATE

保存日期，例如：

```text
2026-09-17
```

只有年月日，没有时间。

## 7. TIME 与 TIMESTAMP、TIMESTAMPTZ

```text
2024-01-01 13:45:30.123
```

- `DATE`：只有日期。
- `TIME`：只有时间（`13:45:30`）。
- `TIMESTAMP`：日期 + 时间，但**不带时区**。
- `TIMESTAMPTZ`（`TIMESTAMP WITH TIME ZONE`）：日期 + 时间 + 时区，是 PostgreSQL 自己很常用的时间类型。

**注意！！！** 业务里存"绝对时间点"强烈建议用 `TIMESTAMPTZ`。`TIMESTAMP` 不带时区，跨时区部署时会出现"同一时刻在不同机器上读出来不一样"的坑。

## 8. INTERVAL（时间间隔）

PostgreSQL 独有的好用类型，表示"一段时间长度"：

```sql
SELECT NOW() - enrolled_at AS study_duration;   -- 结果是 interval
SELECT NOW() + INTERVAL '7 days';              -- 7 天后
```

它可以做日期加减：`enrolled_at + INTERVAL '30 days'` 就是"选课 30 天后"。

## 9. BOOLEAN

```text
TRUE
FALSE
NULL
```

注意：

```text
NULL 不是 FALSE
```

**注意！！！** 这是布尔类型里最容易踩的坑。`BOOLEAN` 列允许三种状态：真、假、未知（NULL）。如果你写了 `WHERE is_active = FALSE`，那么 `is_active` 为 NULL 的行会被过滤掉（因为 `NULL = FALSE` 是 UNKNOWN），它们不会出现在结果里。如果你想"把非 TRUE 的都捞出来"，得写 `WHERE is_active IS NOT TRUE`。

## 10. JSON / JSONB（PostgreSQL 半结构化类型）

PostgreSQL 允许你把一个没有固定结构的 JSON 直接存进列里：

```sql
CREATE TABLE users (
    user_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    config JSONB
);

INSERT INTO users (config)
VALUES ('{"theme": "dark", "notifications": true}');

SELECT config->>'theme' FROM users;   -- 取出 dark
```

- `JSON`：原样存储文本，查询时再解析。
- `JSONB`：二进制存储，会解析、去掉空白、还能建索引，查询更快。生产环境优先用 `JSONB`。

## 11. 其他常用类型速览

| 类型 | 用途 | 例子 |
|---|---|---|
| `UUID` | 全局唯一 ID，分布式系统主键 | `a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11` |
| `SERIAL` / `BIGSERIAL` | PostgreSQL 老式自增主键 | `student_id SERIAL PRIMARY KEY` |
| `MONEY` | 货币金额（带本地化格式） | PostgreSQL 特有 |
| `INET` / `CIDR` | IPv4/IPv6 地址、网段 | `'192.168.1.1'`、`'10.0.0.0/8'` |
| `数组类型` | 一列存多个值 | `TEXT[]`、`INTEGER[]` |
| `ENUM` | 枚举（自定义取值集合） | 见建表文档 |

## 12. 类型转换

标准 SQL 形式：

```sql
CAST('123' AS INTEGER)
```

PostgreSQL 还支持简写：

```sql
'123'::integer
```

后者非常方便，但属于 PostgreSQL 风格（`::` 是 PG 专属，Oracle 用 `TO_NUMBER()`，MySQL 用 `CAST` 或隐式转换）。

### 显式转换 vs 隐式转换

- **显式转换**：你主动写 `CAST(x AS 类型)` 或 `x::类型`，意图明确，结果可控。
- **隐式转换**：数据库自动帮你转。例如 `SELECT '123' + 1` 在某些数据库里会自动把字符串 `'123'` 转成数字再相加。

**注意！！！** 隐式转换是双刃剑：它让语句能跑，但可能在你不知道的时候悄悄选错了类型，导致**索引用不上**（列上的函数/转换会让索引失效，这点在性能文档里会展开）。生产环境建议：跨类型运算一律显式 `CAST`，别依赖数据库猜。

---

# 第六部分：运算符与函数补充

## 1. 位运算

PostgreSQL 支持对整数按二进制位操作：

| 运算符 | 含义 | 例子 |
|---|---|---|
| `&` | 按位与 | `5 & 3 = 1` |
| `\|` | 按位或 | `5 \| 3 = 7` |
| `#` | 按位异或 | `5 # 3 = 6` |
| `~` | 按位取反 | `~5 = -6` |
| `<<` | 左移 | `1 << 4 = 16` |
| `>>` | 右移 | `16 >> 2 = 4` |

位运算常用于权限位、标志位压缩存储。

## 2. 字符串常用函数

| 函数 | 作用 | 例子 |
|---|---|---|
| `LENGTH(s)` | 字符个数 | `LENGTH('Alice') = 5` |
| `SUBSTRING(s FROM a FOR b)` | 取子串 | `SUBSTRING('Hello' FROM 1 FOR 3) = 'Hel'` |
| `UPPER(s)` | 转大写 | `UPPER('abc') = 'ABC'` |
| `LOWER(s)` | 转小写 | `LOWER('ABC') = 'abc'` |
| `TRIM(s)` | 去掉首尾空格 | `TRIM('  hi  ') = 'hi'` |
| `REPLACE(s, from, to)` | 替换 | `REPLACE('a-b', '-', '_') = 'a_b'` |
| `POSITION(sub IN s)` | 子串位置 | `POSITION('bc' IN 'abc') = 2` |

## 3. 模式匹配运算符（预览）

- `LIKE`：简单通配（`%` `_`），详见排序文档。
- `~`：正则表达式匹配（PostgreSQL 特有），详见排序文档。

---

# 第七部分：SQL 标准与各方言差异概述

同一个 SQL，在 PostgreSQL / MySQL / Oracle / SQL Server 里写法可能不同。初学以 PostgreSQL 为主，但要知道差异在哪。

| 关注点 | PostgreSQL | MySQL | Oracle | SQL Server |
|---|---|---|---|---|
| 字符串引号 | 单引号 `'` | 单引号（双引号取决于 sql_mode） | 单引号 | 单引号 |
| 不等于 | `<>` 或 `!=` | `<>` 或 `!=` | `<>` 或 `!=` | `<>` 或 `!=` |
| 字符串拼接 | `\|\|` 或 `CONCAT()` | `CONCAT()`（`\|\|` 默认是逻辑或） | `\|\|` 或 `CONCAT()` | `+` 或 `CONCAT()` |
| 类型转换 | `x::类型` 或 `CAST` | `CAST` 或隐式 | `TO_NUMBER/TO_DATE` | `CAST` 或 `CONVERT` |
| 分页 | `LIMIT n OFFSET m` | `LIMIT m, n` | `ROWNUM` / `FETCH FIRST` | `TOP n` / `OFFSET FETCH` |
| 自增主键 | `GENERATED ... AS IDENTITY` 或 `SERIAL` | `AUTO_INCREMENT` | `GENERATED AS IDENTITY` / 序列 | `IDENTITY(1,1)` |
| 当前时间 | `CURRENT_TIMESTAMP` / `NOW()` | `NOW()` / `CURRENT_TIMESTAMP` | `SYSDATE` | `GETDATE()` |
| 布尔类型 | 原生 `BOOLEAN` | `TINYINT(1)` 模拟 | 无原生 BOOLEAN | `BIT` |
| 空串与 NULL | 严格区分 | 某些模式下 `''` 等于 NULL | `''` 当 NULL 处理 | 区分 |

**记忆法：** 标准 SQL（`CAST`、`<>`、`||`、`UNION`）在谁那里都能跑；一旦你看到 `::`、`ILIKE`、`SERIAL`、`LIMIT ... OFFSET`，那就是 PostgreSQL 方言。

---

# 本文件覆盖的原文档知识点清单

1. SQL 固定骨架（SELECT/FROM/WHERE/GROUP BY/HAVING/ORDER BY）与骨架记忆法
2. DDL/DML/DQL/TCL/DCL 分类表及各自关键字
3. 学习路径（先五句，后进阶）
4. 关键字大小写不影响含义，建议大写关键字+小写表名列名+下划线命名
5. 字符串用单引号，字符串大小写是内容的一部分
6. 单引号里写单引号用两个单引号转义
7. 双引号是标识符不是字符串，`'...'` 与 `"..."` 记忆法
8. PostgreSQL 未加引号标识符折叠为小写的规则，及双引号命名的坑
9. 分号的作用（psql 中结束并执行）
10. 单行注释 `--` 与多行注释 `/* */`
11. 一个值 / 一列 / 表达式的概念，SELECT 后可放表达式
12. 算术运算 `+ - * /`，`*` 在不同上下文含义不同
13. 字符串拼接 `||`，与 `CONCAT` 函数
14. 比较运算 `= <> != < > <= >=`，`<>` 是标准 SQL
15. 逻辑运算 `AND OR NOT`
16. 运算优先级不确定就加括号（AND 高于 OR）
17. NULL 不是 0 / '' / FALSE
18. 不要写 `= NULL` 或 `<> NULL`，判断用 `IS NULL` / `IS NOT NULL`
19. 三值逻辑 UNKNOWN 的由来，WHERE 只保留 TRUE
20. CHECK 约束只有 FALSE 才违反（TRUE 和 UNKNOWN 都通过）的详细表格
21. COALESCE 从左往右找第一个非 NULL 值
22. INTEGER（int2/int4/int8）
23. VARCHAR(n) 变长字符串
24. NUMERIC(p,s) 精确小数
25. DATE、TIMESTAMP（及 timestamptz 预告）
26. BOOLEAN（TRUE/FALSE/NULL，NULL 不是 FALSE）
27. 类型转换 `CAST` 标准形式与 PostgreSQL 的 `::` 运算符
28. 【补充】三值逻辑 AND/OR/NOT 完整真值表
29. 【补充】NULLIF 函数及其与 CASE、防除零的关系
30. 【补充】GREATEST/LEAST 函数及对 NULL 的处理
31. 【补充】CHAR(n) 与 VARCHAR(n)、TEXT 的区别
32. 【补充】FLOAT/REAL/DOUBLE PRECISION 近似类型
33. 【补充】JSON/JSONB、UUID、SERIAL/BIGSERIAL、MONEY、INET/CIDR、数组类型、INTERVAL
34. 【补充】位运算符、字符串函数（LENGTH/SUBSTRING/UPPER/LOWER/TRIM/REPLACE/POSITION）
35. 【补充】隐式类型转换与显式类型转换的区别及陷阱
36. 【补充】SQL 标准与 PostgreSQL/MySQL/Oracle/SQL Server 方言差异表
