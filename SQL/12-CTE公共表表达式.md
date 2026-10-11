# CTE 公共表表达式（WITH）

## 一、是什么？

CTE 的全称是 **Common Table Expressions（通用表表达式）**。

你可以把它理解为：在一个复杂的 SQL 查询语句内部，**临时定义的一个或多个"虚拟结果表"**，并给它们起了名字。在这一次查询执行期间，你随时可以像引用普通表一样引用这些 CTE。

---

## 二、为什么有这个东西？

在没有 CTE 之前，如果我们想基于一个复杂的统计结果进行下一步过滤，只能使用**嵌套子查询**。

例如："查出所有学生，从中过滤出分数大于 80 的，再从中过滤出名字带 A 的……"

这就导致 SQL 变成了"洋葱代码"，一层套一层，极其难以阅读，你必须从最里面那一层往外读才能看懂逻辑。

有了 CTE 之后，你可以把逻辑**扁平化**：第一步算什么（起名叫 A），第二步算什么（起名叫 B），最后 `SELECT * FROM B`。它让 SQL 变得像写普通代码一样，有清晰的自上而下的逻辑流。

---

## 三、语法是什么？

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

几个要点：
- 多个 CTE 之间用**逗号**隔开；
- **后面的 CTE 可以引用前面的 CTE**（CTE_Name2 用了 CTE_Name1），但反过来不行；
- 最后的主查询可以引用前面定义的所有 CTE。

---

## 四、简单例子

假设我们有一张全国城市人口表 `cities`，你想找出"人口超过平均值的北方城市"。

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

---

## 五、常见用法是什么？

- **多步聚合分析**：比如第一步求出每门课的平均分，第二步求出整个学校的所有课程平均分的"平均分"。
- **复用子查询**：如果你的最终查询中，需要好几次用到同一段复杂的 `JOIN` 结果（比如又要做分子又要做分母），你可以把它定义成一个 CTE，然后多次引用。
- **递归查询（进阶）**：`WITH RECURSIVE` 甚至可以用来查询树形结构（如员工上下级关系，多级分类菜单等）。

---

## 六、重要注意事项

- **生命周期极短**：CTE 仅仅在**当前这一条完整的 SQL 语句**执行期间存在！它绝不是真正的建表。这个查询执行完毕，CTE 就自动消失了。
- **性能评估**：在大多数现代数据库（包括较新的 PostgreSQL）中，CTE 相当于语法糖，优化器会自动将其展开并做全局优化。但在某些旧版本或特殊情况下，它可能成为性能瓶颈。不过作为初学者，现在请**无脑优先使用 CTE 以提升代码可读性**。

---

## 七、CTE 的物化 vs 内联（进阶）

这是理解 CTE 性能的关键，必须讲清楚。

### 1. PostgreSQL 12 之前：CTE 总是物化

在旧版本 PostgreSQL 里，CTE 是**强制物化（materialized）**的：优化器不会把 CTE 的内容"塞回"主查询里，而是把 CTE 当成一个**独立的临时结果集，先完整算一次存起来**，然后主查询再来读它。

好处：
- CTE 如果被主查询**引用多次**，只算一次，很省。

坏处：
- 主查询的 WHERE 条件**无法下推**到 CTE 内部。比如 CTE 里查了整张 enrollments，主查询再 `WHERE course_id = 1`，旧版本会先把整张表算出来，再过滤——明明只需要 course_id=1 的数据。

### 2. PostgreSQL 12 及之后：CTE 可以内联

从 PostgreSQL 12 开始，优化器默认**可以把 CTE 内联（inline）**：把 CTE 的定义展开合并进主查询，做全局优化，就像它是个子查询一样。这样主查询的 WHERE 条件就能下推到 CTE 内部，性能更好。

### 3. MATERIALIZED / NOT MATERIALIZED 关键字（PostgreSQL 12+）

你可以显式控制：

```sql
-- 强制物化：CTE 只算一次，结果固定（适合被多次引用、或想隔离计算）
WITH cte AS MATERIALIZED (
    SELECT * FROM enrollments
)
SELECT * FROM cte a JOIN cte b ON a.student_id = b.student_id;

-- 允许内联：优化器自由展开、条件下推（默认行为）
WITH cte AS NOT MATERIALIZED (
    SELECT * FROM enrollments
)
SELECT * FROM cte WHERE course_id = 1;
```

| 关键字 | 行为 | 适用场景 |
|---|---|---|
| `MATERIALIZED` | 强制算出一次结果 | CTE 被多次引用、计算昂贵且结果固定 |
| `NOT MATERIALIZED` | 允许内联展开 | 想让外层条件下推、单次引用 |

> 注意！！！这两个关键字是 **PostgreSQL 12+ 特有**，MySQL、SQL Server、Oracle 没有。它们是用来"调优"的，不是日常必写。初学者知道"现代 CTE 通常会被自动优化"就够了。

---

## 八、CTE 与子查询的性能对比（进阶）

