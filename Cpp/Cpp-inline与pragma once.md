下面把你的问题拆开讲清楚：**`#pragma once` 和 `inline` 解决的是完全不同层面的“重复”问题**。你举的例子如果写错，会触发 **ODR 违反**，程序行为未定义。

---

## 1. 先看你的例子

```cpp
// common.h
#pragma once

inline int global = 0;   // C++17
static int counter = 0;
```

假设：

- `b.cpp` 包含了 `common.h`
- `a.cpp` 里自己写了 `inline int global = 1;`
- 最后 `a.cpp` 和 `b.cpp` 一起链接

会发生什么？要分两种情况。

### 情况 1：`a.cpp` 也包含了 `common.h`

那么 `a.cpp` 在预处理后，会变成类似：

```cpp
// common.h 展开
inline int global = 0;

// a.cpp 自己写的
inline int global = 1;
```

在**同一个翻译单元**里，`global` 被定义了两次：

```cpp
inline int global = 0;
inline int global = 1;
```

这是 **重定义错误**，编译直接失败。  
即使加了 `inline` 也不行。`inline` 允许的是“不同翻译单元中出现相同定义”，不是“同一个翻译单元里出现多个不同定义”。

所以：**编译错误：redefinition of 'global'**。

---

### 情况 2：`a.cpp` 没有包含 `common.h`

那么：

- `a.cpp` 翻译单元里有：`inline int global = 1;`
- `b.cpp` 翻译单元里有：`inline int global = 0;`（来自 `common.h`）

两个翻译单元都定义了同一个外部链接的 `inline` 变量 `global`，但**初始化值不同**。

C++ 标准规定：

> 一个 `inline` 变量可以在多个翻译单元中定义，但所有定义必须完全相同（token 序列相同、名字查找相同等）。如果不同，则程序 ill-formed，无需诊断，也就是 **未定义行为（UB）**。

实际链接时可能发生：

- 链接器把 `inline` 变量当作 **weak symbol**，看到两个弱符号，就随便选一个。
- 可能选 `a.cpp` 的 `global = 1`，也可能选 `b.cpp` 的 `global = 0`。
- 取决于链接顺序、编译器、优化等级、目标文件顺序等。
- 最终 `a.cpp` 和 `b.cpp` 看到的 `global` 是同一个实体，但值可能是 0 或 1，不确定。
- 有些链接器会报错，有些不会。

所以结论是：

> **`a.cpp` 写 `inline int global = 1;`，`b.cpp` 包含 `common.h` 得到 `inline int global = 0;`，如果两个翻译单元都参与链接，程序违反 ODR，行为未定义。不能这样写。**

正确做法：`inline` 变量的所有定义必须完全一致。比如都在头文件里写 `inline int global = 0;`，所有 `.cpp` 都包含这个头文件，这样每个翻译单元定义相同，链接器合并成一个实体。

---

## 2. `#pragma once` 是什么？

`#pragma once` 是**预处理指令**，作用非常单纯：

> 保证同一个头文件在**同一个翻译单元**中只被包含一次。

翻译单元 = 一个 `.cpp` 文件加上它所有 `#include` 展开后的内容。  
`#pragma once` 只在这个预处理阶段起作用。

例如：

```cpp
// common.h
#pragma once
inline int global = 0;
```

`a.cpp` 如果写了：

```cpp
#include "common.h"
#include "common.h"
```

预处理器看到第二次 `#include "common.h"` 时，因为 `#pragma once`，不会再展开。  
所以 `a.cpp` 中只出现一次 `inline int global = 0;`。

如果没有 `#pragma once`，也没有 include guard，那么 `a.cpp` 中会出现两次：

```cpp
inline int global = 0;
inline int global = 0;
```

这仍然是同一个翻译单元中的重定义，**编译错误**。  
所以 `#pragma once` 防止的是这种“同一个 TU 内重复展开同一个头文件”的问题。

`#pragma once` 不是 C++ 标准，但几乎所有编译器都支持。标准做法是 include guard：

```cpp
#ifndef COMMON_H
#define COMMON_H

inline int global = 0;

#endif
```

效果类似：同一个 TU 中只展开一次。

---

## 3. `#pragma once` 和 `inline` 的区别

| 对比项                                         | `#pragma once` / include guard                        | `inline`                                                |
| ---------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------- |
| 层面                                           | 预处理阶段                                            | 语言语义、链接阶段                                      |
| 作用范围                                       | **同一个翻译单元内**                                  | **多个翻译单元之间**                                    |
| 目的                                           | 防止同一个头文件在同一个 TU 中被多次展开              | 允许同一个函数/变量在多个 TU 中定义相同版本，链接器合并 |
| 能否解决多个 `.cpp` 包含头文件导致的多重定义？ | 不能                                                  | 可以（对于 `inline` 函数/变量）                         |
| 能否解决同一个 TU 中重复定义？                 | 能（防止头文件重复展开）                              | 不能。同一个 TU 中即使 `inline` 也不能重复定义          |
| 是否标准                                       | `#pragma once` 非标准但广泛支持；include guard 是标准 | 标准 C++ 关键字                                         |
| 典型用途                                       | 所有头文件都应该加                                    | 头文件中定义函数、变量、类静态成员时使用                |

