# C++ 函数指针与回调详细笔记（与 C 语言逐块对照）

> **一句话总纲（C++ 版）** ：C++ 继承了 C 的函数指针，但现代 C++ 的默认答案已经不是“裸函数指针 + `void*`”，而是 **Lambda 用于就地定义，`std::function` 用于类型擦除存储，模板参数用于零开销泛型**。函数指针退居“与 C 互操作”和“极致性能”两个角落。
>
> 记忆口诀：**C 用指针加 void*，C++ 用闭包和 function；有捕获的 Lambda 不是函数指针，成员函数指针需要 this；热路径用模板，冷路径用 function。**


## 一、C++ 中“函数指针”是什么（继承与差异）

### 1.1 C++ 直接继承了 C 的函数指针语法

C++ 中普通自由函数的指针与 C 几乎完全一致：

```cpp
int add(int a, int b) { return a + b; }
int (*fp)(int, int) = add;          // 与 C 相同
int r = fp(3, 4);                    // 8
```

`add` 在表达式中退化为 `int (*)(int, int)`，`&add` 与 `add` 等价，`(*fp)(3,4)` 与 `fp(3,4)` 等价——这些规则 C++ 全部沿用。

### 1.2 但 C++ 多出了几类“不能直接塞进函数指针”的实体

这是 C++ 与 C 最根本的分水岭：

| 实体                                 | C 中是否存在 | 能否存入普通函数指针 `int(*)(int)`      |
| ------------------------------------ | ------------ | --------------------------------------- |
| 自由函数 / 静态成员函数              | 是 / —       | **能**                                  |
| 非静态成员函数                       | —            | **不能**（类型不同，见第十二节）        |
| 有捕获的 Lambda                      | —            | **不能**（无隐式转换）                  |
| 有捕获的 Lambda（无捕获）            | —            | **能**（C++11 起的隐式转换）            |
| 函数对象（重载了 `operator()` 的类） | —            | 不能直接存，但可被 `std::function` 容纳 |

**关键结论**：在 C++ 中，“函数指针”只覆盖了可调用实体的一小部分。C 的回调范式（函数指针 + `void*`）在 C++ 中仍然可用，但它不再是表达能力足够的抽象。

### 1.3 C++ 的“可调用对象”概念

C++ 标准把满足 `f(args...)` 合法的一切实体统称为 **可调用对象**，包括：自由函数、函数指针、Lambda、函数对象、成员函数指针（配合对象或 `std::invoke`）、`std::bind` 表达式。C 只有“函数指针”这一种，因此 C++ 的回调设计首先要回答的问题是：  **用哪个类型来承载“任意可调用对象”？**


## 二、由来与为什么：C++ 为什么不再以裸函数指针为默认

### 2.1 C 回调范式的根本缺陷

C 的回调必须写成：

```c
typedef void (*cb_t)(void *ctx, int event);
void register_cb(cb_t cb, void *ctx);
```

它的表达能力被两件事锁死：

1. **上下文只能靠 `void*` 手工传递**，类型不安全，每一次回调内部都要 `reinterpret_cast`。
2. **生命周期完全靠调用者手工管理**，库无法知道 `ctx` 何时失效，也无法在回调对象销毁时自动清理。

C++ 的演进目标是：**让回调既携带状态，又保持类型安全，并且尽可能把生命周期管理交给语言机制。**

### 2.2 C++ 给出的三层替代方案

C++ 不是用一个特性取代函数指针，而是提供了一条“从零开销到最大灵活”的光谱：

| 机制                           | 引入版本     | 核心能力                                        | 开销                                  |
| ------------------------------ | ------------ | ----------------------------------------------- | ------------------------------------- |
| **无捕获 Lambda → 函数指针**   | C++11        | 就地定义，可传给 C API                          | 零开销（就是函数指针）                |
| **模板参数接受任意可调用对象** | C++11 起可用 | 保留原始类型，可内联                            | 零运行时开销，但可能代码膨胀          |
| **`std::function`**            | C++11        | 类型擦除，统一存储任意可调用对象                | 一次间接调用 + 可能的堆分配           |
| **`std::move_only_function`**  | C++23        | 支持仅可移动的可调用对象（如捕获 `unique_ptr`） | 与 `std::function` 类似，但无拷贝要求 |

