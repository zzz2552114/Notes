# C++ 头文件、源文件、`#include`、声明定义那些事

## 1. 先把最核心的一句话说了

`#include` 不是“导入模块”，也不是“链接”，它干的事特别朴素：

> **预处理阶段，把目标文件的内容原封不动地复制粘贴到这一行。**

比如你写：

```cpp
#include "math_utils.h"
```

预处理器就找到 `math_utils.h`，把它里面的所有文字复制到 `math_utils.cpp` 里这一行的位置。然后编译器再对合并后的这一大坨代码进行编译。

所以：

- `#include` 不关心文件扩展名。
- 你可以 `#include "a.h"`，也可以 `#include "a.cpp"`，甚至 `#include "a.txt"`，只要路径找得到。
- 但通常我们只 `#include` 头文件，因为头文件是拿来放声明、给别的文件看的。

一个 `.cpp` 文件加上它 `#include` 进来的所有东西，预处理之后形成的东西，叫一个 **翻译单元**。编译器一次编译一个翻译单元，生成一个目标文件 `.o` 或 `.obj`。最后链接器把这些目标文件拼成可执行文件。

---

## 2. 声明和定义，这是理解一切的关键

C++ 里，“声明”和“定义”是两码事。

### 声明

声明就是告诉编译器：

> “有这么个东西，名字叫啥，类型是啥，参数是啥。你先记着，具体在哪定义我后面再说。”

比如：

```cpp
int add(int a, int b);          // 函数声明
extern int global_count;        // 变量声明
class Foo;                      // 类的前置声明
```

声明不分配内存，不产生实际代码。它只是让编译器知道这个名字合法。

### 定义

定义就是：

> “这个东西具体是啥，我在这里把它做出来。”

比如：

```cpp
int add(int a, int b) {         // 函数定义
    return a + b;
}

int global_count = 0;           // 变量定义，分配空间
```

函数定义给出函数体；变量定义分配存储空间。

### 单一定义规则

C++ 有个规则叫 **ODR**，单一定义规则：

- 一个函数、一个全局变量，在整个程序里只能有一个定义。
- 但可以有多个声明。
- `inline` 函数、`inline` 变量、模板、类定义等例外，它们允许多个翻译单元里有相同定义。

理解了这个，后面很多问题就通了。

---

## 3. 头文件里放什么，`.cpp` 里放什么

一般来说：

### 头文件 `.h` / `.hpp` 里放：

- 函数声明
- 类定义
- 模板定义
- `inline` 函数
- `constexpr` 变量
- 类型别名、`using`
- 宏
- `extern` 全局变量声明
- 其他头文件的 `#include`

### `.cpp` 里放：

- 函数定义
- 全局变量定义
- 类的成员函数定义（非模板、非 inline 的）
- 只在本文件用的辅助函数、内部变量

举个新例子，不用你原来的 `stats`：

```cpp
// math_utils.h
#pragma once

#include <vector>
#include <string>

namespace math_utils {

    // 函数声明
    double average(const std::vector<int>& nums);
    long long sum(const std::vector<int>& nums);

    // 全局变量声明：注意是 extern
    extern int debug_level;

    // C++17 起可以这样在头文件里定义 inline 变量
    inline constexpr double pi = 3.1415926;

    // 类定义通常放头文件
    class Calculator {
    public:
        int add(int a, int b) const;
    };

}
```

```cpp
// math_utils.cpp
#include "math_utils.h"   // 自己的头文件放最前
#include <vector>         // 自己用到就包含

namespace math_utils {

    // 全局变量定义，只能有一处
    int debug_level = 0;

    double average(const std::vector<int>& nums) {
        if (nums.empty()) return 0.0;
        return static_cast<double>(sum(nums)) / nums.size();
    }

    long long sum(const std::vector<int>& nums) {
        long long res = 0;
        for (int x : nums) res += x;
        return res;
    }

    int Calculator::add(int a, int b) const {
        return a + b;
    }

}
```

