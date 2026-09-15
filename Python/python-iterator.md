# Python-iterator

```python
class itermodel:
    def __init__(self, a):
        self.a = a
    # 表示要传一个 a 进来

    def __iter__(self):
        return self

    def __next__(self):
        # 后面会不断调用这个方法直到 StopIteration
        if self.a > 100:
            raise StopIteration
        self.a += 1
        return self.a


it = itermodel(66)

for i in it:
    print(i)
```

## 2. 运行结果

```text
67
68
69
...
100
101
```

说明：初始 `a = 66`，每次调用 `__next__` 时先判断 `self.a > 100`，不满足则 `self.a += 1` 并返回。因此会输出 `67` 到 `101`。当 `self.a` 变成 `101` 后，下一次调用 `__next__` 会抛出 `StopIteration`，`for` 循环结束。

---

## 3. 迭代器简介

### 3.1 可迭代对象与迭代器

| 概念       | 要求                          | 说明                                          |
| ---------- | ----------------------------- | --------------------------------------------- |
| 可迭代对象 | 实现 `__iter__`               | 可以被 `for` 遍历，例如 `list`、`str`、`dict` |
| 迭代器     | 实现 `__iter__` 和 `__next__` | 表示一个数据流，能不断返回下一个值            |
| `__iter__` | 返回迭代器对象                | 迭代器通常返回 `self`                         |
| `__next__` | 返回下一个值                  | 没有更多数据时抛出 `StopIteration`            |

### 3.2 `for` 循环内部流程

当执行：

```python
for i in it:
    print(i)
```

大致等价于：

```python
iterator = iter(it)  # 调用 it.__iter__()
while True:
    try:
        i = next(iterator)  # 调用 iterator.__next__()
    except StopIteration:
        break
    print(i)
```

所以：

1. `for` 会先调用 `iter(it)`，也就是 `it.__iter__()`。
2. `__iter__` 返回一个迭代器对象。
3. 然后不断调用 `next(iterator)`，也就是 `iterator.__next__()`。
4. 当 `__next__` 抛出 `StopIteration` 时，循环结束。



### 补充

**遍历 `list` 时，`for` 循环也会先对这个 `list` 实例调用 `iter()`，从而调用它的 `__iter__` 方法，获得一个专门的迭代器对象。**

不过 `list` 和前面自定义的 `itermodel` 有一个重要区别：

> `list` 本身是**可迭代对象**，但它不是迭代器。
> `list.__iter__()` 返回的是一个独立的 `list_iterator` 对象。

```python
lst = [1, 2, 3]
it = iter(lst)
print(type(it))
# <class 'list_iterator'>
```

---

## 4. 本例说明

```python
class itermodel:
    def __init__(self, a):
        self.a = a

    def __iter__(self):
        return self

    def __next__(self):
        if self.a > 100:
            raise StopIteration
        self.a += 1
        return self.a
```

- `itermodel` 是一个迭代器类。
- `it = itermodel(66)` 创建了一个迭代器对象。
- `it` 既是迭代器，也是可迭代对象，因为它同时实现了：
  - `__iter__`
  - `__next__`
- `__iter__` 返回 `self`，表示“我自己就是迭代器”。
- `__next__` 负责返回下一个值，并在耗尽时抛出 `StopIteration`。

因此，原注释可以更准确地写成：

> `itermodel` 是一个迭代器类，`it` 是迭代器，同时也是可迭代对象。  
> 在 `for i in it` 时，会先调用 `__iter__` 方法获得迭代器；  
> 然后不断调用 `__next__` 方法获取下一个值，直到抛出 `StopIteration`。

---

## 5. 注意点

1. 迭代器是一次性的。  
   上面的 `it` 被遍历完后，`self.a` 已经变成 `101`。再次 `for i in it` 不会重新输出，而是立即结束。

2. 如果想重复遍历，可以重新创建对象：

   ```python
   for i in itermodel(66):
       print(i)
   ```

3. `StopIteration` 是用来告诉 `for` 循环“没有更多数据了”的标准异常。

4. 迭代器的优点是惰性求值：不需要一次性把所有数据放进内存，而是按需生成下一个值。