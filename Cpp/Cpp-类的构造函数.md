下面以现代 C++（C++11/14/17/20 为主）说明。严格说，C++ 标准里的“特殊成员函数”中，构造函数主要有三类：**默认构造函数、拷贝构造函数、移动构造函数**；但日常说“构造函数有哪些情况”，还包括用户自己写的各种重载、委托构造、继承构造、转换构造、initializer_list 构造等。

## 1. 构造函数的基本性质

构造函数：

- 名字与类名相同；
- 没有返回类型；
- 可以重载；
- 创建对象时自动调用；
- 不能声明为 `virtual`、`static`、`const`、`volatile`；
- 可以有 `explicit`、`noexcept`、`constexpr`、`consteval`、`=default`、`=delete` 等修饰；
- 可以是 `public`、`protected`、`private`。

例子：

```cpp
class A {
public:
    A() {}              // 默认构造函数
    A(int x) {}         // 带参构造函数
    A(const A& other) {}// 拷贝构造函数
    A(A&& other) {}     // 移动构造函数
};
```

注意：拷贝赋值运算符 `operator=`、移动赋值运算符 `operator=`、析构函数都不是构造函数。

---

## 2. 构造函数的各种情况

### 2.1 默认构造函数

可以无参调用的构造函数叫默认构造函数。形式可以是：

```cpp
class A {
public:
    A() {}                 // 无参
    // A(int x = 0) {}     // 也可以无参调用，也算默认构造函数
};
```

隐式默认构造函数：

- 如果类没有用户声明的任何构造函数，编译器可能隐式生成一个默认构造函数；
- 如果用户声明了任意构造函数，编译器通常不再隐式生成默认构造函数；
- 可以用 `A() = default;` 显式要求生成；
- 如果类有引用成员、`const` 成员且没有默认初始化器、或者有无默认构造的基类/成员，隐式默认构造函数可能被定义为删除。

例子：

```cpp
class A {
public:
    A() = default;
};

A a;   // 调用默认构造函数
A b{}; // 值初始化
```

---

### 2.2 普通带参构造函数

用户定义、接受参数的构造函数。

```cpp
class Point {
    int x, y;
public:
    Point(int x_, int y_) : x(x_), y(y_) {}
};

Point p(1, 2);
```

也可以有默认参数：

```cpp
class A {
public:
    A(int x = 0, int y = 0) {}
};
```

这种 `A(int x = 0, int y = 0)` 也可以无参调用，所以它同时算默认构造函数。

---

### 2.3 拷贝构造函数

拷贝构造函数用于用同类型对象初始化新对象。典型形式：

```cpp
class A {
public:
    A(const A& other) {
        // 拷贝 other 的资源
    }
};
```

也可以是非 `const` 引用、`volatile` 引用等，但最常见是 `const A&`。

调用时机：

```cpp
A a;
A b(a);        // 拷贝构造
A c = a;       // 拷贝构造
```

编译器隐式生成拷贝构造的条件很复杂，大致是：

- 如果用户没有声明拷贝构造函数，编译器可能生成；
- 如果用户声明了移动构造函数或移动赋值运算符，拷贝构造函数可能被定义为删除；
- 如果成员不可拷贝，隐式拷贝构造也会被删除。

---

### 2.4 移动构造函数

C++11 引入，用于“窃取”临时对象或右值对象的资源。典型形式：

```cpp
#include <utility>

class A {
    int* data;
public:
    A(A&& other) noexcept {
        data = other.data;
        other.data = nullptr;
    }
};
```

调用：

```cpp
A a;
A b(std::move(a)); // 移动构造
```

移动构造函数通常声明为 `noexcept`，因为标准容器在扩容时更愿意使用不会抛异常的移动构造。

隐式生成移动构造的条件：

- 用户没有声明拷贝构造、拷贝赋值、移动赋值、析构函数；
- 成员可移动；
- 否则可能不生成或被删除。

---

### 2.5 委托构造函数

一个构造函数在初始化列表中调用同一个类的另一个构造函数，叫委托构造。