| 对比项 | CTE | 子查询 |
|---|---|---|
| 可读性 | 高，自上而下分步 | 嵌套深了像洋葱 |
| 旧 PostgreSQL | 物化，算一次，多次引用高效，但条件难下推 | 通常内联，条件可下推 |
| 现代 PostgreSQL | 可内联，行为接近子查询 | 可内联 |
| 被多次引用 | 定义一次、多处用，方便 | 要复制粘贴或复用 |

结论：**可读性优先选 CTE；单次引用、且担心旧版本物化问题时，子查询也可。** 现代数据库两者性能基本打平。

---

## 九、递归 CTE（WITH RECURSIVE）——重点进阶

这是 CTE 最强大、也最绕的功能。用来处理**层级结构、树形结构、图遍历、序列生成**。

### 1. 语法结构

```sql
WITH RECURSIVE cte_name AS (
    -- ① 初始项（non-recursive term）：不引用自己，只执行一次
    SELECT ...
    UNION ALL
    -- ② 递归项（recursive term）：引用 cte_name 自己
    SELECT ...
    FROM cte_name
    WHERE ...
)
SELECT * FROM cte_name;
```

### 2. 执行机制（逐字讲清楚）

1. **先执行初始项**，得到一个初始结果集，放进工作表。
2. **执行递归项**：它从"上一次的结果"出发，算出新一轮结果。
3. **把新一轮结果并入总结果**，然后用新一轮结果作为"上一次的结果"，再跑一次递归项。
4. **重复步骤 2-3，直到递归项返回空集**（没有新行产生）。
5. 所有轮次的结果 `UNION ALL` 合并，就是最终输出。

> 记忆法：**初始项是"种子"，递归项是"生长规则"，一直长到长不出新东西为止。**

### 3. 应用场景一：层级查询（员工上下级）

假设有 `employees(emp_id, name, manager_id)`，要从某个老板开始，一层层找出所有下属。

```sql
WITH RECURSIVE org_tree AS (
    -- 初始项：从 CEO 开始（manager_id IS NULL 的人）
    SELECT emp_id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- 递归项：找 manager_id 等于上一轮 emp_id 的人
    SELECT e.emp_id, e.name, e.manager_id, ot.level + 1
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.emp_id
)
SELECT * FROM org_tree;
```

原理：先拿出 CEO（level=1），然后递归项找所有 manager_id=CEO 的员工（level=2），再找他们的下属（level=3）……直到某层没有新员工为止。`level` 记录了这个人在组织树的第几层。

### 4. 应用场景二：树形结构遍历（多级分类/楼中楼评论）

多级分类菜单：`categories(cat_id, name, parent_id)`，要展开成一棵带缩进的树。

```sql
WITH RECURSIVE cat_tree AS (
    SELECT cat_id, name, parent_id, 1 AS depth
    FROM categories
    WHERE parent_id IS NULL          -- 顶层分类

    UNION ALL

    SELECT c.cat_id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN cat_tree ct ON c.parent_id = ct.cat_id
)
SELECT depth, name FROM cat_tree;
```

评论楼中楼（一条主评论下的所有回复回复）同理，用 parent_comment_id 递归。

### 5. 应用场景三：生成序列（数字、日期）

