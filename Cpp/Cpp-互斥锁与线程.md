## 互斥锁与线程

### 1. 进程和线程

- **进程**：一个正在运行的程序。比如你双击一个可执行文件，操作系统会创建一个进程。进程拥有自己的内存空间、文件句柄等资源。
- **线程**：进程内部的执行流。一个进程至少有一个线程，通常叫“主线程”。主线程执行 `main()` 函数。
- **多线程**：一个进程里同时有多个执行流在运行。它们可以“并发”或“并行”执行。

打个比方：

- 进程像一个工厂。
- 线程像工厂里的工人。
- 工厂里的白板、工具、仓库，多个工人可以共同使用，这些就是“共享资源”。

在这段代码里：

- `main()` 所在的是主线程。
- `std::thread t1(add_count);` 创建了一个新线程，这个新线程去执行 `add_count()`。
- `std::thread t2(add_count);` 又创建了一个新线程，也去执行 `add_count()`。
- 所以最后有三个线程：主线程、t1、t2。

### 2. 共享资源

多个线程在同一个进程里，它们共享：

- 全局变量
- 堆内存
- 静态变量
- 文件描述符等

但每个线程有自己的：

- 栈
- 寄存器状态
- 程序计数器

这段代码里：

```cpp
int count = 0;
std::mutex m;
```

`count` 和 `m` 都是全局变量，所以三个线程都能看到、都能访问同一个 `count` 和同一个 `m`。

### 3. 为什么共享资源会出问题？

因为 `count += 1;` 看起来是一行代码，但在 CPU 和编译器看来，它通常不是“一步完成”的，而是分成几步：

1. 从内存读取 `count` 到寄存器。
2. 寄存器里的值加 1。
3. 把结果写回内存中的 `count`。

如果两个线程同时执行，可能出现这种情况：

- 初始 `count = 0`。
- 线程 t1 读取 `count`，得到 0。
- 线程 t2 也读取 `count`，也得到 0。
- t1 计算 0 + 1 = 1，写回 `count`，`count = 1`。
- t2 计算 0 + 1 = 1，写回 `count`，`count = 1`。

两个线程都加了 1，但最终 `count` 却是 1，而不是 2。这叫**丢失更新**。

更正式地说，这叫**数据竞争**：两个线程同时访问同一个非原子对象，至少有一个是写操作，而且没有同步措施。数据竞争在 C++ 中是**未定义行为**，可能结果错误，也可能看起来正常，但不可依赖。

### 4. 锁、互斥锁、mutex 是什么？

- **锁**：一个抽象概念。进入某个需要保护的代码前“加锁”，离开时“解锁”。加锁后，其他人不能同时进入。
- **互斥锁**：英文是 mutex，全称 mutual exclusion，意思是“互相排斥”。它保证同一时刻最多只有一个线程能持有这把锁。
- **`std::mutex`**：C++ 标准库提供的互斥锁类。你可以创建一个 `std::mutex` 对象，然后调用它的 `lock()` 和 `unlock()`。
- **临界区**：被锁保护起来、不能同时被多个线程执行的那段代码。

用生活比喻：

- 厕所只有一个坑位，门上有锁。
- 一个人进去后锁门。
- 其他人来了发现锁着，只能在门外等。
- 里面的人出来后开锁，下一个人才可能进去。

在这段代码里：

- `m` 就是那把锁。
- `m.lock()` 就是进厕所锁门。
- `count += 1;` 就是在厕所里做事。
- `m.unlock()` 就是开门出来。

### 5. 原子操作

“原子”在计算机里意思是“不可分割的操作”。一个原子操作要么全部完成，要么完全没发生，中间不会被其他线程观察到。

`count += 1;` 本身不是原子操作，因为它包含读、加、写。但在这段代码里，`count += 1;` 被 `m.lock()` 和 `m.unlock()` 包住了。对于所有也使用同一个 `m` 的线程来说，这段代码表现得像原子操作：同一时间只有一个线程能执行它。

所以代码注释说“原子地将 count 变量增加 1”，更准确的理解是：

> 通过互斥锁保护，使得对同样使用这把锁的线程来说，这个自增过程表现为不可分割。

但严格说，`count += 1;` 这条语句本身不是原子操作。

### 6. `std::thread` 是什么？

`std::thread` 是 C++11 标准库提供的线程类。它用来创建和管理线程。

用法：

```cpp
std::thread t(可调用对象);
```

构造 `std::thread` 对象时，如果传入了函数名，就会启动一个新线程，并在这个新线程里执行这个函数。

在这段代码里：

```cpp
std::thread t1(add_count);
```

意思是：创建一个名为 `t1` 的线程对象，并让新线程执行 `add_count()` 函数。

`join()` 的作用：

```cpp
t1.join();
```

意思是：当前线程，也就是主线程，阻塞在这里，等待 `t1` 执行完毕。`t1` 结束后，主线程才继续往下走。

如果没有 `join()`，主线程可能不等 t1、t2 完成，就直接打印 `count`，那结果可能是不完整的。