```cpp
class A {
    int x;
public:
    A() : A(0) {}          // 委托给 A(int)
    A(int x_) : x(x_) {}
};
```

规则：

- 委托构造的初始化列表里只能有这一个委托调用；
- 不能再同时初始化其他成员或基类；
- 被委托的构造函数先执行完，然后才执行当前构造函数体；
- 不能形成委托环。

```cpp
A() : A(0) {} // 正确
// A() : A(0), x(1) {} // 错误：委托时不能再初始化其他成员
```

---

### 2.6 继承构造函数

C++11 允许派生类继承基类构造函数：

```cpp
struct Base {
    Base(int);
    Base(int, int);
};

struct Derived : Base {
    using Base::Base; // 继承 Base 的构造函数
};

Derived d(1);     // 调用 Base(int)
Derived e(1, 2);  // 调用 Base(int, int)
```

一般说，继承构造函数不会继承默认构造、拷贝构造、移动构造；这些仍按派生类自己的规则生成或删除。

---

### 2.7 转换构造函数与 explicit

如果一个非 `explicit` 构造函数可以用一个实参调用，它就可以作为隐式转换：

```cpp
class A {
public:
    A(int x) {} // 转换构造函数
};

void f(A);

f(1); // int 隐式转换为 A
// 等价于 f(A(1));
```

加上 `explicit` 后禁止隐式转换：

```cpp
class B {
public:
    explicit B(int x) {}
};

// B b = 1;   // 错误，但是不加explicit就是正确的，会变成 B b = B(1);
B b(1);       // 正确
B b2 = B(1);  // 正确
```

C++11 后，多参数构造函数也可能参与列表初始化的转换：

```cpp
struct P {
    P(int, int);
};

P p = {1, 2}; // 如果 P(int,int) 不是 explicit，可以
```

---

### 2.8 initializer_list 构造函数

用于列表初始化：

```cpp
#include <initializer_list>
#include <vector>

class Vec {
    std::vector<int> data;
public:
    Vec(std::initializer_list<int> il) : data(il) {}
};

Vec v{1, 2, 3};
```

当使用 `{}` 初始化时，如果存在 `initializer_list` 构造函数，通常会优先匹配它。

---

### 2.9 模板构造函数 / 完美转发构造函数

构造函数也可以是模板：

```cpp
#include <utility>

class Any {
public:
    template<class T>
    Any(T&& value) {
        // 完美转发
    }

    Any(const Any&) = default; // 仍需要拷贝构造
};
```

注意：模板构造函数不是拷贝构造函数。它不会替代编译器生成的拷贝/移动构造函数，有时还会“劫持”重载，所以常用 `enable_if` 或 C++20 概念约束。

---

### 2.10 默认、删除、显式默认

```cpp
class A {
public:
    A() = default;                 // 显式要求默认构造
    A(const A&) = delete;          // 禁止拷贝构造
    A(A&&) noexcept = default;     // 显式默认移动构造
};
```

`= delete` 表示该构造函数被删除，调用会编译错误。

---

### 2.11 constexpr / consteval / noexcept

```cpp
class A {
    int x;
public:
    constexpr A(int x_) : x(x_) {} // 可用于编译期
    A(A&&) noexcept = default;     // 不抛异常
};
```

C++20 还有 `consteval` 立即函数构造函数。

---

### 2.12 聚合初始化不是构造函数

聚合类可以没有用户声明的构造函数：

```cpp
struct Agg {
    int x;
    double y;
};

Agg a{1, 2.0}; // 聚合初始化，不调用用户定义构造函数
```

它属于对象初始化方式，但不是“调用某个构造函数”。

---

## 3. 除了用冒号，可以在函数体里赋值吗？

**可以，但要看成员类型。**  
冒号后面的是“成员初始化列表”，它做的是初始化。函数体里的赋值只是赋值，不是初始化。

构造函数执行顺序是：

1. 虚基类；
2. 直接基类；
3. 非静态数据成员，按声明顺序初始化；
4. 最后执行构造函数体。

所以函数体执行时，成员已经“初始化过了”。如果没有在初始化列表写某个成员：

