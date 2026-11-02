# Python 异步、协程、事件循环与线程学习笔记

这份笔记从一个查询文章列表的函数出发，依次解释：

1. 什么情况下需要 `async`。
2. 协程如何创建、执行、暂停和恢复。
3. `await`、`Task`、事件循环分别负责什么。
4. 为什么一路 `await` 下去，可能仍然没有并发收益。
5. 协程任务与线程有什么联系和区别。
6. 异步为什么适合处理等待，以及怎样与多线程、多核结合。
7. 在普通脚本和 FastAPI 中分别应该怎么使用。

文中的主要讨论范围是 **Python 的** `**asyncio**` **模型**。其他语言和异步运行时可能采用不同的调度方式。

## 1．从原函数出发：这里的 `async` 有必要吗？

原函数：

```
async def list_public_posts(skip: int, limit: int):
    tmp = []

    for post in POSTS:
        if post.get("status") == "published":
            tmp.append(post)

    # Python 切片的边界超出列表范围，不会因此报错
    return tmp[skip:skip + limit]
```

假设 `POSTS` 是已经保存在内存中的普通 Python 列表，这个函数只做了：

```
读取内存数据
    ↓
遍历列表
    ↓
判断文章状态
    ↓
构造新列表
    ↓
切片并返回
```

**这里没有等待数据库、网络或者其他外部结果。**

作为普通业务辅助函数，可以直接写成：

```
def list_public_posts(skip: int, limit: int):
    tmp = []

    for post in POSTS:
        if post.get("status") == "published":
            tmp.append(post)

    return tmp[skip:skip + limit]
```

然后直接调用：

```
posts = list_public_posts(0, 10)
```

如果保留 `async def`，内部也可以完全不变：

```
posts = await list_public_posts(0, 10)
```

但这段遍历、判断和切片依然会连续执行。**添加** `**async**` **不会让普通循环自动变成异步循环，也不会让筛选操作自动加速。**

需要区分：

- **语法是否合法：**没有 `await` 的 `async def` 是合法的。
- **是否有实际收益：**这个函数本身没有可重叠的等待，因此没有体现异步 I/O 的收益。
- **接口是否需要：**有时为了满足统一的异步接口，仍然可以保留 `async def`。

不要为了“凑出异步”给 `append()` 加 `await`，也不需要把普通列表遍历改成 `async for`。

## 2．先分清几个概念

| 概念        | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| 进程        | 程序运行的资源环境，通常有独立的地址空间                     |
| 线程        | 操作系统调度的执行单元，负责执行代码                         |
| 协程函数    | 使用 `async def` 定义的函数                                  |
| 协程对象    | 调用协程函数后得到的对象，保存相应执行过程的状态             |
| 任务 `Task` | 事件循环用来管理协程执行、完成状态、结果和异常的对象         |
| 事件循环    | 推进任务、等待事件，并安排任务恢复执行的调度机制             |
| 可等待对象  | 能够出现在 `await` 后面的对象，例如协程对象、`Task`、`Future` |

理解这些概念时，最重要的是分开两个层次：

> **任务表示有一份工作要完成；线程表示由谁执行工作。**

创建一个任务，不等于创建一个线程。

一份工作还没完成，也不意味着必须一直有一个专属线程停在那里等待它。

## 3．协程不是本来就存在的：它有明确的创建和启动过程

### 3.1 普通函数调用

```
def get_number():
    print("函数开始执行")
    return 42


number = get_number()
```

执行 `get_number()` 时，函数体立即开始执行，最后返回 `42`。

### 3.2 协程函数调用

```
async def get_number():
    print("函数开始执行")
    return 42
```

这时执行：

```
coro = get_number()
```

**只会创建协程对象，函数体还没有开始执行。**

因此：

```
async def get_number():
    ...
```

是定义函数。

```
coro = get_number()
```

是创建协程对象。

```
number = await coro
```

是在当前协程执行过程中，执行并等待这个协程的结果。

### 3.3 一个完整的启动示例

```
import asyncio


async def get_number():
    print("函数开始执行")
    return 42


async def main():
    coro = get_number()
    print("已经创建协程对象")

    number = await coro
    print("结果：", number)


asyncio.run(main())
```

输出：

```
已经创建协程对象
函数开始执行
结果： 42
```

这里的启动过程是：

