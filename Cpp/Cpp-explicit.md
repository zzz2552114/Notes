# C++ `explicit` 

---

## 一、设计初衷：为什么需要 `explicit`？

### 1.1 历史背景

C++ 早期设计时，Bjarne Stroustrup 希望用户定义类型能像内置类型一样自然。内置类型有隐式转换（如 `int` → `long`），所以用户定义类型也被赋予了隐式转换能力。当时的观点是："没有人会写愚蠢的代码，每个人都会享受良好隐式转换带来的好处。"

### 1.2 现实打脸

实践证明，隐式转换在很多场景下会造成**难以发现的 bug**。最核心的问题是：

> **隐式转换让本该报错的代码编译通过了。**

它不是在帮程序员偷懒，而是在**掩盖类型错误**。编译器不再报"类型不匹配"，而是自作主张地帮你转换，导致代码意图被隐藏、重构时炸弹遍地、重载决议被污染。

### 1.3 `explicit` 的诞生

C++ 标准委员会引入 `explicit`，把"是否允许隐式转换"的决定权从编译器手里夺回来，交还给程序员。它是一个**选择性加入（opt-in）**的安全机制。

**为什么不做成默认？** 向后兼容性。改变已有构造函数的含义会让大量现有代码无法编译。委员会最关心的问题之一就是保持向后兼容。

---

## 二、`explicit` 是什么？

`explicit` 是 C++ 中的一个**函数说明符**，用于修饰：

- 类的**构造函数**（C++98 起）
- **转换运算符**（C++11 起）
- **推导指引**（C++17 起）

其核心作用是：**禁止编译器将该构造函数或转换运算符用于隐式类型转换和拷贝初始化。**

---

## 三、理解 `explicit` 的前提：隐式转换的两个方向

这是理解一切的关键。C++ 中用户自定义类型的隐式转换有**两个完全独立的方向**：

| 方向                  | 机制                                         | 写在哪个类里 | 例子              |
| --------------------- | -------------------------------------------- | ------------ | ----------------- |
| **其他类型 → 类类型** | 转换构造函数（非 explicit 的单参数构造函数） | 目标类里     | `Foo(int)`        |
| **类类型 → 其他类型** | 转换运算符（`operator T()`）                 | 源类里       | `operator bool()` |

### 3.1 方向一：其他类型 → 类类型

```cpp
class Foo {
public:
    Foo(int foo) : m_foo(foo) { }   // 转换构造函数
    int GetFoo() { return m_foo; }
private:
    int m_foo;
};

void DoBar(Foo foo) { ... }

int main() {
    DoBar(42);   // 42 被隐式转换为 Foo
}
```

**逐步分析：**

1. `DoBar` 期望参数类型是 `Foo`，你传的实参是 `int`
2. 编译器要问："能不能把 `int` 变成 `Foo`？"——方向是 **int → Foo**
3. 在 `Foo` 中找到 `Foo(int)`，它不是 `explicit`，可以用作隐式转换
4. 编译器把 `DoBar(42)` 改写成 `DoBar(Foo(42))`

**所以：`42` 隐式转换为 `Foo`。**

### 3.2 方向二：类类型 → 其他类型

```cpp
class Boolean {
    bool value;
public:
    Boolean(bool v) : value(v) { }
    operator bool() const { return value; }   // 转换运算符
};

Boolean b(true);
if (b) { }          // Boolean 被隐式转换为 bool
bool x = b;         // Boolean 被隐式转换为 bool
```

**逐步分析：**

1. `if` 需要一个可判定真假的值，`b` 的类型是 `Boolean`
2. 编译器要问："能不能把 `Boolean` 变成 `bool`？"——方向是 **Boolean → bool**
3. 在 `Boolean` 中找到 `operator bool()`，它不是 `explicit`
4. 编译器把 `if (b)` 改写成 `if (b.operator bool())`

**所以：`Boolean` 隐式转换为 `bool`。**

### 3.3 两个方向的对比

```
DoBar(42):     int  ──隐式转换──►  Foo     （由 Foo(int) 提供，写在目标类）
if (b):        Boolean  ──隐式转换──►  bool  （由 operator bool() 提供，写在源类）
```

**两个例子方向完全相反！** 它们由完全不同的机制控制，写在不同的类里。`GetFoo()` 只是普通成员函数，不参与隐式转换。

### 3.4 一个类可以同时拥有两个方向

```cpp
class Foo {
public:
    Foo(int x);              // 方向一：int → Foo
    operator int() const;    // 方向二：Foo → int
};

Foo f = 42;   // int → Foo
int x = f;    // Foo → int
```

