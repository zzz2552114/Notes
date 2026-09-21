## 1. `throw` 的基本语法

C++ 里 `throw` 常见只有两种核心形式：

### 1.1 抛出一个异常对象

```cpp
throw 表达式;
```

例如：

```cpp
throw std::runtime_error("something failed");
throw std::out_of_range("index out of range");
throw std::invalid_argument("bad argument");
throw MyException("custom error");
throw 42;
throw "error";
```

其中 `表达式` 的结果会被用来初始化一个“异常对象”。  
异常对象会被传播到匹配的 `catch`。

比如：

```cpp
#include <stdexcept>

void f(int i) {
    if (i < 0) {
        throw std::out_of_range("i must be >= 0");
    }
}
```

调用：

```cpp
try {
    f(-1);
} catch (const std::out_of_range& e) {
    std::cerr << e.what() << '\n';
}
```

这里：

- `throw` 是关键字；
- `std::out_of_range("i must be >= 0")` 创建一个临时异常对象；
- `catch (const std::out_of_range& e)` 捕获它；
- `e.what()` 返回描述信息。

### 1.2 重新抛出当前异常

```cpp
throw;
```

这种形式没有表达式，也没有括号。  
它只能在当前有异常正在处理时使用，常见于 `catch` 块：

```cpp
try {
    do_something();
} catch (const std::exception& e) {
    log(e.what());
    throw; // 继续把原异常抛出去
}
```

`throw;` 和 `throw e;` 不一样：

```cpp
catch (const std::exception& e) {
    throw;   // 重新抛出原异常，保留原始动态类型
    // throw e; // 会复制 e，可能发生对象切片，丢失派生类信息
}
```

所以如果你只是想“处理一下再继续抛”，用：

```cpp
throw;
```

不要用：

```cpp
throw e;
```

---

## 2. 为什么不是 `throw(...)`？

因为 `throw` 不是函数名。  
函数调用的形式是：

```cpp
函数名(参数列表)
```

而 `throw` 的语法是：

```cpp
throw 表达式;
```

所以：

```cpp
throw std::out_of_range("msg");
```

解析成：

```text
throw  +  std::out_of_range("msg")  +  ;
```

其中：

```cpp
std::out_of_range("msg")
```

才是构造对象。

你也可以把整个表达式括起来：

```cpp
throw (std::out_of_range("msg"));
```

这也是合法的，因为 `(std::out_of_range("msg"))` 仍然是一个表达式。  
但这里括号不是 `throw` 的语法要求，只是表达式分组。

注意一个特殊历史语法：

```cpp
void f() throw(int); // 旧式动态异常规格，C++11 弃用，C++17 移除
void g() throw();    // C++17 起等价于 noexcept
```

这不是“抛出异常”的常用写法，而是函数声明后的异常规格。  
现代 C++ 不要用它，应该用：

```cpp
void f() noexcept;
void g() noexcept(false);
```

所以不要把：

```cpp
throw(std::out_of_range("msg"));
```

和：

```cpp
void f() throw();
```

混为一谈。前者是抛异常表达式，后者是旧式异常规格。

---

## 3. 常用标准异常

C++ 标准库已经有一整套异常类型，最常用的是：

```cpp
#include <stdexcept>
```

常见类型：

| 异常类型                   | 常见用途                 |
| -------------------------- | ------------------------ |
| `std::exception`           | 所有标准异常的基类       |
| `std::logic_error`         | 程序逻辑错误             |
| `std::invalid_argument`    | 参数非法                 |
| `std::out_of_range`        | 下标/范围越界            |
| `std::length_error`        | 超出最大长度             |
| `std::runtime_error`       | 运行时错误               |
| `std::range_error`         | 结果超出范围             |
| `std::overflow_error`      | 算术上溢                 |
| `std::underflow_error`     | 算术下溢                 |
| `std::bad_alloc`           | `new` 分配失败           |
| `std::bad_cast`            | `dynamic_cast` 引用失败  |
| `std::bad_optional_access` | `optional::value()` 无值 |
| `std::bad_variant_access`  | `variant` 访问错误       |

最常用的是：

```cpp
throw std::runtime_error("...");
throw std::invalid_argument("...");
throw std::out_of_range("...");
```

比如：

```cpp
int safe_get(const std::vector<int>& v, std::size_t i) {
    if (i >= v.size()) {
        throw std::out_of_range("index out of range");
    }
    return v[i];
}
```

捕获：

```cpp
try {
    int x = safe_get(v, 100);
} catch (const std::out_of_range& e) {
    std::cerr << "越界: " << e.what() << '\n';
} catch (const std::exception& e) {
    std::cerr << "其他标准异常: " << e.what() << '\n';
}
```

注意捕获顺序：**派生类在前，基类在后**。  
`std::out_of_range` 继承自 `std::logic_error`，最终继承自 `std::exception`。  
如果把 `catch (const std::exception&)` 放前面，后面的 `catch` 就永远轮不到。

---

## 4. 自定义异常

常用做法是继承 `std::runtime_error` 或 `std::logic_error`：

```cpp
#include <stdexcept>
#include <string>

class MyError : public std::runtime_error {
public:
    explicit MyError(const std::string& msg)
        : std::runtime_error(msg) {}
};
```

抛出：

```cpp
throw MyError("something bad happened");
```

捕获：

```cpp
try {
    throw MyError("something bad happened");
} catch (const MyError& e) {
    std::cerr << "MyError: " << e.what() << '\n';
} catch (const std::exception& e) {
    std::cerr << "std exception: " << e.what() << '\n';
}
```

也可以继承构造函数：

```cpp
class MyError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};
```

