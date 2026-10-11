# 条件表达式：CASE-END

SQL 不是只有"查表"和"算数"，它还能在查询里做**判断**：满足什么条件就返回什么值。这个"根据条件分叉"的能力，就是 `CASE`。

---

## 一、最基本的形状

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
```

记忆结构：

```text
CASE
    WHEN 条件 THEN 结果
    WHEN 条件 THEN 结果
    ELSE 结果
END
```

它本质上就是很多编程语言里的 `if / else if / else`，只不过写在 SQL 里，而且必须用 `END` 收尾。`CASE ... END` 整体是一个**表达式**——它会算出一个值，可以放在任何能写值的地方（SELECT 列、WHERE、ORDER BY、UPDATE 的 SET 后面……）。

> 注意！！！`END` 这个词不能漏。漏了 `END`，数据库根本不知道你的 CASE 在哪里结束，直接报语法错误。

---

## 二、最重要的一个坑：只有 TRUE 才不进 ELSE

注意：***只有 WHEN 结果为 TRUE 才不进入 ELSE，而 UNKNOWN 会进入 ELSE***。

这句话要结合 SQL 的三值逻辑来理解。SQL 里的布尔判断结果不是"真/假"两种，而是**真（TRUE）/ 假（FALSE）/ 未知（UNKNOWN）**三种。任何和 `NULL` 做比较的表达式，结果都是 `UNKNOWN`。

看这个例子：

```sql
SELECT
    student_name,
    CASE
        WHEN age >= 18 THEN 'adult'
        ELSE 'minor'
    END AS age_group
FROM students;
```

假设有个学生 `age` 是 `NULL`：

- `age >= 18` 这个判断 = `NULL >= 18` = **UNKNOWN**，不是 TRUE。
- 既然不是 TRUE，就**不满足**这个 WHEN，于是落到 `ELSE`。
- 所以这个年龄为 NULL 的学生，会被归到 `'minor'`。

注意：当 age 为 NULL 时，`age >= 18` 的结果是 UNKNOWN，会进入 ELSE 分支，所以 NULL 年龄也归为 minor。这是**三值逻辑的自然结果**，不是巧合。

> 记忆法：WHEN 是一个"门"，只有 TRUE 这一种状态能通过；FALSE 和 UNKNOWN 都被挡在门外，一起掉进 ELSE。

如果你想把 NULL 单独区分出来，必须显式写：

```sql
SELECT
    student_name,
    CASE
        WHEN age IS NULL THEN 'unknown age'
        WHEN age >= 18 THEN 'adult'
        ELSE 'minor'
    END AS age_group
FROM students;
```

> 注意！！！判断 NULL **必须用 `IS NULL`，不能写 `WHEN age = NULL`**。因为 `age = NULL` 的结果永远是 UNKNOWN，这个 WHEN 永远不会通过。这是 CASE 里最隐蔽的坑，后面"CASE 与 NULL 的交互"一节还会细讲。

---

## 三、CASE 的两种形式

`CASE` 其实有两种写法。初学者最常用的是"搜索式"，但你必须知道还有一个"简单式"，否则看别人代码会懵。

### 1. 搜索式 CASE（Searched CASE）——最灵活

```sql
CASE
    WHEN 布尔条件 THEN 结果
    WHEN 布尔条件 THEN 结果
    ...
    ELSE 结果
END
```

`WHEN` 后面跟的是**任意布尔表达式**，可以是 `>=`、`BETWEEN`、`LIKE`、`IS NULL`、`AND/OR` 组合，想怎么写就怎么写。前面所有例子都是这种。

```sql
SELECT
    student_name,
    CASE
        WHEN score >= 90 THEN '优秀'
        WHEN score >= 80 THEN '良好'
        WHEN score >= 60 THEN '及格'
        WHEN score < 60  THEN '不及格'
        ELSE '缺考'
    END AS grade
FROM enrollments;
```

### 2. 简单式 CASE（Simple CASE）——等值比较的简写

```sql
CASE 表达式
    WHEN 值1 THEN 结果1
    WHEN 值2 THEN 结果2
    ...
    ELSE 结果
END
```

它只在"把一个表达式依次和几个值做**等值比较**"时才好用。

```sql
SELECT
    student_name,
    CASE major
        WHEN 'CS'  THEN '计算机学院'
        WHEN 'Math' THEN '数学学院'
        WHEN 'Physics' THEN '物理学院'
        ELSE '其他学院'
    END AS college