**选型原则**：能模板就模板（零开销）；需要存储/延迟调用就用 `std::function`；必须对接 C API 时才退回“无捕获 Lambda + 函数指针”；可调用对象不可拷贝时用 `std::move_only_function`。


## 三、原理：C++ 回调机制的底层实现

### 3.1 `std::function` 的类型擦除与虚表分派

`std::function<R(Args...)>` 内部并不是“一个函数指针”，而是一个**类型擦除的多态包装器**。概念结构：

```text
std::function<int(int)> f = [x](int y){ return x + y; };

┌──────────────────────────────┐
│ std::function 对象            │
│  ┌────────────────────────┐  │
│  │ 小对象缓冲区 (SBO)      │  │ ← 小闭包直接内联存储
│  │ （通常 16–48 字节）     │  │
│  └────────────────────────┘  │
│  vtable* → invoke / copy /  │ ← 类型擦除的调用接口
│           destroy            │
│  ┌────────────────────────┐  │
│  │ 堆指针 → 大闭包         │  │ ← 大闭包堆分配
│  └────────────────────────┘  │
└──────────────────────────────┘
```

**每一次 `std::function` 调用都经过一次间接分派**，即使被存储的对象很小、即使启用了小对象优化（SBO），虚表指针这一层分派也无法消除。SBO 只解决“是否堆分配”，不解决“是否间接调用”。

不同实现的 SBO 缓冲区大小不同：libstdc++ 约 16 字节，libc++ 约 24 字节，MSVC 在调试模式下可达 48 字节。超过这个大小的闭包（例如捕获了一个大 `std::array` 或多个变量）就会触发堆分配。

### 3.2 Lambda 的本质：编译器生成的函数对象

Lambda **不是函数**，而是一个**匿名类**（闭包类型）的实例，该类重载了 `operator()`：

```cpp
auto lam = [x](int y) { return x + y; };
// 编译器近似生成：
struct __lambda_xyz {
    int x;                          // 捕获的变量成为成员
    int operator()(int y) const { return x + y; }
};
```

**捕获变量 = 类的数据成员**。这解释了为什么：

- 无捕获 Lambda 的大小通常为 1 字节，可以退化为函数指针（不需要 `this`，没有状态）。
- 有捕获 Lambda **不能**转换为函数指针：函数指针只是代码地址，没有地方存放捕获的数据。
- Lambda 可以传给模板参数，编译器能看到闭包的具体类型，从而**内联 `operator()`**，实现零开销。

### 3.3 为什么有捕获的 Lambda 不能转函数指针（C++ 标准硬约束）

C++ 标准规定：**只有不带捕获的非泛型 Lambda**，其闭包类型才具有到函数指针的转换函数。原因很直接：

> “函数指针就是指向某些代码位的指针。无数据。带捕获的 lambda 还需要捕获的数据，以便提供有用的结果。这是不可能自动工作的。”

**应对策略**：

- 如果目标 API 只接受 C 函数指针，**只能使用无捕获 Lambda**，或者写一个无捕获的包装函数（但包装函数仍需要某种全局/静态途径拿到状态，不推荐用于可重入场景）。
- 如果目标 API 是 C++ 接口，**不要设计成接受裸函数指针**，应接受模板参数或 `std::function`，这样调用方就可以自由使用有捕获的 Lambda。


## 四、怎么用：C++ 中回调的四种主要写法

### 4.1 无捕获 Lambda 直接传给 C 风格函数指针参数

这是与 C API 对接时的标准做法：

```cpp
#include <cstdlib>

int data[] = {3, 1, 4, 2};
std::qsort(data, 4, sizeof(int),
    [](const void *a, const void *b) -> int {   // 无捕获 → 隐式转函数指针
        int x = *static_cast<const int*>(a);
        int y = *static_cast<const int*>(b);
        return (x > y) - (x < y);
    });
```