`explicit` 可以分别应用到两个方向上，独立控制。

---

## 四、`explicit` 怎么用？

### 4.1 基本语法

```cpp
class MyClass {
public:
    explicit MyClass(int x);   // explicit 构造函数
    explicit operator bool() const;  // explicit 转换运算符（C++11）
};
```

### 4.2 只能写在类内部的声明处

```cpp
class MyClass {
public:
    explicit MyClass(int x);  // ✅ 类内部声明
};

// MyClass::explicit MyClass(int x) { }  // ❌ 错误！类外不能写 explicit
MyClass::MyClass(int x) { }              // ✅ 类外定义不写 explicit
```

### 4.3 四种初始化方式的完整对照

| 语法            | 名称           | 允许 explicit？ |
| --------------- | -------------- | --------------- |
| `Foo f(42);`    | 直接初始化     | ✅ 允许          |
| `Foo f{42};`    | 直接列表初始化 | ✅ 允许          |
| `Foo f = 42;`   | 拷贝初始化     | ❌ 禁止          |
| `Foo f = {42};` | 拷贝列表初始化 | ❌ 禁止          |

**核心规则：`explicit` 只禁止拷贝初始化（隐式转换），不禁止直接初始化。**

### 4.4 `explicit` 后的合法与非法写法

```cpp
class Foo {
public:
    explicit Foo(int x);
};

Foo f1(42);                      // ✅ 直接初始化
Foo f2{42};                      // ✅ 直接列表初始化
Foo f3 = 42;                     // ❌ 拷贝初始化，禁止
Foo f4 = {42};                   // ❌ 拷贝列表初始化，禁止
Foo f5 = Foo(42);                // ✅ 显式构造临时对象再拷贝
Foo f6 = static_cast<Foo>(42);   // ✅ 显式转换
```

---

## 五、`explicit` 的完整使用规则

### 5.1 单参数构造函数（最常见）

任何可以用**单个参数**调用的构造函数——包括只有一个参数、或第一个参数之后所有参数都有默认值的构造函数——都应该考虑加上 `explicit`。

```cpp
class MyString {
public:
    explicit MyString(int size);   // 防止 print(3) 意外构造字符串
};

class Array {
public:
    explicit Array(int size);      // 防止 processArray(10) 意外构造数组
};
```

### 5.2 多参数构造函数（C++11 起有意义）

**C++11 之前：** `explicit` 对多参数构造函数无意义，因为多参数构造函数本来就不能参与隐式转换。

**C++11 起：** 列表初始化 `{...}` 让多参数构造函数第一次获得了参与隐式转换的能力，所以 `explicit` 变得有意义。

```cpp
class Point {
public:
    explicit Point(int x, int y);
};

void f(Point p);

f({1, 2});          // ❌ 拷贝列表初始化，explicit 被排除
f(Point{1, 2});     // ✅ 直接列表初始化临时对象
f(Point(1, 2));     // ✅ 显式构造临时对象
```

### 5.3 转换运算符（C++11 起）

```cpp
class Boolean {
    bool value;
public:
    Boolean(bool v) : value(v) { }
    explicit operator bool() const { return value; }
};

Boolean b(true);
if (b) { }                       // ✅ 布尔语境特例，仍然允许
bool x = b;                      // ❌ 拷贝初始化，禁止
bool y = static_cast<bool>(b);   // ✅ 显式转换
bool z(b);                       // ✅ 直接初始化
```

**关键特例：** 即使 `operator bool()` 是 `explicit` 的，在**布尔语境**（`if`、`while`、`for`、`!`、`&&`、`||`、三元运算符条件）中仍然允许使用。这个设计让智能指针等类型既能安全地在 `if (ptr)` 中使用，又不会在 `bool x = ptr;` 中被意外转换。

`std::unique_ptr`、`std::shared_ptr`、`std::optional` 等标准库类型都使用 `explicit operator bool()`。

### 5.4 拷贝构造函数也可以 explicit

```cpp
class C {
public:
    explicit C(const C&) { }   // 罕见的用法
};

C f(C c) {
    return c;   // ❌ 错误！return 需要拷贝初始化，但拷贝构造函数是 explicit
}
```

实际中很少使用，但在需要严格控制对象复制的场景下可能有用。

---

## 六、`explicit` 与 `{...}` 列表初始化的关系

### 6.1 `{1, 2}` 本身没有类型

`{1, 2}` 是**花括号初始化列表**（braced-init-list），**不是表达式**，因此**没有类型**。它只有在初始化上下文中才有意义，含义完全取决于目标类型。

