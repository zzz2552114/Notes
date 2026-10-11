# PostgreSQL 专属特性（快速预览）

至此，标准 SQL 的主线已经走通。在真实开发中，PostgreSQL 提供了一些极其舒爽的方言和独有特性，它们能极大提高开发效率。你不需要现在就精通，但务必混个眼熟。

## 1. RETURNING 子句

- **痛点**：以往 `INSERT` 进去一条新数据，往往它的 ID 是自增生成的。如果后端代码紧接着要用到这个新 ID，不得不再发一条 `SELECT max(id)` 或者调一个 `LAST_INSERT_ID`，麻烦且并发下不安全。
- **PG 方案**：直接写 `INSERT INTO students (name) VALUES ('Tom') RETURNING student_id, created_at;`，插入完毕后，立刻直接把这行新数据的这两个字段返回给你！

## 2. ON CONFLICT (UPSERT 功能)

- **痛点**：我想插入一条数据，但万一这人已经存在了（触发了唯一主键冲突），我该怎么办？以往要先 `SELECT` 判断存不存在，存在就发 `UPDATE`，不存在发 `INSERT`，并发时容易死锁。
- **PG 方案**：

```sql
INSERT INTO students (student_id, age) VALUES (1, 18)
ON CONFLICT (student_id) DO UPDATE SET age = 18;
```

俗称"不存在则插入，存在则更新"，一条搞定，绝对并发安全。

## 3. 强大的类型转换符 `::`

- **痛点**：标准 SQL 的类型转换很繁琐：`CAST('123' AS INTEGER)`。
- **PG 方案**：直接写 `'123'::INTEGER` 或者 `date_string::TIMESTAMP`，极其简练，还能嵌套使用。

## 4. ILIKE 不区分大小写匹配

`LIKE 'abc%'` 是严格区分大小写的。如果在 MySQL 里想忽略，常常要改排序规则。在 PG 里，直接用 `student_name ILIKE '%alice%'` 就能找出 'Alice', 'ALICE', 'alice'，非常实用。

## 5. 无敌的 JSONB 支持

PG 允许你把一个巨大的、没有固定结构的 JSON 数据直接塞进名为 JSONB 的列里。不仅能存，你还能直接在 SQL 里用专属箭头运算符提取内部节点：`SELECT config->>'theme' FROM users;`。甚至还能为 JSON 内部的某个深度属性建立索引，查询飞快。PG 凭借此特性完美兼顾了关系型（SQL）和非关系型（NoSQL/MongoDB）的优势。

---

# 进阶一：RETURNING 子句深度讲解

## 6.1 语法

```sql
-- INSERT 后返回新行（包括自动生成的 ID、DEFAULT 值）
INSERT INTO students (student_name) VALUES ('Tom')
RETURNING student_id, created_at;

-- UPDATE 后返回更新后的值
UPDATE students SET major = 'Math' WHERE student_id = 1
RETURNING student_id, student_name, major;

-- DELETE 后返回被删除的行（删除前的值）
DELETE FROM students WHERE student_id = 99
RETURNING student_name, major;
```

## 6.2 与应用程序的交互

一条 SQL 完成"写入 + 读回新 ID"，省掉一次网络往返，而且是并发安全的——不会因为两个请求同时 INSERT 而拿到错误的 max(id)。

## 6.3 在 CTE 中使用（可修改 CTE）

```sql
-- 插入一个学生，同时把他的 ID 写入选课表
WITH inserted AS (
    INSERT INTO students (student_name) VALUES ('Jerry')
    RETURNING student_id
)
INSERT INTO enrollments (student_id, course_id)
SELECT student_id, 1 FROM inserted;
```

---

# 进阶二：ON CONFLICT（UPSERT）深度讲解

## 7.1 完整语法

```sql
INSERT INTO table (col1, col2)
VALUES (v1, v2)
ON CONFLICT (冲突目标)
DO UPDATE SET col1 = new_value
WHERE 条件;
-- 或者
ON CONFLICT (...) DO NOTHING;
```

## 7.2 冲突目标

- 列名：`ON CONFLICT (email)`
- 约束名：`ON CONFLICT ON CONSTRAINT students_email_key`
- 部分唯一索引的谓词：`ON CONFLICT (email) WHERE is_active = TRUE`

## 7.3 EXCLUDED 伪表

冲突时，`EXCLUDED` 代表"本来要插入但冲突了的那一行"：

```sql
INSERT INTO students (student_id, name, age)
VALUES (1, 'Tom', 20)
ON CONFLICT (student_id) DO UPDATE
SET name = EXCLUDED.name,
    age  = EXCLUDED.age;
-- 等价于：存在就用新值覆盖旧值
```

