## 1. `std::atomic` 是什么

`std::atomic<T>` 是 C++11 引入的模板类，表示一个原子类型。

```cpp
std::atomic<int> c{0};
std::atomic<bool> flag{false};
std::atomic<long> count{0};
std::atomic<size_t> progress{0};
```

对它的操作：

- `load()`：原子读
- `store()`：原子写
- `fetch_add()`：原子加
- `fetch_sub()`：原子减
- `exchange()`：原子交换
- `compare_exchange_weak/strong()`：原子比较并交换
- `++`、`--`、`+=`、`-=`、`&=`、`|=`、`^=` 等运算符

这些操作不会产生数据竞争。

数据竞争的定义是：

> 多个线程同时访问同一个非原子对象，至少有一个是写，并且没有同步，就是数据竞争。  
> 数据竞争导致未定义行为。

`std::atomic` 通过原子操作避免了这个问题。

---

## 2. 为什么需要 `std::atomic`

假设多个线程同时给一个普通 `int` 加一：

```cpp
int count = 0;

void worker() {
    for (int i = 0; i < 100000; ++i) {
        ++count; // 错误：数据竞争
    }
}
```

`++count` 看起来是一行，实际可能是：

1. 从内存读 `count` 到寄存器
2. 寄存器加 1
3. 写回内存

两个线程可能同时读到相同的值，然后各自加 1 写回，导致丢失更新。  
最终结果小于预期，甚至是未定义行为。

用 `mutex` 可以解决：

```cpp
int count = 0;
std::mutex m;

void worker() {
    for (int i = 0; i < 100000; ++i) {
        std::lock_guard<std::mutex> lk(m);
        ++count;
    }
}
```

但每次自增都要加锁解锁，开销大。

用 `atomic`：

```cpp
std::atomic<int> count{0};

void worker() {
    for (int i = 0; i < 100000; ++i) {
        ++count;
    }
}
```

更简单，通常也更快。

---

## 3. 声明、初始化和模板参数

```cpp
std::atomic<int> c{0};
```

- `std::atomic` 是模板。
- `int` 是模板参数，表示被包装的类型。
- `c` 是对象名。
- `{0}` 是初始值。

要求：`T` 必须是可平凡复制类型，简单理解就是“可以像 `int` 一样按位复制”的类型。  
`int`、`long`、`bool`、指针都可以。  
自定义复杂类型通常不行，除非满足条件。

注意：

```cpp
std::atomic<int> a; // C++20 前可能未初始化
std::atomic<int> b{0}; // 推荐
```

`std::atomic` 不可拷贝、不可移动：

```cpp
std::atomic<int> a{0};
std::atomic<int> b = a; // 错误
std::atomic<int> c = std::move(a); // 错误
```

所以不能把它放进需要拷贝的容器里，也不能按值传参。

---

## 4. 常见操作和返回值

### 4.1 `load()`：原子读

```cpp
std::atomic<int> c{42};
int v = c.load();
```

返回当前值。  
默认内存序 `seq_cst`。

### 4.2 `store()`：原子写

```cpp
c.store(0);
```

无返回值。  
默认内存序 `seq_cst`。

### 4.3 `fetch_add()`：原子加，返回旧值

```cpp
std::atomic<int> c{5};
int old = c.fetch_add(1); // old == 5, c == 6
```

### 4.4 `++c`：原子自增，返回新值

```cpp
std::atomic<int> c{5};
int now = ++c; // now == 6, c == 6
```

### 4.5 `--c`、`+=`、`-=`

```cpp
--c;
c += 10;
c -= 3;
```

都是原子操作。

### 4.6 `exchange()`：原子交换，返回旧值

```cpp
std::atomic<int> c{5};
int old = c.exchange(10); // old == 5, c == 10
```

### 4.7 `compare_exchange_strong()`：比较并交换

```cpp
std::atomic<int> c{0};
int expected = 0;

bool ok = c.compare_exchange_strong(expected, 1);
```

含义：

- 如果 `c` 当前值等于 `expected`，就把 `c` 改成 `1`，返回 `true`。
- 否则，把 `c` 当前值写入 `expected`，返回 `false`。

常用于实现无锁的“检查+修改”：

```cpp
std::atomic<int> c{0};

void set_if_zero() {
    int expected = 0;
    if (c.compare_exchange_strong(expected, 1)) {
        // 成功从 0 改成 1
    } else {
        // 已经被别人改了，expected 现在是当前值
    }
}
```

`compare_exchange_weak` 类似，但可能“伪失败”，通常放在循环里。

### 4.8 `is_lock_free()`

```cpp
bool lock_free = c.is_lock_free();
```

判断这个原子类型在当前平台是否真正无锁。  
不是所有 `std::atomic<T>` 都保证无锁。  
`std::atomic_flag` 保证无锁。

---

## 5. 常用用法

### 5.1 计数器

```cpp
std::atomic<long> done{0};

void worker() {
    // 做任务
    done.fetch_add(1);
}

int main() {
    std::vector<std::thread> ts;
    for (int i = 0; i < 4; ++i) {
        ts.emplace_back(worker);
    }
    for (auto& t : ts) t.join();

    std::cout << done.load() << '\n';
}
```