```cpp
int arr[] = {1, 2};              // 数组初始化器
std::vector<int> v = {1, 2};     // 初始化 initializer_list
std::pair<int,int> p = {1, 2};   // 分别初始化 first 和 second
Point pt = {1, 2};               // 调用 Point(int, int)
auto il = {1, 2};                // 推导为 initializer_list<int>
```

### 6.2 列表初始化的优势

1. **消除"最令人烦恼的解析"**：`Foo f();` 是函数声明，`Foo f{};` 才明确是对象创建
2. **禁止窄化转换**：`int x{3.14};` 编译错误，`int x = 3.14;` 只警告
3. **统一聚合体和类的初始化语法**

### 6.3 著名的 `vector` 陷阱

```cpp
std::vector<int> v1(10);      // 10 个元素，每个都是 0
std::vector<int> v2{10};      // 1 个元素，值是 10
std::vector<int> v3{10, 20};  // 2 个元素：10 和 20
```

**原因：** 花括号会优先匹配 `initializer_list` 构造函数。这是列表初始化的一个副作用。

---

## 七、常见使用场景（什么时候用 `explicit`？）

### 7.1 强烈建议使用 `explicit`

1. **接受数值参数的构造函数**：`MyString(int size)`、`Array(int size)` 等
2. **接受指针的构造函数**：`SmartPtr(T* ptr)`，防止裸指针隐式变智能指针
3. **转换运算符**：`operator bool()`、`operator int()` 等
4. **C++11 多参数构造函数**：因为列表初始化允许多参数构造函数参与隐式转换
5. **所有单参数构造函数，除非你明确需要隐式转换**——这是 C++ 社区广泛认可的最佳实践

### 7.2 可以不加 `explicit` 的场景

**当隐式转换是设计意图的一部分时：**

```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
};

Rational r = 5;        // 5 → Rational(5, 1)，符合直觉
Rational sum = r + 3;  // 3 → Rational(3, 1)，符合直觉
```

这里让整数隐式转换为有理数是有意义的、符合直觉的。

### 7.3 为什么标准库的 `vector` 的 `size` 构造函数是 `explicit` 的

```cpp
std::vector<int> v2 = 10;   // ❌ 现代 C++ 标准里会直接报错
```

因为标准委员会把它标记为 `explicit`，防止这种"逻辑上完全不对"的代码悄悄通过编译。

---

## 八、常见误区与注意事项

### 误区一：`explicit` 会影响直接初始化

**错误。** `explicit` 只禁止隐式转换和拷贝初始化。直接初始化（`Foo f(42)` 或 `Foo f{42}`）始终允许。

### 误区二：`explicit` 只能用于单参数构造函数

**错误（C++11 起）。** 多参数构造函数和转换运算符都可以使用 `explicit`。

### 误区三：加了 `explicit` 就无法进行类型转换了

**错误。** 仍然可以使用显式转换（`static_cast`、函数式转换 `Foo(42)`、`Foo{42}`）。

### 误区四：`explicit` 写在类外定义处

**错误。** `explicit` 只能写在类内部的声明处，类外定义时不能重复写。

### 误区五：`explicit` 会阻止所有隐式转换

**不完全正确。** 对于 `explicit operator bool()`，在布尔语境中仍然允许隐式使用。

### 注意事项一：`explicit` 只控制"用哪个构造函数"，不控制"能不能构造"

它的作用是**禁止隐式转换路径**，而不是禁止构造对象本身。

### 注意事项二：拷贝构造函数也能被 `explicit` 修饰

这会导致 `return c;` 这类需要拷贝初始化的操作失败。

### 注意事项三：`explicit` 与重载决议的交互

隐式转换会让重载决议变得极其复杂：

```cpp
void f(Array arr);
void f(int n);

f(10);   // 如果有隐式转换，两个都可行 → 歧义错误！
```

### 注意事项四：真实世界的 bug 不长成 `processArray(10)` 这样

真正危险的是这些场景：

```cpp
// 场景 A：变量名掩盖类型
int count = getElementCount();
processArray(count);   // 你以为是"传个数"，实际构造了 Array

// 场景 B：模板代码里的隐式转换
std::vector<Array> v;
v.push_back(10);       // 看起来像塞整数，实际塞的是 Array

// 场景 C：运算符重载被意外触发
Complex d = c + 3.5;   // 3.5 被隐式转换成 Complex(3.5, 0)
```

隐式转换的危险不在于"程序员明知故犯地写错类型"，而在于：
1. 编译器不再帮你检查类型
2. 代码意图被隐藏
3. 重构时的定时炸弹
4. 重载决议、模板实例化、运算符匹配被污染