一句话：

> `#pragma once` 管的是“**同一个编译单元内，头文件不要被重复粘贴**”。  
> `inline` 管的是“**多个编译单元之间，允许出现相同的定义，最后合并成一个实体**”。  
> 它们不是替代关系，而是**配合关系**。

---

## 4. 为什么头文件里写 `inline` 变量还需要 `#pragma once`？

因为 `inline` 不能防止同一个翻译单元中重复定义。

例子：

```cpp
// common.h
inline int global = 0;
```

`a.cpp`：

```cpp
#include "common.h"
#include "common.h"   // 没有 #pragma once 或 include guard
```

预处理后 `a.cpp` 变成：

```cpp
inline int global = 0;
inline int global = 0;
```

这是同一个翻译单元中的重定义，编译错误。  
`inline` 允许的是不同翻译单元中的重复定义，不是同一个翻译单元中的重复定义。

所以头文件里定义 `inline` 变量时，仍然必须加 `#pragma once` 或 include guard。

---

## 5. `static` 在头文件中的行为

你的 `common.h` 里还有：

```cpp
static int counter = 0;
```

`static` 在命名空间作用域表示**内部链接**。  
每个包含 `common.h` 的翻译单元，都会得到自己的 `counter`。

例如：

- `a.cpp` 包含 `common.h`，有自己的 `counter`。
- `b.cpp` 包含 `common.h`，有自己的 `counter`。
- 两个 `counter` 地址不同，互不影响。
- 不会发生链接冲突，因为它们是内部链接，链接器看不到它们之间的冲突。

但注意：

- `#pragma once` 防止同一个 TU 中重复包含 `common.h`，所以每个 TU 中只有一个 `counter`。
- 如果没有 `#pragma once`，同一个 TU 中会出现两个 `static int counter = 0;`，那也是重定义错误。
- `static` 和 `inline` 的区别：`inline` 变量全程序一个实体，`static` 变量每个 TU 一个实体。

---

## 6. 正确写法总结

如果你想在头文件中定义一个全局变量，并让所有 `.cpp` 共享同一个实体，C++17 应该这样：

```cpp
// common.h
#pragma once

inline int global = 0;   // 所有包含它的 TU 共享同一个 global
```

所有 `.cpp` 都只包含这个头文件，不要自己再写一个不同的 `inline int global = 1;`。  
如果某个 `.cpp` 需要修改 `global`，直接赋值：

```cpp
// a.cpp
#include "common.h"

void set_global() {
    global = 1;   // 修改的是同一个全局变量
}
```

如果你希望每个 `.cpp` 有自己的独立变量，用：

```cpp
// common.h
#pragma once

static int counter = 0;   // 每个 TU 一份
```

或者 C++17 更推荐用匿名命名空间：

```cpp
// common.h
#pragma once

namespace {
    int counter = 0;   // 内部链接，每个 TU 一份
}
```

---

## 7. 关于 ODR 的准确表述

C++ 的 ODR（One Definition Rule）对 `inline` 变量的要求：

- 一个 `inline` 变量可以在多个翻译单元中定义。
- 每个定义必须出现在不同的翻译单元中。
- 所有定义必须完全相同。
- 如果定义不同，程序 ill-formed，无需诊断，即未定义行为。

所以：

```cpp
// a.cpp
inline int global = 1;

// b.cpp
inline int global = 0;
```

这是 **ODR 违反**。  
链接器可能合并成一个，但选哪个不确定。  
这不是“谁覆盖谁”的可靠行为，而是未定义行为。

---

## 8. 最终结论

1. **`#pragma once` 和 `inline` 不是一回事。**
   - `#pragma once`：预处理层面，防止同一个头文件在同一个翻译单元中重复展开。
   - `inline`：语言/链接层面，允许在多个翻译单元中定义相同实体，最后合并成一个。

2. **你的例子中，如果 `a.cpp` 写 `inline int global = 1;`，而 `b.cpp` 包含 `common.h` 得到 `inline int global = 0;`：**
   - 如果 `a.cpp` 也包含 `common.h`，则 `a.cpp` 中 `global` 重定义，编译错误。
   - 如果 `a.cpp` 不包含 `common.h`，则两个翻译单元中的 `inline` 定义不同，违反 ODR，程序行为未定义。实际可能链接器随便选一个，值不确定。

3. **头文件中定义 `inline` 变量，必须同时使用 `#pragma once` 或 include guard。**
   - 否则同一个 TU 多次包含头文件会导致重定义。
   - `inline` 不能解决同一个 TU 内的重复定义。

4. **`static` 在头文件中表示内部链接，每个 TU 一份，不会冲突，但也不是全程序共享。**
   - `inline` 是全程序一个实体。
   - `static` 是每个 TU 一个实体。

5. **正确做法：**
   - 要全程序共享：`inline int global = 0;` + `#pragma once`，所有 TU 都包含同一个头文件，不要各自写不同定义。
   - 要每个 TU 独立：`static int counter = 0;` 或匿名命名空间。