# 重载运算符



## 1. 最基本写法

运算符重载本质是函数，函数名是：

```cpp
operator 符号
```

比如：

```cpp
operator+
operator==
operator<<
operator[]
```

语法：

```cpp
返回类型 operator符号(参数列表) {
    // 实现
}
```

例如给二维点 `Point` 重载 `+`：

```cpp
struct Point {
    int x, y;

    Point operator+(const Point& other) const {
        return {x + other.x, y + other.y};
    }
};
```

使用：

```cpp
Point a{1, 2};
Point b{3, 4};
Point c = a + b;  // 等价于 a.operator+(b)
```

编译器看到 `a + b`，如果 `a` 是 `Point`，就会去找：

```cpp
a.operator+(b)
```

或者全局的：

```cpp
operator+(a, b)
```

运算符重载不是魔法，只是函数调用的语法糖。

---

## 2. 成员函数 vs 非成员函数

运算符重载有两种常见位置：

### 2.1 写成成员函数

写在 `struct` 里面：

```cpp
struct Point {
    int x, y;

    Point operator+(const Point& other) const {
        return {x + other.x, y + other.y};
    }
};
```

特点：

- 左操作数就是当前对象 `*this`。
- 二元运算符成员函数只写一个参数。
- `a + b` 变成 `a.operator+(b)`。

### 2.2 写成非成员函数

写在 `struct` 外面：

```cpp
struct Point {
    int x, y;
};

Point operator+(const Point& a, const Point& b) {
    return {a.x + b.x, a.y + b.y};
}
```

特点：

- 左右操作数都作为参数。
- `a + b` 变成 `operator+(a, b)`。
- 如果成员是 `private`，需要在 `struct` 里声明为 `friend`。

### 2.3 怎么选？

| 运算符            | 推荐写法     | 原因                     |
| ----------------- | ------------ | ------------------------ |
| `=`               | 成员         | 必须是成员               |
| `[]`              | 成员         | 必须是成员               |
| `()`              | 成员         | 必须是成员               |
| `->`              | 成员         | 必须是成员               |
| 类型转换          | 成员         | 必须是成员               |
| `+=` `-=` `*=`    | 成员         | 通常修改自身             |
| `++` `--`         | 成员         | 通常修改自身             |
| `+` `-` `*` `/`   | 非成员       | 对称，允许两边隐式转换   |
| `==` `!=` `<` `>` | 成员或非成员 | 常可成员，C++20 可默认   |
| `<<` `>>`         | 非成员       | 左操作数是流，不是你的类 |

一句话：

> 会修改自己的，常用成员；对称的，常用非成员；流输入输出，必须非成员。

---

## 3. 一个完整例子：Vec2

下面这个例子可以直接编译运行：

```cpp
#include <iostream>
#include <cstddef>

struct Vec2 {
    double x{}, y{};

    Vec2() = default;
    Vec2(double x, double y) : x(x), y(y) {}

    // 复合赋值：成员函数，返回引用，支持 a += b += c
    Vec2& operator+=(const Vec2& rhs) {
        x += rhs.x;
        y += rhs.y;
        return *this;
    }

    Vec2& operator-=(const Vec2& rhs) {
        x -= rhs.x;
        y -= rhs.y;
        return *this;
    }

    // 一元负号：不修改自己，加 const
    Vec2 operator-() const {
        return {-x, -y};
    }

    // 比较：不修改自己，加 const
    bool operator==(const Vec2& rhs) const {
        return x == rhs.x && y == rhs.y;
    }

    bool operator!=(const Vec2& rhs) const {
        return !(*this == rhs);
    }

    // 下标：非 const 版本返回引用，可以修改
    double& operator[](std::size_t i) {
        return i == 0 ? x : y;
    }

    // 下标：const 版本返回 const 引用，只能读
    const double& operator[](std::size_t i) const {
        return i == 0 ? x : y;
    }

    // 前置 ++：返回引用
    Vec2& operator++() {
        ++x;
        ++y;
        return *this;
    }

    // 后置 ++：返回值，int 参数只是占位
    Vec2 operator++(int) {
        Vec2 old = *this;
        ++(*this);
        return old;
    }
};

// 非成员 operator+：对称，推荐基于 += 实现
Vec2 operator+(Vec2 lhs, const Vec2& rhs) {
    lhs += rhs;
    return lhs;
}

// 流输出：必须非成员，返回 ostream& 支持链式
std::ostream& operator<<(std::ostream& os, const Vec2& v) {
    return os << '(' << v.x << ", " << v.y << ')';
}

int main() {
    Vec2 a{1, 2};
    Vec2 b{3, 4};

    Vec2 c = a + b;
    c += {1, 1};

    std::cout << c << '\n';       // (5, 7)
    std::cout << (a == b) << '\n'; // 0
    std::cout << c[0] << ' ' << c[1] << '\n'; // 5 7

    ++c;
    std::cout << c << '\n';       // (6, 8)
    std::cout << c++ << '\n';     // (6, 8)
    std::cout << c << '\n';       // (7, 9)
}
```

