## 锁的另一个包装器，以及锁的cv

先建立一个整体认识：

- **`std::unique_lock`**：是互斥锁的“智能包装器”，比 `lock_guard` 更灵活。它能延迟加锁、手动解锁、转移所有权，最重要的是——**`std::condition_variable` 必须用它**。
- **`std::condition_variable`**：是“条件变量”，用来让一个线程等待某个条件成立，另一个线程在条件成立时通知它。它本身不存储条件，条件由共享变量表示，必须和互斥锁配合使用。

下面从由来、原理、用法，再到具体代码逐字逐句讲。

---

## 一、`std::unique_lock` 是什么？

### 1. 由来

`std::lock_guard` 是最简单的 RAII 锁：构造时加锁，析构时解锁。但它不够灵活：

- 不能手动解锁。
- 不能延迟加锁。
- 不能转移所有权。
- 不能配合 `condition_variable` 的 `wait`。

于是标准库提供了 `std::unique_lock`，它是更通用的互斥锁包装器。

### 2. 原理

`std::unique_lock` 也是 RAII 类，但内部多了一个状态：**是否持有锁**。

- 构造时可以立即加锁，也可以不加锁。
- 析构时，如果它持有锁，就自动解锁；如果没持有，就什么都不做。
- 它支持移动语义，可以把锁的所有权从一个 `unique_lock` 转移到另一个。
- 它不能复制。

可以把它想象成一把“可拆卸的锁把手”：你可以拿着它，也可以暂时不拿，还可以把它交给别人。

### 3. 基本用法

头文件：

```cpp
#include <mutex>
```

常见构造方式：

```cpp
std::mutex m;

// 1. 构造时立即加锁
std::unique_lock<std::mutex> lock1(m);

// 2. 延迟加锁：构造时不加锁
std::unique_lock<std::mutex> lock2(m, std::defer_lock);
// 之后手动加锁
lock2.lock();

// 3. 接管已经加锁的互斥量
m.lock();
std::unique_lock<std::mutex> lock3(m, std::adopt_lock);

// 4. 尝试加锁，不阻塞
std::unique_lock<std::mutex> lock4(m, std::try_to_lock);
if (lock4.owns_lock()) {
    // 加锁成功
}
```

常用成员函数：

- `lock()`：加锁。
- `unlock()`：解锁。
- `try_lock()`：尝试加锁，不阻塞。
- `owns_lock()`：返回是否持有锁。
- `release()`：释放与互斥量的关联，但**不解锁**，返回互斥量指针。之后 `unique_lock` 不再管理它。
- `operator bool()`：等价于 `owns_lock()`。

移动语义：

```cpp
std::unique_lock<std::mutex> lock1(m);
std::unique_lock<std::mutex> lock2 = std::move(lock1);
// 现在 lock2 持有锁，lock1 不再持有
```

### 5. 在之前 `add_count` 例子中的情况

之前的代码：

```cpp
void add_count() {
  m.lock();
  count += 1;
  m.unlock();
}
```

可以改成 `unique_lock`：

```cpp
void add_count() {
  std::unique_lock<std::mutex> lock(m);
  count += 1;
}
```

也可以改成 `lock_guard`：

```cpp
void add_count() {
  std::lock_guard<std::mutex> lock(m);
  count += 1;
}
```

这里两者都能用，但 **`lock_guard` 更简单、开销更小**。因为这里不需要手动解锁，也不需要延迟加锁，更不需要配合条件变量。所以 `unique_lock` 在这里是“杀鸡用牛刀”。

---

## 二、`std::condition_variable` 是什么？

### 1. 由来

多线程编程中经常有这样的需求：

- 消费者线程发现队列为空，需要等待。
- 生产者线程往队列里放了数据，需要通知消费者。

如果消费者不停地循环检查队列是否为空，这叫**忙等待**，会浪费 CPU。更好的办法是：消费者线程睡眠，等生产者通知它再醒来。这就是条件变量。

C++11 把 POSIX 的 `pthread_cond_t` 封装成了 `std::condition_variable`，头文件：

```cpp
#include <condition_variable>
```

### 2. 原理

`std::condition_variable` 内部维护一个等待队列。基本操作：

- `wait`：当前线程等待，直到被通知。
- `notify_one`：唤醒一个等待线程。
- `notify_all`：唤醒所有等待线程。

关键点：**`wait` 必须和一个互斥锁配合使用**。为什么？

因为“条件”通常由共享变量表示，比如队列是否为空。检查这个条件需要锁保护。而且 `wait` 必须原子地连续完成两件事：

1. 释放互斥锁。
2. 让线程进入睡眠。

如果先解锁，经过一些操作再睡眠，中间可能被其他线程插入：其他线程修改了条件并发出通知，但当前线程还没睡，通知就丢失了，当前线程会永远睡下去。所以 `wait` 必须原子地“解锁并睡眠”。

当线程被唤醒后，`wait` 会重新获取互斥锁，然后返回。这样线程醒来时，又持有锁，可以安全地重新检查条件。

### 3. 基本用法

典型模式：

```cpp
std::mutex m;
std::condition_variable cv;
bool ready = false;

// 等待线程
void wait_thread() {
    std::unique_lock<std::mutex> lock(m);
    cv.wait(lock, [] { return ready; });
    // 条件成立，继续执行
}

// 通知线程
void notify_thread() {
    {
        std::lock_guard<std::mutex> lock(m);
        ready = true;
    }
    cv.notify_one();
}
```

