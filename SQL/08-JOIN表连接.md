# JOIN 表连接

到现在为止，我们一次只处理一张表。真实问题往往要同时用多张表。还是这三张：

```text
students      学生是谁
courses       课程是什么
enrollments   谁选了哪门课、成绩多少
```

它们之间通过键连接起来。`students.student_id`、`courses.course_id` 是主键，`enrollments` 里存 `student_id` 和 `course_id` 作为外键，把两张表勾连起来。

---

## 一、INNER JOIN（内连接）

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

`ON` 是"按什么条件把两边对上"。`JOIN ... ON ...` 这个形状很像自然语言："把 A 和 B 接起来，条件是……"——`ON` 后面永远写"两边怎么对应"。

记：

```text
INNER JOIN = 只看匹配上的
```

`JOIN` 默认就是 `INNER JOIN`，写不写 `INNER` 都行。没选过课的学生，在 enrollments 里没有对应行，内连接后**不会出现**；反过来，没有任何学生选的课也不会出现。

---

## 二、LEFT JOIN（左外连接）

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

`LEFT JOIN` 完整写法是 `LEFT OUTER JOIN`，`OUTER` 可省略。左表每一行**至少出现一次**，右表能对上就拼上，对不上右表那几列填 NULL。这正好能查出"哪些学生一门课都没选"：

```sql
SELECT s.student_name
FROM students s
LEFT JOIN enrollments e
    ON e.student_id = s.student_id
WHERE e.course_id IS NULL;   -- 右表没匹配上的行
```

---

## 三、RIGHT JOIN / FULL JOIN

知道含义即可：

```text
RIGHT JOIN → 右表全保留
FULL JOIN  → 两边都尽量保留
```

实际写业务 SQL 时，LEFT JOIN 通常更常见——因为人思考问题时，通常把"主表"放在左边，用 LEFT JOIN 把附属信息挂上去。`RIGHT JOIN` 和 `FULL JOIN` 知道怎么回事就行，遇到了能读懂。

`FULL JOIN` 同时保留左右两表所有行，两边对不上的部分各自填 NULL。它最典型的用途是**对比两表差异**（两边都可能有对方没有的行）。

---

## 四、CROSS JOIN（笛卡尔积）

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

这就是笛卡尔积。它不带 ON 条件，纯粹把左边每一行和右边每一行配对。日常业务里很少直接写 CROSS JOIN，但它有正经用途：比如"把每个学生和每门课都组合出来，再看哪些已经选了、哪些没选"——用它做一张"全量配对底表"。

---

## 五、多表 JOIN 链式写法

三张表一起连，就是把 JOIN 一个接一个串起来：

```sql
SELECT
    s.student_name,
    c.course_name,
    e.score
FROM students s
JOIN enrollments e ON e.student_id = s.student_id
JOIN courses c     ON c.course_id = e.course_id;
```

执行顺序上，先把 students 和 enrollments 连出一个中间结果，再把这个中间结果和 courses 连。每加一张表，就多一个 `ON`。

> 注意！！！多表 JOIN 时，**列名如果在多张表里都存在（比如 student_id、course_id），必须用表名/表别名限定**，否则数据库不知道你指的是哪张表的列，直接报"列名歧义"错误。这就是为什么前面都用 `s.`、`e.`、`c.` 做前缀。

---

## 六、极其重要的 LEFT JOIN 大坑：条件放 ON 还是放 WHERE？

这是原文档反复强调、也是初学者最容易翻车的地方。看这个 SQL：

```sql
SELECT ...
FROM students s
LEFT JOIN enrollments e
    ON e.student_id = s.student_id
WHERE e.score >= 60;
```

很多人会以为：

> 我只是想筛选成绩 >= 60 的记录，同时保留没选课的学生。

但由于：

```text
WHERE
```

是在连接结果形成**之后**再过滤行的，所以 `e.score IS NULL` 的学生（也就是没选课、右表填 NULL 的那些行）会被 `WHERE e.score >= 60` 筛掉——因为 `NULL >= 60` 是 UNKNOWN，不满足 WHERE。

于是这个 LEFT JOIN 在效果上可能变得很像 INNER JOIN。你辛辛苦苦保留的"没选课的学生"，被自己的 WHERE 删掉了。