- 类类型成员：会先调用它的默认构造函数；
- 内置类型成员：默认初始化，通常值不确定；
- 然后进入函数体，你可以在函数体里给它赋值。

例子：可以函数体赋值

```cpp
#include <string>

class A {
    int x;
    std::string s;
public:
    A(int a, const char* p) {
        x = a;   // 可以：x 先默认初始化，然后赋值
        s = p;   // 可以：s 先默认构造，然后调用 operator=
    }
};
```

但这不是初始化，而是“先默认构造，再赋值”。对 `std::string` 这种类型，可能多一次默认构造和赋值，效率不如初始化列表：

```cpp
A(int a, const char* p) : x(a), s(p) {} // 直接初始化，更好
```

必须用初始化列表的情况：

### 3.1 const 成员

```cpp
class A {
    const int c;
public:
    A(int x) : c(x) {} // 必须
    // A(int x) { c = x; } // 错误：const 不能赋值
};
```

### 3.2 引用成员

```cpp
class A {
    int& r;
public:
    A(int& ref) : r(ref) {} // 必须
    // A(int& ref) { r = ref; } // 错误：引用必须初始化
};
```

引用一旦绑定，不能重新绑定。函数体里的 `r = ref` 不是重新绑定，而是给引用所指对象赋值；如果引用成员没初始化，本身就编译错误。

### 3.3 无默认构造函数的类类型成员

```cpp
struct NoDefault {
    NoDefault(int);
};

class A {
    NoDefault n;
public:
    A(int x) : n(x) {} // 必须
    // A(int x) { n = NoDefault(x); } // 错误：n 没有默认构造，无法先默认构造再赋值
};
```

### 3.4 无默认构造函数的基类

```cpp
struct Base {
    Base(int);
};

struct Derived : Base {
    Derived(int x) : Base(x) {} // 必须
    // Derived(int x) { } // 错误：Base 没有默认构造
};
```

### 3.5 需要直接初始化或效率要求高时

```cpp
class A {
    std::string s;
public:
    A(const char* p) : s(p) {} // 直接构造
    // A(const char* p) { s = p; } // 先默认构造，再赋值，效率较低
};
```

---

## 4. 函数体赋值的其他限制

### 4.1 静态成员不能在初始化列表初始化

```cpp
class A {
    static int n;
public:
    A() {
        n = 1; // 可以，在函数体赋值
    }
    // A() : n(1) {} // 错误：静态成员不属于对象
};

int A::n = 0;
```

### 4.2 数组成员不能整体赋值

```cpp
class A {
    int arr[3];
public:
    A() : arr{1, 2, 3} {} // 可以
    // A() { arr = {1, 2, 3}; } // 错误：数组不能整体赋值
};
```

函数体里可以逐元素赋值。

### 4.3 初始化顺序按声明顺序，不按冒号顺序

```cpp
class A {
    int a;
    int b;
public:
    A(int x) : b(x), a(b) {} 
    // 实际先初始化 a，再初始化 b！
    // a 使用未初始化的 b，危险
};
```

正确写法应保持声明顺序一致：

```cpp
class A {
    int a;
    int b;
public:
    A(int x) : a(x), b(a) {}
};
```

---

## 5. 综合例子

```cpp
#include <initializer_list>
#include <string>
#include <utility>

struct Base {
    int b;
    Base() : b(0) {}
    Base(int x) : b(x) {}
};

class Demo : public Base {
    int a;
    const int c;
    int& r;
    std::string s;

public:
    Demo() : Demo(0) {} // 委托构造

    Demo(int x)
        : Base(x), a(x), c(x), r(a), s("hi") {} // 普通带参构造

    Demo(const Demo& o)
        : Base(o), a(o.a), c(o.c), r(a), s(o.s) {} // 拷贝构造

    Demo(Demo&& o) noexcept
        : Base(std::move(o)), a(o.a), c(o.c), r(a), s(std::move(o.s)) {} // 移动构造

    Demo(std::initializer_list<int> il)
        : Base(), a(0), c(0), r(a), s() {
        for (int v : il) a += v;
    } // initializer_list 构造

    explicit Demo(int x, int y)
        : Base(x), a(x + y), c(0), r(a), s() {} // explicit 多参构造

    Demo(double d) : Demo(static_cast<int>(d)) {} // 转换构造 + 委托构造
};

struct Derived : Base {
    using Base::Base; // 继承构造
};
```

