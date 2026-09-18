下面把 C++ 的 lambda 从“是什么”到“怎么用、什么时候用、有什么坑”尽量细致地讲一遍。你可以把它当成一篇 C++ lambda 的长篇说明书。

---

## 1. 一句话本质：lambda 不是函数，而是“闭包对象”

C++ 的 lambda 表达式：

```cpp
auto f = [](int x) { return x + 1; };
```

编译器看到它，大致会生成一个**唯一的匿名类**，类似：

```cpp
class __Lambda_xxx {
public:
    int operator()(int x) const {
        return x + 1;
    }
};
```

然后：

```cpp
auto f = __Lambda_xxx{};
```

所以你调用 `f(10)`，本质是调用这个匿名对象的 `operator()`。

因此：

- lambda 的类型是**编译器生成的唯一闭包类型**，不是普通函数类型。
- 无捕获 lambda 可以转换成函数指针。
- 有捕获 lambda 不能直接转换成函数指针，因为它要保存状态。
- lambda 通常能被内联，性能很好。
- 一旦放进 `std::function`，就可能发生类型擦除和间接调用，有额外开销。

---

## 2. 基本语法总览

一个完整 lambda 长得像：

```cpp
[捕获列表](参数列表) mutable noexcept -> 返回类型 { 函数体 }
```

更完整一点：

```cpp
[捕获列表](参数列表) 说明符 异常规格 属性 -> 返回类型 { 函数体 }
```

例如：

```cpp
int x = 10;

auto f = [x](int y) mutable noexcept -> int {
    x += y;
    return x;
};
```

其中：

- `[x]`：捕获列表，把外部变量 `x` 按值捕获进来。
- `(int y)`：参数列表。
- `mutable`：允许修改按值捕获的副本。
- `noexcept`：表示不抛异常。
- `-> int`：显式返回类型。
- `{ x += y; return x; }`：函数体。

很多部分可以省略：

```cpp
auto f1 = [] { return 42; };
auto f2 = [](int x) { return x * 2; };
auto f3 = [x] { return x; };
auto f4 = [&x] { return x; };
```

---

## 3. 捕获列表：lambda 的“背包”

捕获列表决定 lambda 能把外部哪些变量装进自己的闭包对象里。

### 3.1 不捕获：`[]`

```cpp
auto f = [](int x) { return x + 1; };
```

它不依赖外部变量，最干净。无捕获 lambda 可以转函数指针：

```cpp
int (*fp)(int) = [](int x) { return x + 1; };
```

### 3.2 按值捕获：`[x]`

```cpp
int x = 10;
auto f = [x] { return x; };
x = 20;
std::cout << f(); // 输出 10
```

按值捕获发生在 lambda 创建时，之后外部 `x` 改变不影响 lambda 里的副本。

### 3.3 按引用捕获：`[&x]`

```cpp
int x = 10;
auto f = [&x] { return x; };
x = 20;
std::cout << f(); // 输出 20
```

按引用捕获不复制，lambda 内部访问的是外部变量本身。危险在于：如果 lambda 活得比变量久，就会悬垂。

### 3.4 隐式按值捕获：`[=]`

```cpp
int a = 1, b = 2;
auto f = [=] { return a + b; };
```

`[=]` 表示函数体里用到的外部变量都按值捕获。但注意：在成员函数里，`[=]` 会隐式捕获 `this` 指针，而不是复制整个对象。

### 3.5 隐式按引用捕获：`[&]`

```cpp
int a = 1, b = 2;
auto f = [&] { a++; b++; };
```

`[&]` 表示函数体里用到的外部变量都按引用捕获。非常方便，也非常容易悬垂。

### 3.6 混合捕获：`[=, &x]` 和 `[&, x]`

```cpp
int a = 1, b = 2;

auto f1 = [=, &a] { a++; return a + b; }; // b 按值，a 按引用
auto f2 = [&, b] { a++; return a + b; };  // a 按引用，b 按值
```

规则：

- 默认捕获是 `=` 时，显式捕获必须用 `&`。
- 默认捕获是 `&` 时，显式捕获必须用值。
- `[=, x]` 和 `[&, &x]` 通常是错误的，因为和默认捕获形式重复。

### 3.7 捕获 `this`：`[this]`

在成员函数里：

```cpp
struct S {
    int value = 42;

    auto getLambda() {
        return [this] { return value; };
    }
};
```

`[this]` 捕获的是 `this` 指针，不是对象副本。如果对象销毁了，再调用这个 lambda，就会悬垂。

### 3.8 捕获 `*this`：`[*this]`，C++17

```cpp
struct S {
    int value = 42;

    auto getLambda() {
        return [*this] { return value; };
    }
};
```

`[*this]` 会复制当前对象。即使原对象销毁，lambda 里的副本还在。

C++20 中，`[=]` 隐式捕获 `this` 被弃用，建议显式写 `[=, this]` 或 `[*this]`。

### 3.9 初始化捕获，也叫广义捕获：C++14

可以给捕获的变量起新名字，也可以移动捕获：