**注意**：如果写成 `[&]` 或 `[=]`，编译会直接失败，因为 qsort 要求的是 `int(*)(const void*, const void*)`。这是 C++ 中“Lambda 与函数指针”边界最常踩的坑。

### 4.2 `std::function` 作为回调类型（现代 C++ 的“默认存储形式”）

```cpp
#include <functional>

class EventSource {
    std::function<void(int)> cb_;
public:
    void register_cb(std::function<void(int)> cb) {
        cb_ = std::move(cb);
    }
    void fire(int event) {
        if (cb_) cb_(event);   // 空 function 调用会抛 std::bad_function_call
    }
};

int main() {
    EventSource src;
    int counter = 0;
    src.register_cb([&counter](int event) {   // 有捕获，完全合法
        counter += event;
    });
    src.fire(5);
}
```

与 C 的 `void(*)(void*, int)` 相比，**上下文不再需要手工 `void*`**，捕获的变量由闭包类型自动持有，类型由编译器保证。这就是 `std::function` 最核心的价值： “它允许用户提供回调所需的任何上下文；普通函数指针不提供。” 

### 4.3 模板参数：零开销的回调接口

如果回调的生命周期仅限于调用期间、不需要存储，**模板是比 `std::function` 更好的选择**：

```cpp
template <typename F>
void for_each(int *arr, size_t n, F&& f) {
    for (size_t i = 0; i < n; ++i) f(arr[i]);
}

for_each(arr, n, [&](int x) { sum += x; });  // 无类型擦除，可内联
```

**代价**：每个不同的可调用类型都会实例化一份 `for_each`，可能导致代码膨胀；并且**无法把 `F` 存进一个统一类型的成员变量**（除非再用 `std::function` 包装一次）。

### 4.4 C++23 `std::move_only_function`：当闭包不可拷贝时

`std::function` 要求存储的可调用对象**可拷贝**。如果 Lambda 捕获了 `std::unique_ptr`，它只能移动、不能拷贝，`std::function` 无法容纳。C++23 的 `std::move_only_function` 移除了拷贝要求，只要求可移动构造：

```cpp
#include <functional>
#include <memory>

std::move_only_function<void()> f =
    [p = std::make_unique<int>(42)]() { /* 使用 *p */ };
// f 只能 std::move，不能拷贝
```

注意：**调用空的 `std::move_only_function` 是未定义行为**，而调用空的 `std::function` 会抛出 `std::bad_function_call`，两者语义不同。


## 五、C++ 特有：成员函数指针与回调

### 5.1 非静态成员函数指针的类型与 C 函数指针完全不同

```cpp
struct Fred { int f(char, float); };

int (*pf)(char, float);            // 普通函数指针
int (Fred::*pmf)(char, float);     // 成员函数指针：多了一个 Fred::*
```

**非静态成员函数有一个隐藏的 `this` 参数**。它的调用必须绑定一个 `Fred` 对象：

```cpp
Fred obj;
(obj.*pmf)(...)                    // 或 (obj_ptr->*pmf)(...)
```

C 的函数指针没有地方放 `this`，所以**非静态成员函数指针不能直接当作 C 回调使用**。

### 5.2 静态成员函数可以当作普通函数指针

静态成员函数没有 `this` 参数，类型与自由函数一致：

```cpp
struct Widget {
    static void on_event(int e);
};
void (*fp)(int) = &Widget::on_event;   // 合法，与普通函数指针兼容
```

因此，**如果必须把类成员暴露为 C 风格回调，写成静态成员函数是正确路径**；状态通过 C 回调的 `void*` 参数传入。

### 5.3 现代 C++ 中调用成员函数指针：`std::invoke` / `std::function` / Lambda

**不要手工写 `(obj.*pmf)(...)` 的分派逻辑**。现代写法：

```cpp
#include <functional>
std::function<void(int)> f = std::bind(&Fred::f, &obj, std::placeholders::_1);
// 或更推荐：
std::function<void(int)> g = [&obj](int x) { obj.f(x); };
```

