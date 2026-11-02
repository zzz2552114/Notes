Q：

```py
import asyncio

async def fetch_number(x):
    print("开始", x)
    await asyncio.sleep(1)      # 模拟一个耗时 1 秒的慢操作（比如数据库查询）
    print("结束", x)
    return x * 10

async def main():
    a = await fetch_number(1)   # 等它完成
    print("a =", a)
    # 同时跑两个，总耗时约 1 秒而不是 2 秒：
    results = await asyncio.gather(fetch_number(2), fetch_number(3))
    print("results =", results)

asyncio.run(main())
```
```text
开始 1
结束 1
a = 10
开始 2
开始 3
结束 2
结束 3
results = [20, 30]
```
这里是因为，本来   ` a = await fetch_number(1)`   可以交换控制权，但是下面立马用到了a，所以必须在这里等待完成对吗？`await`不是异步等待吗？那为什么我加一个    
```py
a = await fetch_number(1)
print(123)
print("a =", a)
```
这个123也会在函数结束之后才打印

A:

对，你的核心理解方向是对的，但有一个关键点要纠正：

> `await` 的“异步等待”不是“跳过等待、继续执行 `await` 后面的代码”，而是“当前协程在这里暂停，把控制权交回事件循环；等被等待的东西完成后再从这一行恢复继续”。

所以：

```python
a = await fetch_number(1)
print(123)
print("a =", a)
```

`print(123)` 一定在 `fetch_number(1)` 完成、`a` 被赋值之后才执行。因为 `print(123)` 是 `main` 协程里 `await` 后面的下一行，而 `main` 在 `await` 处被挂起了。它不是只挂起“赋值”这个动作，而是挂起整个 `main` 协程。

时间线大概是这样：

```text
main 执行到 a = await fetch_number(1)
  fetch_number(1) 开始执行
  打印：开始 1
  执行到 await asyncio.sleep(1)
      fetch_number 挂起，控制权交回事件循环
      main 也在等 fetch_number 完成，所以 main 也挂起
      此时没有别的任务可跑，事件循环等 1 秒
  1 秒后 sleep 完成
  fetch_number 恢复，打印：结束 1
  fetch_number return 10
main 恢复：
  a = 10
  打印：123
  打印：a = 10
```

所以输出是：

```text
开始 1
结束 1
123
a = 10
```

这完全符合协程的顺序语义。

---

### 纠正一个说法

你说：

> 本来 `a = await fetch_number(1)` 可以交换控制权，但是下面立马用到了 `a`，所以必须在这里等待完成对吗？

不完全是。不是因为“下面立马用到 `a`”才等待，而是：

```python
a = await fetch_number(1)
```

这行本身的语义就是：

> 当前协程暂停，直到 `fetch_number(1)` 完成并返回结果，然后把结果赋给 `a`，再继续下一行。

即使你下面不用 `a`，写成：

```python
await fetch_number(1)
print(123)
```

`print(123)` 仍然会在 `fetch_number(1)` 完成后才执行。

---

### `await` 到底“异步”在哪里？

对比同步阻塞：

```python
time.sleep(1)  # 整个线程卡住 1 秒
```

异步等待：

```python
await asyncio.sleep(1)  # 当前协程挂起 1 秒，事件循环可以去跑别的任务
```

区别是：

- `time.sleep(1)`：整个线程被阻塞，别的协程也跑不了。
- `await asyncio.sleep(1)`：当前协程暂停，事件循环可以调度其他任务。

但注意：

> 事件循环可以去跑别的任务，不代表当前协程会继续执行 `await` 后面的代码。

当前协程的顺序仍然是：

```text
await 之前 -> 等 await 完成 -> await 之后
```

---

### 为什么 `gather` 能同时跑两个？

```python
results = await asyncio.gather(fetch_number(2), fetch_number(3))
```

`gather` 会把两个协程包装成两个任务并发调度。事件循环可以这样跑：

```text
任务2：开始 2 -> 遇到 sleep 挂起
任务3：开始 3 -> 遇到 sleep 挂起
1 秒后：
任务2恢复：结束 2
任务3恢复：结束 3
gather 完成
main 恢复：results = [20, 30]
```

所以总耗时约 1 秒，而不是 2 秒。

而：

```python
a = await fetch_number(1)
```

只是等待一个协程完成。它内部遇到 `await sleep(1)` 时确实会让出控制权，但此时事件循环里没有别的任务可跑，于是看起来就是单纯等 1 秒。并且 `main` 本身在 `await fetch_number(1)` 处暂停，不能执行后面的 `print(123)`。

---

### 如果你想先打印 123，再等结果

可以这样：

```python
async def main():
    task = asyncio.create_task(fetch_number(1))
    print(123)
    a = await task
    print("a =", a)
```