逐点解释：

- `operator+=` 是成员，因为要修改当前对象。
- 返回 `Vec2&`，因为要支持 `a += b += c`。
- `operator==` 加了 `const`，因为比较不应该修改对象。
- `operator[]` 写了两个版本：非 const 能改，const 只能读。
- 前置 `++` 返回引用，后置 `++` 返回值。
- `operator+` 是非成员，左右对称。
- `operator<<` 是非成员，因为 `std::cout << v` 中左操作数是 `std::cout`，不是 `Vec2`。

---

## 4. 流运算符：让 cout 认识你的类型

流输出通常这样写：

```cpp
std::ostream& operator<<(std::ostream& os, const Vec2& v) {
    return os << '(' << v.x << ", " << v.y << ')';
}
```

为什么返回 `std::ostream&`？

为了支持链式：

```cpp
std::cout << a << b << c;
```

等价于：

```cpp
((std::cout << a) << b) << c;
```

如果返回 `void`，就不能继续 `<< b`。

流输入：

```cpp
std::istream& operator>>(std::istream& is, Vec2& v) {
    return is >> v.x >> v.y;
}
```

注意第二个参数不是 `const`，因为要修改 `v`。



### 如果成员是 `private`，需要在 `struct` 里声明友元：

```cpp
struct Money {
private:
    long cents;

public:
    Money(long c) : cents(c) {}

    friend Money operator+(const Money& a, const Money& b);
    friend std::ostream& operator<<(std::ostream& os, const Money& m);
};

Money operator+(const Money& a, const Money& b) {
    return Money(a.cents + b.cents);
}

std::ostream& operator<<(std::ostream& os, const Money& m) {
    return os << m.cents / 100.0;
}
```

`friend` 的意思是：这个外部函数是我朋友，可以访问我的私有成员。

---

## 5. C++20：默认比较，爽到飞起

C++20 之前，你要写：

```cpp
bool operator==(const Point&) const;
bool operator!=(const Point&) const;
bool operator<(const Point&) const;
bool operator<=(const Point&) const;
bool operator>(const Point&) const;
bool operator>=(const Point&) const;
```

C++20 可以用三路比较 `<=>`：

```cpp
#include <compare>

struct Point {
    int x, y;

    auto operator<=>(const Point&) const = default;
};
```

这一行会按成员声明顺序自动生成：

```cpp
== != < <= > >=
```

例如：

```cpp
Point a{1, 2};
Point b{1, 3};

std::cout << (a < b) << '\n'; // 1
```

默认比较规则：先比 `x`，如果相等再比 `y`，类似字典序。

---

## 6. 特殊运算符示例

### 6.1 函数调用 `operator()`

让对象像函数一样使用：

```cpp
struct Adder {
    int base;

    int operator()(int x) const {
        return base + x;
    }
};

Adder add{10};
std::cout << add(5); // 15
```

这叫函数对象，也叫仿函数。

### 6.2 类型转换 `operator bool`

```cpp
struct Flag {
    bool ok;

    explicit operator bool() const {
        return ok;
    }
};
```

使用：