---

## 九、`explicit` 的原理

### 9.1 编译器层面的机制

`explicit` 不是一个运行时机制，而是**编译期的语义约束**。编译器在以下场景中会检查 `explicit`：

1. **拷贝初始化**：`Foo f = expr;` — 不考虑 explicit 构造函数
2. **函数实参传递**：`f(expr)` 其中参数类型是 `Foo` — 属于拷贝初始化，不考虑 explicit
3. **函数返回值**：`return expr;` — 属于拷贝初始化，不考虑 explicit
4. **列表拷贝初始化**：`Foo f = {expr};` — 不考虑 explicit

编译器在重载决议的第一阶段就会**排除所有 explicit 构造函数**（当处于拷贝初始化上下文时），所以它们根本不会进入候选集。

### 9.2 与 `explicit` 相关的标准条款

- **C++98**：`explicit` 仅适用于构造函数，且仅对单参数构造函数有意义
- **C++11**：扩展到转换运算符和多参数构造函数（因列表初始化）
- **C++17**：扩展到推导指引
- **C++20**：条件 explicit（`explicit(bool)`），允许根据编译期条件决定是否 explicit

### 9.3 `explicit(bool)`（C++20）

```cpp
template<typename T>
class Wrapper {
public:
    explicit(std::is_integral_v<T>) Wrapper(T value);
};
```

当 `T` 是整型时，构造函数是 `explicit` 的；否则不是。这让泛型代码可以精细控制隐式转换。

---

## 十、用户的疑问及解答

### 疑问 1：这些例子是不是把调用者写得太蠢了？

**解答：** 你的质疑合理，但有一个关键误解——以为"类型不匹配会阻止编译"。**隐式转换恰恰让编译器闭嘴了**，本该报错的地方被静默修复。真实世界的 bug 不长成 `processArray(10)` 这样，而是隐藏在变量名、模板代码、重载决议中。`std::vector<int> v2 = 10;` 逻辑上完全不对，**这正是为什么现代标准里 `vector` 的 `size` 构造函数是 `explicit` 的**——你的直觉是对的，`explicit` 就是把你的直觉强加给编译器。

### 疑问 2：`{1, 2}` 在 C++ 里本身是什么类型？

**解答：** **它没有类型。** `{1, 2}` 是花括号初始化列表（braced-init-list），**不是表达式**，不能独立存在，只在初始化上下文中才有意义，含义完全取决于目标类型。`f({1, 2})` 中，编译器看到参数类型是 `Point`，触发列表初始化，查找 `Point` 中可行的构造函数，找到 `Point(int, int)`。但因为函数实参属于**拷贝初始化**，`explicit` 构造函数被排除。

### 疑问 3：`if (b)` 和 `bool x = b;` 的隐式转换是怎么实现的？

**解答：** 通过 `operator bool()` 转换运算符实现。

- `if (b)`：`if` 要求布尔语境，编译器查找从 `Boolean` 到 `bool` 的隐式转换，找到 `operator bool()`，改写成 `if (b.operator bool())`
- `bool x = b;`：拷贝初始化，编译器同样找到 `operator bool()`，改写成 `bool x = b.operator bool();`
- 加上 `explicit` 后：`if (b)` **仍然 OK**（布尔语境特例），`bool x = b;` **报错**

### 疑问 4：`Foo f{42};` 是什么语法？

**解答：** 这是 **C++11 的直接列表初始化**。设计目的是统一初始化语法、禁止窄化转换、消除"最令人烦恼的解析"。它与 `explicit` 的交互规则是：**直接列表初始化允许 `explicit`，拷贝列表初始化不允许**。著名陷阱：`vector<int> v1(10)` 是 10 个元素，`vector<int> v2{10}` 是 1 个元素值为 10，因为花括号优先匹配 `initializer_list` 构造函数。

### 疑问 5：`DoBar(42)` 中，到底是 42 隐式转换为 Foo，还是 Foo 隐式转换为 int？

**解答：** **`42` 隐式转换为 `Foo`。** 因为：
1. `DoBar` 期望参数类型是 `Foo`，你传的是 `int`
2. `Foo` 有非 explicit 的 `Foo(int)` 转换构造函数
3. 所以 `int → Foo` 的转换成立
4. `Foo → int` **不可能发生**，因为 `Foo` 里根本没有 `operator int()`，`GetFoo()` 只是普通函数

**你的困惑源于把两个相反方向的隐式转换混为一谈**：`DoBar(42)` 是"普通类型 → 类类型"（由转换构造函数提供，写在目标类），`if (b)` 是"类类型 → 普通类型"（由转换运算符提供，写在源类）。它们是两个完全独立、方向相反的机制。