```cpp
auto ptr = std::make_unique<int>(42);

auto f = [p = std::move(ptr)] {
    return *p;
};
```

也可以捕获表达式结果：

```cpp
int x = 10;
auto f = [y = x + 1] { return y; };
```

也可以捕获引用：

```cpp
int x = 10;
auto f = [&r = x] { r++; };
```

初始化捕获是 C++14 非常重要的特性，让 lambda 可以捕获只能移动的对象。

### 3.10 捕获参数包：C++20

```cpp
template <class... Args>
auto makeLambda(Args... args) {
    return [args...] {
        return (args + ...);
    };
}
```

按引用捕获包：

```cpp
template <class... Args>
auto makeLambda(Args&... args) {
    return [&args...] {
        ((args++), ...);
    };
}
```

### 3.11 捕获结构化绑定：C++20

```cpp
auto [a, b] = std::pair{1, 2};
auto f = [a, b] { return a + b; };
```

C++20 起可以捕获结构化绑定。C++17 里这方面限制更多。

---

## 4. 参数列表：从普通参数到泛型、模板、显式对象参数

### 4.1 普通参数

```cpp
auto f = [](int x, int y) { return x + y; };
```

### 4.2 泛型 lambda：C++14

```cpp
auto add = [](auto a, auto b) {
    return a + b;
};

std::cout << add(1, 2);       // 3
std::cout << add(1.5, 2.5);   // 4.0
```

泛型 lambda 本质是：

```cpp
struct Lambda {
    template <class T, class U>
    auto operator()(T a, U b) const {
        return a + b;
    }
};
```

### 4.3 模板 lambda：C++20

可以显式写模板参数：

```cpp
auto f = []<typename T>(T x) {
    return x + x;
};
```

也可以配合参数包：

```cpp
auto print = []<typename... Ts>(Ts... args) {
    ((std::cout << args << ' '), ...);
};
```

### 4.4 显式对象参数：C++23

C++23 允许 lambda 使用“推导 this”：

```cpp
auto factorial = [](this auto&& self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};

std::cout << factorial(5); // 120
```

这让递归 lambda 写起来更自然，不必依赖 `std::function`。

---

## 5. 说明符：mutable、constexpr、consteval、noexcept

### 5.1 `mutable`

lambda 的 `operator()` 默认是 `const`。按值捕获的变量在 lambda 内部默认不能修改。

```cpp
int x = 0;

auto f = [x]() mutable {
    x++;
    return x;
};

std::cout << f(); // 1
std::cout << f(); // 2
std::cout << x;   // 0
```

`mutable` 修改的是 lambda 内部的副本，不影响外部 `x`。

### 5.2 `constexpr`：C++17

如果 lambda 满足 constexpr 要求，它可以隐式成为 constexpr。

```cpp
constexpr auto f = [](int x) {
    return x * x;
};

constexpr int y = f(5); // 25
```

也可以显式写：

```cpp
auto f = [](int x) constexpr {
    return x * x;
};
```

### 5.3 `consteval`：C++20

```cpp
auto f = [](int x) consteval {
    return x * x;
};

constexpr int y = f(5);
```

`consteval` 表示必须在编译期求值。

### 5.4 `noexcept`

```cpp
auto f = [](int x) noexcept {
    return x + 1;
};
```

### 5.5 属性

C++23 允许在 lambda 上使用更多属性，例如：

```cpp
auto f = [](int x) [[nodiscard]] {
    return x + 1;
};
```

---

## 6. 返回类型推导

C++11：

- 如果函数体只有一条 `return expr;`，返回类型推导为 `decltype(expr)`。
- 否则返回 `void`。
- 所以复杂 lambda 常需要显式返回类型：

```cpp
auto f = [](int x) -> int {
    if (x > 0) return x;
    return -x;
};
```

C++14 起返回类型推导更强，多分支也能推导，但所有 `return` 类型必须一致。

```cpp
auto f = [](int x) {
    if (x > 0) return x;
    return -x;
};
```

---

## 7. 闭包类型、函数指针、默认构造

### 7.1 每个 lambda 都有唯一类型

```cpp
auto f1 = [] {};
auto f2 = [] {};

// decltype(f1) 和 decltype(f2) 是不同类型
```

所以两个 lambda 即使代码一样，也不能互相赋值。

### 7.2 无捕获 lambda 转函数指针

```cpp
int (*fp)(int) = [](int x) { return x + 1; };
```

有捕获就不行：

```cpp
int y = 10;
// int (*fp)(int) = [y](int x) { return x + y; }; // 错误
```

### 7.3 无捕获 lambda 默认构造：C++20

```cpp
auto l = [] {};
decltype(l) l2; // C++20 起可以
```

C++20 之前，闭包类型通常没有默认构造函数。

---

## 8. 版本演进速查表