---

## 二、这段程序的由来和目的

在单线程程序里，代码从上到下顺序执行，不需要考虑“两个执行流同时改同一个变量”的问题。

但现代 CPU 是多核的，程序也常常需要同时做多件事，比如：

- 一边下载文件，一边更新界面。
- 一边处理网络请求，一边写日志。
- 一边计算，一边等待用户输入。

多线程能提高效率，但也会带来共享资源竞争问题。

C++11 之前，C/C++ 在 Unix/Linux 上通常用 POSIX 线程库，也就是 `pthread`。C++11 之后，标准库引入了：

- `std::thread`
- `std::mutex`
- `std::lock_guard`
- `std::atomic`
- `std::condition_variable`

这段程序就是展示 `std::mutex` 的最简单例子：两个线程各自把 `count` 加 1，用锁保证结果正确。

---

## 三、逐字逐句解释代码

完整代码：

```cpp
#include <iostream>
#include <mutex>
#include <thread>

int count = 0;

std::mutex m;

void add_count() {
  m.lock();
  count += 1;
  m.unlock();
}

int main() {
  std::thread t1(add_count);
  std::thread t2(add_count);
  t1.join();
  t2.join();

  std::cout << "Printing count: " << count << std::endl;
  return 0;
}
```

下面逐部分解释。

### 注释

```cpp
// std::mutex类提供了互斥锁同步原语。
```

“同步原语”是指操作系统或标准库提供的基本同步工具。常见同步原语有：

- 互斥锁 mutex
- 信号量 semaphore
- 条件变量 condition variable
- 原子操作 atomic

### 定义全局互斥锁 m

```cpp
std::mutex m;
```

- `std`：C++ 标准库命名空间。标准库里的名字通常都放在 `std` 里。
- `mutex`：互斥锁类。
- `m`：这个互斥锁对象的名字。
- `;`：语句结束。

这行声明并默认初始化了一个全局 `std::mutex` 对象 `m`。

默认构造的 `std::mutex` 处于“未锁定”状态。

它也是全局变量，所以两个线程用的是**同一把锁**。这一点非常关键。如果每个线程各有一把锁，就锁不住同一个共享变量。

### add_count 函数

```cpp
void add_count() {
```

这个函数将被两个线程执行。

```cpp
  m.lock();
```

调用 `m` 的成员函数 `lock()`。

作用：

- 尝试获取锁。
- 如果锁当前没有被其他线程持有，当前线程就获得锁，继续往下执行。
- 如果锁已经被其他线程持有，当前线程就阻塞，也就是停在这里等待，直到锁被释放。

“阻塞”可以理解为：这个线程暂时不能继续执行，操作系统会把它挂起，等条件满足再唤醒。

```cpp
  count += 1;
```

由于它被夹在 `m.lock()` 和 `m.unlock()` 之间，所以同一时刻只有一个线程能执行它。

这就是临界区。

```cpp
  m.unlock();
```

释放锁。

作用：

- 当前线程不再持有 `m`。
- 如果有其他线程正在等待这把锁，它们中的一个可能被唤醒并获得锁。
- 当前线程继续往下执行，离开 `add_count()`。

注意：手动 `lock()` 和 `unlock()` 必须成对出现。如果中间抛出异常，`unlock()` 可能不会执行，导致锁永远不释放，其他线程永远等待。本例中 `count += 1` 不会抛异常，所以没问题。但实际工程中更推荐 `std::lock_guard`。

### 6. main 函数

```cpp
  std::thread t1(add_count);
```

创建一个 `std::thread` 对象，名字叫 `t1`。

`add_count` 是函数名。这里把它作为新线程要执行的可调用对象。

这行执行后：

- 新线程被创建。
- 新线程开始执行 `add_count()`。
- 主线程继续执行下一行。

注意：新线程不一定立刻运行，具体什么时候运行由操作系统调度决定。

```cpp
  std::thread t2(add_count);
```

再创建一个线程 `t2`，也执行 `add_count()`。

现在：

- 主线程在 `main` 里继续走。
- t1 在执行 `add_count()`。
- t2 也在执行 `add_count()`。

t1 和 t2 会竞争同一个全局锁 `m`。

```cpp
  t1.join();
```

`join()` 的意思是：主线程等待 `t1` 结束。

主线程执行到这一行时，如果 t1 还没执行完，主线程就会阻塞在这里，直到 t1 完成。

```cpp
  t2.join();
```

主线程再等待 `t2` 结束。

有了这两个 `join()`，主线程一定会在 t1 和 t2 都完成后，才继续往下执行。

如果一切正常，这里会输出：

```text
Printing count: 2
```

---

## 四、这个程序实际运行过程

假设初始状态：

```text
count = 0
m 未锁定
```

可能的时间线如下：

