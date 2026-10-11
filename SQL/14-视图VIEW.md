# 视图（VIEW）：保存查询的捷径

## 1. 是什么？

视图是**一段被保存下来的 SQL 查询语句，并在数据库里被当做一张"虚拟表"来对待**。

一旦建好，你就可以对这个视图使用 `SELECT * FROM 视图名`，就像查询真表一样。

拆开看这个定义里有两个关键词：

```text
"被保存下来的 SQL 查询语句"  →  它本质上是一段 SELECT，不是数据
"被当做一张虚拟表来对待"      →  你可以像查表一样查它，但它本身不存数据
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

## 2. 为什么有这个东西？

### 用途一：隐藏复杂逻辑（化繁为简）

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

### 用途二：安全与权限隔离

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

### 用途三：兼容性隔离（底层表改了，上层代码不动）

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

## 3. 语法是什么？

### 最基本的创建语法

```sql
CREATE VIEW 视图名称 AS
SELECT 字段1, 字段2...
FROM 表名
WHERE 条件...;
```

### 创建或替换（如果已存在则覆盖）

```sql
CREATE OR REPLACE VIEW 视图名称 AS
SELECT ...;
```

> `OR REPLACE` 的意思是：如果这个名字的视图已经存在，就用新的定义替换掉它，而不是报错。这在迭代开发中非常常用——你改了视图逻辑，直接重跑这条语句就行。

但要注意：`CREATE OR REPLACE VIEW` 对已有列有严格限制——已有列的**名字、顺序和数据类型**必须保持不变。不过，它**允许在列的末尾追加新列**。如果你需要删除列、在中间插入列、改变已有列的顺序或类型，就需要先 `DROP VIEW` 再 `CREATE VIEW`。

### 删除视图

```sql
DROP VIEW 视图名称;
```

如果视图不存在会报错，可以加 `IF EXISTS` 避免：

```sql
DROP VIEW IF EXISTS 视图名称;
```

### 给视图的列起别名

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

## 4. 简单例子

### 例子一：查看哪些课程没人选

```sql
CREATE VIEW empty_courses AS
SELECT c.course_name
FROM courses c
LEFT JOIN enrollments e ON c.course_id = e.course_id
WHERE e.student_id IS NULL;

-- 以后直接查：
SELECT * FROM empty_courses;
```

### 例子二：只看活跃学生

```sql
CREATE VIEW active_students AS
SELECT student_id, student_name, major, created_at
FROM students
WHERE is_active = TRUE;
```

以后任何查询只要想排除已退学的学生，直接 `FROM active_students` 就行，不用每次都写 `WHERE is_active = TRUE`。

### 例子三：把三表 JOIN 封装成成绩单宽表

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

## 5. 常见用法是什么？

- **把基础过滤条件封在视图里**：比如所有表都有 `is_deleted = false`、`status = 'active'` 这种软删除/状态过滤，写一个视图统一处理，避免每个查询都漏写。
- **把核心几张表 JOIN 好做宽表**：数据分析师经常需要学生+选课+课程的关联数据，建一个视图一次性 JOIN 好，分析师直接查视图。
- **给不同角色提供不同粒度的数据**：管理层看汇总视图，普通员工看明细视图，通过视图控制每个人能看到什么。
- **兼容老代码**：表结构重构后，用视图模拟旧表结构，让上层应用零改动过渡。

---

## 6. 重要注意事项（最容易犯错的地方！）

### 注意一：普通视图绝对不存数据！！

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

### 注意二：视图能更新吗？

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

### 注意三：WITH CHECK OPTION——防止通过视图插入"看不见"的数据

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

答案是：**默认可以**。插入成功了，但这个学生不会出现在 `active_students` 视图里（因为他不活跃），你"插进去就找不到了"。

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

> 记忆法：`WITH CHECK OPTION` = "通过这个视图插入或修改的数据，必须能被这个视图自己看到"。

### 注意四：视图可以嵌套，但别套太深

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

### 注意五：视图里的 ORDER BY 不可靠

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

### 注意六：物化视图（Materialized View）——真的存数据的视图

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

# 进阶一：可更新视图的详细规则

## 7.1 什么时候视图是可更新的

PostgreSQL 规定，一个视图要能被 `INSERT/UPDATE/DELETE`，必须同时满足：

1. `FROM` 后面只能有**一个**表（或一个可更新视图），不能是多表 JOIN。
2. 视图定义里**不能有** `DISTINCT`、`GROUP BY`、`HAVING`、`LIMIT`、`OFFSET`。
3. 视图定义里**不能有**聚合函数（SUM/COUNT/AVG...）、窗口函数。
4. 视图定义里**不能有** `UNION`、`INTERSECT`、`EXCEPT`。
5. 视图的 SELECT 列表里不能有表达式、函数调用（除非是透传列）。

## 7.2 INSERT 的额外限制

通过视图 INSERT 时，基表里所有"NOT NULL 且没有 DEFAULT 值"的列，都必须在视图的列列表里出现。否则数据库不知道该往那个列填什么。

```sql
-- 假设 students 表：student_id (主键自增), name (NOT NULL), email (NOT NULL, 有 DEFAULT '')
CREATE VIEW student_names AS
SELECT student_id, name FROM students;