`std::function` 可以直接存储成员函数指针，但**调用时仍然需要绑定对象**；Lambda 或 `std::bind` 负责完成“对象 + 成员函数”的绑定。日常代码中，**Lambda 是首选**，因为可读性远好于 `std::bind`（见第十节）。


## 六、C++ 常见用法与代码示例

### 6.1 `std::sort` 比较器：C++ 相对于 `qsort` 的质变

```cpp
#include <algorithm>
#include <vector>

std::vector<int> v = {3, 1, 4, 2};
std::sort(v.begin(), v.end(),
    [](int a, int b) { return a > b; });   // 降序，有/无捕获均可
```

**与 C 的 `qsort` 的本质差别**：

- `std::sort` 是**模板**，比较器的实际类型是闭包类型，编译器**可以看到并内联 `operator()`**，每次比较没有函数指针的间接调用开销。
- `qsort` 接受的是函数指针，**每一次比较都是一次通过指针的间接调用**，内联通常无法发生。实测和文献中常见 `std::sort` 比 `qsort` 快数倍，根本原因就在这里。
- `std::sort` 的比较器返回 `bool`（严格弱序），`qsort` 返回 `int`（负/零/正），语义不同，不要混用。

**C++ 中除非强制要求与 C 接口保持一致，否则不应使用 `qsort`。**

### 6.2 事件/回调系统：`std::function` 的标准形态

```cpp
class Button {
    std::function<void()> on_click_;
public:
    void set_on_click(std::function<void()> cb) {
        on_click_ = std::move(cb);
    }
    void click() {
        if (on_click_) on_click_();
    }
};

Button btn;
int count = 0;
btn.set_on_click([&count] { ++count; });
btn.click();
```

与 C 的 `void(*)(void*)` 对比：**C++ 版本不需要调用方显式传 `void*`，捕获的 `count` 由闭包自动携带；回调类型在编译期被检查，不会出现 `void*` 强转错误。**

### 6.3 命令分派表：`std::unordered_map` + `std::function`

```cpp
#include <functional>
#include <string>
#include <unordered_map>

std::unordered_map<std::string, std::function<void()>> cmds;
cmds["help"] = [] { /* ... */ };
cmds["quit"] = [] { /* ... */ };

if (auto it = cmds.find(name); it != cmds.end())
    it->second();
```

C 版本通常用“字符串 + 函数指针数组”线性查找；C++ 版本可以把有状态的对象、成员函数绑定、Lambda 全部放进同一个映射，表达能力和可维护性高出一个量级。

### 6.4 驱动/策略对象：函数对象（Functor）

```cpp
struct HashPolicy {
    size_t operator()(const std::string& s) const { /* ... */ }
};
std::unordered_set<std::string, HashPolicy> set;
```

函数对象是“可调用类型”而非“函数指针”，**可以被模板内联**，同时可以携带状态。C 只能通过“函数指针 + 全局变量/`void*`”模拟，既不安全也不可组合。

### 6.5 与 C 线程/信号 API 对接

```cpp
extern "C" void c_thread_start(void (*fn)(void*), void* arg);

struct Task { int id; };

// 必须无捕获，才能转函数指针
c_thread_start([](void* p) {
    auto* t = static_cast<Task*>(p);
    // ...
}, &task);
```

这是 C++ 中**仍然必须使用“裸函数指针 + `void*`”** 的典型场景。此时 C++ 的优势仅限于 Lambda 让包装函数写得更紧凑，**上下文和生命周期管理的负担并没有消失**。


## 七、C++ 中的常见注意事项与重要事项

### 7.1 有捕获 Lambda ≠ 函数指针（最高频错误）

**规则**：只有无捕获的非泛型 Lambda 能隐式转换为函数指针。任何 `[&]`、`[=]`、`[x]`、`[this]` 都会切断这条转换路径。**如果目标接口是 `void(*)(...)`，只能传无捕获 Lambda 或自由函数/静态成员函数。**

### 7.2 `std::function` 的调用空指针语义与 C 不同

- C 中 `fp == NULL` 时调用是未定义行为，通常需要 `if (fp)` 保护。
- `std::function` 空对象可以直接用 `if (f)` 判断；**调用空的 `std::function` 会抛出 `std::bad_function_call`**，而不是静默崩溃。
- `std::move_only_function` 空调用是 **未定义行为**，没有异常保护。