```cpp
// main.cpp
#include "math_utils.h"
#include <iostream>

int main() {
    math_utils::debug_level = 1;
    std::vector<int> v{1, 2, 3};
    std::cout << math_utils::average(v) << '\n';
    return 0;
}
```

这个例子里：

- `math_utils.h` 里 `average` 和 `sum` 只是声明。
- `math_utils.cpp` 里给出定义。
- `debug_level` 在头文件里是 `extern int debug_level;`，在 `math_utils.cpp` 里是 `int debug_level = 0;`。
- `main.cpp` 只要 `#include "math_utils.h"`，就能直接用 `math_utils::debug_level`，**不需要再写 `extern`**，因为头文件里的 `extern` 已经通过 `#include` 复制进来了。

## 编译的时候

### 方式一：一步到位

```bash
g++ main.cpp global.cpp -o app
```

这条命令看起来像“一起编译”，实际上 `g++` 帮你做了：

- 分别编译 `main.cpp` 和 `global.cpp` 成临时目标文件；
- 然后调用链接器把它们链接成 `app`。

所以，**对，最终可执行文件需要这两个 `.cpp` 都参与**。

### 方式二：分步编译再链接

```bash
g++ -c main.cpp -o main.o
g++ -c global.cpp -o global.o
g++ main.o global.o -o app
```

这样更清楚：

- `-c` 表示只编译，不链接。
- 前两步分别生成 `main.o` 和 `global.o`。
- 最后一步链接，把两个目标文件合成 `app`。

如果项目大，通常用 Makefile、CMake 等工具管理，但本质一样。

---

## 4. 为什么 `.cpp` 要 `#include` 自己的 `.h`？

你原来的 `stats.cpp` 里写了：

```cpp
#include "stats.h"
```

这是必须的吗？要看情况。

- 如果你不包含 `stats.h`，但自己在 `stats.cpp` 里 `#include <vector>`，并且把 `Sum` 的定义放到 `Average` 前面，那么编译确实能过。
- 但工程上强烈建议包含自己的头文件。

原因：

1. **让编译器检查声明和定义是否一致。**
   如果 `stats.h` 里声明的是 `double Average(const std::vector<int>&)`，而 `stats.cpp` 里定义成了 `float Average(...)`，不包含头文件的话，编译器可能不会发现这个不一致，但别的文件用头文件调用时就会链接错。

2. **保证接口统一。**
   头文件是给别人看的接口。`.cpp` 包含自己的 `.h`，相当于自己先检查一遍：“我实现的这个东西，和对外承诺的接口一样吗？”

3. **避免依赖间接包含。**
   你可能因为 `stats.h` 里包含了 `<vector>`，所以在 `stats.cpp` 里不写 `<vector>` 也能过。但万一哪天 `stats.h` 不再包含 `<vector>`，你的 `stats.cpp` 就突然编译不过了。

所以标准做法：

```cpp
#include "math_utils.h"   // 自己的头文件放最前
#include <vector>         // 自己直接用到，就自己包含
```

---

## 5. `#include <vector>` 到底该放头文件还是 `.cpp`？

结论：

- **头文件里必须包含 `<vector>`**，因为头文件里的函数声明用到了 `std::vector<int>`。
- **`.cpp` 里也建议包含 `<vector>`**，因为实现里也直接用了 `std::vector<int>`。

这叫 **include what you use**：你用到什么，就包含什么。不要依赖别人已经包含过了。

重复包含会不会出问题？不会。标准库头文件都有 include guard，重复包含会被忽略。

所以：

```cpp
// math_utils.h
#pragma once
#include <vector>   // 必须，因为接口用了 std::vector

namespace math_utils {
    double average(const std::vector<int>& nums);
}
```

```cpp
// math_utils.cpp
#include "math_utils.h"
#include <vector>   // 建议，自己也直接用了

namespace math_utils {
    double average(const std::vector<int>& nums) {
        // ...
    }
}
```

