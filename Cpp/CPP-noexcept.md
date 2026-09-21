先一句话：**`noexcept` 是 C++ 里的“异常承诺书”。你签了它，就等于告诉编译器、标准库和调用者：“这个函数不会让异常跑出去。”如果异常真的跑出去，程序不会正常传播异常，而是直接 `std::terminate`，通常就是崩掉。**

它和移动构造关系极大：`T(T&&) noexcept` 不只是“写给人看”，它会直接影响 `std::vector` 扩容时到底移动还是拷贝你的对象。

---

## 1. `noexcept` 的两种身份

### 身份一：异常说明符，写在函数后面

```cpp
void f() noexcept;              // 承诺不抛
void g() noexcept(true);        // 同上
void h() noexcept(false);       // 明确可能抛，基本等于没写
void k() noexcept(noexcept(x)); // 条件承诺：看表达式结果
```

构造函数、移动构造、析构、成员函数都能写：

```cpp
struct A {
    A(const A&);            // 拷贝构造，可能分配内存，可能抛
    A(A&&) noexcept;        // 移动构造，通常只偷指针，承诺不抛
    ~A() noexcept;          // 析构，通常也不该抛
};
```

### 身份二：`noexcept` 运算符，用来问“这个表达式会不会抛”

```cpp
noexcept(f())   // 返回 bool 常量，不调用 f()
```

注意：**`noexcept(expr)` 不会真的执行 `expr`**。它只在编译期判断这个表达式是否被声明为不抛异常。

常见写法：

```cpp
void may_throw();

void f() noexcept(noexcept(may_throw())) {
    may_throw();
}
```

如果 `may_throw()` 本身不是 `noexcept`，那么 `f()` 就不是 `noexcept`。这叫“条件 noexcept”。

---

## 2. 签了 `noexcept`，但异常跑出去会怎样？

看这个：

```cpp
void f() noexcept {
    throw std::runtime_error("boom");
}
```

`f()` 里可以写 `throw`，编译器不一定禁止。但异常一旦试图逃出 `f()`，就会调用 `std::terminate`。默认行为通常是直接终止程序，而且**标准不保证会正常栈展开、调用局部对象析构**。所以：

> `noexcept` 不是“函数里不能有 throw”，而是“异常不能逃出这个函数边界”。

这是个很硬的契约。你乱签，程序就可能直接炸。

---

## 3. 为什么 C++11 要引入 `noexcept`？

老 C++ 有动态异常规范，比如：

```cpp
void f() throw(int, std::string);
```

这东西运行时检查、性能差、和泛型代码配合糟糕。C++11 把它弃用，C++17 移除，改用 `noexcept`。

`noexcept` 的好处：

1. **编译期就知道**，不是运行时才检查。
2. **能进入函数类型系统**。C++17 起，`noexcept` 是函数类型的一部分。
3. **标准库可以根据它做策略选择**，最典型就是 `std::vector` 扩容。
4. **调用者可以据此写异常安全代码**，不用准备 catch。

---

## 4. 最关键作用：影响 `std::vector` 扩容时移动还是拷贝

你前面那段说：

> 只要类型定义了移动构造，`T b = std::move(a)` 才真的会走移动。

这只说对了一半。对于 `std::vector` 这种容器，还要看移动构造是不是 `noexcept`。

`std::vector` 扩容时，要把旧元素搬到新内存。它心里会想：

- 如果移动构造是 `noexcept`：放心移动，因为不会在搬家途中爆炸。
- 如果移动构造可能抛，而类型又可以拷贝：那就拷贝，因为拷贝失败时原对象还在，vector 还能回滚，保证强异常安全。
- 如果类型不可拷贝：没办法，只能移动，但异常安全保证会弱一些。

标准库用的工具叫 `std::move_if_noexcept`，逻辑大致是：

```cpp
if (std::is_nothrow_move_constructible_v<T> || !std::is_copy_constructible_v<T>)
    return std::move(x);   // 移动
else
    return x;              // 拷贝
```

所以：

```cpp
struct BadMove {
    BadMove(const BadMove&);   // 可拷贝
    BadMove(BadMove&&);        // 移动构造没写 noexcept
};

std::vector<BadMove> v;
v.reserve(100); // 扩容时可能拷贝旧元素，而不是移动
```

```cpp
struct GoodMove {
    GoodMove(const GoodMove&);
    GoodMove(GoodMove&&) noexcept; // 关键
};

std::vector<GoodMove> v;
v.reserve(100); // 扩容时敢移动旧元素
```

性能差距可能巨大：一个只是偷指针，一个是深拷贝。

所以实践口诀：

> **移动构造、移动赋值，能 `noexcept` 就 `noexcept`。**

---

## 5. `noexcept` 对异常安全的意义

异常安全通常分三级：

1. **基本保证**：不泄漏，对象有效，但状态可能变了。
2. **强保证**：失败就回滚，像没发生过。
3. **不抛保证**：也就是 `noexcept`，根本不失败。

`noexcept` 是最高等级。标准库很多强保证依赖它：

- `vector::reserve` 想提供强保证。
- 如果元素移动不抛，它可以移动旧元素。
- 如果移动可能抛，它只能拷贝，或者接受较弱保证。

