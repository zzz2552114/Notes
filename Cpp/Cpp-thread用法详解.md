## 1. `std::thread` 的构造函数和参数

`std::thread` 的构造函数大致长这样：

```cpp
template <class F, class... Args>
explicit thread(F&& f, Args&&... args);
```

- `F&& f`：线程要执行的可调用对象，可以是：
  - 普通函数
  - 函数指针
  - lambda 表达式
  - 函数对象
  - 成员函数指针
- `Args&&... args`：传给 `f` 的参数包，数量不限。

所以：

```cpp
std::thread t(worker, 42, std::string("x"));
```

拆开看：

| 部分               | 含义                       |
| ------------------ | -------------------------- |
| `std::thread`      | 线程类                     |
| `t`                | 线程对象                   |
| `worker`           | 新线程要执行的函数         |
| `42`               | 传给 `worker` 的第一个参数 |
| `std::string("x")` | 传给 `worker` 的第二个参数 |

如果 `worker` 定义成：

```cpp
void worker(int n, std::string s) {
    std::cout << n << " " << s << '\n';
}
```

那么新线程实际执行的就是：

```cpp
worker(42, std::string("x"));
```

输出类似：

```text
42 x
```

注意：新线程可能立刻开始执行，也可能稍后开始，这由操作系统调度决定。

---

## 2. 参数传递的细节：不是直接传引用

`std::thread` 构造时，会把 `worker`、`42`、`std::string("x")` **复制或移动**到线程内部存储中。这个过程叫 `decay-copy`。

简单理解：

1. `42` 被复制到线程内部存储。
2. `std::string("x")` 被移动到线程内部存储。
3. 新线程启动时，再把这些内部存储的参数传给 `worker`。
4. 传给 `worker` 时，参数是以**右值**形式传递的。

因此：

- 如果 `worker` 参数是 `std::string s`，会发生移动构造，效率高。
- 如果 `worker` 参数是 `const std::string&`，可以绑定到内部存储的临时对象，线程执行期间有效。
- 如果 `worker` 参数是 `std::string&` 非 const 左值引用，直接传 `std::string("x")` 会编译失败，因为临时对象不能绑定到非 const 左值引用。
- 如果你想传一个已有变量的引用，必须用 `std::ref`：

```cpp
void inc(int& x) { ++x; }

int x = 0;
std::thread t(inc, std::ref(x)); // 正确，传引用
t.join();
```

但要注意：`x` 必须在线程执行期间一直存活。

---

## 3. 返回值

### `std::thread` 构造函数的返回值

`std::thread` 的构造函数**没有返回值**。它只是构造一个线程对象，并启动线程。

### 线程函数的返回值

`worker` 如果有返回值，**返回值会被忽略**。

例如：

```cpp
int worker(int n, std::string s) {
    return n + s.size();
}

std::thread t(worker, 42, std::string("x")); // 返回值被丢弃
t.join();
```

如果你想获取线程函数的返回值，不能用 `std::thread` 直接拿。常用方案：

1. `std::async` + `std::future`
2. `std::packaged_task` + `std::thread`
3. `std::promise` + `std::future`

例如：

```cpp
#include <future>

std::future<int> fut = std::async(std::launch::async, worker, 42, std::string("x"));
int result = fut.get(); // 获取返回值
```

### 其他成员函数的返回值

| 函数                                  | 返回值            | 说明                    |
| ------------------------------------- | ----------------- | ----------------------- |
| `t.join()`                            | `void`            | 阻塞等待线程结束        |
| `t.detach()`                          | `void`            | 分离线程，让它后台运行  |
| `t.joinable()`                        | `bool`            | 是否可以 join 或 detach |
| `t.get_id()`                          | `std::thread::id` | 获取线程 ID             |
| `std::thread::hardware_concurrency()` | `unsigned int`    | 硬件并发线程数建议值    |

---

## 4. 用处是什么？

`std::thread` 的主要用处是**并发执行任务**：

- 利用多核 CPU 并行计算。
- 执行后台任务，比如日志、网络监听、定时任务。
- 把一个大任务拆成多个子任务同时跑。
- 与互斥量、条件变量配合，实现生产者消费者模型。
- 实现线程池的底层工作线程。

例如：

```cpp
std::thread t1(compute, 1);
std::thread t2(compute, 2);
t1.join();
t2.join();
```

这样 `compute(1)` 和 `compute(2)` 可能同时运行。

---

## 5. 常用用法

### 5.1 普通函数

```cpp
#include <iostream>
#include <thread>
#include <string>

void worker(int n, std::string s) {
    std::cout << "worker: " << n << ", " << s << '\n';
}

int main() {
    std::thread t(worker, 42, std::string("x"));
    t.join(); // 必须等待，否则 t 析构时可能 terminate
}
```