### 7.3 `std::function` 的性能成本必须心里有数

`std::function` 的每次调用都要经过**类型擦除的间接分派**（通常是虚表） 。即使 SBO 避免了堆分配，**间接调用这一层成本始终存在**。

**性能光谱（从低到高）** ：

1. **模板参数 / 直接函数调用**：零开销，可内联。
2. **函数指针**：一次间接调用，通常不可内联。
3. **`std::function`（SBO）** ：间接分派 + 可能的虚表，不可内联。
4. **`std::function`（堆分配）** ：在上述基础上多一次堆分配/释放。

**实践原则**：UI 回调、事件处理、初始化代码中 `std::function` 的开销完全可以接受；**热路径（每帧执行、紧循环、高频交易）中应避免 `std::function`**，改用模板参数或裸函数指针。

### 7.4 捕获列表的生命周期陷阱（C++ 版比 C 更隐蔽）

```cpp
std::function<void()> f;
{
    int local = 42;
    f = [&local] { use(local); };   // 危险：捕获的是引用
}                                    // local 已销毁
f();                                 // 悬空引用，未定义行为
```

C 的 `void*` 陷阱是“指针悬空”，C++ 的捕获陷阱是“引用/指针悬空”，**本质相同，但语法更隐蔽**。规则：**异步或延迟调用的回调，捕获必须用值捕获或 `shared_ptr`，禁止用引用捕获局部变量。**

### 7.5 `std::bind` 在现代 C++ 中已不推荐

`std::bind` **在标准中从未被废弃**，C++20/C++23 仍然有效。但主流工程共识是 **“能用 Lambda 就不用 `bind`”** ，原因是：

- **可读性差**：多层嵌套或占位符 `_1`、`_2` 混用时，代码难以理解。
- **`std::bind` 对成员函数的绑定语义容易出错**，且无法像 Lambda 一样直接内联。
- Abseil 等工业级代码库明确建议避免 `std::bind`，改用 Lambda 或 `std::bind_front`。

**唯一仍有一定存在感的场景**：需要把参数预绑定后生成一个**函数指针**去对接 C API（Lambda 有捕获时不能转函数指针，而无捕获 Lambda 无法携带绑定参数；但即便如此，写一个无捕获的包装函数通常更清晰）。

### 7.6 不要用函数指针的思维设计 C++ 回调接口

如果一个 C++ 接口写成：

```cpp
void register_cb(void (*cb)(void*), void* ctx);   // 这是 C 风格
```

它**强迫调用方回到 C 的手工 `void*` 模式**，放弃了 C++ 的闭包和类型安全。现代 C++ 接口应当二选一：

- **热路径/泛型**：`template <typename F> void register_cb(F&& f);`
- **需要存储/类型擦除**：`void register_cb(std::function<void()> cb);`

只有在**必须保持 C ABI 兼容**时，才使用 `void(*)(void*) + void*`。

### 7.7 成员函数指针不是函数指针，不要强转

`int (Fred::*)(int)` 与 `int (*)(Fred*, int)` 在大多数实现上“看起来”可以强转，但这是**未定义行为**，并且完全不可移植。正确做法是用 Lambda 或 `std::function` 做适配，而不是用 `reinterpret_cast` 糊弄类型系统。

### 7.8 `std::function` 的拷贝可能很昂贵

拷贝一个 `std::function` 会拷贝其底层的可调用对象。如果闭包捕获了大容器，拷贝成本很高。**传递 `std::function` 时优先 `const std::function&` 或按值 `std::move`**，避免不必要的深拷贝。

### 7.9 空 `std::function` 的判断与调用

```cpp
std::function<void()> f;
if (f) f();          // 正确：先判断
// f();              // 抛 std::bad_function_call
```

与 C 一样，**注册前可能为空、注销后可能为空**，接口设计时应明确“空回调”的语义：是忽略、是错误、还是用默认行为。


## 八、C++ 与 C 回调的逐项对比总结