FROM students;
```

这等价于搜索式：

```sql
CASE
    WHEN major = 'CS'       THEN '计算机学院'
    WHEN major = 'Math'     THEN '数学学院'
    WHEN major = 'Physics'  THEN '物理学院'
    ELSE '其他学院'
END
```

### 3. 两种形式对比

| 对比项 | 搜索式 CASE | 简单式 CASE |
|---|---|---|
| 语法 | `CASE WHEN 条件 THEN ...` | `CASE 表达式 WHEN 值 THEN ...` |
| 能做的判断 | 任意布尔表达式（>, <, BETWEEN, LIKE, IS NULL...） | 只能等值比较（`表达式 = 值`） |
| 灵活性 | 高 | 低 |
| 可读性（等值场景） | 啰嗦 | 简洁 |
| NULL 处理 | `WHEN x IS NULL` 能命中 | `WHEN NULL` **命不中**（见下文坑） |

> 记忆法：简单式只是搜索式在"等值比较"这一种情况下的语法糖。只要你的判断里出现了大小于号、区间、模糊匹配，就老老实实用搜索式。

> 注意！！！简单式 CASE 里 `CASE major WHEN NULL THEN ...` 是**永远不会命中**的，因为它展开成 `major = NULL`，结果恒为 UNKNOWN。想匹配 NULL 只能用搜索式写 `WHEN major IS NULL`。

---

## 四、ELSE 子句与省略 ELSE 的默认行为

### 1. 写了 ELSE

前面的例子都写了 `ELSE`，表示"上面所有 WHEN 都不满足时，统一返回这个值"。

### 2. 省略 ELSE 会怎样？

如果你**不写 ELSE**，那么当所有 WHEN 都不满足时，CASE 返回的不是空字符串，不是 0，而是 **`NULL`**。

```sql
SELECT
    student_name,
    CASE
        WHEN score >= 60 THEN '及格'
    END AS grade
FROM enrollments;
```

- score = 80 → `'及格'`
- score = 40 → 所有 WHEN 都不满足 → 返回 **`NULL`**
- score = NULL → `score >= 60` 是 UNKNOWN → 也返回 **`NULL`**

> 注意！！！省略 ELSE ≠ "什么都不返回"，而是"返回 NULL"。这在做 `SUM(CASE WHEN ...)` 条件聚合时很关键：如果你漏了 ELSE，那些不满足条件的行贡献的是 NULL，而 SUM 会忽略 NULL，结果碰巧对；但如果是 COUNT 或者别的场景，这个 NULL 就会咬你一口。

### 3. 什么时候省略 ELSE 是安全的？

最典型的就是条件聚合：

```sql
SUM(CASE WHEN score >= 60 THEN 1 ELSE 0 END)
```

这里显式写了 `ELSE 0`，表示"不及格的行算 0 个"。如果你写成：

```sql
SUM(CASE WHEN score >= 60 THEN 1 END)   -- 不写 ELSE
```

不及格的行贡献 NULL，SUM 忽略 NULL，最后加起来的还是"及格的行数"——结果一样。但**强烈建议写 `ELSE 0`**，因为：
1. 一眼就能看懂意图；
2. 换成别的聚合函数（比如 AVG）行为就变了，写 ELSE 0 更安全。

---

## 五、CASE 在 SELECT 中的使用

这是最常见的用法，分两类。

### 1. 条件列：给每一行打标签

把原始值翻译成更友好的分类标签：

```sql
SELECT
    student_name,
    age,
    CASE
        WHEN age < 18 THEN '未成年'
        WHEN age < 25 THEN '青年'
        WHEN age < 35 THEN '中青年'
        ELSE '其他'
    END AS age_phase
FROM students;
```

### 2. 条件聚合：SUM(CASE WHEN ...)

以后做统计题时，这个写法会非常常见：

```sql
SUM(
    CASE WHEN score >= 60 THEN 1 ELSE 0 END
)
```

它的含义是：**遍历每一行，如果 score >= 60 就贡献 1，否则贡献 0，最后加起来**——也就是"及格的人数"。

完整例子：统计每门课的及格人数和不及格人数。

```sql
SELECT
    course_id,
    SUM(CASE WHEN score >= 60 THEN 1 ELSE 0 END) AS pass_cnt,
    SUM(CASE WHEN score <  60 THEN 1 ELSE 0 END) AS fail_cnt,
    COUNT(*) AS total_cnt
