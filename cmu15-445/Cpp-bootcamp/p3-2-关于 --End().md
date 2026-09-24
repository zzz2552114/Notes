## 关于 --End()

原代码

```cpp
DLLIterator End(){
    return DLLIterator(tail,tail);
}
```

主要错在 **`DLL::End()` 的语义**：它返回了最后一个节点 `tail`，但标准迭代器的 `end()` 应该表示“最后一个元素之后”，即 `nullptr`。因此：

- `[begin, end)` 遍历会漏掉尾节点，所以 6 个元素只遍历到 5 个。
- `--End()` 时会先执行 `curr = curr->prev`，于是从 `tail` 跑到 `tail->prev`，得到倒数第二个值 5，而不是最后一个值 6。
- 后续段错误通常也是反向遍历/继续 `--` 时对 `nullptr` 取 `prev` 导致。

你代码中 `DLLIterator` 已经保存了 `tail`，就是为这种情况准备的：`end()` 的 `curr` 应该是 `nullptr`，但 `tail` 仍然保存链表尾节点，以便 `--` 能从尾后回到尾节点。

需要改两处：

### 1. 修改 `End()`

```cpp
DLLIterator End(){
    return DLLIterator(nullptr, tail);
}
```

而不是：

```cpp
DLLIterator End(){
    return DLLIterator(tail, tail);
}
```

### 2. 修改 `operator--()`，让从 `end()` 递减能回到 `tail`

```cpp
DLLIterator& operator--(){
    if(curr == nullptr)
        curr = tail;
    else
        curr = curr->prev;
    return *this;
}
```

后置 `--` 也应复用前缀逻辑，否则从 `end()` 做 `it--` 仍会对 `nullptr` 取 `prev`：

```cpp
DLLIterator operator--(int){
    DLLIterator tmp = *this;
    --(*this);
    return tmp;
}
```

同理，后置 `++` 建议也改成复用前缀：

```cpp
DLLIterator operator++(int){
    DLLIterator tmp = *this;
    ++(*this);
    return tmp;
}
```

- 空链表：`Begin().curr == nullptr`，`End().curr == nullptr`，仍然相等。
- 正向遍历：从 `head` 一直走到 `tail`，再 `++` 到 `nullptr`，等于 `End()`，不会漏尾节点。
- 反向：`auto it = End(); --it;` 会从 `nullptr` 回到 `tail`，所以 `*it == 6`。

所以测试失败的两个点和段错误基本都来自这里：**`end()` 不应该指向 `tail`，而应该指向 `tail` 后面的空位置。**