所以 `noexcept` 不只是优化开关，它是**异常安全策略的输入**。

---

## 6. `noexcept` 对编译器优化的作用

编译器知道一个函数不会抛异常，可以：

- 不为它生成复杂的异常处理表。
- 不准备 landing pad。
- 调用点少写 catch 路径。
- 更容易内联和优化。

但别误会：`noexcept` 不会自动让函数变快。真正的大收益往往来自标准库策略，比如 `vector` 选择移动而不是拷贝。

---

## 7. 默认规则和常见坑

### 坑 1：写了拷贝构造，移动构造不自动生成

更准确地说，如果你声明了拷贝构造、拷贝赋值、移动构造、移动赋值、析构中的任何一个，编译器通常就不再隐式生成移动构造。

所以：

```cpp
struct A {
    A(const A&); // 自己写了拷贝构造
    // 移动构造不会隐式生成
};
```

想保留移动，必须显式写：

```cpp
A(A&&) noexcept = default;
```

或者自己实现。

### 坑 2：写了移动构造，但忘了 `noexcept`

```cpp
A(A&&); // 有移动构造，但 vector 可能不敢用
```

标准库一查：`is_nothrow_move_constructible<A>` 是 false。如果 `A` 可拷贝，vector 扩容就拷贝。

### 坑 3：乱加 `noexcept`

```cpp
struct A {
    std::string s;
    A(A&&) noexcept : s(std::move(s)) {} // 通常没问题
};
```

但如果移动里可能分配内存、可能调用用户未知代码，就别乱加。加了又抛，直接 `terminate`。

### 坑 4：以为 `noexcept(expr)` 会执行 expr

```cpp
noexcept(f()); // 不会调用 f()
```

它只是编译期问：`f()` 这个表达式是否声明为不抛。

### 坑 5：析构函数抛异常

析构函数默认倾向 `noexcept(true)`，除非成员或基类析构可能抛。析构里抛异常非常危险，尤其栈展开时，很容易直接 `terminate`。实践中析构不要抛。

---

## 8. 条件 `noexcept`：模板里最常用

模板类型不知道 `T` 的移动会不会抛，所以要条件写：

```cpp
template<class T>
struct Wrapper {
    T value;

    Wrapper(Wrapper&& other)
        noexcept(std::is_nothrow_move_constructible<T>::value)
        : value(std::move(other.value))
    {}
};
```

如果 `T` 的移动构造不抛，`Wrapper` 的移动构造也不抛；否则就不是。这叫“异常规范传播”。

标准库大量类型都这么干：

- `std::optional`
- `std::variant`
- `std::pair`
- `std::tuple`
- 各种容器和智能指针

比如 `std::unique_ptr` 的移动构造就是 `noexcept`，所以 `std::vector<std::unique_ptr<T>>` 扩容时能高效移动。

---

## 9. 虚函数、函数指针里的 `noexcept`

虚函数覆盖有规则：派生类覆盖函数的异常规范不能比基类更宽松。

```cpp
struct Base {
    virtual void f() noexcept;
};

struct Derived : Base {
    void f() override; // 错误：基类承诺不抛，派生不能变成可能抛
};
```

反过来可以：

```cpp
struct Base {
    virtual void f();
};

struct Derived : Base {
    void f() noexcept override; // 可以，承诺更强
};
```

C++17 起，`noexcept` 也是函数类型的一部分：

```cpp
void f() noexcept;

void (*p)() noexcept = f; // 可以
void (*q)() = p;          // 可以：noexcept 函数指针可转成普通函数指针
// void (*r)() noexcept = q; // 不行：普通函数指针不能反转为 noexcept
```

---

## 10. 实践清单：什么时候写 `noexcept`

适合写：

- 移动构造、移动赋值：通常只转移资源，不分配。
- `swap`：通常只是交换指针/成员。
- 析构函数：默认就该不抛。
- 简单 getter、不分配不抛的访问函数。
- 标准库风格的模板：用条件 `noexcept`。

不要写：

- 可能分配内存的函数。
- 可能调用用户回调、虚函数、未知代码的函数。
- 拷贝构造，除非你确定它绝不抛。
- 任何你无法保证所有路径都不抛的函数。

模板里推荐：

```cpp
T(T&&) noexcept(std::is_nothrow_move_constructible_v<Member>);
```

---

## 11. 生动总结

把 `noexcept` 想成一张“不炸承诺书”：

- 你签了，调用者就敢让你走快速通道。
- `std::vector` 搬家时，看到你签了，就敢移动你；没签，就宁愿拷贝。
- 你没签，性能可能下降。
- 你乱签，结果真炸了，程序直接 `std::terminate`，不给你正常抛异常的机会。

所以对你的移动构造：

```cpp
T(T&&) noexcept = default;
```

或者：

```cpp
T(T&& other) noexcept
    : resource(std::exchange(other.resource, nullptr))
{}
```

这是最常见的正确姿势。

一句话口诀：

> **移动构造加 `noexcept`，vector 才敢移动；模板条件 `noexcept`，别乱签合同；签了还抛异常，`terminate` 不留情。**