---

## 6. 全局变量和 `extern` 那点事

很多人会踩这个坑：

```cpp
// global.h
int g_count;   // 这是定义，不是声明
```

然后 `a.cpp` 和 `b.cpp` 都 `#include "global.h"`。

预处理之后：

- `a.cpp` 里有一份 `int g_count;`
- `b.cpp` 里也有一份 `int g_count;`

编译都能过，因为每个翻译单元里都看到了定义。但链接的时候，链接器发现两个目标文件里都有 `g_count` 的定义，于是报错：

```
multiple definition of `g_count'
```

这就是重复定义。

正确做法是：

```cpp
// global.h
#pragma once
extern int g_count;   // 声明，不分配空间
```

```cpp
// global.cpp
#include "global.h"
int g_count = 0;      // 定义，只能有一处
```

```cpp
// a.cpp
#include "global.h"
void f() {
    g_count = 10;     // 直接用，不用再写 extern
}
```

```cpp
// b.cpp
#include "global.h"
void h() {
    g_count = 20;     // 也能直接用
}
```

所以：

- 头文件里放 `extern` 声明。
- 某一个 `.cpp` 里放定义。
- 使用的地方只要包含头文件，就能直接用，**不需要自己再写 `extern`**。

如果你只在某一个 `.cpp` 里用这个全局变量，头文件里直接 `int g_count;` 也能用，但一旦多个文件包含，就会炸。

C++17 之后还有个新玩法：

```cpp
// global.h
#pragma once
inline int g_count = 0;   // C++17 inline 变量，允许多个翻译单元包含
```

`inline` 变量允许在多个翻译单元里出现相同定义，链接器会把它们合并成一个。这样就不用拆成 `extern` + 一个 `.cpp` 定义了。

另外，`const int g = 0;` 在命名空间作用域下默认是内部链接，每个翻译单元一份，可以放头文件，但每个 `.cpp` 里的 `g` 其实是不同的对象，一般不拿来共享。

---

## 7. 可以 `#include` 一个 `.cpp` 文件吗？

技术上可以。因为 `#include` 就是文本插入，不管你文件叫啥。

但工程上几乎总是禁止。原因还是重复定义。

比如你在 `main.cpp` 里写：

```cpp
#include "math_utils.cpp"
```

那么 `math_utils.cpp` 的全部内容会被复制到 `main.cpp` 里。`math_utils.cpp` 里又有 `sum` 的定义。同时 `math_utils.cpp` 自己通常也会被单独编译成 `math_utils.o`。链接时：

- `main.o` 里有一份 `sum` 定义
- `math_utils.o` 里也有一份 `sum` 定义

链接器就报 `multiple definition`。

特殊场景可以包含 `.cpp` 或类似文件：

- 模板实现文件，比如 `.tpp` / `.ipp`，被头文件包含，因为模板需要看到实现才能实例化。
- Unity build，把多个 `.cpp` 合并成一个编译单元，加快编译。
- 你非常清楚所有函数都是 `inline` 或 `static`，不会重复定义。

但日常写代码，不要 `#include "xxx.cpp"`。

---

## 8. `#pragma once` 和 include guard

头文件通常要防止被同一个翻译单元重复包含。两种写法：

```cpp
#pragma once
```

或者：

```cpp
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

// 内容

#endif
```

`#pragma once` 不是 C++ 标准，但几乎所有编译器都支持，写起来简单。  
include guard 是标准写法，可移植性最好。

它们只防止同一个翻译单元里重复包含。不同 `.cpp` 各自包含一次头文件是正常的，也是必须的。

---

## 9. 常见错误和注意事项

### 9.1 头文件里放普通函数定义

```cpp
// bad.h
int add(int a, int b) {
    return a + b;
}
```

如果多个 `.cpp` 包含 `bad.h`，链接时就会 `multiple definition of add`。  
要放头文件，得加 `inline`：

```cpp
inline int add(int a, int b) {
    return a + b;
}
```

