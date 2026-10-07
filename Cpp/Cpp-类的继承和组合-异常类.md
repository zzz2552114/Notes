## 一、你这句代码到底在干什么？

```cpp
class ClosedError : public std::runtime_error {
```

意思是：定义一个类 `ClosedError`，它**公有继承**自 `std::runtime_error`。

`std::runtime_error` 是 C++ 标准库里的异常类，表示“运行时错误”。它继承自 `std::exception`，并且可以保存一段错误消息，通过 `what()` 返回。

```cpp
public:
    ClosedError() : std::runtime_error("BlockingQueue is closed") {}
```

这是在定义 `ClosedError` 的默认构造函数。

冒号后面的：

```cpp
: std::runtime_error("BlockingQueue is closed")
```

叫**成员初始化列表**。这里用来调用基类 `std::runtime_error` 的构造函数，把异常消息设成 `"BlockingQueue is closed"`。

因为 `std::runtime_error` 没有默认构造函数，所以你必须显式调用它的某个构造函数。否则编译器会报错。

最后的 `{}` 是构造函数体，这里为空。

```cpp
} ;
```

类定义末尾必须有分号。你写了分号，这是对的。

---

## 二、为什么“非得”继承一个类？不继承行不行？

**不是非得继承。**  
C++ 允许你抛任何类型：

```cpp
throw 42;
throw "error";
throw MyStruct{};
```

但实际工程里，自定义异常通常都会继承 `std::exception` 或它的派生类，比如 `std::runtime_error`。原因不是为了“语法必须”，而是为了**统一异常处理**。

### 1. 不继承会怎样？

假设你定义：

```cpp
struct ClosedError {
    const char* msg;
};
```

然后：

```cpp
try {
    throw ClosedError{"queue closed"};
} catch (const std::exception& e) {
    // 捕获不到 ClosedError
}
```

因为 `ClosedError` 不是 `std::exception` 的派生类，所以 `catch (const std::exception&)` 抓不住它。别人必须专门写：

```cpp
catch (const ClosedError& e) {
    // 才能抓住
}
```

这会导致使用你代码的人必须知道你的具体异常类型，耦合很重。

### 2. 继承 `std::runtime_error` 会怎样？

```cpp
class ClosedError : public std::runtime_error {
public:
    ClosedError() : std::runtime_error("BlockingQueue is closed") {}
};
```

现在：

```cpp
try {
    throw ClosedError();
} catch (const std::exception& e) {
    std::cout << e.what() << '\n';  // 输出 BlockingQueue is closed
}
```

因为 `ClosedError` 是一种 `std::runtime_error`，而 `std::runtime_error` 又是一种 `std::exception`，所以它可以被 `std::exception` 的 catch 捕获。

这就是继承的意义：**把你的异常挂到标准异常家族树上，让别人可以用统一方式处理。**

---

## 三、C++ 异常体系长什么样？

大致是这样：

```text
std::exception
├── std::logic_error
│   ├── std::invalid_argument
│   ├── std::domain_error
│   ├── std::length_error
│   └── std::out_of_range
└── std::runtime_error
    ├── std::range_error
    ├── std::overflow_error
    └── std::underflow_error
```

`ClosedError` 表示“阻塞队列已关闭”，这是运行时才能发现的错误，所以继承 `std::runtime_error` 很合适。

如果你写的是“参数非法”，可以继承 `std::invalid_argument`；  
如果是“下标越界”，可以继承 `std::out_of_range`；  
如果只是通用错误，也可以直接继承 `std::exception`。

---

## 四、C++ 继承到底是怎么回事？

继承表达的是 **is-a** 关系：派生类“是一种”基类。

例如：

```cpp
class Animal {
public:
    void breathe() { std::cout << "breathing\n"; }
};

class Dog : public Animal {
public:
    void bark() { std::cout << "woof\n"; }
};
```

`Dog` 是一种 `Animal`。所以：

```cpp
Dog d;
d.breathe();  // 继承自 Animal
d.bark();     // 自己的
```

也可以把 `Dog` 当作 `Animal` 使用：

```cpp
Animal& a = d;
a.breathe();
```

### 继承的三种方式

```cpp
class D : public B { ... };     // 公有继承
class D : protected B { ... };  // 保护继承
class D : private B { ... };    // 私有继承
```

