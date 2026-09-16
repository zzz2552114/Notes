## 1. 类模板：`Foo`、`Foo2`

```cpp
template<typename T>
class Foo {
    T var_;
public:
    Foo(T var) : var_(var) {}
    void print() { std::cout << var_ << std::endl; }
};
```

`Foo` 本身不是类，是“类的蓝图”。  
真正用的是 `Foo<int>`、`Foo<float>` 这种，它们是**具体类型**。

代码里：

```cpp
Foo<int> a(3);      // 显式指定 T = int
Foo b(3.4f);        // C++17 类模板实参推导，自动推成 Foo<float>
Foo<float> b(3.4);
```

`Foo<int>` 和 `Foo<float>` 是两个不同类型。  
虽然 `print()` 看起来差不多，但编译器眼里它们就是两个类。

多类型参数：

```cpp
template<typename T, typename U>
class Foo2 { ... };

Foo2<int, float> c(3, 3.2f);
```

就是一次抽两个类型出来。

---

## 2. 函数模板：`add`、`print_two_values`、`print_msg`、`add3`

```cpp
template <typename T> T add(T a, T b) { return a + b; }
```

用法：

```cpp
add<int>(3, 5);        // 显式指定 T = int
add<float>(2.8, 3.7);  // 显式指定 T = float
add(3, 5);             // 自动推导 T = int
```

多类型：

```cpp
template<typename T, typename U>
void print_two_values(T a, U b);
```

特化：

```cpp
template <typename T> void print_msg() { std::cout << "Hello world!\n"; }
template <> void print_msg<float>() { std::cout << "print_msg called with float type!\n"; }
// 注意这里不是 template <float> void print_msg() {xxx}
// 调用的时候 print_msg<type>()，不是传值
```

`print_msg<int>()` 走通用版，`print_msg<float>()` 走特化版。  
特化就是“给某个具体类型开小灶”。

非类型模板参数也能用在函数上：

```cpp
template <bool T> int add3(int a) {
    if (T) return a + 3;
    return a;
}

add3<true>(3);   // 6
add3<false>(3);  // 3
```

这里 `T` 不是类型，是编译期的 `bool` 值。

---

## 3. 特化：`FooSpecial<float>`

主模板：

```cpp
template<typename T>
class FooSpecial {
    T var_;
public:
    FooSpecial(T var) : var_(var) {}
    void print() { std::cout << var_ << std::endl; }
};
```

给 `float` 开小灶：

```cpp
template<> class FooSpecial<float> {
    float var_;
public:
    FooSpecial(float var) : var_(var) {}
    void print() { std::cout << "hello float! " << var_ << std::endl; }
};
```

于是：

```cpp
FooSpecial<int> d(5);      // 普通版，打印 5
FooSpecial<float> e(4.5);  // 特化版，打印 hello float! 4.5
```

注意：写特化之前，必须先有主模板。  
`template<>` 和后面的 `class` 之间可以有空格、换行、注释，但不能插别的代码。它俩必须组成同一个声明。

---

## 4. 非类型模板参数：`Bar<150>`

```cpp
template<int T>
class Bar {
public:
    Bar() {}
    void print_int() { std::cout << "print int: " << T << std::endl; }
};
```

用法：

```cpp
Bar<150> f;
f.print_int();  // print int: 150
```

这里的 `150` 不是构造函数参数，是**模板实参**。  
`T` 是编译期整数常量，它成了类型的一部分。

所以：

- `Bar<150>` 和 `Bar<151>` 是**两个不同类型**。
- 它们不能互相赋值，指针也不兼容。
- 可以写函数重载：`void f(Bar<1>)` 和 `void f(Bar<2>)` 是不同参数类型。
- `T` 通常不占对象内存，因为它在类型里，不在对象里。

---

## 5. 值当模板参数 vs 构造函数参数：`Bar<150>` vs `Bar2(10)`

```cpp
class Bar2 {
    int a;
public:
    Bar2(int a) : a(a) {}
    void print_int() { std::cout << "print int: " << a << std::endl; }
};
```

对比：

|          | `Bar<150>`                      | `Bar2(10)`          |
| -------- | ------------------------------- | ------------------- |
| 值在哪   | 类型里                          | 对象里              |
| 确定时机 | 编译期                          | 运行期              |
| 类型     | `Bar<150>`、`Bar<151>` 不同类型 | 所有对象都是 `Bar2` |
| 互相赋值 | 不行                            | 可以                |
| 指针兼容 | 不行                            | 都兼容              |
| 函数重载 | 可以按 `Bar<1>`、`Bar<2>` 重载  | 不能按 `a` 的值重载 |
| 运行时改 | 不能                            | 可以                |

一句话：  
`template<int T>` 把整数提升到了**类型层面**；  
`Bar2(int a)` 把整数留在了**对象层面**。

---

## 6. `Bar2` 的移动构造

`Bar2` 只有 `int a`，没有用户声明的析构、拷贝、移动等。  
所以编译器会隐式生成移动构造。

```cpp
Bar2 b(10);
Bar2 a = std::move(b);  // 合法，走移动构造
```

但 `int` 的移动就是拷贝，所以 `b.a` 还是 10。

如果 `Bar2` 里是 `std::vector` 或 `std::string`：

```cpp
class Bar2 {
    std::vector<int> v;
    std::string s;
public:
    Bar2(std::vector<int> v, std::string s) : v(std::move(v)), s(std::move(s)) {}
};
```

默认情况下，`Bar2 a = std::move(b);` 会走隐式移动构造，  
`vector`/`string` 的资源会被真正转移，源对象变成“有效但未指定状态”（通常为空）。

但如果你自己声明了析构函数：

```cpp
~Bar2() {}
```

编译器就不隐式生成移动构造了。  
这时 `std::move(b)` 会退化成拷贝构造，深拷贝 `vector`/`string`。

想确保移动，可以显式写：

```cpp
Bar2(Bar2&&) = default;
Bar2& operator=(Bar2&&) = default;
```

或者遵守 Rule of Zero，别乱声明析构和拷贝操作。

---

## 7. 小坑提醒

- 第一份代码 `main` 里先有 `Foo<int> a(3);`，后面又写 `Bar2 a(10);`，变量名重复，实际编译会报错。改个名就行，比如 `Bar2 a2(10);`。
- `Foo b(3.4f);` 是 C++17 的类模板实参推导，老编译器可能不支持。
- 特化之前必须先有主模板。
- `template<>` 和 `class` 中间可以换行、空格、注释，但不能插别的声明。

---

## 8. 最后总结

- 类模板：`Foo<T>`、`Foo2<T,U>`，实例化成不同类型。
- 函数模板：`add<T>`、`print_two_values<T,U>`、`add3<bool T>`。
- 特化：给特定类型开小灶，`FooSpecial<float>`、`print_msg<float>`。
- 非类型模板参数：`Bar<150>`、`add3<true>`，值变成类型的一部分。
- 值模板参数 vs 构造函数参数：一个在类型层面，一个在对象层面。
- 移动构造：默认有，成员是 `vector`/`string` 时能真正转移资源；但用户声明析构等会抑制移动构造，退化成拷贝。

就这些。模板看着花，其实就是让编译器在编译期帮你生成一堆具体代码。