如果你的目的是真正：

> 左表学生全部保留，只把"匹配的成绩"限制在 60 分以上

正确写法是把条件放在 **ON** 里：

```sql
SELECT ...
FROM students s
LEFT JOIN enrollments e
    ON e.student_id = s.student_id
   AND e.score >= 60;
```

这个区别非常值得理解。下面第九节会深度展开原理。

---

## 七、JOIN 的执行算法（进阶）

数据库拿到一个 JOIN 后，底层到底是怎么把两张表拼起来的？主要有三种算法。理解它们，你写 SQL 时才能预判性能。

### 1. Nested Loop Join（嵌套循环连接）

原理最直白：**外层表每取一行，就拿着这行的连接键去内层表找匹配的行**。

```text
for each row r in 外表:
    for each row s in 内表:
        if r.join_key = s.join_key:
            输出 (r, s)
```

- 适用场景：**小表驱动大表**。外表（驱动表）越小，外层循环次数越少；如果内表的连接键上有**索引**，内层就不用全表扫，而是用索引精确查找，速度极快。
- 没有索引时，外表 N 行 × 内表 M 行 = N×M 次比较，性能很差。
- Materialize 优化：优化器可能把内表物化成一份带索引的临时结构，避免重复扫描。

### 2. Hash Join（哈希连接）

原理分两个阶段：
- **Build 阶段**：读入**较小的那张表**，把它的连接键作为 key、整行作为 value，构建一张哈希表。
- **Probe 阶段**：逐行读入大表，用它的连接键去哈希表里查，命中就输出拼接行。

```text
Build: 小表 → 哈希表（key = 连接键）
Probe: 大表逐行 → 查哈希表 → 命中输出
```

- 适用场景：**大表等值连接、且没有索引可用**。两张表都很大、又没有索引时，Hash Join 比 Nested Loop 快得多（总体约 O(N+M)）。
- 代价：要在内存里建哈希表；内存装不下时会**溢出到磁盘**（partition），性能下降。

### 3. Merge Join（合并连接）

原理：**两张表都按连接键排好序，然后像归并两个有序数组一样，两个指针从头往后扫，键相等就配对**。

- 适用场景：**输入本来就已经有序**（比如连接键上有索引，索引本身就是有序的），或者连接条件是范围比较（`>`、`<`、`BETWEEN`）。
- 代价：如果输入没排序，要先排序，排序本身是 O(N log N) 的成本。

### 4. 三种算法对比与选择原则

| 算法 | 原理 | 最佳场景 | 代价 |
|---|---|---|---|
| Nested Loop | 外表逐行去内表找 | 小表驱动大表 + 内表有索引 | 无索引时 N×M 很慢 |
| Hash Join | 小表建哈希表，大表探测 | 大表等值连接、无索引 | 内存占用，可能溢磁盘 |
| Merge Join | 两有序输入归并扫描 | 输入已有序、范围连接 | 需排序则有排序成本 |

> 记忆法：**有索引用 Nested Loop，大表无索引用 Hash Join，数据本来就有序用 Merge Join**。绝大多数时候你不用手选，优化器会自动决定，但看懂执行计划里的选择，能帮你判断是不是该加索引。

---

## 八、自连接（Self Join）：表自己连自己

一张表自己和自己 JOIN，叫自连接。最典型的场景是**层级关系**——表里存了"上级 id"，要在同一张表里把上下级配对出来。

假设有一张员工表 `employees`：

```text
emp_id   name   manager_id
  1      张三     NULL
  2      李四      1
  3      王五      1
  4      赵六      2
```

要查出"每个员工及其直属上级的名字"，就得把 employees 当成两张不同的表（员工视角 e、上级视角 m）来连：

```sql
SELECT
    e.name  AS employee_name,
    m.name  AS manager_name
FROM employees e
LEFT JOIN employees m
    ON e.manager_id = m.emp_id;
```

结果：

```text
employee_name   manager_name
   张三           NULL
   李四           张三
   王五           张三
   赵六           李四
```

> 注意！！！自连接必须给表起**不同的别名**（e 和 m），否则数据库分不清哪个是员工、哪个是上级。物理上是同一张表，逻辑上你把它当成两张用。

