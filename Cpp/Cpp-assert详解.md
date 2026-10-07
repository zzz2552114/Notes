这句话的核心是：**C/C++ 的 `assert` 是一个受预处理宏 `NDEBUG` 控制的宏。只要在包含 `<assert.h>` / `<cassert>` 之前定义了 `NDEBUG`，`assert(expr)` 就会被展开成空操作，`expr` 根本不会被求值。** 所以如果把有副作用的代码写进 `assert`，Release 构建里这段代码会“消失”。

下面详细说。

## 1. `NDEBUG` 是什么？

`NDEBUG` 是 C/C++ 里一个约定俗成的预处理宏，通常读作 “No Debug” 或 “No Debugging”。它不是关键字，不是变量，也不是标准库函数，而是一个宏名。

它唯一的重要作用就是：**控制 `assert` 是否生效。**

C 标准大致规定：

- 如果包含 `<assert.h>` 时，`NDEBUG` **没有被定义**：
  - `assert(expr)` 会检查 `expr`。
  - 如果 `expr` 为假，程序打印诊断信息，然后调用 `abort()` 终止。
- 如果包含 `<assert.h>` 时，`NDEBUG` **已经被定义**：
  - `assert(expr)` 被展开为类似 `((void)0)` 的东西。
  - 也就是说，它变成一条空语句。
  - `expr` 完全不会被求值，不会生成检查代码。

C++ 的 `<cassert>` 行为相同。

例如：

```cpp
#include <cassert>

void f(int x) {
    assert(x > 0);
}
```

Debug 构建不定义 `NDEBUG`：

```cpp
assert(x > 0);
```

会真的检查 `x > 0`，如果 `x <= 0` 就 abort。

Release 构建定义了 `NDEBUG`：

```cpp
assert(x > 0);
```

预处理后基本变成：

```cpp
((void)0);
```

什么也不做。

## 2. `NDEBUG` 怎么定义？

常见方式：

- GCC/Clang 编译选项：

```bash
-DNDEBUG
```

- MSVC 编译选项：

```bash
/DNDEBUG
```

- 源码里：

```cpp
#define NDEBUG
#include <cassert>
```

注意：**`#define NDEBUG` 必须放在 `#include <cassert>` 或 `#include <assert.h>` 之前。** 如果你已经包含了 `<cassert>`，再定义 `NDEBUG`，通常不会影响已经定义好的 `assert` 宏。

另外，标准判断的是“`NDEBUG` 是否被定义”，不是它的值。所以：

```cpp
#define NDEBUG 0
```

也会关闭 `assert`。不要以为写 0 就是开启。通常直接写 `#define NDEBUG` 或用 `-DNDEBUG`。

## 3. Debug / Release 与 `NDEBUG`

Debug 和 Release 不是 C/C++ 语言标准里的概念，而是构建系统的配置。

常见约定：

- Debug 构建：不定义 `NDEBUG`，保留断言，方便发现错误。
- Release 构建：定义 `NDEBUG`，去掉断言，减少运行时开销和代码体积。
- CMake 的 Release 配置通常会自动带上 `-DNDEBUG`。
- MSVC 的 Release 配置通常有 `/DNDEBUG`。

但要注意：**优化等级本身不会自动关闭 assert。**  
`g++ -O2 main.cpp` 不会自动定义 `NDEBUG`，断言仍然有效。  
只有你或构建系统显式加了 `-DNDEBUG`，断言才会消失。

## 4. 为什么不要把有副作用的表达式放进 `assert`？

副作用就是：表达式求值时不只计算一个值，还会改变程序状态。

例如：

```cpp
int pop(); // 从栈中弹出一个元素，并返回它
```

坏例子：

```cpp
assert(pop() == 3);
```

Debug 构建时，`NDEBUG` 未定义，`assert` 生效：

```cpp
pop(); // 真的会调用
```

Release 构建时，`NDEBUG` 已定义，`assert` 变成空操作：

```cpp
((void)0);
```

于是 `pop()` 根本不会被调用，栈的状态就变了，程序逻辑在 Debug 和 Release 下不一致。可能 Debug 正常，Release 出错。

类似坏例子：

```cpp
assert(++i < n);
assert((p = malloc(sizeof(int))) != NULL);
assert(fclose(fp) == 0);
```

这些都会在 Release 下丢失副作用。

正确做法是：**副作用放在 `assert` 外面，`assert` 只检查结果。**

```cpp
int v = pop();
assert(v == 3);
```

如果这个结果必须处理，那就不应该用 `assert`，而应该用真正的错误处理：

```cpp
int v = pop();
if (v != 3) {
    // 错误处理
}
```

`assert` 适合检查“程序内部不应该发生”的不变量、前置条件、后置条件，不适合处理用户输入或可恢复错误。

## 5. 对 `P6.3` 里 `assert(capacity >= 1)` 的理解

`assert(capacity >= 1);` 这个表达式里：

- `capacity` 只是被读取、比较。
- 没有函数调用，没有赋值，没有 `++`，没有修改任何状态。
- 所以它没有副作用。

因此：

- Debug 构建下：如果 `capacity < 1`，程序立即断言失败，打印类似 `Assertion 'capacity >= 1' failed`，然后 abort。这样能更早发现调用者误用。
- Release 构建下：`NDEBUG` 定义了，这行变成空操作，检查消失，但因为没有副作用，程序逻辑不会因为这行消失而改变。

所以参考实现里写：

```cpp
assert(capacity >= 1);
```

意思是：**这个实现假设 `capacity` 至少为 1。** 写不写都可以：

- 写了：Debug 下更早暴露误用。
- 不写：正常逻辑也不受影响，只是少了一个调试检查。
- 如果 `capacity` 来自用户输入或外部数据，不能只靠 `assert` 防止错误，因为 Release 下它不会检查。此时应该用 `if (capacity < 1) throw ...` 或返回错误码。

## 6. 总结

一句话：

**`NDEBUG` 是关闭 `assert` 的开关。定义它后，`assert(expr)` 变成空操作，`expr` 不求值。所以不要把 `pop()`、赋值、自增、函数调用等有副作用的代码放进 `assert`。`assert(capacity >= 1)` 只是只读检查，没有副作用，Release 下消失也不会影响逻辑，只是少了一个 Debug 期的早期检查。**