```cpp
Flag f{true};

if (f) {
    std::cout << "yes\n";
}
```

建议加 `explicit`，防止它被乱转换成 `int` 等类型。

### 6.3 赋值 `operator=`

```cpp
struct MyString {
    char* data;
    std::size_t size;

    MyString& operator=(const MyString& other) {
        if (this == &other) {
            return *this;
        }

        // 释放旧资源，复制 other 的资源
        // ...

        return *this;
    }
};
```

要点：

- 必须返回 `*this`。
- 注意自赋值：`a = a`。
- 管理资源时，默认拷贝赋值可能不够用。

### 6.4 下标 `operator[]`

```cpp
struct IntArray {
    int data[10];

    int& operator[](int i) {
        return data[i];
    }

    const int& operator[](int i) const {
    // 这里第一个 const 标识返回值类型
    // 第二个 const 标识 this* IntArray 的内容在函数内部是不可以修改的
        return data[i];
    }
};
```

非 const 版本可以写：

```cpp
arr[0] = 100;
```

const 版本只能读：

```cpp
const IntArray& ca = arr;
std::cout << ca[0];
```

---

## 7. 哪些运算符不能重载？

不能重载：

```cpp
::      // 作用域解析
.       // 成员访问
.*      // 成员指针访问
?:      // 三目条件
sizeof
typeid
alignof
noexcept
static_cast
dynamic_cast
reinterpret_cast
const_cast
```

也不能发明新运算符，比如 `**`、`<>` 之类。

不能改变：

- 运算符优先级
- 结合性
- 操作数个数

例如 `a + b * c` 永远先算 `b * c`。

---

## 8. 重要坑点

1. **该加 `const` 就加**

   ```cpp
   bool operator==(const Vec2& rhs) const;
   ```

   最后的 `const` 表示不修改当前对象。没有它，`const Vec2` 不能调用。

2. **参数常用 `const T&`**

   避免拷贝：

   ```cpp
   Vec2 operator+(const Vec2& rhs) const;
   ```

3. **返回值要合理**

   - 新建对象：按值返回，如 `operator+`
   - 修改自己：返回引用，如 `operator+=`、`operator++`
   - 比较：返回 `bool`
   - 流：返回 `std::ostream&`

4. **不要返回局部变量的引用**

   ```cpp
   Vec2& bad() {
       Vec2 v;
       return v; // 错！
   }
   ```

5. **前置 ++ 和后置 ++ 不同**

   ```cpp
   Vec2& operator++();      // ++v
   Vec2 operator++(int);    // v++
   ```

   后置的 `int` 只是占位，不写名字。

6. **不要随便重载 `&&`、`||`、`,`**

   重载 `&&` 和 `||` 会失去短路求值：

   ```cpp
   if (a && b) // 内置逻辑会短路
   ```

   重载后不一定短路，容易出坑。

7. **不要滥用运算符重载**

   运算符语义要直观。`+` 就应该像加法，`==` 就应该像比较。不要用 `+` 做删除，用 `<<` 做加密，那会让人抓狂。

---

## 9. 一句话总结口诀

> 成员管自身，非成员管对称；  
> 修改返回引用，新建按值回；  
> 输入 `const` 引用，输出流引用；  
> `const` 不修改，`explicit` 防乱转；  
> 流操作用友元，C++20 默认比较爽。

最常用的套路就是：

```cpp
struct T {
    // 修改自身的用成员
    T& operator+=(const T& rhs) { /*...*/ return *this; }

    // 比较用成员，加 const
    bool operator==(const T& rhs) const { /*...*/ }

    // 下标用成员，成对写 const / 非 const
    int& operator[](int i) { /*...*/ }
    const int& operator[](int i) const { /*...*/ }
};

// 对称二元用非成员
T operator+(T lhs, const T& rhs) {
    lhs += rhs;
    return lhs;
}

// 流输出用非成员
std::ostream& operator<<(std::ostream& os, const T& v) {
    return os << /*...*/;
}
```

这样，你的 `struct` 就能像内置类型一样自然地被加减、比较、打印了。