## Reversed

`reversed()` 是 Python 的内置函数，用于返回一个反向迭代器，它可以以逆序的方式遍历序列（如列表、元组、字符串、`range` 等）或其他支持反向迭代协议的对象。下面详细解释其用法、特性及注意事项。

---

## 1. 语法

```python
reversed(seq)
```

- **参数 `seq`**：必须是一个实现了 `__reversed__()` 方法的对象，或者支持序列协议（即实现了 `__len__()` 和 `__getitem__()` 方法，并且索引从 0 开始）。常见的序列类型（`list`、`tuple`、`str`、`range` 等）都满足条件。
- **返回值**：一个反向迭代器（`reversed` 对象），可用于 `for` 循环或转换为其他数据结构。

---

## 2. 基本用法示例

### 对列表使用 `reversed()`
```python
a = [1, 2, 3, 4]
rev = reversed(a)
# 由于 reversed() 返回值是一个迭代器，所以后面必须用 list 复原成列表
# rev 的类型 <class 'list_reverseiterator'>
print(list(rev))   # 输出: [4, 3, 2, 1]
print(a)           # 原列表不变: [1, 2, 3, 4]
```

### 对字符串使用 `reversed()`
```python
s = "hello"
rev_s = ''.join(reversed(s))
print(rev_s)       # 输出: "olleh"
```

### 对元组使用 `reversed()`
```python
t = (1, 2, 3)
rev_t = tuple(reversed(t))
print(rev_t)       # 输出: (3, 2, 1)
```

### 对 `range` 使用 `reversed()`
```python
r = range(5, 10)
rev_r = list(reversed(r))
print(rev_r)       # 输出: [9, 8, 7, 6, 5]
```

---

## 3. `reversed()` 与 `list.reverse()` 的区别

| 特性                   | `reversed(seq)`                | `list.reverse()`           |
| ---------------------- | ------------------------------ | -------------------------- |
| 适用类型               | 任何支持反向迭代的序列或对象   | 仅限 `list` 类型           |
| 返回值                 | 反向迭代器（不修改原对象）     | `None`（原地修改列表）     |
| 内存占用               | 迭代器，按需生成元素，节省内存 | 原地反转，不额外占用大内存 |
| 原对象是否改变         | 否                             | 是                         |
| 是否可对不可变对象使用 | 是（如字符串、元组）           | 否（不可变对象无此方法）   |

```python
# 原地反转列表
lst = [1, 2, 3]
lst.reverse()
print(lst)          # 输出: [3, 2, 1]
```

---

## 4. `reversed()` 与切片反转 `[::-1]` 的区别

- **切片反转 `seq[::-1]`**：创建一个新的序列（如新列表、新字符串），包含所有元素的反转副本。对于大型序列，这会占用额外内存。
- **`reversed(seq)`**：返回一个迭代器，逐个产生反转后的元素，不创建副本，内存效率更高。但若需要多次访问反转结果，可能需转换为列表。

```python
# 切片反转（生成新列表）
new_lst = lst[::-1]

# reversed 反转（生成迭代器）
rev_iter = reversed(lst)
```

---

## 5. 在自定义类中支持 `reversed()`

要使自定义类的实例支持 `reversed()`，有两种方式：

### 方法一：实现 `__reversed__()` 特殊方法
```python
class MyRange:
    def __init__(self, start, end):
        self.start = start
        self.end = end

    def __iter__(self):
        return iter(range(self.start, self.end))

    def __reversed__(self):
        # 返回反向迭代器
        return iter(range(self.end - 1, self.start - 1, -1))

mr = MyRange(1, 5)
print(list(reversed(mr)))   # 输出: [4, 3, 2, 1]
```

### 方法二：实现序列协议（`__len__` 和 `__getitem__`）
```python
class MySequence:
    def __init__(self, data):
        self.data = data

    def __len__(self):
        return len(self.data)

    def __getitem__(self, index):
        return self.data[index]

seq = MySequence([10, 20, 30])
print(list(reversed(seq)))   # 输出: [30, 20, 10]
```
当没有 `__reversed__` 时，`reversed()` 会回退到使用 `__len__` 和 `__getitem__` 从后向前获取元素。

---

## 6. 注意事项

- **惰性求值**：`reversed()` 返回的迭代器在遍历时才实际获取元素，因此如果原对象在迭代过程中被修改，可能会产生不可预料的结果（甚至抛出异常）。
- **只能使用一次**：迭代器耗尽后无法再次使用，若需多次遍历，应转换为列表。
- **对无序类型无效**：如集合（`set`）和字典（`dict`）没有明确的顺序，不能直接使用 `reversed()`（Python 3.7+ 中字典保留插入顺序，但 `reversed()` 不支持直接用于字典，需先转换为列表如 `list(dict)` 再反转）。

---

## 7. 扩展：`reversed()` 与 `sorted(..., reverse=True)`

- `reversed()` 只是单纯反转元素的**现有顺序**。
- `sorted(iterable, reverse=True)` 先对元素进行排序（按值大小或指定规则），然后以逆序输出。

```python
data = [3, 1, 4, 2]
print(list(reversed(data)))       # [2, 4, 1, 3]  # 反转原顺序
print(sorted(data, reverse=True)) # [4, 3, 2, 1]  # 排序后逆序
```

---

## 8. 总结

- `reversed()` 是一个强大且高效的内置函数，用于以反向顺序遍历序列或自定义对象。
- 它不会修改原对象，返回一个迭代器，适合处理大型数据或只需一次反向遍历的场景。
- 与切片反转相比，它更节省内存；与 `list.reverse()` 相比，它更通用且不会改变原数据。

通过合理运用 `reversed()`，可以写出简洁、高效的 Python 代码。