`create_task` 表示“把协程提交给事件循环并发调度”，但不立刻等待它。`main` 可以继续往下执行，直到 `await task` 才等结果。

或者更简单：

```python
print(123)
a = await fetch_number(1)
print("a =", a)
```

把 `print(123)` 放在 `await` 前面。

---

总结一句：

> `await` 是“当前协程非阻塞地等待”，不是“当前协程不等待”。它会暂停当前协程，让事件循环去跑其他任务；但当前协程 `await` 后面的代码，必须等被等待对象完成后才会继续执行。
> 

## 一、`await` 暂停的是什么？

暂停的是**当前这个协程的帧（frame）**，也就是当前 `Task` 里正在执行的那段代码。具体说：

```text
事件循环 (event loop)
  └─ Task (包装 main 协程)
       └─ main 的协程帧  ← 执行到 await 这一行时，这个帧被挂起
```

`await` 那一行的语义是：

1. 把被 `await` 的对象（协程 / Future / Task）交给事件循环；
2. **当前协程帧挂起**，局部变量都留在帧里（不会被销毁）；
3. 控制权 return 回事件循环，事件循环去干别的（等 IO、跑其他 Task、等定时器）；
4. 被 `await` 的东西完成后，事件循环把当前协程帧**从 `await` 的下一行**恢复执行。

所以不是"暂停线程"，不是"暂停事件循环"，也不是"暂停整个程序"——只是"暂停这一条协程的进度"。线程里的其他协程、定时器、IO 回调都还能跑。

可以用一句对照记住：

| 写法 | 谁被挂起 | 谁能继续 |
|---|---|---|
| `time.sleep(1)` | 整个线程 | 谁都不能 |
| `await asyncio.sleep(1)` | 当前协程帧 | 事件循环里的其他 Task |

---

## 二、最简单的观察办法：放一个"心跳任务"

不用 `gather`，用 `create_task` 起一个后台任务一直打印，就能肉眼看到 `await` 期间事件循环没停：

```python
import asyncio, time

async def ticker():
    while True:
        print(f"[ticker] {time.strftime('%X')}")
        await asyncio.sleep(0.5)

async def slow():
    print("[slow] start")
    await asyncio.sleep(2)
    print("[slow] end")

async def main():
    t = asyncio.create_task(ticker())   # 后台任务
    await slow()                        # main 在这里被挂起 2 秒
    t.cancel()

asyncio.run(main())
```

输出大概：

```text
[slow] start
[ticker] 12:00:00
[ticker] 12:00:00
[ticker] 12:00:01
[ticker] 12:00:01
[slow] end
```

`slow` 里 `await asyncio.sleep(2)` 挂起了 `slow` 协程，但 `ticker` 每 0.5 秒还在跑——这就是"`await` 期间事件循环还能做别的事"的直接证据。

把 `await asyncio.sleep(2)` 换成 `time.sleep(2)` 再看：`ticker` 会卡住不动，2 秒后才一次性接着跳。对比非常直观。

---

## 三、几个更轻量的观察技巧

### 1. 打印时间戳

```python
async def work(name, sec):
    print(f"{time.strftime('%X')} {name} start")
    await asyncio.sleep(sec)
    print(f"{time.strftime('%X')} {name} end")
```

两个协程交错打印的时间戳，一眼就能看出并发。

### 2. 用 `asyncio.all_tasks()` 看当前有哪些任务

```python
async def main():
    print("tasks:", asyncio.all_tasks())
    t = asyncio.create_task(slow())
    print("tasks now:", asyncio.all_tasks())
    await t
```

能看到 `slow` 任务在 `await` 期间存在于任务集合里。

### 3. 用 `asyncio.current_task()`

```python
async def slow():
    print("current task:", asyncio.current_task())
```

确认"暂停的是这个 Task 里的协程帧"。

### 4. `asyncio.sleep(0)` 主动让出控制权

```python
async def step(name):
    for i in range(3):
        print(name, i)
        await asyncio.sleep(0)   # 不真睡，只是让出控制权

async def main():
    await step("A")
    await step("B")
```

`await asyncio.sleep(0)` 相当于"我这一帧先挂起，让别人跑一轮"。这能帮你体会"`await` 的本质是交还控制权，而睡不睡是另一回事"。

---

## 四、回到你原来的疑问

```python
a = await fetch_number(1)
print(123)
```

`print(123)` 一定在 `fetch_number(1)` 完成后执行，因为 `print(123)` 属于 `main` 这个协程帧，而 `main` 在 `await` 处被挂起了。`await` 保证的是"本协程的顺序"，不保证"本协程之后的代码立刻执行"。

`await` 期间能"同时进行"的是**别的 Task**（`create_task` 出来的、事件循环里已有的、`gather` 包装出来的）。这些 Task 里的协程帧，在 `main` 挂起时会被事件循环捡起来跑——这就是"异步"。