---

## 十一、你没提到但很重要的补充

### 11.1 `explicit` 与 `std::initializer_list` 的交互

当一个类同时有 `initializer_list` 构造函数和普通构造函数时，花括号会**优先匹配 `initializer_list`**：

```cpp
class Widget {
public:
    Widget(int x, int y);                          // 普通构造函数
    Widget(std::initializer_list<int> il);         // initializer_list 构造函数
};

Widget w1(10, 20);   // 调用 Widget(int, int)
Widget w2{10, 20};   // 调用 Widget(initializer_list<int>)！
```

即使 `initializer_list` 构造函数是 `explicit` 的，直接列表初始化仍然优先匹配它。

### 11.2 `explicit` 与完美转发

在泛型代码中，`explicit` 会影响 `std::is_convertible` 和 `std::is_constructible` 的判断：

```cpp
std::is_convertible<int, Foo>::value;      // 如果 Foo(int) 是 explicit，为 false
std::is_constructible<Foo, int>::value;    // 无论是否 explicit，都为 true
```

`is_convertible` 检查的是**隐式转换**能力，`is_constructible` 检查的是**能否构造**。这个区别在模板元编程中非常重要。

### 11.3 `explicit` 与 `std::function`、`std::tuple` 等标准库类型

很多标准库类型使用 `explicit` 构造函数来防止意外的隐式转换：

```cpp
std::optional<int> opt = 42;   // ✅ OK：optional 有非 explicit 的转换构造函数
std::optional<int> opt2 = {42}; // ✅ OK
```

但 `std::tuple` 的某些构造函数是 `explicit` 的，防止 `std::tuple<int> t = 42;` 这样的代码。

### 11.4 `explicit` 与隐式转换的优先级

当同时存在多个可行的隐式转换路径时，编译器会选择**最佳匹配**。`explicit` 的作用是在重载决议的第一阶段就排除 explicit 构造函数，让它们根本不进入候选集。

### 11.5 `explicit` 与 `operator T()` 的对称性

```cpp
class Foo {
public:
    explicit Foo(int);              // 禁止 int → Foo 的隐式转换
    explicit operator int() const;  // 禁止 Foo → int 的隐式转换
};

Foo f(42);              // ✅ 直接初始化
// Foo f2 = 42;         // ❌ 拷贝初始化
int x = static_cast<int>(f);  // ✅ 显式转换
// int y = f;           // ❌ 隐式转换
```

两个方向的 `explicit` 是**独立**的，可以只加一个方向。

### 11.6 为什么 `explicit operator bool()` 在 `if` 中仍然有效

这是 C++ 标准的一个特殊规定。**布尔语境**（contextual conversion to bool）包括：

- `if (expr)`、`while (expr)`、`for (; expr; )`
- `!expr`、`expr && expr`、`expr || expr`
- `expr ? a : b` 的条件部分

在这些语境中，即使 `operator bool()` 是 `explicit` 的，也会被自动调用。这个设计是为了让智能指针等类型安全地用于条件判断，同时防止它们被意外转换为 `bool` 变量或参与算术运算。

### 11.7 `explicit` 与 C++20 的 `explicit(bool)`

```cpp
template<typename T>
class Optional {
public:
    explicit(std::is_trivially_copyable_v<T>) Optional(const T& value);
};
```

这允许根据编译期条件决定构造函数是否 `explicit`，在泛型库设计中非常有用。

### 11.8 最佳实践总结

1. **默认给所有单参数构造函数加 `explicit`**，除非有充分理由允许隐式转换
2. **给所有转换运算符加 `explicit`**，尤其是 `operator bool()`
3. **C++11 起，多参数构造函数也应考虑 `explicit`**
4. **隐式转换是设计意图的一部分时才不加 `explicit`**，如 `Rational(int, int)`
5. **优先使用直接初始化 `Foo f{42}`**，避免"最令人烦恼的解析"
6. **注意 `initializer_list` 构造函数的优先级陷阱**
7. **在泛型代码中用 `std::is_convertible` 和 `std::is_constructible` 检查 explicit 的影响**

---

## 十二、一句话总结

> **`explicit` 是 C++ 提供的安全机制，把"是否允许隐式转换"的决定权从编译器手里夺回来交还给程序员。它控制两个方向的隐式转换（转换构造函数：普通类型 → 类类型；转换运算符：类类型 → 普通类型），只禁止拷贝初始化，不禁止直接初始化。除非你有充分理由允许隐式转换，否则给每个可以用单个参数调用的构造函数都加上 `explicit`。**