FROM enrollments
GROUP BY course_id;
```

一行 SQL 就把"及格几人、不及格几人"全算出来了，不用写两遍 WHERE。这就是条件聚合的威力。

---

## 六、CASE 在 WHERE 中的使用

CASE 可以出现在 WHERE 里做"条件过滤"。虽然大多数时候直接写布尔表达式更直观，但有些动态判断场景必须靠 CASE。

比如："如果是 CS 专业，只看年龄 >= 20 的；其他专业不限制年龄。"

```sql
SELECT student_name, major, age
FROM students
WHERE
    CASE
        WHEN major = 'CS' THEN age >= 20
        ELSE TRUE
    END;
```

解释：
- CS 专业的行，CASE 返回 `age >= 20` 这个布尔值，年龄不够 20 就被过滤掉；
- 其他专业的行，CASE 返回 `TRUE`，全部保留。

> 注意！！！WHERE 里的 CASE 最终要算出一个布尔值（TRUE/FALSE/UNKNOWN）。如果 CASE 返回的是 UNKNOWN，这一行就会被过滤掉（和 WHERE 的正常三值逻辑一致）。

---

## 七、CASE 在 ORDER BY 中的使用：自定义排序

默认 `ORDER BY` 只能按列的自然顺序（数字大小、字典序）排。但业务里经常要"按特定顺序排"，比如把"优秀"排最前、"良好"其次、"不及格"最后——而这些中文标签按字典序排不出这个效果。

这时用 CASE 造一个**排序键**：

```sql
SELECT
    student_name,
    CASE
        WHEN score >= 90 THEN '优秀'
        WHEN score >= 60 THEN '及格'
        ELSE '不及格'
    END AS grade
FROM enrollments
ORDER BY
    CASE grade
        WHEN '优秀' THEN 1
        WHEN '及格' THEN 2
        WHEN '不及格' THEN 3
    END;
```

这里 ORDER BY 里的 CASE 给每个等级算出一个数字（1/2/3），然后按数字升序，就实现了"优秀 → 及格 → 不及格"的自定义顺序。

再举一个原文档要求的例子：**按特定专业顺序排序**（比如想让 CS 排最前，Math 其次，Physics 再其次）：

```sql
SELECT student_name, major
FROM students
ORDER BY
    CASE major
        WHEN 'CS'     THEN 1
        WHEN 'Math'   THEN 2
        WHEN 'Physics' THEN 3
        ELSE 4
    END;
```

> 记忆法：CASE 在 ORDER BY 里的套路就是"给每个类别编个号，然后按号排"。你想让谁在前，就给谁小编号。

---

## 八、CASE 在 GROUP BY 中的使用：按条件分组

GROUP BY 后面通常跟列名，但也可以跟一个表达式——包括 CASE。这样就能"按条件把行折成组"。

比如不按具体分数分组，而是按分数段分组：

```sql
SELECT
    CASE
        WHEN score >= 90 THEN '优秀(90+)'
        WHEN score >= 60 THEN '及格(60-89)'
        ELSE '不及格(<60)'
    END AS score_band,
    COUNT(*) AS cnt
FROM enrollments
GROUP BY
    CASE
        WHEN score >= 90 THEN '优秀(90+)'
        WHEN score >= 60 THEN '及格(60-89)'
        ELSE '不及格(<60)'
    END;
```

结果：

```text
优秀(90+)      12
及格(60-89)   30
不及格(<60)    8
```

> 注意！！！GROUP BY 里的 CASE 表达式必须和 SELECT 里的**一模一样**（很多数据库允许直接用 SELECT 里的别名，但标准写法是把表达式重写一遍，或者用子查询/CTE 包一层）。为了可移植，推荐用子查询或 CTE：

```sql
WITH banded AS (
    SELECT
        CASE
            WHEN score >= 90 THEN '优秀(90+)'
            WHEN score >= 60 THEN '及格(60-89)'
            ELSE '不及格(<60)'
        END AS score_band
    FROM enrollments
)
SELECT score_band, COUNT(*)
FROM banded
GROUP BY score_band;
```

---

## 九、CASE 在 UPDATE SET 中的使用：条件更新

UPDATE 的 SET 后面也能用 CASE，实现"不同行更新成不同值"，一条 SQL 搞定本该写很多条 UPDATE 的事。

比如：给所有学生加平时分，但加分规则按成绩段区分：90 分以上加 2 分（封顶 100），60-89 加 5 分，60 以下加 10 分。

```sql
UPDATE enrollments
SET score =
    CASE
        WHEN score >= 90 THEN LEAST(score + 2, 100)
        WHEN score >= 60  THEN score + 5
        ELSE score + 10
    END;