| 标准  | lambda 新能力                                                |
| ----- | ------------------------------------------------------------ |
| C++11 | 引入 lambda、捕获列表、`mutable`、`noexcept`、尾置返回类型、无捕获转函数指针 |
| C++14 | 泛型 lambda、初始化捕获、返回类型推导增强                    |
| C++17 | `constexpr` lambda、`[*this]` 捕获                           |
| C++20 | 模板 lambda、捕获参数包、捕获结构化绑定、无捕获 lambda 默认构造、`consteval` lambda、未求值上下文 lambda |
| C++23 | 有说明符时可省略空参数列表、显式对象参数 `this auto&&`、属性位置更灵活 |

---

## 9. 什么情况下用 lambda？典型场景

### 9.1 STL 算法谓词

```cpp
std::vector<int> v = {3, 1, 4, 1, 5};

std::sort(v.begin(), v.end(), [](int a, int b) {
    return a > b;
});
```

`std::find_if`：

```cpp
auto it = std::find_if(v.begin(), v.end(), [](int x) {
    return x % 2 == 0;
});
```

`std::transform`：

```cpp
std::transform(v.begin(), v.end(), v.begin(), [](int x) {
    return x * 2;
});
```

`std::for_each`：

```cpp
std::for_each(v.begin(), v.end(), [](int x) {
    std::cout << x << ' ';
});
```

### 9.2 自定义删除器

```cpp
auto deleter = [](FILE* f) {
    if (f) std::fclose(f);
};

std::unique_ptr<FILE, decltype(deleter)> fp(std::fopen("a.txt", "r"), deleter);
```

### 9.3 线程函数

```cpp
std::thread t([&] {
    std::cout << "running in thread\n";
});
t.join();
```

### 9.4 回调函数

```cpp
button.onClick([=] {
    std::cout << "clicked\n";
});
```

### 9.5 延迟执行 / 作用域清理

```cpp
auto cleanup = [&] {
    std::cout << "cleanup\n";
};

// 在合适的时候调用 cleanup();
```

### 9.6 立即调用 lambda

```cpp
int x = [] {
    return 42;
}();
```

常用于复杂初始化：

```cpp
const auto config = [] {
    Config c;
    c.load();
    c.validate();
    return c;
}();
```

### 9.7 递归 lambda

传统方式用 `std::function`：

```cpp
std::function<int(int)> fib = [&](int n) -> int {
    return n <= 1 ? 1 : fib(n - 1) + fib(n - 2);
};
```

C++23 可以用显式对象参数：

```cpp
auto fib = [](this auto&& self, int n) -> int {
    return n <= 1 ? 1 : self(n - 1) + self(n - 2);
};
```

### 9.8 编译期计算

```cpp
constexpr auto square = [](int x) constexpr {
    return x * x;
};

constexpr int y = square(5);
```

### 9.9 泛型算法组件

```cpp
auto add = [](auto a, auto b) {
    return a + b;
};

std::cout << add(1, 2) << '\n';
std::cout << add(std::string("a"), std::string("b")) << '\n';
```

### 9.10 捕获移动对象

```cpp
auto ptr = std::make_unique<int>(42);

auto f = [p = std::move(ptr)] {
    return *p;
};
```

---

## 10. 常见陷阱与最佳实践

### 10.1 按引用捕获导致悬垂

```cpp
std::function<void()> f;

{
    int x = 10;
    f = [&x] { std::cout << x; };
}

f(); // 危险：x 已经销毁
```

### 10.2 `[=]` 在成员函数里捕获的是 `this`

```cpp
struct S {
    int value = 42;

    auto f() {
        return [=] { return value; }; // 实际捕获 this
    }
};
```

如果对象销毁，lambda 再调用就悬垂。想复制对象用：

```cpp
return [*this] { return value; };
```

### 10.3 `mutable` 修改的是副本

```cpp
int x = 0;
auto f = [x]() mutable { x++; };
f();
f();
std::cout << x; // 0
```

### 10.4 `std::function` 有开销

```cpp
std::function<int(int)> f = [](int x) { return x + 1; };
```

`std::function` 可能堆分配、类型擦除、间接调用。能用 `auto` 就用 `auto`。

### 10.5 避免无脑 `[&]` 和 `[=]`

显式捕获更安全：

```cpp
auto f = [&x, y] { ... };
```

### 10.6 注意 lambda 生命周期

lambda 对象本身也可能悬垂，尤其是返回捕获了局部变量引用的 lambda：

```cpp
auto makeLambda() {
    int x = 10;
    return [&x] { return x; }; // 危险
}
```

---

## 11. 总结成一句话

C++ lambda 是“带捕获状态的匿名函数对象”：

- 语法核心是 `[捕获](参数) 说明符 -> 返回 { 体 }`。
- 捕获列表决定它背包里装什么。
- `mutable` 决定能不能改副本。
- 泛型 lambda、模板 lambda、初始化捕获、`*this`、C++23 显式对象参数让它越来越强。
- 最适合：STL 算法、回调、线程、删除器、延迟执行、编译期计算、泛型小函数。
- 最大坑：按引用捕获和 `this` 捕获导致悬垂。
- 最佳实践：显式捕获、注意生命周期、少用 `std::function`、优先 `auto`。

如果你愿意，我也可以继续给你整理一份“C++ lambda 面试题 + 易错点 + 代码输出题”的版本。