另一个自连接场景是**同表比较**：比如找出"成绩相同的不同学生对"。

```sql
SELECT a.student_id, b.student_id, a.score
FROM enrollments a
JOIN enrollments b
  ON a.score = b.score
 AND a.student_id < b.student_id;   -- 避免重复配对和自己连自己
```

---

## 九、WHERE 与 ON 的条件放置差异（深度展开 LEFT JOIN 大坑）

### 1. INNER JOIN 中，WHERE 和 ON 效果相同

```sql
-- 写法 A
SELECT ...
FROM students s
JOIN enrollments e ON e.student_id = s.student_id AND e.score >= 60;

-- 写法 B
SELECT ...
FROM students s
JOIN enrollments e ON e.student_id = s.student_id
WHERE e.score >= 60;
```

内连接只保留两边都匹配上的行。条件放在 ON 里，是"匹配时就要求 score>=60"；放在 WHERE 里，是"先全部匹配完再筛 score>=60"。因为内连接本来就丢弃了不匹配的行，两种写法最终结果**完全一样**。

### 2. LEFT JOIN 中，WHERE 和 ON 效果天差地别

```sql
-- 写法 A：条件放 ON —— 保留所有学生，只挂出 60 分以上的选课
SELECT ...
FROM students s
LEFT JOIN enrollments e
    ON e.student_id = s.student_id
   AND e.score >= 60;

-- 写法 B：条件放 WHERE —— 连接完再筛，没选课的学生(score IS NULL)被删
SELECT ...
FROM students s
LEFT JOIN enrollments e
    ON e.student_id = s.student_id
WHERE e.score >= 60;
```

原理：
- ON 里的 `AND e.score >= 60` 是**连接条件的一部分**：右表连不上（或连上了但 score<60）时，这一行照样保留左表，右表填 NULL。
- WHERE 是**连接完成后对最终结果集的过滤**：此时 score 为 NULL 的行已经存在于结果里，`WHERE e.score >= 60` 把它们连同 score<60 的行一起筛掉，LEFT JOIN 退化成 INNER JOIN。

### 3. 实际案例对比

假设有学生张三（没选课）、李四（选了课，score=50）、王五（选了课，score=90）。

| 写法 | 张三 | 李四 | 王五 |
|---|---|---|---|
| ON 里 AND score>=60 | 保留(score=NULL) | 保留(score=NULL，被ON过滤掉选课) | 保留(score=90) |
| WHERE score>=60 | **被删** | **被删** | 保留 |

一目了然：想"保留全部学生、只挂高分选课"就放 ON；想"只要及格的记录"就放 WHERE。

### 4. RIGHT JOIN 和 FULL JOIN 中的类似问题

- `RIGHT JOIN`：把右表条件放 WHERE，同样会把左表没匹配上的行删掉，退化成内连接。
- `FULL JOIN`：WHERE 里对某一侧非空列加条件，会把另一侧没匹配上的行删掉，削弱全外连接的效果。

> 记忆法：**外连接里，想保留哪一侧的"没匹配上的行"，过滤条件就不能写成对另一侧非空列的 WHERE 条件；要限制"参与连接的右表行"，就把条件挪进 ON。**

---

## 十、USING vs ON

`USING(col)` 是 `ON a.col = b.col` 的简写。

```sql
SELECT ...
FROM students s
JOIN enrollments e
USING (student_id);
```

等价于：

```sql
ON e.student_id = s.student_id
```

区别在于输出列：

| 写法 | 连接键列在结果里 |
|---|---|
| `USING (student_id)` | 只输出**一列** student_id（两表的同名列被合并） |
| `ON e.student_id = s.student_id` | 输出**两列**（s.student_id 和 e.student_id 都在） |

> 注意！！！`USING` 要求两边的列**同名**。如果 students 的键叫 student_id、enrollments 的键也叫 student_id，能用；如果叫别的，就只能用 ON。

---

## 十一、自然连接 NATURAL JOIN（不推荐）

`NATURAL JOIN` 会**自动**把两张表里所有同名的列作为连接条件，不用你写 ON：

```sql
SELECT ...
FROM students
NATURAL JOIN enrollments;
```

它自动按同名列（比如 student_id）连接。听起来很省事，但**非常危险**：