## 7.4 DO NOTHING

```sql
-- 冲突就什么都不做（插入被忽略）
INSERT INTO students (student_id, name) VALUES (1, 'Tom')
ON CONFLICT DO NOTHING;
```

## 7.5 与其他数据库对比

| 数据库 | UPSERT 语法 |
|---|---|
| PostgreSQL | `INSERT ... ON CONFLICT ... DO UPDATE/NOTHING` |
| MySQL | `INSERT ... ON DUPLICATE KEY UPDATE` |
| SQL Server / Oracle | `MERGE INTO ... WHEN MATCHED ...` |

PG 的写法最干净，且是原子操作不会死锁。

---

# 进阶三：类型转换符 `::`

```sql
'123'::INTEGER              -- 字符串转整数
'2024-01-01'::DATE          -- 字符串转日期
123::TEXT                   -- 数字转字符串
'{"a":1}'::JSONB            -- 字符串转 JSONB
ARRAY[1,2,3]::TEXT[]        -- 数组类型转换

-- 嵌套转换
'123'::TEXT::INTEGER
```

等价于 `CAST('123' AS INTEGER)`，但写起来短得多。

---

# 进阶四：ILIKE 深度讲解

```sql
-- 不区分大小写匹配
SELECT * FROM students WHERE student_name ILIKE '%alice%';
-- 能匹配 'Alice', 'ALICE', 'alice', 'aLiCe'
```

通配符 `%`（任意多字符）和 `_`（单个字符）和 LIKE 完全一致。

性能提示：ILIKE 默认不走普通 B-Tree 索引。要加速可以：
- 建表达式索引：`CREATE INDEX ON students(LOWER(student_name));` 然后用 `WHERE LOWER(student_name) = LOWER('%alice%')`。
- 装 `pg_trgm` 扩展做三元组索引，支持任意模糊匹配。

---

# 进阶五：JSONB 深度讲解

## 10.1 JSON vs JSONB

| | JSON | JSONB |
|---|---|---|
| 存储格式 | 原始文本 | 解析后的二进制 |
| 保留空格/键顺序 | 是 | 否 |
| 能否建索引 | 不能 | 能（GIN 索引） |
| 查询速度 | 慢 | 快 |

推荐用 JSONB。

## 10.2 常用运算符

```sql
-- -> 取字段，返回 JSONB
SELECT data->'name' FROM users;

-- ->> 取字段，返回 TEXT
SELECT data->>'name' FROM users;

-- #> 按路径取
SELECT data#>'{address, city}' FROM users;

-- @> 包含（适合查"role 是 admin 的所有用户"）
SELECT * FROM users WHERE data @> '{"role": "admin"}';

-- ? 键是否存在
SELECT * FROM users WHERE data ? 'email';

-- || 合并两个 JSONB
SELECT '{"a":1}'::JSONB || '{"b":2}'::JSONB;

-- - 删除键
SELECT data - 'temp_field' FROM users;
```

## 10.3 JSONB 索引

```sql
-- 通用 GIN 索引
CREATE INDEX idx_users_data ON users USING GIN(data);

-- 针对 @> 包含查询的更快索引
CREATE INDEX idx_users_data ON users USING GIN(data jsonb_path_ops);

-- 针对某个具体 key 的表达式索引
CREATE INDEX idx_users_role ON users((data->>'role'));
```

## 10.4 JSONB 更新

```sql
-- jsonb_set 更新嵌套字段
UPDATE users
SET data = jsonb_set(data, '{theme}', '"dark"')
WHERE id = 1;
```

---

# 进阶六：DISTINCT ON（PG 特有）

按某列去重，每组保留排序后的第一行：

```sql
SELECT DISTINCT ON (user_id)
       user_id, order_id, created_at
FROM orders
ORDER BY user_id, created_at DESC;
-- 每个 user_id 只保留最新一条订单
```

等价于 `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) = 1`，但写法更简洁。必须配合 ORDER BY（DISTINCT ON 的列要放最前面），否则结果不确定。

---

# 进阶七：LATERAL 横向连接

子查询可以引用 FROM 中前面出现的表的列：

```sql
-- 每个用户查他最新的 3 条订单
SELECT u.user_id, u.username, o.order_id, o.amount
FROM users u
LEFT JOIN LATERAL (
    SELECT order_id, amount
    FROM orders
    WHERE orders.user_id = u.user_id
    ORDER BY created_at DESC
    LIMIT 3
) o ON TRUE;
```