- `public` 继承：最常用，表示 is-a。基类的 public 成员在派生类中还是 public，外部可以把派生类当成基类。
- `protected` 继承：基类的 public/protected 成员在派生类中变成 protected，外部不能把派生类当成基类。
- `private` 继承：基类所有成员在派生类中变成 private，外部完全不能把派生类当成基类。通常表示“用基类来实现”，而不是 is-a。

对于异常类，**必须用 public 继承**。  
如果你写：

```cpp
class ClosedError : std::runtime_error { ... };
```

因为 `class` 默认是 private 继承，所以 `catch (const std::runtime_error&)` 抓不住它。这是常见错误。

所以必须写：

```cpp
class ClosedError : public std::runtime_error { ... };
```

---

## 五、构造顺序和初始化列表

继承时，构造顺序是：

1. 先构造基类；
2. 再构造成员变量；
3. 最后执行派生类构造函数体。

析构顺序反过来。

所以：

```cpp
class ClosedError : public std::runtime_error {
public:
    ClosedError() : std::runtime_error("BlockingQueue is closed") {}
};
```

执行 `ClosedError()` 时：

1. 先调用 `std::runtime_error("BlockingQueue is closed")`；
2. 然后执行 `ClosedError` 的构造函数体 `{}`。

因为 `std::runtime_error` 没有默认构造函数，所以你不能写：

```cpp
ClosedError() {}
```

否则编译器会尝试调用 `std::runtime_error()`，但不存在，编译失败。

---

## 六、类之间的关系有哪些？

除了继承，类之间还有这些常见关系：

### 1. 继承：is-a

```cpp
class Dog : public Animal { ... };
```

`Dog` 是一种 `Animal`。

### 2. 组合：has-a

```cpp
class Car {
    Engine engine_;
};
```

`Car` 有一个 `Engine`。

组合通常优于继承。如果你只是需要“有一个”关系，不要用继承。

### 3. 聚合：弱 has-a

```cpp
class Car {
    Engine* engine_;
};
```

`Car` 关联一个 `Engine`，但不负责它的生命周期。

### 4. 依赖：uses-a

```cpp
void drive(Engine& e);
```

函数 `drive` 使用了 `Engine`。

### 5. 友元：friend

```cpp
class A {
    friend class B;
private:
    int x_;
};
```

`B` 可以访问 `A` 的私有成员。

### 6. 接口继承

只继承纯虚函数，定义接口：

```cpp
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};
```

### 7. 多重继承

```cpp
class D : public A, public B { ... };
```

一个类继承多个基类。可能引起菱形问题，用虚继承解决。异常类通常不需要多重继承。

---

## 七、自定义异常的常见写法

### 1. 固定消息

```cpp
#include <stdexcept>

class ClosedError : public std::runtime_error {
public:
    ClosedError() : std::runtime_error("BlockingQueue is closed") {}
};
```

### 2. 允许自定义消息

```cpp
#include <stdexcept>
#include <string>

class ClosedError : public std::runtime_error {
public:
    explicit ClosedError(const std::string& msg = "BlockingQueue is closed")
        : std::runtime_error(msg) {}
};
```

### 3. 带额外数据

```cpp
class ClosedError : public std::runtime_error {
    int code_;
public:
    ClosedError(int code, const std::string& msg = "BlockingQueue is closed")
        : std::runtime_error(msg), code_(code) {}

    int code() const noexcept { return code_; }
};
```

### 4. 抛出

```cpp
throw ClosedError();
throw ClosedError("queue already closed");
throw ClosedError(1001, "queue closed");
```

### 5. 捕获

```cpp
try {
    // ...
} catch (const ClosedError& e) {
    std::cerr << "ClosedError: " << e.what() << '\n';
} catch (const std::runtime_error& e) {
    std::cerr << "runtime_error: " << e.what() << '\n';
} catch (const std::exception& e) {
    std::cerr << "exception: " << e.what() << '\n';
}
```

注意：**具体类型在前，基类在后**。  
如果先写 `catch (const std::exception&)`，那么 `ClosedError` 也会被它抓住，你就丢失了具体类型信息。

---

## 八、异常使用的注意事项