关键函数：

- `wait(lock)`：解锁并等待，被唤醒后重新加锁。可能虚假唤醒。
- `wait(lock, pred)`：等价于 `while (!pred()) wait(lock);`，处理虚假唤醒。推荐用这个。
- `wait_for(lock, duration, pred)`：等待一段时间。
- `wait_until(lock, time_point, pred)`：等待到某个时间点。
- `notify_one()`：唤醒一个等待线程。
- `notify_all()`：唤醒所有等待线程。

**虚假唤醒**：操作系统可能在没有 `notify` 的情况下唤醒等待线程。所以必须用循环检查谓词。`wait(lock, pred)` 内部已经帮你循环了。

### 5. 生产者消费者例子逐字逐句讲解

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <condition_variable>
#include <queue>

std::mutex m;
std::condition_variable cv;
std::queue<int> q;

void producer() {
    for (int i = 0; i < 5; ++i) {
        {
            std::lock_guard<std::mutex> lock(m);
            q.push(i);
            std::cout << "Produced " << i << std::endl;
        }
        cv.notify_one();
    }
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(m);
        cv.wait(lock, [] { return !q.empty(); });
        int value = q.front();
        q.pop();
        lock.unlock();
        std::cout << "Consumed " << value << std::endl;
        if (value == 4) break;
    }
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
```

逐行解释：

- `std::mutex m;`：保护共享队列 `q`。
- `std::condition_variable cv;`：用于生产者和消费者之间的通知。
- `std::queue<int> q;`：共享队列，生产者放数据，消费者取数据。

生产者：

```cpp
for (int i = 0; i < 5; ++i) {
    {
        std::lock_guard<std::mutex> lock(m);
        q.push(i);
        std::cout << "Produced " << i << std::endl;
    }
    cv.notify_one();
}
```

- 循环 5 次，生产 0 到 4。
- `{ ... }` 是一个作用域，`lock_guard` 在这里构造时加锁，离开作用域时自动解锁。
- `q.push(i);` 修改共享队列，必须在锁保护下。
- 解锁后调用 `cv.notify_one();` 通知一个等待的消费者。通知可以在锁内也可以在锁外，这里放锁外是为了减少消费者被唤醒后立刻又阻塞在锁上的情况。

消费者：

```cpp
while (true) {
    std::unique_lock<std::mutex> lock(m);
    cv.wait(lock, [] { return !q.empty(); });
    int value = q.front();
    q.pop();
    lock.unlock();
    std::cout << "Consumed " << value << std::endl;
    if (value == 4) break;
}
```

- `std::unique_lock<std::mutex> lock(m);`：加锁。这里必须用 `unique_lock`，因为 `cv.wait` 需要它。
- `cv.wait(lock, [] { return !q.empty(); });`：
  - 如果 `q` 非空，谓词为真，`wait` 立即返回，不睡眠。
  - 如果 `q` 为空，谓词为假，`wait` 原子地解锁 `m`，然后阻塞当前线程。
  - 当被 `notify_one` 唤醒后，`wait` 重新获取锁，再次检查谓词。如果还是假，继续等待。
  - 这样即使有虚假唤醒，也会重新检查队列是否非空。
- `int value = q.front(); q.pop();`：取出数据。此时仍持有锁，安全。
- `lock.unlock();`：手动解锁。因为后面打印不需要保护队列。手动解锁是 `unique_lock` 的能力之一，`lock_guard` 做不到。
- `std::cout << "Consumed " << value << std::endl;`：打印。
- `if (value == 4) break;`：取到 4 就退出循环。

主函数：

- 创建生产者线程 `t1` 和消费者线程 `t2`。
- `t1.join(); t2.join();` 等待两个线程结束。

这个例子展示了 `condition_variable` 的典型用法：**等待方用 `unique_lock` + `wait`，通知方修改共享数据后 `notify_one`。**

---

## 三、总结对比

| 工具                          | 作用                       | 必须配合                           | 典型场景                         |
| ----------------------------- | -------------------------- | ---------------------------------- | -------------------------------- |
| `std::lock_guard`             | 简单 RAII 锁               | 无                                 | 简单临界区                       |
| `std::unique_lock`            | 灵活 RAII 锁               | 无，但 `condition_variable` 需要它 | 延迟加锁、手动解锁、配合条件变量 |
| `std::scoped_lock`            | 同时锁多个互斥量，避免死锁 | 无                                 | 需要一次锁多把锁                 |
| `std::condition_variable`     | 等待/通知                  | `std::unique_lock<std::mutex>`     | 生产者消费者、任务队列           |
| `std::condition_variable_any` | 等待/通知                  | 任何 `BasicLockable`               | 需要配合非 `std::mutex` 的锁     |

一句话记住：

- **`unique_lock` 是锁的灵活包装器。**
- **`condition_variable` 是等待/通知机制，必须和 `unique_lock<std::mutex>` 一起用。**
- **之前 `add_count` 那个简单例子，用 `lock_guard` 就够了；`unique_lock` 和 `condition_variable` 是为更复杂的同步场景准备的。**