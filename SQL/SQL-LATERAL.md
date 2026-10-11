这张图在演示 SQL 中 **`LATERAL` 关键字**的用法。我一步一步给你拆明白。

------

## 一、先看完整 SQL

```
SELECT * FROM
  (SELECT 1 AS x) AS t1,
  LATERAL (SELECT t1.x + 1 AS y) AS t2;
```

查询结果：

| t1.x | t2.y |
| ---- | ---- |
| 1    | 2    |

------

## 二、逐段拆解

### 第 1 段：构造左表 t1

```
(SELECT 1 AS x) AS t1
```

这是一个**派生表（derived table）**，本质就是一张只有一行一列的临时表：

| x    |
| ---- |
| 1    |

别名 `t1`，列名 `x`，值为 `1`。

### 第 2 段：构造右表 t2（核心）

```
LATERAL (SELECT t1.x + 1 AS y) AS t2
```

这里做了两件事：

1. 又是一个派生表，别名叫 `t2`，列名叫 `y`。
2. **关键**：子查询内部引用了 `t1.x`——也就是前一张表的列。

因为 `t1.x = 1`，所以 `t1.x + 1 = 2`，最终 t2 也是一行一列：

| y    |
| ---- |
| 2    |

### 第 3 段：SELECT * 输出

两张表做**笛卡尔积（逗号连接）**，各只有一行，拼起来就是：

| t1.x | t2.y |
| ---- | ---- |
| 1    | 2    |

------

## 三、LATERAL 到底解决了什么问题？

### 没有 LATERAL 会怎样？

如果写成：

```
SELECT * FROM
  (SELECT 1 AS x) AS t1,
  (SELECT t1.x + 1 AS y) AS t2;   -- 去掉 LATERAL
```

绝大多数数据库会**直接报错**，类似：

> ```
> Unknown column 't1.x' in 'field list'
> ```

原因是 SQL 标准规定：**FROM 子句中的每个派生表默认是独立求值的，不能引用同层 FROM 中其他表的列**。t2 这个子查询在解析时"看不到"t1。

### 加上 LATERAL 之后

`LATERAL` 的作用就是**打破这个隔离**，告诉数据库：

> "这个子查询不是独立的，它依赖前面表 t1 的当前行值，请按 t1 的每一行来计算我。"

换句话说，`LATERAL` 把一个普通子查询变成了**关联子查询（correlated subquery）**，并且可以放在 FROM 里当表用。

------

## 四、执行逻辑（伪代码视角）

可以把它理解成这样一个循环：

```
for each row r in t1:
    用 r.x 代入子查询 (SELECT r.x + 1 AS y)
    得到 t2 的一行或多行
    把 r 和 t2 的行拼接，输出
```

在这个例子里 t1 只有一行，所以只循环一次。

------

## 五、为什么这个语法很重要？实际用途

`LATERAL` 最常见的场景是：**左表每一行，都要生成一个动态的右表**。

### 典型场景 1：配合表值函数

比如 PostgreSQL 里的 `unnest`、`generate_series`，或按每行参数调用一个函数：

```
SELECT t.id, u.name
FROM users t,
LATERAL unnest(t.friend_ids) AS u(name);
```

把每行 users 里的数组 `friend_ids` 展开成多行。

### 典型场景 2：Top-N  per group

对每个部门，取工资最高的 3 个人：

```
SELECT d.dept_name, e.emp_name, e.salary
FROM departments d,
LATERAL (
    SELECT emp_name, salary
    FROM employees
    WHERE dept_id = d.dept_id
    ORDER BY salary DESC
    LIMIT 3
) e;
```

没有 LATERAL，这种"每组前 N"写起来要麻烦得多（需要窗口函数 + 子查询嵌套）。

### 典型场景 3：复用计算结果

把复杂表达式的结果"提出来"，让后续 SELECT 或 WHERE 直接用，避免重复写：

```
SELECT t.raw_price, t.discounted, t.profit
FROM orders o,
LATERAL (SELECT o.price * 0.8 AS discounted) t1,
LATERAL (SELECT t1.discounted - o.cost AS profit) t2;
```

------

## 六、和其他写法的对比

| 写法                                  | 能否引用前表列          | 位置                              |
| ------------------------------------- | ----------------------- | --------------------------------- |
| 普通派生表 `(SELECT ...) AS t`        | 不能                    | FROM 中                           |
| `LATERAL (SELECT ...) AS t`           | **能**                  | FROM 中                           |
| 关联子查询 `WHERE col = (SELECT ...)` | 能                      | WHERE / SELECT 中，但只能返回标量 |
| 窗口函数                              | 能（通过 PARTITION BY） | SELECT 中                         |

`LATERAL` 的独特价值是：**既能引用前表列，又能返回多行多列，还能放在 FROM 里参与连接**。

------

## 七、数据库支持情况

- **PostgreSQL**：很早就支持，用法最灵活
- **MySQL**：8.0.14+ 开始支持
- **Oracle**：12c+ 支持（用 `LATERAL` 或 `CROSS APPLY` / `OUTER APPLY`）
- **SQL Server**：用 `CROSS APPLY` / `OUTER APPLY`，等价于 `LATERAL`
- **SQLite**：3.38.0+ 支持

------

## 一句话总结

> `LATERAL` 让 FROM 子句里的子查询能够"看见"前面表的列，从而对前表的**每一行**动态生成一个结果集再连接。图中的例子就是最简化的演示：t1 给 x=1，LATERAL 子查询拿到 x 算出 y=2，最终拼出一行两列。