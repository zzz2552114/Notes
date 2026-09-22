## 1. 由来：为什么需要读者-写者锁？

普通 `std::mutex` 很霸道：**同一时刻只允许一个线程访问共享数据**。不管你是读还是写，都得排队。

但现实里很多场景是**读多写少**：

- 配置表：程序天天读配置，偶尔才更新一次。
- 缓存：很多线程查缓存，少数线程写缓存。
- 数据库：多个查询可以同时跑，但更新要独占。

如果多个线程只是“读”，它们其实互不干扰，完全可以同时进行。让读读也排队，纯属浪费时间。

所以就有了**读者-写者锁**：

- **读锁（共享锁）**：多个线程可以同时拿。
- **写锁（独占锁）**：同一时刻只能一个线程拿，而且不能和任何读锁共存。

规则一句话：  
**读读共享，读写互斥，写写互斥。**

---

## 2. C++ 里是什么？

C++ 没有单独一个叫 `reader_writer_lock` 的类，但标准库给了三件套来模拟：

| 工具                | 作用                               |
| ------------------- | ---------------------------------- |
| `std::shared_mutex` | 底层互斥量，支持共享锁定和独占锁定 |
| `std::shared_lock`  | RAII 读锁，管理共享锁定            |
| `std::unique_lock`  | RAII 写锁，管理独占锁定            |

需要 C++17，头文件：

```cpp
#include <shared_mutex>  // shared_mutex, shared_lock
#include <mutex>         // unique_lock
```

---

## 3. 本质：不是“变成”，而是两套接口

你之前理解得大方向对，但有个关键点要纠正：

**`std::shared_mutex` 本身不是“变成”读锁或写锁，而是它同时提供两套成员函数。**

| 模式       | 加锁            | 解锁              | 谁能同时持有 |
| ---------- | --------------- | ----------------- | ------------ |
| 共享（读） | `lock_shared()` | `unlock_shared()` | 多个线程     |
| 独占（写） | `lock()`        | `unlock()`        | 一个线程     |

你可以手动调用：

```cpp
std::shared_mutex m;

m.lock_shared();   // 读锁
// 读操作
m.unlock_shared();

m.lock();          // 写锁
// 写操作
m.unlock();
```

但实际写代码时，我们不用手动，而是用 RAII 包装器：

- `std::shared_lock lk(m);` → 构造时调用 `m.lock_shared()`，析构时调用 `m.unlock_shared()`。
- `std::unique_lock lk(m);` → 构造时调用 `m.lock()`，析构时调用 `m.unlock()`。

所以，**用哪个包装器，就决定了用哪种锁定模式**。不是 `shared_mutex` 自己变来变去。

---

## 4. 怎么用？基本套路

读函数：

```cpp
std::shared_mutex m;

void read_data() {
    std::shared_lock lk(m);   // 拿读锁
    // 只读共享数据
}                             // 自动放读锁
```

写函数：

```cpp
void write_data() {
    std::unique_lock lk(m);   // 拿写锁
    // 修改共享数据
}                             // 自动放写锁
```

多个读线程可以同时进 `read_data()`；  
写线程进 `write_data()` 时，必须等所有读锁释放，而且写线程之间也互斥。

---

## 5. 例子一：之前的 `count` 程序

核心代码：

```cpp
int count = 0;
std::shared_mutex m;

void read_value() {
    std::shared_lock lk(m);
    std::cout << "Reading value " << count << "\n";
}

void write_value() {
    std::unique_lock lk(m);
    count += 3;
}
```

重点：

- `read_value()` 用 `shared_lock`，多个读线程可以同时读 `count`。
- `write_value()` 用 `unique_lock`，写的时候独占，其他读和写都得等。
- 两个写线程各加 3，最终 `count = 6`。
- 读线程可能读到 0、3、6，取决于调度顺序。

这里 `shared_lock` 只用于读，`unique_lock` 只用于写。这是约定，也是正确用法。

---

## 6. 例子二：配置表

更贴近实际：

```cpp
std::shared_mutex config_mutex;
std::map<std::string, std::string> config;

std::string get_config(const std::string& key) {
    std::shared_lock lk(config_mutex);   // 读锁
    auto it = config.find(key);
    if (it == config.end()) return "";
    return it->second;
}

void set_config(const std::string& key, const std::string& value) {
    std::unique_lock lk(config_mutex);   // 写锁
    config[key] = value;
}
```

多个线程同时 `get_config` 没问题；  
`set_config` 时会独占，保证不会读到写了一半的 map。

---

## 7. 重点注意事项（坑都在这儿）

1. **`shared_lock` 不能用于普通 `std::mutex`**  
   普通 `std::mutex` 没有 `lock_shared()`，编译会报错。  
   `shared_lock` 只能用于 `std::shared_mutex` 或 `std::shared_timed_mutex`。

2. **`unique_lock` 可以用于普通 `std::mutex`，也可以用于 `shared_mutex` 的写锁**  
   它比较通用。

3. **读锁里不要写数据**  
   `shared_lock` 只是“共享”，不是“安全”。如果你在读锁里修改数据，多个读线程同时写，照样数据竞争。  
   约定：`shared_lock` 保护只读，`unique_lock` 保护写。

4. **不要嵌套同一把 `shared_mutex`**  
   比如先拿 `shared_lock`，再想拿 `unique_lock`，会死锁：写锁等读锁释放，读锁是你自己持有的。  
   同一个线程也不能同时持有同一把 `shared_mutex` 的读锁和写锁。

5. **公平性不保证**  
   标准不保证读优先还是写优先。可能写者饥饿，也可能读者饥饿，取决于标准库实现。  
   如果业务对公平性敏感，要自己加机制。

6. **读多写少才划算**  
   `shared_mutex` 比普通 `mutex` 开销大。如果写很多、读很少，可能普通 `mutex` 更简单更快。

7. **C++17 才能用 `std::shared_mutex`**  
   编译要加 `-std=c++17 -pthread`。  
   C++14 可以用 `std::shared_timed_mutex`，功能类似，但支持超时。

8. **优先 RAII，别手动 lock/unlock**  
   手动容易忘记解锁，异常也不安全。  
   `shared_lock` 和 `unique_lock` 离开作用域自动解锁。

9. **锁的作用域要覆盖共享数据访问**  
   别只在声明处加锁，结果访问数据时锁已经释放了。

10. **如果数据初始化后从不修改，可以不加锁**  
    只读不写，且没有并发写，就不需要锁。但只要有写，读也必须加共享锁。

---

## 8. 和其他工具的关系

- **普通 `std::mutex`**：读读也互斥，简单但并发差。
- **`std::shared_mutex`**：读读共享，读写互斥，写写互斥，适合读多写少。
- **`std::atomic`**：适合单个变量的原子操作，不适合保护复杂数据结构。
- **`std::condition_variable`**：等待/通知机制，通常配 `unique_lock<std::mutex>`。它不能直接和 `shared_lock` 搭配；如果非要配共享锁，得用 `condition_variable_any`，但更复杂。

---

## 9. 一句话总结

**`std::shared_mutex` 是底层读写互斥量，同时提供共享和独占两套接口；`std::shared_lock` 是读锁的 RAII 包装器，`std::unique_lock` 是写锁的 RAII 包装器。读多写少时用它，读读并发，读写/写写互斥。用哪个包装器，就决定了用哪种锁。**