1. 主线程执行 `main`。
2. 创建 t1，t1 开始执行 `add_count()`。
3. 创建 t2，t2 也开始执行 `add_count()`。
4. t1 和 t2 都调用 `m.lock()`。
5. 假设 t1 先抢到锁：
   - t1 获得 `m`。
   - t2 调用 `m.lock()` 时发现锁已被占用，于是阻塞等待。
6. t1 执行 `count += 1;`
   - 读取 `count`，得到 0。
   - 加 1，得到 1。
   - 写回 `count`，`count = 1`。
7. t1 执行 `m.unlock();`
   - 释放锁。
   - t2 可能被唤醒。
8. t2 获得锁。
9. t2 执行 `count += 1;`
   - 读取 `count`，得到 1。
   - 加 1，得到 2。
   - 写回 `count`，`count = 2`。
10. t2 执行 `m.unlock();`
11. 主线程在 `t1.join()` 和 `t2.join()` 处等待，直到两个线程都结束。
12. 主线程打印 `count`，得到 2。

如果 t2 先抢到锁，过程对称，最终结果也是 2。

所以最终输出：

```text
Printing count: 2
```

---

## 六、锁的保护对象到底是什么？

规则是：

> 任何线程想修改 `count`，都必须先获得全局互斥锁 `m`。

因为两个线程都用了同一个 `m`，所以它们互斥。

如果有一个线程用了 `m`，另一个线程没用 `m`，那还是可能竞争。

如果两个线程各用不同的 mutex，也锁不住同一个 `count`。

所以关键点：

- 锁必须是同一把。
- 所有访问共享资源的地方都要加锁。
- 锁的范围要覆盖共享资源的读写。

---

## 八、几个容易混淆的点

### 1. `std::mutex` 不能复制

```cpp
std::mutex m2 = m; // 错误
```

互斥锁通常不允许复制。因为它代表一个唯一的锁状态，复制没有意义。

### 2. `std::thread` 也不能随便复制

`std::thread` 表示一个线程的所有权，通常只能移动，不能复制。

### 3. `join()` 必须调用，或者 `detach()`

如果一个 `std::thread` 对象仍然关联着可 join 的线程，而它被析构了，程序会调用 `std::terminate` 终止。

所以要么：

```cpp
t.join();
```

等待线程结束。

要么：

```cpp
t.detach();
```

让线程在后台独立运行。

但 `detach()` 后就不能再控制它，主线程也可能先结束，容易出问题。本例用 `join()` 是正确的。

### 4. 编译时需要 C++11 或更高，并链接线程库

例如在 Linux 上用 g++：

```bash
g++ -std=c++11 -pthread main.cpp -o main
```

`-pthread` 是链接 POSIX 线程库的选项。`std::thread` 在类 Unix 系统上通常基于 pthread 实现。

---



## RAII智能锁

在你之前的例子里，`std::lock_guard` 和 `std::scoped_lock` **都可以用**，效果完全一样。不过它们的设计目标和适用场景有区别。

### 核心区别：单锁 vs. 多锁

两者都是 RAII 风格的锁管理工具，构造时加锁，析构时自动解锁。

*   **`std::lock_guard`**：C++11 引入，**专为锁定单个互斥量设计**。它只能管理一把锁，没有无参构造函数。
*   **`std::scoped_lock`**：C++17 引入，是 `lock_guard` 的**增强版**。它支持**可变参数**，可以同时锁定多个互斥量，并使用死锁避免算法（与 `std::lock` 相同）来安全地获取所有锁。

### 为什么之前的例子两者都能用？

因为你的例子只保护一个全局互斥量 `m`。`std::scoped_lock` 在只接收一个互斥量时，其行为与 `lock_guard` 完全相同：构造时调用 `m.lock()`，析构时调用 `m.unlock()`。

两种写法如下：

**使用 `std::lock_guard`：**
```cpp
void add_count() {
  std::lock_guard<std::mutex> lock(m);
  count += 1;
}
```

**使用 `std::scoped_lock`：**
```cpp
void add_count() {
  std::scoped_lock lock(m);  // C++17 后模板参数可自动推导
  count += 1;
}
```

### 一个重要的安全差异

在只锁一把锁的场景下，`lock_guard` 有一个细微的**安全性优势**。

如果误写成 `std::scoped_lock lock;`（没有传入互斥量），这在 `scoped_lock` 中是合法的（因为它支持零个互斥量），但**什么也不会做**，锁不会被创建，导致共享资源失去保护，且编译器不会报错。

而如果误写成 `std::lock_guard lock;`，**编译会直接失败**，因为 `lock_guard` 没有无参构造函数，这能让你在编译期就发现错误。

### 总结：应该用哪个？

*   **只锁一把互斥量**：用 `std::lock_guard` 即可。它更简单，且能避免“忘记传入互斥量”的运行时错误。
*   **需要同时锁多把互斥量**：必须用 `std::scoped_lock`。这是它存在的核心意义，能自动避免死锁。

在新代码中，如果经常需要锁多把锁，直接统一用 `std::scoped_lock` 也很方便；如果只锁一把，`lock_guard` 是更稳妥的选择。



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