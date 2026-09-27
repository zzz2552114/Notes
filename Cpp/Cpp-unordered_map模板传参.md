Q: 

umap实际上<>尖括号里能传几个东西？首先是键类型，然后值类型，然后哈希，那我想提供operator==，传什么呢？已经在hash里被包含了吗？



A:

这是一个非常好的问题，说明你在主动类比 `set` 的设计。答案是：**`operator==` 不通过哈希传递，而是通过第 4 个模板参数 `KeyEqual` 传递，默认值 `std::equal_to<Key>` 会自动调用你类型的 `operator==`。** 我逐层讲清楚。

## 一、`unordered_map` 尖括号里到底能传几个？

完整声明是 **5 个**模板参数，不是 4 个：

```
template<
    class Key,                                    // 1. 键类型
    class T,                                      // 2. 值类型
    class Hash = std::hash<Key>,                  // 3. 哈希函数对象类型
    class KeyEqual = std::equal_to<Key>,          // 4. 相等比较函数对象类型
    class Allocator = std::allocator<std::pair<const Key, T>>  // 5. 分配器
> class unordered_map;
```

后三个都有默认值，所以平时你只写两个：`std::unordered_map<std::string, int>`。

## 二、`operator==` 在哪里？——第 4 个参数 `KeyEqual`

和 `set` 的设计完全对称，我放在一起对比：

| 容器                 | 比较需求 | 模板参数           | 默认值               | 默认值内部做什么                   |
| -------------------- | -------- | ------------------ | -------------------- | ---------------------------------- |
| `std::set`           | 小于比较 | 第 2 个 `Compare`  | `std::less<Key>`     | `return a < b;` → 调 `operator<`   |
| `std::unordered_map` | 哈希     | 第 3 个 `Hash`     | `std::hash<Key>`     | `return hash(key);` → 调哈希函数   |
| `std::unordered_map` | 相等比较 | 第 4 个 `KeyEqual` | `std::equal_to<Key>` | `return a == b;` → 调 `operator==` |

`std::equal_to<Key>` 的实现和 `std::less<Key>` 一样简单：

```
template<class T>
struct equal_to {
    bool operator()(const T& a, const T& b) const {
        return a == b;   // ← 就这一句，调用 T 的 operator==
    }
};
```

**所以你在自定义类型里重载了 `operator==` 之后，第 4 个参数根本不用写，默认的 `std::equal_to<Key>` 会自动调到它。**

## 三、完整代码示例

### 最常见的写法：自定义哈希 + 类内 `operator==`

```
struct Point {
    int x, y;

    // 重载 operator==，供 std::equal_to<Point> 调用
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

// 自定义哈希（作为第3个参数传入）
struct PointHash {
    std::size_t operator()(const Point& p) const {
        return std::hash<int>()(p.x) ^ (std::hash<int>()(p.y) << 1);
    }
};

// 只传3个参数：Key, T, Hash
// 第4个参数 KeyEqual 用默认的 std::equal_to<Point>
// 它内部会调用 Point::operator==
std::unordered_map<Point, int, PointHash> m;

m[{1, 2}] = 10;   // ✅ 正常工作
```

这里 `operator==` 没有出现在尖括号里，但它**通过默认的第 4 个参数被间接调用**了。这和你之前 `std::set<MyString>` 只传一个参数、`operator<` 通过默认的 `std::less` 被调用，是**完全一样的机制**。

### 也可以显式传第 4 个参数（自定义相等比较器）

如果你不想在类里写 `operator==`，或者需要多套相等规则，可以像自定义哈希一样写一个相等比较器结构体：

```
struct PointEqual {
    bool operator()(const Point& a, const Point& b) const {
        return a.x == b.x && a.y == b.y;
    }
};

// 4个参数全写
std::unordered_map<Point, int, PointHash, PointEqual> m;
```

这和 `set` 里写 `ByLen` 比较器是一个思路——把比较逻辑外置成函数对象。

### 最简洁的写法：特化 `std::hash` + 类内 `operator==`

```
struct Point {
    int x, y;
    bool operator==(const Point&) const = default;  // C++20 可自动生成
};

// 特化 std::hash<Point>
namespace std {
    template<> struct hash<Point> {
        size_t operator()(const Point& p) const {
            return hash<int>()(p.x) ^ (hash<int>()(p.y) << 1);
        }
    };
}

// 两个参数都不用额外传！
// 第3个用默认 std::hash<Point>（你特化过的）
// 第4个用默认 std::equal_to<Point>（调 operator==）
std::unordered_map<Point, int> m;
```

这就是你提到的"为类型特化 `std::hash`"方案——特化之后连第 3 个参数都省了。

## 四、为什么 `unordered_map` 需要两个东西（哈希 + 相等），而 `set` 只需要一个（小于）？

这是由底层数据结构决定的：

- **`set` 底层是红黑树**，只需要"小于"就能确定元素的位置和唯一性。两个元素 `a` 和 `b`，只要 `!(a < b) && !(b < a)`，就认为等价。
- **`unordered_map` 底层是哈希表**，分两步定位：
  1. 用**哈希函数**算桶号：`bucket = hash(key) % bucket_count`
  2. 同一个桶里可能有多个元素（哈希碰撞），用**相等比较**逐个比对，找到真正匹配的那个

所以哈希和相等是**两个独立的职责**，分别由第 3、第 4 个模板参数负责，互不包含。哈希函数不负责判断相等，相等比较器也不负责算哈希。

## 五、一个必须注意的约束：哈希和相等必须一致

这是 `unordered_map` 最容易踩的坑：

> **如果 `a == b`，那么必须有 `hash(a) == hash(b)`。**

反过来不要求（哈希碰撞是正常的），但正向必须成立。否则两个相等的元素被算到不同的桶里，`find` 永远找不到。

比如你的 `Point`：

- `operator==` 比较 `x` 和 `y`
- 哈希函数也必须基于 `x` 和 `y` 来算

如果你哈希只算了 `x`，但 `operator==` 比较了 `x` 和 `y`，那 `{1,2}` 和 `{1,3}` 哈希相同但不相等——这没问题（碰撞）；但如果你哈希算了 `x+y`，而 `operator==` 只比较 `x`，那 `{1,2}` 和 `{1,3}` 被认为相等但哈希不同——**这就出 bug 了**。

## 六、总结对照表

```
std::set<Key, Compare = std::less<Key>>
         ↑       ↑
       键类型   小于比较器，默认调 operator<

std::unordered_map<Key, T, Hash = std::hash<Key>, KeyEqual = std::equal_to<Key>>
                   ↑    ↑      ↑                    ↑
                 键类型 值类型  哈希函数，默认调 std::hash   相等比较器，默认调 operator==
```

**`operator==` 没有被"包含在 hash 里"，它是独立的第 4 个模板参数 `KeyEqual`，默认值 `std::equal_to<Key>` 会自动调用你类型的 `operator==`。所以只要你在类里重载了 `operator==`，尖括号里只需要额外传哈希（第 3 个参数），第 4 个参数用默认值即可。**