-- 通过这个视图 INSERT：email 列有 DEFAULT，所以可以不出现
INSERT INTO student_names (name) VALUES ('Tom');
-- 实际等价于：INSERT INTO students (name, email) VALUES ('Tom', '');
```

## 7.3 自动更新视图 vs INSTEAD OF 触发器

如果视图不满足自动可更新的条件（比如它是个 JOIN 视图），你还可以给它加一个 `INSTEAD OF` 触发器，自己定义"通过这个视图 INSERT/UPDATE/DELETE 时该干什么"：

```sql
-- 先写一个触发器函数
CREATE OR REPLACE FUNCTION transcript_update_fn() RETURNS TRIGGER AS $$
BEGIN
    UPDATE enrollments SET score = NEW.score
    WHERE student_name = NEW.student_name AND course_name = NEW.course_name;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 把触发器挂到视图上
CREATE TRIGGER transcript_upd
INSTEAD OF UPDATE ON transcript
FOR EACH ROW EXECUTE FUNCTION transcript_update_fn();
```

这样即使 transcript 是三表 JOIN 的复杂视图，也能通过它 UPDATE 了——具体改哪张表由你写的触发器函数决定。

---

# 进阶二：WITH LOCAL CHECK OPTION vs WITH CASCADED CHECK OPTION

当视图嵌套时，检查条件到底检查哪一层？

## 8.1 两种级别

- **WITH LOCAL CHECK OPTION**：只检查**当前视图自己**的 WHERE 条件。
- **WITH CASCADED CHECK OPTION**：检查当前视图的条件，**以及所有底层视图**的 WHERE 条件。这是 PostgreSQL 的默认行为（不写 LOCAL/CASCADED 时默认 CASCADED）。

## 8.2 详细示例展示差异

假设有两层视图：

```sql
-- 底层视图：只看活跃学生
CREATE VIEW v_active AS
SELECT * FROM students WHERE is_active = TRUE;

-- 上层视图：在 v_active 基础上只看 CS 专业
CREATE VIEW v_cs AS
SELECT * FROM v_active WHERE major = 'CS'
WITH LOCAL CHECK OPTION;   -- 注意：LOCAL
```

现在通过 `v_cs` 插入一条数据：

```sql
INSERT INTO v_cs (student_name, major, is_active)
VALUES ('Bob', 'Math', TRUE);
```

- `major = 'Math'`：违反 v_cs 自己的 `major='CS'` 条件，LOCAL 会检查到，报错。✅
- 但如果插入 `major='CS', is_active=FALSE`：v_cs 自己的条件满足了（major=CS），LOCAL 不检查底层 v_active 的 `is_active=TRUE`，所以**能插进去**！但插进去后在 v_active 和 v_cs 里都看不到这条记录。

如果改成 `WITH CASCADED CHECK OPTION`：

```sql
CREATE VIEW v_cs AS
SELECT * FROM v_active WHERE major = 'CS'
WITH CASCADED CHECK OPTION;
```

再插入 `major='CS', is_active=FALSE`：CASCADED 会连带检查底层 v_active 的条件，`is_active=FALSE` 不满足，直接报错。

| 视图定义 | 插入 (major='CS', is_active=FALSE) |
|---|---|
| WITH LOCAL CHECK OPTION | 成功（但数据"隐身"了） |
| WITH CASCADED CHECK OPTION（默认） | 报错 |

---

# 进阶三：视图与权限

## 9.1 基本授权

```sql
-- 给角色授予查视图的权限
GRANT SELECT ON public_emps TO intern_role;