```
执行 main()
    ↓
创建 main 的协程对象
    ↓
asyncio.run() 创建事件循环
    ↓
为这个协程建立主任务并运行
    ↓
进入 main 的函数体
    ↓
执行 await coro
    ↓
进入 get_number 的函数体
    ↓
取得结果，继续执行 main
```

**协程对象不会仅仅因为被创建，就自动开始运行。**

如果既不等待它，也不把它交给调度机制，通常还会出现“协程未被等待”的警告。[Python：协程与任务](https://docs.python.org/3.13/library/asyncio-task.html#coroutines)

## 4．事件循环怎么创建？是否需要额外创建一个线程？

### 4.1 独立脚本通常使用 `asyncio.run()`

```
import asyncio


async def main():
    await asyncio.sleep(1)
    print("完成")


asyncio.run(main())
```

`asyncio.run()` 负责：

1. 创建事件循环。
2. 运行传入的顶层协程。
3. 完成必要的清理。
4. 关闭事件循环。

通常把它作为独立异步程序的入口即可，不需要自己手写底层事件循环管理代码。[Python：运行异步程序](https://docs.python.org/3.13/library/asyncio-runner.html#running-an-asyncio-program)

### 4.2 不需要专门新建一个“事件循环线程”

事件循环需要在线程中执行，但这个线程可以就是主线程。

常见关系是：

```
一个进程
└── 主线程
    └── 运行事件循环
        ├── 推进任务 A
        ├── 推进任务 B
        └── 推进任务 C
```

不是每个任务一个线程，也不是调用 `asyncio.run()` 就会额外建立一个专门执行协程的新线程。

### 4.3 已经有事件循环时，由外层环境管理

例如 FastAPI 配合 Uvicorn 运行时，由服务器管理事件循环和请求调度。

这时路由内部直接使用 `await`，不要再嵌套调用 `asyncio.run()`。

`asyncio.run()` 不能在同一线程已经运行事件循环时再次调用。

## 5．`await` 的准确含义：等待结果，必要时暂停当前任务

看这段代码：

```
async def handle_request():
    user_id = 123

    user = await query_user(user_id)

    return user["name"]
```

假设 `query_user()` 使用支持异步的数据库访问方式，查询需要等待。

可以按下面的过程理解：

```
进入 handle_request
    ↓
保存 user_id = 123
    ↓
进入 query_user，发起查询
    ↓
查询结果尚未就绪
    ↓
当前任务暂停
    ↓
事件循环推进其他可运行的任务
    ↓
查询结果就绪
    ↓
当前任务获得继续执行的机会
    ↓
将结果赋给 user
    ↓
执行 return user["name"]
```

暂停时，相应的执行位置、局部变量和等待关系会被保留下来。

### 5.1 当前任务的执行顺序没有被打乱

```
result = await query_database()
print(result)
```

`print(result)` 必须等查询结果回来后才能执行。

**异步不会让依赖结果的下一行代码提前完成。**

它允许的是：当前任务等待时，运行它的线程可以推进其他任务。

### 5.2 `await` 不保证每次都会切换任务

如果被等待的操作可以立即完成，执行可能直接继续。

例如原来的内存筛选函数：

```
posts = await list_public_posts(0, 10)
```

如果函数内部没有实际暂停操作，它可以直接连续执行到 `return`，然后执行调用方的下一行。

因此：

> `**await**` **提供等待和暂停的机制，但不代表每次遇到它都一定发生任务切换。**

### 5.3 不是所有对象都能 `await`

可以等待：

```
result = await some_coroutine()
result = await some_task
```

不能随意等待普通对象：

```
# 如果 POSTS 是普通列表，这样写会报错
result = await POSTS
```

普通函数是否耗时，与它返回的对象是否可等待，是两回事。

## 6．为什么一路 `await` 下去，仍然可能没有并发收益？

### 6.1 多层协程调用，通常仍然属于一个任务

```
async def query_database():
    ...


async def list_public_posts():
    return await query_database()


async def main():
    posts = await list_public_posts()
    print(posts)
```

这里会产生多个协程对象，但直接等待子协程时，**不会自动为每一层建立独立的** `**Task**`。

关系可以理解成：

```
任务 A
└── main()
    └── await list_public_posts()
        └── await query_database()
            └── 等待数据库结果
```

这是一条存在依赖关系的任务链：

```
数据库返回
    ↓
业务函数拿到结果
    ↓
main 拿到结果
    ↓
打印结果
```

**调用层数变多，并不等于独立工作变多。**

### 6.2 如果没有其他工作，仍然只能等待

```
async def main():
    posts = await query_database()
    print(posts)
```

如果程序只有这条任务链，查询期间也没有其他工作可推进，那么：

- 查询需要多久，仍然需要等多久。
- 不会因为写了 `async` 而自动加速。
- 事件循环不会自动找一项新业务替程序执行。

没有可运行任务时，事件循环可以等待 I/O、定时器等事件就绪，不需要持续空转。

### 6.3 异步收益常常体现为两种情况

第一种：**等待重叠，缩短多项独立工作的总耗时。**

例如两个独立网络请求都需要等约 1 秒，安排并发后，它们的等待可以重叠。

第二种：**一项工作等待时，程序仍能响应其他工作。**

例如慢查询还没结束，但健康检查、其他用户请求或者进度输出仍能继续。

## 7．怎样创建能够独立推进的任务？

### 7.1 使用 `asyncio.create_task()`

```
task = asyncio.create_task(report_progress())
```

它可以拆成：

```
# 调用协程函数，取得协程对象
coro = report_progress()

# 将协程包装成 Task，交给事件循环调度
task = asyncio.create_task(coro)
```

当前任务之后可以继续执行其他代码，在需要时再等待：

```
result = await task
```

`create_task()` 接收的准确对象是**协程对象**，而不是函数本身。

```
# 正确
task = asyncio.create_task(report_progress())

# 错误：传入了函数对象
task = asyncio.create_task(report_progress)
```

### 7.2 普通 `def` 函数不能直接这样提交

```
def report_progress():
    print("报告进度")
    return 123
```

执行：

```
task = asyncio.create_task(report_progress())
```

实际发生的是：

```
先在当前线程执行 report_progress()
    ↓
打印内容，返回 123
    ↓
执行 create_task(123)
    ↓
因为参数不是协程对象而报错
```

如果普通函数需要执行 10 秒，就会先执行这 10 秒，之后才发生参数类型错误。

`create_task()` 不会自动把普通函数变成异步函数。

严格来说，普通 `def` 也可能返回协程对象，此时返回值可以提交；日常使用最清楚的写法仍然是：

```
async def work():
    ...


task = asyncio.create_task(work())
```

### 7.3 先创建任务，再依次等待，仍然可以并发

下面是依次进行：

```
a = await query_a()
b = await query_b()
```

因为调用 `query_b()` 的语句必须等前一行完成后才会执行。

下面可以并发推进：

```
task_a = asyncio.create_task(query_a())
task_b = asyncio.create_task(query_b())

a = await task_a
b = await task_b
```

两个任务已经提前安排好了。最后两句依次等待结果，不等于它们依次启动。

也可以用：

```
a, b = await asyncio.gather(
    query_a(),
    query_b(),
)
```

`gather()` 会为这里传入的协程安排任务，并等待结果。

### 7.4 数据依赖无法通过创建任务消除

例如：

```
user = await query_user()
orders = await query_orders(user["id"])
```

第二次查询需要第一次返回的 ID，因此不能直接把两项工作当成互相独立的操作同时启动。

决定是否能并发的，首先是业务依赖关系。

## 8．一个完整实验：观察任务、线程、等待和阻塞

下面的代码可以作为一个独立 Python 文件运行，只使用标准库，不依赖其他示例文件。

其中 `asyncio.sleep(1)` 用于**模拟可异步等待的数据库延迟**，没有连接真实数据库。

```
import asyncio
import threading
import time


POSTS = [
    {"id": 1, "status": "published"},
    {"id": 2, "status": "draft"},
    {"id": 3, "status": "published"},
]

START = time.perf_counter()


def log(message):
    task = asyncio.current_task()

    print(
        f"{time.perf_counter() - START:4.1f}s"
        f" | 线程={threading.get_ident()}"
        f" | 任务={task.get_name()}"
        f" | {message}"
    )


async def query_posts(label, blocking=False):
    log(f"{label}：开始等待")

    if blocking:
        # 阻塞当前线程
        time.sleep(1)
    else:
        # 暂停当前任务，允许事件循环推进其他任务
        await asyncio.sleep(1)

    log(f"{label}：等待结束")
    return POSTS


async def list_public_posts(
    label,
    skip=0,
    limit=10,
    blocking=False,
):
    log(f"{label}：进入业务函数")

    posts = await query_posts(label, blocking)

    published = [
        post
        for post in posts
        if post.get("status") == "published"
    ]

    return published[skip:skip + limit]


async def main():
    asyncio.current_task().set_name("main")

    print("\n实验一：直接 await，两次查询依次进行")

    start = time.perf_counter()

    await list_public_posts("A")
    await list_public_posts("B")

    print(f"耗时：{time.perf_counter() - start:.2f} 秒")

    print("\n实验二：先创建两个任务，再等待结果")

    start = time.perf_counter()

    task_a = asyncio.create_task(
        list_public_posts("A"),
        name="任务A",
    )

    task_b = asyncio.create_task(
        list_public_posts("B"),
        name="任务B",
    )

    a = await task_a
    b = await task_b

    print(f"耗时：{time.perf_counter() - start:.2f} 秒")
    print("结果：", a, b)

    print("\n实验三：任务仍有两个，但内部阻塞线程")

    start = time.perf_counter()

    task_a = asyncio.create_task(
        list_public_posts("A", blocking=True),
        name="任务A",
    )

    task_b = asyncio.create_task(
        list_public_posts("B", blocking=True),
        name="任务B",
    )

    await task_a
    await task_b

    print(f"耗时：{time.perf_counter() - start:.2f} 秒")


asyncio.run(main())
```

### 8.1 实验一：多层 `await`，仍是一条任务链

核心代码：

```
await list_public_posts("A")
await list_public_posts("B")
```

观察：

- A 完成后，B 才开始。
- 业务函数与查询函数的日志都属于 `main` 任务。
- 总耗时约 2 秒。

```
main 任务：

A 开始 → A 等待 → A 完成 → B 开始 → B 等待 → B 完成
```

### 8.2 实验二：两个任务，共享同一个线程

核心代码：

```
task_a = asyncio.create_task(...)
task_b = asyncio.create_task(...)

a = await task_a
b = await task_b
```

观察：

- A、B 都能在对方等待期间推进。
- 日志中出现两个任务名称。
- 线程编号仍然相同。
- 总耗时约 1 秒。

```
任务 A：开始 → 等待 ─────────→ 完成
任务 B：     开始 → 等待 ────→ 完成
```

等待时间重叠了，但 Python 代码仍由同一个线程交替执行。

### 8.3 实验三：创建了任务，也可能无法交替推进

内部等待变成：

```
time.sleep(1)
```

观察：

- 虽然存在两个任务，但 A 阻塞线程期间，B 无法执行。
- A 返回后，B 才有机会开始。
- 总耗时重新接近 2 秒。

这组实验实际运行的耗时约为：

| 实验                 | 耗时    |
| -------------------- | ------- |
| 顺序等待             | 2.00 秒 |
| 两个任务并发等待     | 1.00 秒 |
| 两个任务内部阻塞线程 | 2.00 秒 |

具体计时会有少量差异。重点是执行顺序、任务名称、线程编号与等待是否重叠。

## 9．`time.sleep()` 和 `await asyncio.sleep()` 的本质差别

| 写法                     | 暂停对象 | 当前事件循环线程能否推进其他任务 |
| ------------------------ | -------- | -------------------------------- |
| `time.sleep(1)`          | 当前线程 | 休眠期间不能                     |
| `await asyncio.sleep(1)` | 当前任务 | 可以                             |

因此：

```
async def work():
    time.sleep(10)
```

仍然会阻塞事件循环线程。

同样：

```
async def work():
    大量同步计算()
```

也不会因为加了 `async def` 就自动把计算移走。

### 不要写 `await time.sleep(1)`

```
await time.sleep(1)
```

执行顺序是：

1. 先调用 `time.sleep(1)`。
2. 当前线程阻塞 1 秒。
3. 函数返回 `None`。
4. 尝试 `await None`。
5. 报错。

`await` 不会先把它后面的普通调用自动改造成异步调用。

## 10．`create_task()` 和 C++ 创建线程有什么相似与不同？

使用形式确实相似。

C++：

```
std::thread t(work);

// 当前线程继续执行其他代码

t.join();
```

Python：

```
task = asyncio.create_task(work())

# 当前任务继续执行其他代码

await task
```

它们都能表达：

```
安排另一份工作
    ↓
自己先继续
    ↓
在需要的时候等待它完成
```

但执行机制不同。

| 比较项                   | C++ `std::thread`                            | Python `asyncio.create_task()`               |
| ------------------------ | -------------------------------------------- | -------------------------------------------- |
| 创建什么                 | 操作系统线程                                 | 事件循环管理的任务                           |
| 工作由谁执行             | 新线程                                       | 运行所属事件循环的线程                       |
| 调度机制                 | 操作系统调度线程                             | 事件循环调度任务                             |
| 能否直接增加并行执行资源 | 可以增加线程资源，实际并行还取决于硬件等条件 | 不会因为创建任务而增加线程                   |
| 等待完成                 | `join()` 通常阻塞调用线程                    | `await` 可以暂停当前任务，释放线程给事件循环 |
| 等待期间的其他工作       | 需要其他线程等执行资源推进                   | 同一事件循环线程可以推进其他任务             |

### 10.1 `await` 和 `join()` 都表达依赖，但暂停层次不同

```
t.join();
```

目标线程尚未结束时，调用 `join()` 的线程通常会阻塞。

```
await task
```

目标任务尚未结束时，当前任务可以暂停，同一个线程继续推进其他任务。

```
join：
    暂停调用线程

await：
    可以暂停当前任务
    线程继续服务事件循环中的其他工作
```

### 10.2 C++ 和 Python 的传参形式也不同

```
std::thread t(work);
```

这里把函数交给新线程调用。

```
asyncio.create_task(work())
```

这里先调用协程函数，得到协程对象，再把对象交给调度机制。

`work()` 是协程函数调用时，并不会立即执行函数体；如果它是普通同步函数调用，就会立即执行。

## 11．单线程异步为什么有价值？它到底是不是“单核”？

### 11.1 单个事件循环中的代码通常由一个线程执行

同一时刻，这个线程只能执行一段代码。

因此，单纯创建多个 `Task`，不会让这些任务的 Python 代码同时跑在多个 CPU 核上。

但“单线程”不等于“永远固定在某个核上”。操作系统可以把线程调度到不同 CPU 核，准确说法是：

> **一个线程在同一时刻只能在一个核上执行。**

应用等待数据库时，数据库进程、网络设备、操作系统等仍然可以在别处推进工作。整个系统并不因此全部串行。

### 11.2 计算和等待是两类不同的问题

如果两项工作都是：

```
持续计算 5 秒
```

那么单线程事件循环无法凭空增加计算能力。

如果两项工作都是：

```
执行少量代码
    ↓
等待网络结果 1 秒
    ↓
执行少量代码
```

它们的等待可以重叠，不要求本地有两个线程同时执行指令。

### 11.3 多线程也能处理等待，为什么还要事件循环？

考虑聊天服务器中的大量连接，大部分连接都在等待下一条消息。

一种设计是：

```
线程 1 等连接 1
线程 2 等连接 2
线程 3 等连接 3
……
```

另一种设计是：

```
少量事件循环线程
    ↓
登记许多连接分别在等什么
    ↓
哪些连接有数据，就推进哪些连接对应的任务
```

第二种设计不需要为每份等待工作保留专属线程。

需要注意：

**阻塞等待的线程通常会被操作系统挂起，不是一直占用 CPU 核空转。**

异步的价值主要在于：

- 用较少线程管理较多等待中的工作。
- 减少大量线程带来的栈、调度状态等资源开销。
- 在支持异步的接口上，让等待和恢复形成统一的调度机制。

线程池也是合理方案。线程和异步并不是只能二选一。

### 11.4 事件循环不是挨个询问所有协程

对于网络 I/O，事件循环可以借助操作系统提供的事件通知机制。

Linux 上常见的是 `epoll` 等 I/O 多路复用设施。

概念上是：

```
登记要关注的连接和事件
    ↓
等待哪些连接已经就绪
    ↓
取得就绪通知
    ↓
安排对应任务继续执行
```

没有可执行工作时，事件循环线程本身也可以等待。[Python：I/O 多路复用](https://docs.python.org/3.13/library/selectors.html)

## 12．异步可以与多线程、多核结合

异步描述的是工作如何等待、如何恢复；多线程、多进程描述的是执行资源如何组织。

它们可以组合：

```
大量逻辑任务 / 协程
        ↓
运行时调度机制
        ↓
一个或多个执行线程
        ↓
操作系统调度
        ↓
一个或多个 CPU 核
```

有些运行时会用多个工作线程推进异步任务。

C++ 的 Boost.Asio 就允许多个线程同时调用同一个 `io_context::run()`，形成执行处理函数的线程池，也提供协程支持。[Boost.Asio：线程模型](https://www.boost.org/latest/doc/html/boost_asio/overview/core/threads.html)

### 12.1 Python `asyncio` 的常见组织方式

```
事件循环线程：
    管理连接、推进任务、等待异步 I/O

线程池：
    接入现有的阻塞式 I/O 工作

进程池：
    执行适合并行的计算工作
```

同一个 `asyncio` 事件循环不会自动把它的任务分配给多个线程执行，但可以把某些工作提交给线程池或进程池。

### 12.2 用 `to_thread()` 接入普通同步函数

```
import asyncio
import time


def blocking_work():
    time.sleep(2)
    return "工作完成"


async def main():
    task = asyncio.create_task(
        asyncio.to_thread(blocking_work)
    )

    print("当前任务可以继续推进其他工作")

    result = await task
    print(result)


asyncio.run(main())
```

这里分两层：

```
asyncio.to_thread(blocking_work)
```

返回一个协程；这个协程执行时，会将同步函数提交给线程池。

```
asyncio.create_task(...)
```

把这个协程安排为事件循环中的任务。

也可以直接等待：

```
result = await asyncio.to_thread(blocking_work)
```

即使当前任务直接等待它，事件循环仍然可以推进其他任务。

注意传入的是：

```
asyncio.to_thread(blocking_work)
```

而不是：

```
asyncio.to_thread(blocking_work())
```

后者会先在当前线程调用同步函数。

`to_thread()` 没有把同步库变成原生异步库。接入数据库等库时，还需要遵守其连接、会话等对象的线程使用要求。[Python：在线程中运行](https://docs.python.org/3.13/library/asyncio-task.html#running-in-threads)

### 12.3 大量计算与 GIL 要单独考虑

在启用 GIL 的 CPython 中，纯 Python 计算通常不能仅靠多个线程获得多核并行，因此常见方案是优化算法、减少数据量，或者使用进程池。

释放 GIL 的原生库、无 GIL 构建等情况需要分别判断。

这是两个不同的问题：

- 单个事件循环由一个线程推进。
- GIL 对多个 Python 线程执行计算的限制。

不要把所有“不能多核并行”的原因都归结为其中某一个。[Python：GIL 与线程性能](https://docs.python.org/3.13/library/threading.html#gil-and-performance-considerations)

## 13．为什么 FastAPI 路由里一路 `await`，仍然能体现异步作用？

假设路由是：

```
@app.get("/posts")
async def get_posts():
    return await list_public_posts()
```

单独看这段函数，只有一条任务链。

但从服务器看，可能是：

```
事件循环
├── 请求 A 的任务
│   └── get_posts()
│       └── 等待数据库
│
├── 请求 B 的任务
│   └── get_user()
│       └── 等待数据库
│
└── 请求 C 的任务
    └── health()
        └── 立即返回
```

**一个请求内部可以一直按照依赖顺序执行；不同请求之间仍然可以并发。**

在 FastAPI 配合 Uvicorn 的常见运行方式里，服务器负责请求任务的安排。

因此，路由里面没有手写 `create_task()`，不代表整个服务没有其他任务。其他任务可能来自其他用户请求。

如果只测试一个请求，等待期间也没有其他工作，通常看不出这种收益。

### 13.1 路由函数和普通辅助函数要区分

FastAPI 对它负责调用的路由和依赖有专门处理；你自己调用普通辅助函数时，则按照普通 Python 调用规则执行。

| 场景                           | 执行方式                               |
| ------------------------------ | -------------------------------------- |
| FastAPI 的 `async def` 路由    | 在事件循环中执行                       |
| FastAPI 的普通 `def` 路由      | 通常在线程池中执行                     |
| FastAPI 管理的普通 `def` 依赖  | 通常在线程池中执行                     |
| 异步路由里直接调用普通辅助函数 | 直接在当前线程执行，不会自动搬到线程池 |
| 异步路由里等待异步辅助函数     | 通过 `await` 进入相应协程调用链        |

例如：

```
def select_public_posts(skip, limit):
    published = [
        post
        for post in POSTS
        if post.get("status") == "published"
    ]

    return published[skip:skip + limit]


@app.get("/posts")
async def get_posts(skip: int = 0, limit: int = 10):
    return select_public_posts(skip, limit)
```

对于少量内存数据，这种快速同步处理是合理的。

但如果辅助函数内部执行同步网络请求或大量计算，就会影响当前事件循环线程。不能仅因为辅助函数写成了 `def`，就认为 FastAPI 会自动把它移走。[FastAPI：并发与 async / await](https://fastapi.tiangolo.com/async/)

## 14．回到文章查询：不同数据来源对应不同写法

### 14.1 数据已经在内存里

使用普通函数：

```
def list_public_posts(skip, limit):
    published = [
        post
        for post in POSTS
        if post.get("status") == "published"
    ]

    return published[skip:skip + limit]
```

### 14.2 数据来自支持异步的数据库访问层

下面是接口示意，具体方法取决于实际使用的库或自行封装的仓储层：

```
async def list_public_posts(skip, limit):
    return await post_repository.list_published(
        skip=skip,
        limit=limit,
    )
```

有意义的等待位于数据库查询处。状态筛选和分页通常也适合交给数据库完成。

### 14.3 必须调用现有的同步 I/O 接口

可以根据运行环境选择：

- 在合适的线程池中调用。
- 在异步函数中使用 `asyncio.to_thread()` 等方式。
- 如果是 FastAPI 同步路由，让框架按同步路由方式执行。

### 14.4 工作主要是大量计算

重点考虑：

- 优化算法。
- 减少需要处理的数据。
- 将筛选、聚合等工作交给合适的数据处理系统。
- 使用适合的进程池或其他并行计算方案。

单纯增加 `async` 或创建更多 `Task`，不会增加计算资源。

## 15．常见误解与判断速查

| 容易产生的理解                                        | 更准确的理解                                                 |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| 写了 `async def` 就会自动并发                         | 它定义协程函数，实际并发还取决于任务安排和能否让出执行权     |
| `async def` 内必须写 `await`                          | 没有 `await` 也合法，只是可能没有实际暂停                    |
| 每个 `await` 都会切换任务                             | 结果立即就绪时可能直接继续                                   |
| `await` 后面的下一行可以提前执行                      | 当前任务仍然按照依赖顺序执行                                 |
| 多层 `await` 就是多个独立任务                         | 直接等待子协程，通常仍属于同一个任务                         |
| 调用协程函数就会执行函数体                            | 调用先创建协程对象，还需要等待或调度                         |
| `create_task()` 接收函数                              | 它接收协程对象，通常写成 `create_task(async_function())`     |
| `create_task()` 会创建线程                            | 它创建任务，不会因此增加操作系统线程                         |
| 最后依次 `await task_a`、`await task_b`，就一定是串行 | 如果两个任务已经提前安排，仍然可以并发推进                   |
| 普通函数放进异步函数里就变异步                        | 它仍然按普通同步调用规则执行                                 |
| `await` 能把同步阻塞函数改造成异步函数                | 普通调用会先执行；其返回值还必须可等待                       |
| 单线程意味着不能同时存在许多未完成工作                | 可以存在许多等待中的任务，只是代码不在多个核上同时执行       |
| 异步必须单线程                                        | 异步可以与多线程、多进程结合                                 |
| 阻塞等待的线程会持续占用 CPU 空转                     | 通常会被操作系统挂起，但仍有线程资源和管理成本               |
| FastAPI 会把所有普通函数自动放到线程池                | 主要针对它负责调用的同步路由、依赖等，普通辅助函数直接调用不会自动移走 |
| 只有一条任务链也能自动获得异步加速                    | 没有其他工作可推进时，通常仍需要等待相同的外部耗时           |

实际选择时，可以按下面的顺序判断：

```
这段工作主要在做什么？
    │
    ├── 快速内存处理
    │   └── 普通函数通常足够
    │
    ├── 等数据库、网络等外部结果
    │   │
    │   ├── 接口支持异步
    │   │   └── async def + await
    │   │
    │   └── 接口只支持同步阻塞调用
    │       └── 考虑线程池、to_thread 或合适的同步路由
    │
    └── 大量计算
        └── 考虑算法、数据量与并行计算资源

再判断：是否存在可以独立推进的工作？
    │
    ├── 没有
    │   └── 当前任务仍需按照依赖顺序等待
    │
    └── 有
        │
        ├── 独立脚本
        │   └── 用 create_task、gather 等安排并发
        │
        └── Web 服务
            └── 跨请求并发通常由服务器安排
```