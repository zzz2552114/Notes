`flush` 是 Python `print()` 的一个关键字参数，用来控制**是否在打印后立即刷新输出缓冲区**。

`print()` 的签名大致是：

```python
print(*objects, sep=' ', end='\n', file=None, flush=False)
```

所以：

```python
print(content, end="", flush=True)
```

意思是：

- `content`：要打印的内容
- `end=""`：打印后不换行，默认是 `end="\n"`
- `flush=True`：打印后**立刻把缓冲区内容写到目标输出流**，通常就是终端/文件

### `flush` 的作用

默认 `flush=False`，输出可能先放在缓冲区里，不一定马上显示。  
设置 `flush=True` 后，相当于打印完立即执行：

```python
file.flush()
```

让内容马上输出。

### 为什么常见于 `end=""`

因为终端通常是“行缓冲”：遇到换行符才更容易立即显示。  
如果你用了：

```python
print(content, end="")
```

没有换行，内容可能不会立刻显示出来。加上：

```python
print(content, end="", flush=True)
```

就能实时显示，常用于：

- 进度条
- 实时日志
- 逐字符输出
- 长时间运行任务的进度提示

### 例子

```python
import time

for i in range(5):
    print(i, end="", flush=True)
    time.sleep(1)
```

这里每个数字会立即显示，而不是等程序结束或缓冲区满后才一起显示。

总结：`flush=True` 表示**每次 print 后立即刷新输出缓冲区，强制马上输出**。