函数体赋值示例：

```cpp
class AssignInBody {
    int x;
    std::string s;
public:
    AssignInBody(int a, const char* p) {
        x = a; // 可以
        s = p; // 可以，但 s 先默认构造再赋值
    }
};

struct NoDefault {
    NoDefault(int);
};

class MustUseInitList {
    const int c;
    int& r;
    NoDefault n;
public:
    MustUseInitList(int a, int& ref)
        : c(a), r(ref), n(a) {} // 必须用初始化列表

    // MustUseInitList(int a, int& ref) {
    //     c = a;          // 错误
    //     r = ref;        // 错误
    //     n = NoDefault(a); // 错误：n 无默认构造
    // }
};
```

---

## 6. 总结

构造函数的常见情况包括：

1. 默认构造函数；
2. 普通带参构造函数；
3. 拷贝构造函数；
4. 移动构造函数；
5. 委托构造函数；
6. 继承构造函数；
7. 转换构造函数；
8. `initializer_list` 构造函数；
9. 模板/完美转发构造函数；
10. 带 `explicit`、`constexpr`、`noexcept`、`=default`、`=delete` 等修饰的构造函数；
11. 聚合初始化不是构造函数，但也是对象初始化方式。

关于“能不能在函数体里赋值”：

- **能**，对普通可赋值成员可以；
- 但函数体里做的是“赋值”，不是“初始化”；
- `const` 成员、引用成员、无默认构造的类类型成员、无默认构造的基类，通常必须用冒号初始化列表；
- 类类型成员如果在函数体赋值，会先默认构造再赋值，可能低效；
- 初始化顺序按成员声明顺序，不按初始化列表书写顺序；
- 静态成员不能在构造初始化列表里初始化；
- 数组成员不能整体赋值。

因此，能用初始化列表时，优先用初始化列表；函数体赋值适合那些已经默认构造、并且确实可以赋值的成员。



## 调用构造函数

对于：

```cpp
explicit Node(int v) : val(v), prev(nullptr), next(nullptr) {}
```

最常见、最直接的写法其实是：

```cpp
Node t(x);     // ✅ 直接初始化，调用 Node(int)
Node t{x};     // ✅ 直接列表初始化，调用 Node(int)
```

`Node t = Node(x);` 也可以：

```cpp
Node t = Node(x);  // ✅ Node(x) 已经显式构造了一个 Node
```

但这里不是“只能这样”，而是右侧先显式调用 `Node(int)`，然后再用它初始化 `t`。C++17 之后通常会直接构造 `t`，省略拷贝/移动。

`explicit` 真正禁止的是这种隐式转换：

```cpp
Node t = x;        // ❌ 错误：不能把 int 隐式转换成 Node
Node t = {x};      // ❌ 错误：复制列表初始化不能用 explicit 构造函数
```

函数传参也一样：

```cpp
void f(Node n);

f(Node(x));        // ✅ 显式构造，可以
f(x);              // ❌ 隐式转换，不可以
```

返回时：

```cpp
Node make() {
    return Node(x); // ✅
    // return x;    // ❌
}
```

容器里：

```cpp
std::vector<Node> v;
v.push_back(Node(x)); // ✅
v.emplace_back(x);    // ✅ 直接构造
v.push_back(x);       // ❌ 隐式转换
```

所以总结：

```cpp
Node t(x);         // ✅ 推荐，直接调用 explicit 构造函数
Node t{x};         // ✅ 也可以
Node t = Node(x);  // ✅ 可以，但不是唯一写法
Node t = x;        // ❌ explicit 禁止
```

你显式写 `Node(x)`、`Node t(x)`、`new Node(x)` 都可以。