```

> 注意！！！UPDATE 里的 CASE **一定要配 WHERE 或小心 ELSE**。如果你写漏了 ELSE，那些不满足任何 WHEN 的行会被 SET 成 NULL，把数据冲掉。这是 UPDATE 里用 CASE 最危险的地方。

---

## 十、NULLIF 与 CASE 的关系

`NULLIF(a, b)` 是一个内置函数，它的行为完全可以用 CASE 展开：

```sql
NULLIF(a, b)
-- 等价于
CASE WHEN a = b THEN NULL ELSE a END
```

它的含义："如果 a 等于 b，就返回 NULL，否则返回 a"。

最常见的用途是**防止除零错误**。比如算 `score / max_score`，万一某行 `max_score = 0`，就会报错：

```sql
SELECT score / NULLIF(max_score, 0) AS ratio
FROM enrollments;
```

展开就是：

```sql
SELECT score / CASE WHEN max_score = 0 THEN NULL ELSE max_score END AS ratio
FROM enrollments;
```

这样当 max_score = 0 时，分母变成 NULL，整个除法结果是 NULL，而不是抛错。

---

## 十一、COALESCE 本质是 CASE 的语法糖

`COALESCE(a, b, c, ...)` 返回参数列表里**第一个非 NULL 的值**。它本质上也是 CASE：

```sql
COALESCE(a, b, c)
-- 等价于
CASE
    WHEN a IS NOT NULL THEN a
    WHEN b IS NOT NULL THEN b
    ELSE c
END
```

如果要把 NULL 的 major 显示为 "Unknown"，用 COALESCE 一行搞定：

```sql
SELECT student_name, COALESCE(major, 'Unknown') AS major
FROM students;
```

展开就是：

```sql
SELECT
    student_name,
    CASE WHEN major IS NOT NULL THEN major ELSE 'Unknown' END AS major
FROM students;
```

> 记忆法：SQL 里很多函数其实都是 CASE 的语法糖。`NULLIF`、`COALESCE`、`GREATEST`/`LEAST`（部分）都能展开成 CASE。理解了 CASE，这些函数就不再是黑盒。

---

## 十二、CASE 中的短路求值（Short-circuit Evaluation）

CASE 的 WHEN 是**按顺序从上到下求值**的，一旦某个 WHEN 算出 TRUE，就立刻返回对应的 THEN 值，**后面的 WHEN 根本不会再被求值**。这叫短路求值。

```sql
CASE
    WHEN score >= 60 THEN '及格'
    WHEN score >= 90 THEN '优秀'   -- 这行永远到不了！
    ELSE '不及格'
END
```

上面这个例子里，score = 95 的行在第一个 WHEN（95 >= 60 = TRUE）就返回"及格"了，第二个 WHEN（>= 90）永远不会被检查到。所以**WHEN 的顺序写反了，结果就错了**。

> 注意！！！写多分支 CASE 时，**条件要从最严格（最具体）写到最宽松（最一般）**。比如先判 `>= 90`，再判 `>= 60`，最后 ELSE。把宽松条件写前面，会把后面的分支全部"堵死"。

短路求值对有副作用的表达式也很重要：

```sql
CASE
    WHEN x IS NOT NULL THEN 100 / x   -- 只有 x 非 NULL 才会算这个除法
    ELSE 0
END
```

因为先判了 `x IS NOT NULL`（TRUE），才会执行 `100 / x`，所以不会除零。如果顺序反过来，就有风险。

> 注意！！！不是所有数据库都严格保证短路求值的执行顺序（SQL 标准对优化器的重排留了余地），但**主流数据库在实践中都是按书写顺序短路的**。你应当始终按"先保护、后计算"的顺序写 WHEN，不要依赖某个数据库的特殊行为。

---

## 十三、CASE 与聚合函数组合的高级用法

### 1. 条件聚合：COUNT / SUM

前面已经见过，再补充 COUNT 形式：

```sql
SELECT
    COUNT(CASE WHEN score >= 60 THEN 1 END) AS pass_cnt