这是"TOP N per group"的另一种解法，比窗口函数更直观。

---

# 进阶八：IDENTITY vs SERIAL

```sql
-- 推荐：标准 SQL IDENTITY
CREATE TABLE students (
    student_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT
);

-- 老写法（不推荐）
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name TEXT
);
```

| | GENERATED ALWAYS AS IDENTITY | SERIAL |
|---|---|---|
| 标准符合性 | SQL 标准 | PG 私有 |
| 能否手动插入值 | 不能（除非 OVERRIDING SYSTEM VALUE） | 能（可能和序列冲突） |
| 推荐度 | ✅ 推荐 | 历史遗留 |

---

# 进阶九：数组类型

```sql
-- 定义数组列
CREATE TABLE users (id INT, tags TEXT[]);

-- 插入
INSERT INTO users VALUES (1, ARRAY['student', 'cs']);
INSERT INTO users VALUES (2, '{teacher, math}');

-- 访问（PG 数组下标从 1 开始！）
SELECT tags[1] FROM users WHERE id = 1;

-- 查询包含某元素的行
SELECT * FROM users WHERE tags @> ARRAY['student'];

-- 展开为行
SELECT unnest(tags) FROM users WHERE id = 1;

-- GIN 索引加速数组查询
CREATE INDEX idx_users_tags ON users USING GIN(tags);
```

---

# 进阶十：范围类型（RANGE）

```sql
-- 时间区间
SELECT '[2024-01-01, 2024-02-01)'::tsrange;

-- 查某段时间是否和已有预约重叠
SELECT * FROM bookings
WHERE time_range && '[2024-01-15, 2024-01-20)'::tsrange;

-- 排他约束：同一个会议室的预约时间段不能重叠
CREATE TABLE meetings (
    room_id INT,
    time_range tsrange,
    EXCLUDE USING GIST (room_id WITH =, time_range WITH &&)
);
```

范围类型 + GiST 索引是做"时间段不重叠预约"的神器。

---

# 进阶十一：全文搜索

```sql
-- 建表
CREATE TABLE articles (id INT, title TEXT, body TEXT);

-- 建全文索引
CREATE INDEX idx_articles_search ON articles
USING GIN (to_tsvector('english', title || ' ' || body));

-- 搜索
SELECT title
FROM articles
WHERE to_tsvector('english', title || ' ' || body)
      @@ plainto_tsquery('english', 'postgresql index');
```

中文搜索需要装 `zhparser` 扩展（基于 jieba 分词）。

---

# 进阶十二：LISTEN / NOTIFY

PostgreSQL 内置的发布-订阅消息机制：

```sql
-- 会话 A：监听频道
LISTEN channel_a;

-- 会话 B：发通知
NOTIFY channel_a, 'new order #123';

-- 会话 A 会立刻收到通知
```

适合做缓存失效通知、实时更新、轻量级任务队列。

---

# 进阶十三：pg_stat_statements

记录所有 SQL 的执行统计（调用次数、总耗时、平均耗时、IO 等），是慢查询分析神器：

```sql
CREATE EXTENSION pg_stat_statements;

-- 查最慢的 10 条 SQL（按总耗时排序）
SELECT query, total_exec_time, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

---

# 进阶十四：扩展（Extension）系统

```sql
CREATE EXTENSION pg_trgm;    -- 三元组索引，模糊搜索
CREATE EXTENSION "uuid-ossp"; -- UUID 生成
CREATE EXTENSION postgis;    -- 地理空间
CREATE EXTENSION pgcrypto;   -- 加密
CREATE EXTENSION citext;     -- 不区分大小写文本
CREATE EXTENSION pg_cron;    -- 定时任务

-- 查看已装扩展
\dx
```

---

# 进阶十五：schema 与 search_path

schema 是数据库内的命名空间，用来组织表：

```sql
CREATE SCHEMA analytics;

-- 把表建在 analytics schema 下
CREATE TABLE analytics.daily_stats (...);

-- 设置搜索路径（查表时按顺序找）
SET search_path TO analytics, public;
-- 之后直接 SELECT * FROM daily_stats 就会找到 analytics.daily_stats
```

多租户隔离、环境隔离、权限管理都靠 schema。

---

# 进阶十六：序列（SEQUENCE）

```sql
CREATE SEQUENCE user_id_seq START WITH 1 INCREMENT BY 1;

