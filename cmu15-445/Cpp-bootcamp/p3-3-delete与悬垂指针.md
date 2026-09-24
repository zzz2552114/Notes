原代码

```cpp
void Clear(){
    N *curr = head;
    while (curr)
    {
        N *tmp = curr->next;
        delete curr;
        curr = tmp;
    }
}
```

`delete` 不是“把指针变量删掉”，而是“释放这个指针所指向的、由 `new` 动态分配出来的对象”。

你的代码里：

```cpp
N *curr = head;
...
delete curr;
```

第一步 `curr == head`，所以效果确实相当于：

```cpp
delete head; // 但注意：删除的是 head 指向的那个节点对象，不是 head 这个变量本身
```

但 `delete head;` 之后：

- `head` 指向的节点对象被析构、内存被释放；
- `head` 这个指针变量本身仍然存在；
- `head` 的值不会自动变成 `nullptr`；
- `head` 仍然保存着原来那个节点的地址，变成“悬垂指针 / 野指针”。

例如：

```cpp
N* head = new N;
N* curr = head;

delete curr;

// 此时：
// curr 仍然等于原来的地址
// head 也仍然等于原来的地址
// 但它们指向的对象已经没了
```

所以 `delete` 不会把 `head` 变成 `nullptr`。它只是释放内存，不修改指针变量。

你的 `Clear()` 循环结束时，所有节点都被 `delete` 了，但 `head` 还指着第一个已经被释放的节点。于是：

```cpp
if (head) { ... }   // 会误判链表非空
head->next;         // use-after-free，未定义行为
Clear();            // 再次从 head 开始 delete，double free
```

所以清空链表后通常要加：

```cpp
head = nullptr;
```

如果还维护尾指针、大小，也应一起清：

```cpp
tail = nullptr;
size = 0;
```

修正后：

```cpp
void Clear() {
    N *curr = head;
    while (curr) {
        N *tmp = curr->next;
        delete curr;
        curr = tmp;;
    }

    head = nullptr;   // 关键：让链表头指针不再悬垂
    // tail = nullptr;
    // size = 0;
}
```

总结：

- `delete curr`：释放 `curr` 指向的节点对象；
- 它不会把 `curr` 或 `head` 置空；
- `head = nullptr`：修改的是“链表头指针变量”，表示链表已空；
- 不加 `head = nullptr`，`head` 就是悬垂指针，后续访问或再次清空会出问题。