FROM enrollments;
```

这里 `CASE WHEN score >= 60 THEN 1 END` 不写 ELSE，及格的行返回 1，不及格的行返回 NULL；`COUNT(列)` 只数非 NULL 的行，所以数出来就是及格人数。

> 对比：`SUM(CASE WHEN ... THEN 1 ELSE 0 END)` 和 `COUNT(CASE WHEN ... THEN 1 END)` 结果一样，但前者靠加 0，后者靠 COUNT 忽略 NULL。理解原理后选哪个都行。

### 2. 行转列（Pivot）：CASE + 聚合

这是 CASE 最强大的用法之一。假设有这样的成绩明细：

```text
student_id   course_id   score
   1            1         80
   1            2         90
   2            1         70
   2            3         85
```

每行是"一个学生一门课"。但你想做成**一行一个学生、每门课一列**的宽表：

```text
student_id   course1   course2   course3
   1           80        90       NULL
   2           70       NULL      85
```

用 CASE + MAX（或 SUM）实现：

```sql
SELECT
    student_id,
    MAX(CASE WHEN course_id = 1 THEN score END) AS course1,
    MAX(CASE WHEN course_id = 2 THEN score END) AS course2,
    MAX(CASE WHEN course_id = 3 THEN score END) AS course3
FROM enrollments
GROUP BY student_id;
```

原理：
- `GROUP BY student_id` 把同一个学生的所有行折成一组；
- 对这一组，`MAX(CASE WHEN course_id = 1 THEN score END)` 只在 course_id=1 的行取出 score，其他行这一列是 NULL；
- MAX 忽略 NULL，所以这一列就是该学生 course1 的成绩；
- 三列分别用不同的 CASE，就把"行"翻转成了"列"。

> 记忆法：行转列 = `GROUP BY 你想当行的东西` + 每一列写一个 `聚合(CASE WHEN 原值=某值 THEN 取值 END)`。

### 3. 多条件交叉统计

统计"每个专业里，及格和不及格的人数"——这就是一个二维交叉表：

```sql
SELECT
    s.major,
    SUM(CASE WHEN e.score >= 60 THEN 1 ELSE 0 END) AS pass_cnt,
    SUM(CASE WHEN e.score <  60 THEN 1 ELSE 0 END) AS fail_cnt
FROM students s
JOIN enrollments e ON e.student_id = s.student_id
GROUP BY s.major;
```

---

## 十四、FILTER 子句（PostgreSQL 特有）

PostgreSQL 提供了一个更优雅的语法 `FILTER (WHERE ...)`，专门给聚合函数加条件，效果等价于 `CASE` 条件聚合：

```sql
SELECT
    course_id,
    COUNT(*) FILTER (WHERE score >= 60) AS pass_cnt,
    COUNT(*) FILTER (WHERE score <  60) AS fail_cnt
FROM enrollments
GROUP BY course_id;
```

它等价于：

```sql
SELECT
    course_id,
    COUNT(CASE WHEN score >= 60 THEN 1 END) AS pass_cnt,
    COUNT(CASE WHEN score <  60 THEN 1 END) AS fail_cnt
FROM enrollments
GROUP BY course_id;
```

### FILTER vs CASE 条件聚合对比

| 对比项 | FILTER (WHERE ...) | SUM/COUNT(CASE WHEN ...) |
|---|---|---|
| 可读性 | 高，聚合和条件分离 | 略啰嗦，CASE 嵌在聚合里 |
| 可移植性 | 仅 PostgreSQL | 所有数据库都支持 |
| 性能 | 等价，优化器处理方式相同 | 等价 |
| 适用 | PostgreSQL 项目 | 通用 |

> 注意！！！`FILTER` 不是标准 SQL，MySQL、SQL Server、Oracle 都不支持。写跨数据库的 SQL 时还是得用 `CASE WHEN`。

---

## 十五、CASE 的类型一致性要求

CASE 所有 `THEN` 和 `ELSE` 返回的值，**类型必须兼容**，数据库会尝试把它们统一成一个类型。

```sql
CASE
    WHEN score >= 60 THEN '及格'   -- 文本
    ELSE 0                         -- 数字