### 5.2 停止标志

```cpp
std::atomic<bool> stop{false};

void worker() {
    while (!stop.load()) {
        // 做一点工作
    }
}

int main() {
    std::thread t(worker);
    // ...
    stop.store(true);
    t.join();
}
```

### 5.3 进度报告

```cpp
std::atomic<size_t> progress{0};

void worker() {
    for (size_t i = 0; i < 1000; ++i) {
        // ...
        progress.store(i + 1);
    }
}
```

### 5.4 一次性标志

```cpp
std::atomic<bool> initialized{false};

void init() {
    bool expected = false;
    if (initialized.compare_exchange_strong(expected, true)) {
        // 只有第一个线程能进来
    }
}
```

### 5.5 自旋锁

```cpp
class SpinLock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // 自旋
        }
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
};
```

初学先了解即可，实际优先用 `std::mutex`。

---

## 6. 什么时候用，什么时候不能用

### 能用

- 单个计数器
- 单个布尔标志
- 单个进度值
- 引用计数
- 无锁数据结构的底层原子字段

### 不能用

- 多个变量必须一起保持一致
- 队列的 `size` 和内容
- 链表的指针和长度
- 哈希表的桶和元素数量
- 任何需要“事务性”修改多个字段的场景

这些必须用 `mutex`，或者非常复杂的无锁算法。

---

## 7. 一个完整例子：多线程计数

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

std::atomic<int> counter{0};

void worker() {
    for (int i = 0; i < 100000; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::vector<std::thread> threads;

    for (int i = 0; i < 4; ++i) {
        threads.emplace_back(worker);
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << counter.load() << '\n';
    return 0;
}
```

输出应该是：

```text
400000
```

这里用了 `memory_order_relaxed`，因为只是计数，不需要同步其他内存。  
但如果你还不熟内存序，就写：

```cpp
counter.fetch_add(1);
```

用默认的 `seq_cst`，最安全。

---

## 8. 内存序：为什么默认 `seq_cst` 最好

`std::atomic` 的操作可以带一个内存序参数：

```cpp
c.load(std::memory_order_seq_cst);
c.store(0, std::memory_order_seq_cst);
c.fetch_add(1, std::memory_order_seq_cst);
```

常见内存序：

| 内存序                 | 含义                                     |
| ---------------------- | ---------------------------------------- |
| `memory_order_seq_cst` | 顺序一致，最严格，默认                   |
| `memory_order_acquire` | 读操作，保证后面的读写不会被重排到它前面 |
| `memory_order_release` | 写操作，保证前面的读写不会被重排到它后面 |
| `memory_order_relaxed` | 只保证原子性，不保证顺序和同步           |
| `memory_order_acq_rel` | acquire + release                        |
| `memory_order_consume` | 不推荐，很少用                           |

初学原则：

> **全部用默认的 `seq_cst`。**  
> 不要碰 `relaxed`、`acquire`、`release`，除非你真正理解 `happens-before` 和内存重排序。

---

## 9. 常见坑

### 9.1 以为 `atomic` 能保护多个变量

错误：

```cpp
std::atomic<size_t> size_;
std::queue<int> q_;
```

`size_` 原子，但 `q_` 不是。  
`q_` 和 `size_` 之间仍然可能不一致。  
必须用 `mutex` 同时保护。

### 9.2 复合逻辑不是原子的

错误：

```cpp
if (c.load() == 0) {
    c.store(1);
}
```

两个线程可能都进入。  
正确用 CAS：

```cpp
int expected = 0;
if (c.compare_exchange_strong(expected, 1)) {
    // 成功
}
```

### 9.3 默认构造可能不初始化

```cpp
std::atomic<int> c; // C++20 前值不确定
```

推荐：

```cpp
std::atomic<int> c{0};
```

### 9.4 不可拷贝

```cpp
std::atomic<int> a{0};
std::atomic<int> b = a; // 错误
```

### 9.5 不是所有平台都无锁

```cpp
std::atomic<BigStruct> x;
```

可能内部用锁实现。  
需要无锁时检查 `is_lock_free()`。

### 9.6 `atomic` 不能替代条件变量

`atomic` 可以轮询，但轮询浪费 CPU。  
等待/通知还是要用 `condition_variable` + `mutex`。

---

## 10. 针对你那段文字的最终总结

你那段话可以浓缩成：

1. `std::atomic<T>` 提供原子操作，避免数据竞争。
2. 对单个计数器、标志、进度，`atomic` 比 `mutex` 更轻、更快。
3. 常见操作：`fetch_add`、`++`、`load`、`store`。
4. 复合逻辑“检查+修改”不能靠 `load` + `store`，要用 `compare_exchange` 或 `mutex`。
5. 多个变量必须一起保持一致时，`atomic` 不够，必须用 `mutex`。
6. 默认内存序 `seq_cst` 最安全；高级内存序不懂就别碰。

一句话：

> `std::atomic` 是给“单个变量”用的无锁原子工具。  
> 它快、简单，但只保证单个变量原子。  
> 一旦涉及多个变量的不变式，就回到 `mutex`。