- 哪天你给表加了个同名列（比如两张表都加了 `created_at`），NATURAL JOIN 会突然把它也当成连接条件，结果悄悄变错。
- 你看不到连接条件，读 SQL 的人不知道到底按什么连的。

> 注意！！！**生产环境基本不用 NATURAL JOIN**。显式写 ON，连接条件一目了然，才是工程上的好习惯。

---

## 十二、隐式连接（逗号连接）vs 显式连接

老写法里，把多张表直接写在 FROM 后面用逗号隔开，连接条件写在 WHERE：

```sql
-- 隐式连接
SELECT s.student_name, e.score
FROM students s, enrollments e
WHERE s.student_id = e.student_id;
```

这在功能上等价于 INNER JOIN，但**为什么显式连接更好**：

1. **可读性**：`JOIN ... ON ...` 一眼看出谁连谁、按什么连；逗号写法里连接条件混在一堆 WHERE 过滤条件里，容易漏。
2. **避免意外笛卡尔积**：逗号连接一旦忘了写连接条件，立刻变成笛卡尔积（行数爆炸）；显式 INNER JOIN 不写 ON 会直接报语法错误，帮你拦住。
3. **外连接语法统一**：`LEFT JOIN` 这种外连接，在逗号写法里要靠很别扭的 `(+)` 语法（Oracle）才能表达，各家还不统一；显式 `LEFT JOIN` 是标准写法。

> 记忆法：**永远用显式 JOIN ... ON，别用逗号连接**。看到老代码里的 `FROM a, b WHERE a.id = b.id`，能读懂就行，自己写就改过来。

---

## 十三、半连接（Semijoin）与反连接（Antijoin）

这两个不是你直接写的 JOIN 类型，而是**优化器在执行计划里对 EXISTS / IN 子查询做的内部转换**。

- **半连接（Semijoin）**："外表里，只要在内表找到**任意一条**匹配行，就输出外表这一行一次"——对应 `EXISTS` / `IN`。注意它不会因为内表有多条匹配而把外表行输出多次（这是它和普通 INNER JOIN 的区别）。
- **反连接（Antijoin）**："外表里，在内表**找不到**匹配行的那些行"——对应 `NOT EXISTS` / `NOT IN`。

你写 `WHERE EXISTS (SELECT 1 FROM enrollments e WHERE e.student_id = s.student_id)`，优化器底层常常把它改写成一个半连接来执行，效率比真的逐行跑子查询高得多。这就是为什么现在很多数据库里 `EXISTS` 和 `IN` 性能差不多——优化器帮你统一了。

---

## 十四、LATERAL JOIN（横向连接，进阶）

普通 JOIN 是"两张表按条件配对"，但**子查询不知道外层当前是哪一行**。`LATERAL` 关键字允许你在 JOIN 里写一个**关联子查询**，让这个子查询能引用外层 FROM 里前面出现的表的列——也就是"外层每一行，都执行一次这个子查询"。

语法：

```sql
SELECT ...
FROM 外表 t
JOIN LATERAL (
    SELECT ...
    FROM 内表
    WHERE 内表.x = t.x      -- 引用外层表 t 的列
) sub ON TRUE;
```

最经典的应用是 **"TOP N per group"**：比如"每个学生，取他成绩最高的两门课"。

```sql
SELECT
    s.student_name,
    top_c.course_id,
    top_c.score
FROM students s
JOIN LATERAL (
    SELECT e.course_id, e.score
    FROM enrollments e
    WHERE e.student_id = s.student_id
    ORDER BY e.score DESC
    LIMIT 2
) top_c ON TRUE;
```

原理：对 students 里的每一个学生，都跑一次 `SELECT ... FROM enrollments WHERE 这个学生 ... ORDER BY score DESC LIMIT 2`，取出他最好的两门课。没有 LATERAL，你是没法在 JOIN 里这样"按外层行逐行取 top 2"的。

> 注意！！！`LATERAL` 是 PostgreSQL 比较强的特性，MySQL 8.0+ 也支持，SQL Server 用类似的 `CROSS APPLY` / `OUTER APPLY`。它和"关联子查询"是近亲，区别在于 LATERAL 子查询能返回**多行多列**，而 SELECT 里的标量子查询只能返回一行一列。