| 维度                | C                                    | C++（现代）                                                  |
| ------------------- | ------------------------------------ | ------------------------------------------------------------ |
| **默认回调载体**    | 函数指针 + `void*`                   | Lambda（就地）+ `std::function`（存储）/ 模板（热路径）      |
| **携带上下文**      | 手工 `void*`，类型不安全             | 闭包捕获，编译期类型检查                                     |
| **有状态回调**      | 函数指针 + 全局/`void*`              | 有捕获 Lambda / 函数对象，天然携带状态                       |
| **生命周期**        | 手工管理，库无法感知                 | 闭包值语义，对象析构时自动清理捕获的资源；引用捕获仍需手工注意 |
| **与 C API 互操作** | 原生                                 | 无捕获 Lambda 可隐式转函数指针；有捕获 Lambda **不能**       |
| **成员函数作回调**  | 不存在                               | 非静态成员函数指针需绑定对象；静态成员函数可退化为普通函数指针 |
| **性能**            | 函数指针：一次间接调用，通常不可内联 | 模板：零开销可内联；`std::function`：间接分派，通常不可内联  |
| **排序比较器**      | `qsort` + 函数指针，难内联           | `std::sort` + Lambda/函数对象，**可内联，通常显著更快**      |
| **空回调**          | `NULL` 检查                          | `std::function` 空调用抛异常；`std::move_only_function` 空调用 UB |
| **仅可移动闭包**    | 不适用                               | C++23 `std::move_only_function` 支持（如捕获 `unique_ptr`）  |
| **历史遗留工具**    | —                                    | `std::bind` 仍可用但**不推荐**，优先 Lambda                  |


## 九、选型决策（实战中直接按这条走）

1. **回调只在本次调用中使用，不存储** → **模板参数** `template <typename F>`，零开销，可内联。
2. **需要把回调存起来，稍后/多次调用，且可拷贝** → **`std::function`**，现代 C++ 的默认选择。
3. **需要存储，但闭包捕获了 `unique_ptr` 等仅移动类型** → **`std::move_only_function`**（C++23）。
4. **必须把回调传给 C API，接口签名是函数指针** → **无捕获 Lambda**（或自由函数/静态成员函数）；上下文仍走 C 的 `void*` 或全局/静态途径，**不要试图用有捕获 Lambda 强转函数指针**。
5. **回调是某个对象的成员函数** → **Lambda 捕获 `this` 或对象指针**，然后交给 `std::function` 或模板；不要把非静态成员函数指针当 C 函数指针使用。
6. **热路径且需要存储回调** → 优先考虑**自定义类型擦除**或**函数指针**，避免 `std::function` 的间接分派；如果性能仍不足，再评估“最快委托”类实现。
7. **不要用 `std::bind` 写新代码**；需要参数绑定时用 Lambda。


## 十、最终总结（C++ 版）

1. **C++ 继承了 C 的函数指针，但现代 C++ 的回调默认答案不是它。**
2. **Lambda** 是就地定义回调的首选；**无捕获 Lambda 可以零开销退化为函数指针**，有捕获则**绝对不能**。
3. **`std::function`** 用类型擦除统一存储任意可调用对象，解决了 C 回调“`void*` + 手工生命周期”的两大痛点，代价是**一次间接分派 + 可能的堆分配**。
4. **模板参数**是热路径下唯一能做到零开销、可内联的回调方式；C++ 的 `std::sort` 相对于 `qsort` 的性能优势根源就在这里。
5. **成员函数指针不是函数指针**；非静态成员函数有隐藏 `this`，必须绑定对象才能调用；静态成员函数才是与 C 函数指针兼容的形态。
6. **`std::bind` 不再推荐**，Lambda 在可读性和可优化性上全面胜出。
7. **C++23 `std::move_only_function`** 补齐了“仅可移动闭包”这一块拼图，使回调系统在移动语义下仍然完整。
8. **最重要的一条实践规则**：**设计 C++ 回调接口时，不要照搬 C 的“函数指针 + `void*`”签名**；要么用模板，要么用 `std::function`，只在必须保持 C ABI 时才退回裸函数指针。