-- 可更新视图还能授权写
GRANT INSERT, UPDATE, DELETE ON active_students TO intern_role;

-- 收回权限
REVOKE ALL ON public_emps FROM intern_role;
```

## 9.2 视图安全隔离的核心

用户**不需要基表的 SELECT 权限**，只需要视图的 SELECT 权限，就能通过视图查到数据。这是视图做权限隔离的根基：

```sql
-- 实习生对 employees 表没有任何权限
REVOKE ALL ON employees FROM intern_role;

-- 但对 public_emps 视图有 SELECT 权限
GRANT SELECT ON public_emps TO intern_role;

-- 实习生执行：
SELECT * FROM public_emps;  -- ✅ 能查到 id, name, department
SELECT * FROM employees;    -- ❌ 报错：permission denied
```

## 9.3 security_invoker 视图（PostgreSQL 15+）

视图默认是以**视图所有者**的权限执行的（security definer）——意思是视图能查到基表数据，是因为视图的创建者有权限，而不是查视图的人有权限。这在大多数情况下是好事（安全隔离），但有时你希望"以查视图的人的权限来查"。

PostgreSQL 15 引入了 `security_invoker` 选项：

```sql
CREATE VIEW my_view WITH (security_invoker = true) AS
SELECT * FROM sensitive_table;

-- 现在查这个视图时，会用"当前用户"对 sensitive_table 的权限来检查
-- 如果当前用户对 sensitive_table 没权限，即使他对视图有权限，也查不到
```

| | security definer（默认） | security_invoker（PG15+） |
|---|---|---|
| 以谁的权限查基表 | 视图创建者 | 当前查询者 |
| 安全隔离 | 强（藏敏感列） | 弱（但权限更直观） |
| 适合场景 | 给不同角色暴露不同视图 | 多租户、行级安全场景 |

---

# 进阶四：物化视图的刷新策略

## 10.1 全量刷新

```sql
REFRESH MATERIALIZED VIEW sales_summary;
```

完全重新执行视图背后的 SQL，把旧结果整个丢掉，写入新结果。**刷新期间会锁住物化视图**，其他查询读不了。

## 10.2 增量刷新（CONCURRENTLY）

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
```

不锁表，其他查询可以继续读。但有两个硬性要求：
1. 物化视图上必须有一个**唯一索引**（所有列都在索引里，或者用了 WHERE 的部分唯一索引）。
2. 物化视图至少被刷新过一次（不能是空的）。

```sql
-- 先建唯一索引，CONCURRENTLY 才有用
CREATE UNIQUE INDEX idx_sales_summary_date ON sales_summary(date);

-- 之后就可以并发刷新了
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
```

## 10.3 刷新方式对比

| | REFRESH（全量） | REFRESH CONCURRENTLY |
|---|---|---|
| 是否锁表 | 锁 | 不锁 |
| 速度 | 快（直接重写） | 慢（要对比新旧版本找差异） |
| 前提 | 无 | 必须有唯一索引 |
| 适合 | 夜间批处理 | 白天高频刷新 |

## 10.4 自动刷新方案

PostgreSQL 原生不会自动刷新物化视图。常见做法：

1. **pg_cron 定时任务**（PostgreSQL 扩展）：

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
-- 每天凌晨 2 点刷新一次
SELECT cron.schedule('refresh-sales-summary', '0 2 * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary');
```

2. **触发器 + NOTIFY/LISTEN**：基表 INSERT/UPDATE 后发通知，应用监听到就刷新（实时性高但实现复杂）。

---

# 进阶五：物化视图日志

Oracle 有物化视图日志（Materialized View Log），基表的每一次变更都会被记录到一张日志表里，增量刷新时只需要把日志里的变更应用到物化视图上，非常高效。

PostgreSQL **原生不支持**物化视图日志。要实现类似的增量刷新，得自己写触发器记录变更，或者用第三方扩展（如 `pg_matviews_ext`）。这是 PG 物化视图相比 Oracle 的一个短板。

---

# 进阶六：视图依赖与 CASCADE

## 11.1 视图依赖谁

视图依赖于它背后的基表和其他视图。你 DROP 基表时，如果有视图依赖它，默认会报错：

```sql
DROP TABLE students;
-- 报错：cannot drop table students because other objects depend on it
-- 依赖对象：view active_students, view transcript
```

## 11.2 CASCADE 级联删除

加 `CASCADE` 会把所有依赖的视图一起删掉：

```sql
DROP TABLE students CASCADE;
-- 不仅删表，还连带把 active_students、transcript 等依赖视图全删了
```

> 注意！！！`CASCADE` 是危险操作，可能删掉你不想删的东西。生产环境慎用，先查清楚依赖关系。

## 11.3 查看视图依赖

```sql
-- 查某个视图依赖哪些对象
SELECT dependent_ns.nspname AS schema, dependent_view.relname AS view_name,
       source_ns.nspname AS base_schema, source_table.relname AS base_table