---

## 十五、非等值连接（Non-equi Join）

ON 条件不一定要用 `=`，也可以用 `<`、`>`、`BETWEEN` 等。这种叫非等值连接。

典型场景：**区间匹配**。假设有一张分数段表 `grade_bands`：

```text
band_name   low_score   high_score
  不及格       0           59
  及格        60           79
  优秀        80          100
```

要给每条选课记录匹配上它属于哪个分数段：

```sql
SELECT
    e.student_id,
    e.score,
    g.band_name
FROM enrollments e
JOIN grade_bands g
  ON e.score BETWEEN g.low_score AND g.high_score;
```

这里 ON 用的是 `BETWEEN ... AND ...`，不是等值。每个 score 落进哪个区间，就拼上哪个 band。

另一个场景是**排名区间**：比如用非等值自连接做"比我分高的人有几个"。

---

## 十六、JOIN 与 NULL 的交互

外连接会**主动制造 NULL**：左表某行在右表没匹配上，右表那几列就填 NULL。这些 NULL 会影响后续操作：

1. **再 WHERE 过滤时被误删**（就是第六节那个坑）。
2. **再 JOIN 时连不上**：如果拿外连接产生的 NULL 列去和别的表做等值 JOIN，`NULL = 任何值` 都是 UNKNOWN，这行就又丢了。
3. **聚合时被忽略**：`SUM`、`AVG`、`COUNT(列)` 会跳过这些外连接产生的 NULL。

> 注意！！！外连接之后，要清楚哪些列是"真 NULL"、哪些列是"没匹配上补的 NULL"。它们在后续 WHERE / JOIN / 聚合里的行为要心里有数，不然结果悄悄变少你都不知道。

---

## 十七、多表 JOIN 的连接顺序与优化器

写 SQL 时你按业务逻辑写 FROM a JOIN b ON ... JOIN c ON ...，但数据库**实际先连哪两张表，往往不是你写的顺序**。优化器会估算各种连接顺序的成本，挑最便宜的执行。

这意味着：
- 你不用太纠结 FROM 后面表写的先后（INNER JOIN 之间交换顺序结果不变）；
- 但 **LEFT JOIN 的左表位置不能随便换**——LEFT JOIN 保留谁是语义，优化器一般不会把 LEFT JOIN 改写成普通连接。
- 想让优化器做出好选择，关键是**统计信息准确、连接键上有索引**，而不是手动调 FROM 顺序。

---

## 本文件覆盖的原文档知识点清单

- [x] 三张表的背景介绍（students / courses / enrollments）
- [x] INNER JOIN：语法、ON 条件、只看匹配上的、ON 后面写"两边怎么对应"
- [x] LEFT JOIN / LEFT OUTER JOIN：保留左表全部行、右表无匹配填 NULL、LEFT = 左边一个都不能丢
- [x] RIGHT JOIN / FULL JOIN：含义、实际业务中 LEFT JOIN 更常见
- [x] CROSS JOIN（笛卡尔积）：每个学生和每门课全部配对、3×4=12 行
- [x] 多表 JOIN 链式写法
- [x] 极其重要的 LEFT JOIN 大坑：WHERE 放 ON 里 vs 放 WHERE 里的区别
- [x] JOIN 的执行算法：Nested Loop / Hash Join / Merge Join 原理、场景与对比
- [x] 自连接（Self Join）：员工上下级、同表比较
- [x] 非等值连接（Non-equi Join）：BETWEEN 区间匹配
- [x] LATERAL JOIN（横向连接）：关联子查询的 JOIN 形式、TOP N per group
- [x] USING vs ON 的区别（合并列 vs 保留两列）
- [x] NATURAL JOIN 的原理与风险、不推荐原因
- [x] JOIN 中 WHERE 与 ON 的条件放置差异深度展开（INNER/LEFT/RIGHT/FULL）
- [x] 隐式连接（逗号连接）vs 显式连接
- [x] 多表 JOIN 的连接顺序和优化器的角色
- [x] JOIN 的列歧义问题和表别名的必要性
- [x] 半连接（Semijoin）和反连接（Antijoin）概念
- [x] JOIN 与 NULL 的交互（外连接产生的 NULL 的影响）