递归 CTE 还能凭空生成一串数字，相当于简化版的 `generate_series`：

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n                -- 初始：从 1 开始
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 10   -- 每次 +1，直到 10
)
SELECT n FROM nums;
```

结果：1, 2, 3, ..., 10。

生成连续日期：

```sql
WITH RECURSIVE dates AS (
    SELECT DATE '2026-01-01' AS d
    UNION ALL
    SELECT d + 1 FROM dates WHERE d < DATE '2026-01-31'
)
SELECT d FROM dates;
```

### 6. 递归深度限制

递归是有深度上限的，防止无限递归把数据库跑爆。

- PostgreSQL：`max_recursion_depth`，默认上限足够日常用（几千层）。
- MySQL：`cte_max_recursion_depth`，默认 1000，可以 `SET cte_max_recursion_depth = 5000;` 调高。

> 注意！！！**递归项里一定要有终止条件**（比如 `WHERE n < 10`、`WHERE 还没到头`）。没有终止条件，递归会一直跑，直到撞深度上限报错。

### 7. UNION vs UNION ALL 在递归里

- `UNION ALL`：不去重，每轮新行都加进去，快。
- `UNION`：去重，如果你的图结构里有环，UNION 能靠去重避免重复（但不能完全防死循环）。

---

## 十、CYCLE 子句（PostgreSQL 14+，检测递归环）

图遍历里如果数据有环（A→B→C→A），普通递归 CTE 会沿着环无限绕，直到撞深度上限。PostgreSQL 14 提供 `CYCLE` 子句自动检测环：

```sql
WITH RECURSIVE graph AS (
    SELECT node, next_node, 1 AS level
    FROM edges WHERE node = 'A'
    UNION ALL
    SELECT e.node, e.next_node, g.level + 1
    FROM edges e
    JOIN graph g ON e.node = g.next_node
)
CYCLE node SET is_cycle TO true DEFAULT false   -- 检测是否成环
SELECT * FROM graph WHERE is_cycle = false;
```

原理：递归过程中记录访问过的节点，如果某个节点再次出现，就把 `is_cycle` 标成 true，帮你跳出环。

---

## 十一、SEARCH 子句（PostgreSQL 14+，控制遍历顺序）

递归 CTE 默认的输出顺序不好控制。`SEARCH` 子句让你显式选择深度优先还是广度优先：

```sql
WITH RECURSIVE org_tree AS (
    SELECT emp_id, name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id, ot.level + 1
    FROM employees e JOIN org_tree ot ON e.manager_id = ot.emp_id
)
SEARCH DEPTH FIRST BY emp_id SET order_col   -- 深度优先
SELECT * FROM org_tree ORDER BY order_col;
```

- `DEPTH FIRST`：一条路走到底再回头（先看完一个分支的所有后代）。
- `BREADTH FIRST`：按层遍历（先看完第一层所有人，再看第二层）。

| 方式 | 特点 | 适用 |
|---|---|---|
| 深度优先 | 沿分支深入 | 打印整棵树、找叶子 |
| 广度优先 | 按层扩展 | 最短路径、按层统计 |

> 注意！！！`CYCLE` 和 `SEARCH` 都是 PostgreSQL 14+ 的高级特性，日常业务很少用到。了解"递归 CTE 能处理树和图、有防环和排序机制"即可。

---

## 十二、CTE 的其他要点

### 1. 链式定义规则

```text
后面的 CTE 可以引用前面的 CTE；
前面的 CTE 不能引用后面的 CTE。
```

所以定义顺序就是依赖顺序：被依赖的先写。

### 2. 列别名定义方式

除了在 CTE 内部给列起别名，还可以在 CTE 名字后面统一列出来：

```sql
WITH avg_tbl(course_id, avg_score) AS (
    SELECT course_id, AVG(score) FROM enrollments GROUP BY course_id
)
SELECT * FROM avg_tbl;
```

### 3. CTE 与 INSERT/UPDATE/DELETE 配合

CTE 甚至可以包一个写操作，用 `RETURNING` 把改动结果带出来：

```sql
WITH updated AS (
    UPDATE enrollments
    SET score = score + 5
    WHERE score < 60
    RETURNING student_id, score
)
SELECT * FROM updated;   -- 查看被更新的行
```

### 4. CTE 与临时表、视图的对比

| 对象 | 生命周期 | 是否真存数据 | 适用 |
|---|---|---|---|
| CTE | 仅当前这一条 SQL | 逻辑上的虚拟表（可能物化） | 单条查询内部分步 |
| 临时表 | 会话期间 | 真存临时数据 | 多条 SQL 复用中间结果 |
| 视图 | 永久（定义保存） | 不存数据，只存查询定义 | 复用的查询逻辑 |

> 记忆法：CTE 是"一次性的、查询内的临时视图"。跨多条语句要用同一个中间结果，才考虑临时表；逻辑要长期复用，才建视图。

---

## 本文件覆盖的原文档知识点清单

- [x] CTE 是什么（Common Table Expressions，查询内临时定义的虚拟结果表并起名）
- [x] 为什么有 CTE（替代嵌套子查询的洋葱代码，逻辑扁平化、自上而下）
- [x] 语法：WITH CTE1 AS (...), CTE2 AS (... FROM CTE1 ...) SELECT * FROM CTE2
- [x] 简单例子（城市人口表，AvgPop→NorthCities→联合查询三步）
- [x] 常见用法：多步聚合分析、复用子查询、递归查询
- [x] 重要注意：生命周期极短（仅当前 SQL 执行期间）、性能评估（现代数据库自动展开，初学者无脑优先用 CTE）
- [x] CTE 的物化 vs 内联（PostgreSQL 12 前总物化、12 后可内联）
- [x] MATERIALIZED / NOT MATERIALIZED 关键字及适用场景
- [x] CTE 与子查询的性能对比与选择
- [x] 递归 CTE（WITH RECURSIVE）：语法结构、执行机制详解
- [x] 递归 CTE 应用场景：层级查询、树形结构遍历、图遍历、序列生成（数字/日期），均有完整示例
- [x] 递归深度限制（max_recursion_depth / cte_max_recursion_depth）
- [x] 递归中 UNION vs UNION ALL（去重 vs 保留）
- [x] CYCLE 子句（PostgreSQL 14+，检测递归环）
- [x] SEARCH 子句（PostgreSQL 14+，深度优先 vs 广度优先）
- [x] 多个 CTE 链式定义和引用规则（后面引用前面）
- [x] CTE 的递归引用（自己引用自己，需 RECURSIVE）
- [x] CTE 与 INSERT/UPDATE/DELETE 配合（RETURNING）
- [x] CTE 的列别名定义方式
- [x] CTE 与临时表、视图的对比
- [x] CTE 在复杂报表中的多步拆解应用模式