FROM pg_depend
JOIN pg_rewrite ON pg_depend.objid = pg_rewrite.oid
JOIN pg_class dependent_view ON pg_rewrite.ev_class = dependent_view.oid
JOIN pg_class source_table ON pg_depend.refobjid = source_table.oid
JOIN pg_namespace dependent_ns ON dependent_view.relnamespace = dependent_ns.oid
JOIN pg_namespace source_ns ON source_table.relnamespace = source_ns.oid
WHERE source_table.relname = 'students';

-- psql 里更简单：
\d+ active_students
```

---

# 进阶七：临时视图（TEMP VIEW）

只在当前数据库会话里存在，会话断开自动删除：

```sql
CREATE TEMP VIEW temp_today_orders AS
SELECT * FROM orders WHERE created_at = CURRENT_DATE;

-- 当前会话里随便查
SELECT * FROM temp_today_orders;
-- 断开重连后，这个视图就没了
```

适合：会话级的临时查询封装、测试调试。

---

# 进阶八：递归视图（RECURSIVE VIEW）

递归视图本质上是递归 CTE 的语法糖：

```sql
CREATE RECURSIVE VIEW org_chart (employee_id, manager_id, depth) AS
WITH RECURSIVE org AS (
    SELECT employee_id, manager_id, 1 AS depth
    FROM employees
    WHERE manager_id IS NULL  -- 从 CEO 开始
    UNION ALL
    SELECT e.employee_id, e.manager_id, o.depth + 1
    FROM employees e JOIN org o ON e.manager_id = o.employee_id
)
SELECT * FROM org;
```

以后查 `SELECT * FROM org_chart` 就能拿到整棵组织树。

---

# 进阶九：视图的系统表和元数据

```sql
-- 查所有普通视图的定义
SELECT viewname, definition FROM pg_views WHERE viewname = 'active_students';

-- 查物化视图
SELECT matviewname, matviewdef FROM pg_matviews;

-- 更灵活地取视图定义
SELECT pg_get_viewdef('active_students'::regclass);

-- psql 里
\d+ active_students   -- 看视图定义和依赖
```

---

# 进阶十：视图与性能优化

## 16.1 视图展开（View Inlining）

大多数情况下，优化器会把视图"展开"——把视图的定义 SQL 直接替换到外层查询里，当成一条大 SQL 来优化。这样视图就不会成为性能瓶颈。

```sql
-- 你写的：
SELECT * FROM active_students WHERE major = 'CS';

-- 优化器实际执行的（视图展开后）：
SELECT * FROM students WHERE is_active = TRUE AND major = 'CS';
```

## 16.2 条件下推（Predicate Pushdown）

外层查询的 WHERE 条件能不能下推到视图内部，决定了视图能不能利用基表的索引。好消息是 PostgreSQL 通常能做到这一点——上面例子里 `major='CS'` 成功下推，就能用 major 列上的索引。

## 16.3 什么情况下视图无法展开

- 视图里有 `LIMIT`、`OFFSET`。
- 视图里有 `VOLATILE` 函数（每次调用结果可能不同）。
- 递归视图。
- 嵌套特别深的视图（优化器可能直接放弃）。

这些情况下视图变成了一个"黑盒"，外层条件不能下推，性能可能变差。

---

# 进阶十一：视图与索引

- **普通视图不能建索引**：因为它不存数据。你在 `active_students(email)` 上建索引，其实是建在 `students(email)` 上的。
- **物化视图可以建索引**：因为它真的存数据。

```sql
-- 物化视图上建索引，查询飞快
CREATE MATERIALIZED VIEW customer_summary AS
SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total
FROM orders GROUP BY customer_id;