### 9.2 头文件里放普通全局变量定义

```cpp
// bad.h
int g_count = 0;
```

多个 `.cpp` 包含就重复定义。正确做法是 `extern` 声明 + 一处定义，或者 C++17 `inline` 变量。

### 9.3 头文件没有自包含

```cpp
// bad.h
#pragma once

void print(const std::string& s);   // 用了 std::string，却没包含 <string>
```

别人包含 `bad.h` 时，如果之前没有包含 `<string>`，就会编译错。  
正确做法：头文件用到什么类型，就包含对应的头文件。

### 9.4 不包含自己的头文件

`.cpp` 不包含自己的 `.h`，可能导致声明和定义不一致，链接时才报错。  
建议 `.cpp` 第一行就 `#include "自己的.h"`。

### 9.5 循环包含

`a.h` 包含 `b.h`，`b.h` 又包含 `a.h`，可能出问题。  
可以用前置声明解决：

```cpp
class B;   // 前置声明

class A {
    B* b;  // 指针或引用可以只用前置声明
};
```

但 `std::vector`、`std::string` 这些标准库类型，不要自己前置声明，直接 `#include` 对应头文件。

### 9.6 默认参数写错地方

默认参数通常写在头文件的声明里，定义里不要再写一遍：

```cpp
// math_utils.h
void foo(int x = 10);

// math_utils.cpp
void foo(int x) {   // 这里不要写 = 10
    // ...
}
```

### 9.7 命名空间要一致

声明在 `namespace math_utils`，定义也要在 `namespace math_utils`，否则链接器找不到。

### 9.8 `using namespace` 不要放头文件全局

头文件里写 `using namespace std;` 会污染所有包含它的文件。最好在 `.cpp` 里用，或者用 `std::` 全称。

### 9.9 模板定义通常放头文件

模板需要编译器看到实现才能实例化，所以模板函数、模板类的定义一般放头文件，或者放 `.tpp` 文件再被头文件包含。

---

## 11. 同一文件夹有没有影响？

没有直接影响。

`#include "math_utils.h"` 用引号，编译器会先在当前目录找，再去系统目录找。  
`#include <vector>` 用尖括号，编译器去系统目录找。

同一个文件夹只是让 `"math_utils.h"` 能被找到，**不代表头文件内容会自动进来**，也不代表你必须包含它。  
是否包含，取决于你的 `.cpp` 里是否需要那些声明。



## 补充

---

## 头文件不参与编译成目标文件

`global.h` 本身**不会**被单独编译成目标文件。  
它只是被 `#include` 到 `main.cpp` 和 `global.cpp` 里，在预处理阶段被复制进去。

所以：

- `global.h` 不需要出现在编译命令里。
- 你写 `g++ main.cpp global.cpp -o app` 就够了。
- 头文件的作用是让每个 `.cpp` 看到声明，保证编译通过。

---

## 如果只编译 `main.cpp` 会怎样？

如果你只写：

```bash
g++ main.cpp -o app
```

那么：

- `main.cpp` 能编译通过，因为它包含了 `global.h`，看到了 `increase` 和 `g_count` 的声明。
- 但链接时，链接器找不到 `increase` 和 `g_count` 的定义，因为 `global.cpp` 没参与。
- 于是报错：`undefined reference to 'increase()'` 或 `undefined reference to 'g_count'`。

所以，**光有声明不够，必须把定义所在的目标文件也交给链接器**。

---

## 如果重复提供定义会怎样？

如果你写：

```bash
g++ main.cpp global.cpp global.cpp -o app
```

 `global.cpp` 被包含了两次，链接器会发现 `increase` 和 `g_count` 有多份定义，报：

```
multiple definition of `increase()'
multiple definition of `g_count'
```

这就是为什么：

- 头文件里放 `extern int g_count;` 声明，而不是 `int g_count = 0;` 定义。
- 全局变量的定义只能在一个 `.cpp` 里出现一次。