然后：

```cpp
throw MyError("msg");
```

`what()` 是虚函数，返回 `const char*`。  
自定义异常通常不要自己管理复杂资源，保持可复制、简单即可。

---

## 5. `throw` 的常见组合用法

### 5.1 在函数中检查条件并抛出

```cpp
int divide(int a, int b) {
    if (b == 0) {
        throw std::invalid_argument("divide by zero");
    }
    return a / b;
}
```

### 5.2 在构造函数中抛出

```cpp
class Socket {
public:
    Socket(int fd) {
        if (fd < 0) {
            throw std::invalid_argument("bad fd");
        }
        // ...
    }
};
```

构造函数抛异常时，对象没有完整构造成功，析构函数不会执行。  
但已经构造完成的成员会被析构。所以资源要用 RAII 管理。

### 5.3 在 `catch` 中包装异常

```cpp
try {
    do_something();
} catch (const std::exception& e) {
    throw std::runtime_error(
        std::string("do_something failed: ") + e.what()
    );
}
```

注意这会丢失原始异常类型。  
如果想保留嵌套异常，可以用 `std::throw_with_nested`，但那属于进阶用法。

### 5.4 在条件表达式中抛出

```cpp
int x = ok ? 42 : throw std::runtime_error("not ok");
```

`throw` 表达式类型是 `void`，但在条件运算符中有特殊规则，可以这样用。

### 5.5 在 lambda 中抛出

```cpp
auto f = [] {
    throw std::runtime_error("lambda error");
};
```

### 5.6 捕获所有异常并重新抛出

```cpp
try {
    do_something();
} catch (...) {
    cleanup();
    throw; // 保留原异常
}
```

`catch (...)` 可以捕获所有异常，但拿不到异常对象，只能重新抛出或做清理。

---

## 6. 不推荐的抛法

### 6.1 抛裸指针

```cpp
throw new std::runtime_error("bad");
```

不推荐。  
因为 `catch` 到指针后还要手动 `delete`，很容易泄漏。

```cpp
catch (std::runtime_error* e) {
    delete e;
}
```

不如直接抛对象：

```cpp
throw std::runtime_error("bad");
```

### 6.2 抛整数、字符串字面量

```cpp
throw 42;
throw "error";
```

可以编译，但不推荐。  
因为它们不属于标准异常体系，`catch (const std::exception&)` 捕获不到，调用者很难统一处理。

### 6.3 抛 `std::exception` 并想带字符串

```cpp
throw std::exception("msg"); // 错
```

`std::exception` 没有接受字符串的构造函数。  
应该用：

```cpp
throw std::runtime_error("msg");
```

### 6.4 `catch` 按值捕获

```cpp
catch (std::exception e) { ... } // 不推荐
```

应该用引用：

```cpp
catch (const std::exception& e) { ... }
```

按值捕获会发生对象切片，丢失派生类部分。

### 6.5 在析构函数中抛异常

析构函数默认是 `noexcept`。  
如果析构函数中抛异常，通常会直接调用 `std::terminate`，程序终止。  
所以析构函数里不要抛异常，必要时自己捕获并处理。

---

## 7. `throw` 和 `noexcept`

现代 C++ 用 `noexcept` 表示函数不抛异常：

```cpp
void f() noexcept;        // 不抛，抛了就 terminate
void g() noexcept(false); // 可以抛
```

如果 `noexcept` 函数中抛出异常：

```cpp
void f() noexcept {
    throw std::runtime_error("bad"); // 会导致 std::terminate
}
```

所以：

```cpp
throw std::runtime_error("bad");
```

只是抛出异常。  
异常能不能被捕获，取决于调用链上有没有匹配的 `catch`，以及中间有没有 `noexcept` 函数。

---

## 8. 异常对象生命周期和切片

```cpp
try {
    throw std::out_of_range("msg");
} catch (const std::out_of_range& e) {
    // e 绑定到异常对象
    std::cout << e.what() << '\n';
}
```

异常对象在 `throw` 时创建，生命周期一直持续到异常处理结束。  
`catch` 的引用绑定到这个异常对象。

如果这样：

```cpp
catch (const std::exception& e) {
    throw e; // 不好
}
```

`e` 的静态类型是 `std::exception`，所以 `throw e;` 可能只抛出基类部分，发生切片。  
应该用：

```cpp
catch (const std::exception& e) {
    throw; // 好，保留原异常
}
```

---

## 9. 总结

`throw` 的正确理解：

```cpp
throw 表达式; // 抛出一个异常对象
throw;        // 重新抛出当前异常
```

`throw` 是关键字，不是函数。  
所以：

```cpp
throw std::out_of_range("msg");
```

不是“调用 throw 函数，参数是 std::out_of_range("msg")”。  
而是：

```text
throw  +  一个表达式  +  ;
```

其中：

```cpp
std::out_of_range("msg")
```

是构造异常对象。

写成：

```cpp
throw(std::out_of_range("msg"));
```

也可以，但括号只是表达式分组，不是 `throw` 的参数列表。

最常用写法：

```cpp
throw std::runtime_error("...");
throw std::invalid_argument("...");
throw std::out_of_range("...");
throw MyException("...");
throw; // 在 catch 中重新抛出
```

捕获时：

```cpp
catch (const std::out_of_range& e) { ... }
catch (const std::exception& e) { ... }
catch (...) { ... }
```

现代 C++ 中不要写函数声明后的 `throw(int)` 这类动态异常规格，用 `noexcept`。  
析构函数、`noexcept` 函数里不要随便抛异常。  
重新抛出用 `throw;`，不要用 `throw e;`。