END
```

这种"有的分支返回字符串、有的分支返回数字"的写法，大多数数据库会报错（或做奇怪的隐式转换）。你必须让所有分支返回同一类东西：

```sql
CASE
    WHEN score >= 60 THEN '及格'
    ELSE '不及格'     -- 都是文本
END
```

> 注意！！！如果一个分支返回整数 1，另一个分支返回小数 1.5，数据库会统一成小数（numeric），这是允许的隐式转换。但"数字 vs 字符串"通常不行。写 CASE 时心里要清楚：这个 CASE 最终算出的是个什么类型的值。

---

## 十六、嵌套 CASE 与可读性

CASE 里还能再套 CASE：

```sql
CASE
    WHEN major = 'CS' THEN
        CASE
            WHEN age >= 20 THEN 'CS青年'
            ELSE 'CS少年'
        END
    ELSE '其他'
END
```

但嵌套越深越难读。**能用 WHEN 条件组合（AND/OR）写清楚的，就别嵌套**：

```sql
CASE
    WHEN major = 'CS' AND age >= 20 THEN 'CS青年'
    WHEN major = 'CS'               THEN 'CS少年'
    ELSE '其他'
END
```

> 记忆法：嵌套 CASE 只在"外层判断完才知道内层该判什么"时才必要。绝大多数场景，把条件用 AND 拼平成一层，可读性更好。

---

## 十七、CASE 与 NULL 的交互（再强调）

最后把 NULL 在 CASE 里的行为集中说一遍，因为这是最高频的踩坑点。

1. **`WHEN x = NULL` 永远不成立**——必须写 `WHEN x IS NULL`。
2. **`WHEN x >= 18` 当 x 是 NULL 时结果是 UNKNOWN**——不通过这个 WHEN，掉进 ELSE。
3. **省略 ELSE 时，所有 WHEN 都不命中返回 NULL**——不是空串、不是 0。
4. **简单式 CASE 里 `CASE x WHEN NULL THEN ...` 命不中**——它展开成 `x = NULL`，恒 UNKNOWN。

一个综合例子，正确处理 NULL：

```sql
SELECT
    student_name,
    CASE
        WHEN age IS NULL THEN '年龄未知'
        WHEN age >= 18   THEN '成年'
        ELSE '未成年'
    END AS age_group
FROM students;
```

> 注意！！！把 `WHEN age IS NULL` 放在**最前面**。因为一旦先写 `WHEN age >= 18`，age 为 NULL 的行会因为 UNKNOWN 掉进后面的分支，你就没机会单独识别它了——顺序在 CASE 里就是一切。

---

## 本文件覆盖的原文档知识点清单

- [x] CASE-END 基本语法和示例（age >= 18 → adult，否则 minor）
- [x] CASE 的逻辑结构记忆：CASE WHEN 条件 THEN 结果 ... ELSE 结果 END
- [x] 重要注意：只有 WHEN 结果为 TRUE 才不进入 ELSE，UNKNOWN 会进入 ELSE
- [x] 条件聚合的常见用法：SUM(CASE WHEN score >= 60 THEN 1 ELSE 0 END)
- [x] 搜索式 CASE 与简单式 CASE 的详细对比与等价转换
- [x] ELSE 子句与省略 ELSE 的默认行为（返回 NULL）
- [x] CASE 在 SELECT 中的使用（条件列、条件聚合）
- [x] CASE 在 WHERE 中的使用（条件过滤）
- [x] CASE 在 ORDER BY 中的使用（自定义排序，含按专业顺序排序示例）
- [x] CASE 在 GROUP BY 中的使用（按条件分组）
- [x] CASE 在 UPDATE SET 中的使用（条件更新不同行）
- [x] NULLIF 与 CASE 的关系（NULLIF(a,b) = CASE WHEN a=b THEN NULL ELSE a END）
- [x] COALESCE 本质是 CASE 的语法糖
- [x] CASE 中的短路求值原理
- [x] CASE 与聚合函数组合：条件聚合、行转列（pivot）、多条件交叉统计
- [x] FILTER 子句（PostgreSQL 特有）与 CASE 条件聚合的对比
- [x] CASE 的类型一致性要求与隐式转换
- [x] 嵌套 CASE 的使用和可读性
- [x] CASE 与 NULL 的交互（WHEN NULL 的陷阱，必须用 IS NULL）