### 5.2 lambda 表达式

```cpp
std::thread t([](int n) {
    std::cout << "lambda: " << n << '\n';
}, 42);

t.join();
```

### 5.3 成员函数

```cpp
class MyClass {
public:
    void run(int n) {
        std::cout << "run: " << n << '\n';
    }
};

int main() {
    MyClass obj;
    std::thread t(&MyClass::run, &obj, 42);
    t.join();
}
```

也可以传对象引用：

```cpp
std::thread t(&MyClass::run, std::ref(obj), 42);
```

### 5.4 传递引用

```cpp
void add(int& x) {
    ++x;
}

int main() {
    int x = 0;
    std::thread t(add, std::ref(x));
    t.join();
    std::cout << x << '\n'; // 1
}
```

### 5.5 移动语义

```cpp
void take(std::unique_ptr<int> p) {
    std::cout << *p << '\n';
}

int main() {
    auto p = std::make_unique<int>(42);
    std::thread t(take, std::move(p));
    t.join();
}
```

### 5.6 detach：后台线程

```cpp
std::thread t([] {
    while (true) {
        // 后台任务
    }
});
t.detach(); // 分离，主线程不再等待它
```

注意：`detach` 后必须保证线程不会访问已经销毁的对象，否则是未定义行为。

### 5.7 获取线程 ID

```cpp
std::thread t(worker, 42, std::string("x"));
std::cout << "t id: " << t.get_id() << '\n';
std::cout << "main id: " << std::this_thread::get_id() << '\n';
t.join();
```

### 5.8 硬件并发数

```cpp
unsigned int n = std::thread::hardware_concurrency();
std::cout << "建议线程数: " << n << '\n';
```

---

## 6. 必须注意的坑

### 6.1 必须 join 或 detach

`std::thread` 对象析构时，如果它仍然 `joinable()`，程序会调用 `std::terminate()` 直接崩溃。

错误示例：

```cpp
void f() {
    std::thread t(worker, 42, std::string("x"));
} // t 析构，但未 join 或 detach -> terminate
```

正确：

```cpp
std::thread t(worker, 42, std::string("x"));
t.join();
```

或者：

```cpp
std::thread t(worker, 42, std::string("x"));
t.detach();
```

### 6.2 不可拷贝，只能移动

```cpp
std::thread t1(worker, 42, std::string("x"));
std::thread t2 = t1;             // 错误，不能拷贝
std::thread t2 = std::move(t1);  // 正确，移动
```

### 6.3 join 只能调用一次

`join()` 后 `joinable()` 变成 `false`，不能再次 `join()`。

### 6.4 线程函数异常未捕获会 terminate

线程函数中抛出的异常如果没有在线程内部捕获，会调用 `std::terminate()`。

```cpp
std::thread t([] {
    throw std::runtime_error("error");
});
t.join(); // 程序终止
```

所以线程函数内部最好自己 try-catch。

### 6.5 参数生命周期

`std::thread` 会复制或移动参数到内部存储，但如果你用 `std::ref` 传引用，就要保证原对象在线程执行期间一直活着。

```cpp
void f(int& x);

std::thread t(f, std::ref(local_x)); // local_x 不能提前销毁
```

### 6.6 detach 后的对象生命周期

```cpp
void f() {
    int x = 0;
    std::thread t([&] {
        std::cout << x << '\n'; // 危险！x 可能已经销毁
    });
    t.detach();
} // x 销毁，线程可能还在访问
```

---

## 7. 针对你这行代码的完整解释

```cpp
std::thread t(worker, 42, std::string("x"));
```

- 创建线程对象 `t`。
- 新线程执行 `worker`。
- `worker` 收到两个参数：`42` 和 `std::string("x")`。
- `42` 被复制到线程内部存储。
- `std::string("x")` 被移动到线程内部存储。
- 新线程启动后，把这些参数传给 `worker`。
- `worker` 的返回值如果有，会被丢弃。
- 必须调用 `t.join()` 或 `t.detach()`，否则 `t` 析构时程序会崩溃。

完整示例：

```cpp
#include <iostream>
#include <thread>
#include <string>

void worker(int n, std::string s) {
    std::cout << "worker: " << n << ", " << s << '\n';
}

int main() {
    std::thread t(worker, 42, std::string("x"));
    t.join(); // 等待新线程结束
    return 0;
}
```

输出：

```text
worker: 42, x
```

---

## 8. 一句话总结

`std::thread t(worker, 42, std::string("x"));` 就是：

> 启动一个新线程，执行 `worker(42, std::string("x"))`。  
> `std::thread` 本身不返回结果，线程函数的返回值会被忽略；  
> 必须 `join()` 或 `detach()`，否则程序会 terminate；  
> 要获取返回值请用 `std::async` 或 `std::future`。