SELECT nextval('user_id_seq');  -- 1
SELECT nextval('user_id_seq');  -- 2
SELECT currval('user_id_seq');  -- 当前会话最后拿到的值
```

注意：ROLLBACK 不会回收已经发出去的序列值，所以序列可能有间隙——这是正常的，序列只保证不重复，不保证连续。

---

# 进阶十七：触发器（TRIGGER）

在 INSERT/UPDATE/DELETE 时自动执行函数：

```sql
-- 自动更新 updated_at 字段
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_updated_at
BEFORE UPDATE ON students
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

`NEW` 代表新行，`OLD` 代表旧行。BEFORE/AFTER 决定时机，FOR EACH ROW 是行级。

---

# 进阶十八：行级安全（RLS）

让用户只能看到自己的数据：

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_orders ON orders
    FOR SELECT
    USING (user_id = current_user_id());  -- 只看到自己的订单
```

多租户 SaaS 应用的标配。

---

# 进阶十九：分区表（Table Partitioning）

把大表按规则切成小物理分区，查询只扫相关分区：

```sql
-- 父表
CREATE TABLE orders (
    order_id INT,
    created_at DATE,
    amount NUMERIC
) PARTITION BY RANGE (created_at);

-- 子分区
CREATE TABLE orders_2024 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

查询 `WHERE created_at = '2024-06-01'` 时，PostgreSQL 只扫 `orders_2024`，不扫其他分区——这叫分区裁剪（partition pruning）。

分区类型：
- `RANGE`：按范围（时间、数值）
- `LIST`：按枚举值列表
- `HASH`：按哈希值均匀分散

---

# 进阶二十：并行查询

PostgreSQL 可以把一个大查询拆给多个 CPU 核心并行跑：

```sql
-- 执行计划里会看到 Gather / Parallel Seq Scan 节点
EXPLAIN SELECT COUNT(*) FROM big_table;
```

大表统计、报表分析场景提速明显。

---

# 进阶二十一：逻辑复制

基于发布-订阅的复制，可以选择性地把某些表的变更同步到另一个库：

```sql
-- 发布端
CREATE PUBLICATION my_pub FOR TABLE orders, customers;

-- 订阅端
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=... dbname=...'
PUBLICATION my_pub;
```

适合跨版本升级、数据汇总、零停机迁移。

---

# 本文件覆盖的原文档知识点清单

- [x] RETURNING 子句（INSERT 后直接返回新 ID，避免 SELECT max(id)）
- [x] ON CONFLICT UPSERT（不存在则插入存在则更新，并发安全）
- [x] 类型转换符 ::（'123'::INTEGER，比 CAST 简洁）
- [x] ILIKE 不区分大小写匹配
- [x] JSONB 支持（箭头运算符 ->/->>，为 JSON 内部属性建索引）
- [x] （进阶补充）RETURNING 完整语法（INSERT/UPDATE/DELETE）、可修改 CTE
- [x] （进阶补充）ON CONFLICT 完整语法、EXCLUDED 伪表、DO NOTHING、与 MySQL/SQL Server/Oracle 对比
- [x] （进阶补充）:: 嵌套转换、数组转换
- [x] （进阶补充）ILIKE 性能（pg_trgm、LOWER 表达式索引）
- [x] （进阶补充）JSON vs JSONB 对比、完整运算符（->/->>/#>/</@/?/||/-）、GIN 索引、jsonb_set 更新
- [x] （进阶补充）DISTINCT ON 组内取第一条
- [x] （进阶补充）LATERAL 横向连接（TOP N per group）
- [x] （进阶补充）GENERATED AS IDENTITY vs SERIAL 对比
- [x] （进阶补充）数组类型、下标从 1 开始、@> 包含、unnest、GIN 索引
- [x] （进阶补充）范围类型 RANGE、&& 重叠、EXCLUDE 排他约束
- [x] （进阶补充）全文搜索 tsvector/tsquery、GIN 索引、zhparser 中文分词
- [x] （进阶补充）LISTEN/NOTIFY 发布订阅
- [x] （进阶补充）pg_stat_statements 慢查询统计
- [x] （进阶补充）扩展系统 CREATE EXTENSION、常用扩展清单
- [x] （进阶补充）schema 与 search_path
- [x] （进阶补充）SEQUENCE 序列、nextval/currval、gap 说明
- [x] （进阶补充）触发器 TRIGGER、NEW/OLD、BEFORE/AFTER
- [x] （进阶补充）行级安全 RLS、CREATE POLICY
- [x] （进阶补充）分区表 RANGE/LIST/HASH、分区裁剪
- [x] （进阶补充）并行查询 Gather 节点
- [x] （进阶补充）逻辑复制 PUBLICATION/SUBSCRIPTION