1. **按 const 引用捕获**  
   ```cpp
   catch (const ClosedError& e)  // 对
   catch (ClosedError e)         // 错，会切片，多态失效
   ```

2. **不要抛裸指针**  
   ```cpp
   throw new ClosedError();  // 不推荐，谁负责 delete？
   ```

3. **析构函数不要抛异常**  
   否则可能导致 `std::terminate`。

4. **异常类要可拷贝**  
   抛出异常时，异常对象会被拷贝或移动到异常存储区。

5. **公有继承标准异常**  
   这样别人可以用 `catch (const std::exception&)` 捕获。

6. **不要从没有虚析构的类继承作为异常**  
   比如不要继承 `std::string`。  
   `std::exception` 有虚析构，所以继承 `std::runtime_error` 是安全的。

7. **注意 `what()` 的签名**  
   ```cpp
   virtual const char* what() const noexcept;
   ```
   如果你要重写，必须写：
   ```cpp
   const char* what() const noexcept override;
   ```

8. **优先组合，而非继承**  
   继承表示 is-a。异常类继承标准异常，是因为异常确实是一种标准异常。其他情况不要滥用继承。

9. **避免在头文件里 `using namespace std;`**  
   容易污染命名空间。写 `std::runtime_error` 更安全。

---



# 补充

## 一、如果你想不继承，只包含一个 `std::runtime_error` 成员，可以这样写

```cpp
#include <stdexcept>
#include <iostream>

class CE {
public:
    CE() : err_("BlockingQueue is closed") {}

    const char* what() const noexcept {
        return err_.what();
    }

private:
    std::runtime_error err_;
};
```

这里 `CE` 和 `std::runtime_error` 是 **组合关系**，也就是 has-a：
`CE` 有一个 `std::runtime_error` 成员。

用法：

```
int main() {
    try {
        throw CE();
    } catch (const CE& e) {
        std::cout << e.what() << '\n';
    }
}
```



但注意：这样 `CE` **不是** `std::exception` 的派生类，所以下面这种捕获抓不住：

cpp

```cpp
catch (const std::exception& e) {
    // 抓不住 CE
}
```

因为 `CE` 不是一种 `std::exception`。

------

## 二、如果想让 `CE` 能被 `catch (const std::exception&)` 捕获，可以让它继承 `std::exception`

```cpp
#include <stdexcept>
#include <iostream>

class CE : public std::exception {
public:
    CE() : err_("BlockingQueue is closed") {}

    const char* what() const noexcept override {
        return err_.what();
    }

private:
    std::runtime_error err_;
};
```

这样 `CE` 是一种 `std::exception`，可以被：

```cpp
catch (const std::exception& e) {
    std::cout << e.what() << '\n';
}
```

捕获。

但这样写比较绕：你既继承了 `std::exception`，又包含了一个 `std::runtime_error` 成员。
更简单的方式是直接继承 `std::runtime_error`：

```cpp
class CE : public std::runtime_error {
public:
    CE() : std::runtime_error("BlockingQueue is closed") {}
};
```

这就是你最开始看到的写法，只是类名从 `ClosedError` 改成了 `CE`。

------

## 三、继承和组合的区别

### 继承：is-a

```cpp
class CE : public std::runtime_error {
public:
    CE() : std::runtime_error("BlockingQueue is closed") {}
};
```

意思是：`CE` **是一种** `std::runtime_error`。
所以：

```cpp
CE e;
std::runtime_error& r = e;   // 合法
std::exception& ex = e;      // 合法
```

因此可以被 `catch (const std::runtime_error&)` 和 `catch (const std::exception&)` 捕获。

### 组合：has-a

```cpp
class CE {
public:
    CE() : err_("BlockingQueue is closed") {}
    const char* what() const noexcept { return err_.what(); }
private:
    std::runtime_error err_;
};
```

意思是：`CE` **有一个** `std::runtime_error` 成员。
但 `CE` 本身不是 `std::runtime_error`，也不是 `std::exception`。
所以 `catch (const std::exception&)` 抓不住它。

对于异常类，通常要的是 **is-a**，也就是继承标准异常。
所以推荐：

```cpp
class ClosedError : public std::runtime_error {
public:
    ClosedError() : std::runtime_error("BlockingQueue is closed") {}
};
```