CREATE UNIQUE INDEX idx_customer_summary_id ON customer_summary(customer_id);
-- 之后按 customer_id 查物化视图，走索引
```

---

# 进阶十二：最佳实践和反模式

## 18.1 什么时候该用视图

- 反复用到的复杂 JOIN，封装一次到处用。
- 做行级权限隔离（敏感列不暴露）。
- 兼容老代码的过渡层。

## 18.2 什么时候不该用视图

- **不要为了"性能"建普通视图**——它不加速。要加速用物化视图。
- **不要嵌套超过 2-3 层**——调试地狱。
- **不要在视图里写 ORDER BY**——不保证生效，外层查的时候再排。
- **不要把视图当"临时变量"到处建**——视图是数据库里的持久对象，建多了管理混乱。

## 18.3 命名规范

```text
v_前缀        → 普通视图，如 v_active_students
mv_前缀       → 物化视图，如 mv_sales_summary
```

---

# 本文件覆盖的原文档知识点清单

- [x] 视图是什么（一段被保存的 SQL 查询，虚拟表，本质是 SELECT 不是数据）
- [x] 视图和真表的区别对比表（存不存数据、数据从哪来、能不能 INSERT、占不占硬盘）
- [x] 记忆法：视图 = 给复杂 SELECT 起名字（Excel 公式命名）
- [x] 用途一：隐藏复杂逻辑（8 个 JOIN 包装成视图）
- [x] 用途二：安全与权限隔离（员工表去掉敏感列，GRANT 视图给实习生）
- [x] 用途三：兼容性隔离（表结构重构后视图模拟旧表，老应用零改动）
- [x] 语法：CREATE VIEW AS
- [x] 语法：CREATE OR REPLACE VIEW（已存在覆盖；已有列名/顺序/类型不变，允许末尾追加列）
- [x] 语法：DROP VIEW / DROP VIEW IF EXISTS
- [x] 语法：给视图列起别名 CREATE VIEW name (col1, col2) AS SELECT ...
- [x] 简单例子：empty_courses（没人选的课）
- [x] 简单例子：active_students（活跃学生）
- [x] 简单例子：transcript（三表 JOIN 成绩单宽表）
- [x] 常见用法：基础过滤条件封装、宽表 JOIN、不同角色不同粒度、兼容老代码
- [x] 注意一：普通视图绝对不存数据（书签比喻，视图不提升性能）
- [x] 注意二：视图能更新吗（简单单表视图可更新；含 GROUP BY/JOIN/聚合/DISTINCT 只读；cannot update a view because it contains aggregates）
- [x] 注意三：WITH CHECK OPTION（防止插入"看不见"的数据；new row violates check option）
- [x] 注意四：视图可以嵌套但别套太深（2-3 层经验法则）
- [x] 注意五：视图里的 ORDER BY 不可靠（只有最外层 ORDER BY 保证排序）
- [x] 注意六：物化视图（CREATE MATERIALIZED VIEW，REFRESH，普通视图 vs 物化视图对比表）
- [x] （进阶补充）可更新视图详细规则、INSERT 限制、INSTEAD OF 触发器
- [x] （进阶补充）WITH LOCAL vs CASCADED CHECK OPTION 对比
- [x] （进阶补充）视图权限 GRANT/REVOKE、security_invoker（PG15+）
- [x] （进阶补充）物化视图刷新策略：全量 vs CONCURRENTLY、唯一索引要求、pg_cron 自动刷新
- [x] （进阶补充）物化视图日志（PG 原生不支持，Oracle 对比）
- [x] （进阶补充）视图依赖、DROP ... CASCADE/RESTRICT、pg_depend
- [x] （进阶补充）临时视图 TEMP VIEW
- [x] （进阶补充）递归视图 RECURSIVE VIEW
- [x] （进阶补充）pg_views / pg_matviews / pg_get_viewdef
- [x] （进阶补充）视图展开、条件下推、无法展开的场景
- [x] （进阶补充）普通视图不能建索引、物化视图可以建索引
- [x] （进阶补充）最佳实践与反模式、命名规范
