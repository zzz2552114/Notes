# 析构函数、delete 与资源释放

原代码

```cpp
void Clear() {
    N* curr = head;
    while (curr) {
        N* tmp = curr->next;  
        delete curr;           
        curr = tmp;            
    }
    head = nullptr;
}
```

## `delete curr` 会调用析构函数吗

会。`delete curr;` 一定会调用 `N` 的析构函数，然后释放 `curr` 指向的那块内存。

`delete` 做两件事：

1. 调用 `curr->~N()`；
2. 调用 `operator delete` 释放内存。

即使 `N` 没有手写析构函数，编译器也会生成一个默认析构函数，`delete curr` 时同样会调用它，只是默认析构不会自动 `delete next`。

## 一、 `delete curr` 不会自动删除 `next`

假设节点定义是：

```cpp
struct N {
    int v;
    N* next;
};
```

它没有自定义析构函数。编译器生成的析构函数类似：

```cpp
~N() {
    // 对成员 v 和 next 做析构
    // 但 next 是裸指针，析构指针本身不会 delete 它指向的对象
}
```

所以 `delete curr;` 只会删除 `curr` 这一个节点，不会删除 `curr->next` 及其后面的节点。因此需要用循环逐个节点删除。

## 二、 如果 `N` 的析构函数里写了 `delete next`

两种设计只能选一种：

### 设计一：节点析构不删除 `next`，由链表类统一删除

推荐这种。

```cpp
struct N {
    int v;
    N* next;
    // 不要 delete next
};

void Clear() {
    N* curr = head;
    while (curr) {
        N* tmp = curr->next;
        delete curr;
        curr = tmp;
    }
    head = nullptr;
}
```

### 设计二：节点析构负责删除 `next`

这时链表类只删除头节点：

```cpp
struct N {
    int v;
    N* next;

    ~N() {
        delete next;
    }
};

void Clear() {
    delete head;   // 递归删除整条链
    // 也就是，删除head的时候会删next，又会调用next的析构不断向下递归
    head = nullptr;
}
```

但递归删除长链表可能导致栈溢出，所以一般不推荐。

## 三、 手写析构函数体是否阻止成员自动析构

不会。你手写的析构函数体只是在成员析构之前额外执行的一段代码。执行顺序是：

1. 先执行你写的函数体；
2. 函数体结束后，编译器自动按声明顺序的逆序析构每个成员；
3. 成员 `next`（`unique_ptr<N>`）析构时，自动 `delete` 它持有的下一个节点，触发下一个节点的 `~N()`；
4. 全部成员析构完，节点的内存才被 `operator delete` 释放。

所以：

```cpp
~N() {
    ++g_frees;
}
```

等价于：

```cpp
~N() {
    ++g_frees;
    // 编译器在这里插入：
    // next.~unique_ptr<N>();   // 自动 delete 下一个节点
    // val.~int();              // int 是平凡类型，无操作
}
```

函数体不会“替代”成员析构，只是插在成员析构之前。

## 四、 为什么有的析构函数里要 delete，编译器为什么不自动插入

编译器只负责析构成员本身，不负责析构成员指向的东西。

对每个成员，编译器生成的析构函数会调用该成员自己的析构函数：

```cpp
struct N {
    int val;
    N* next;
};
```

编译器生成的析构等价于：

```cpp
~N() {
    val.~int();     // int 是平凡类型，析构=什么都不做
    next.~N*();     // 指针是平凡类型，析构=什么都不做
}
```

注意：`next.~N*()` 只是析构指针变量 `next` 本身，不是析构 `next` 指向的那个对象。

指针变量 `next` 里存的是一个地址值。析构这个变量，只是说“这个变量不用了”，它占的那 8 字节空间被回收，但***它指向的堆内存完全没被碰 ！！！***

为什么编译器不自动 delete？因为编译器不知道 `next` 指向的东西该不该删。裸指针只存一个地址，不携带任何所有权信息。看几个例子：

```cpp
N* p1 = new N(1);      // 堆上，该 delete
N n;
N* p2 = &n;            // 指向栈上对象，绝不能 delete
N* p3 = other.next;    // 指向别人的节点，不该由我 delete
N* p4 = nullptr;       // 空，delete 也无所谓
```

这四种情况，类型都是 `N*`，编译器从类型上完全无法区分谁拥有资源。如果编译器一律自动 `delete p`，那 `p2`、`p3` 就会崩，甚至 double free。所以编译器选择：裸指针的析构不做任何事，由程序员自己负责。

那为什么 `unique_ptr` 就会自动 delete？因为 `unique_ptr` 把所有权写进了类型里。它的析构函数长这样（简化）：

```cpp
template<class T>
unique_ptr<T>::~unique_ptr() {
    delete ptr_;   // 明确知道：我拥有 ptr_，我负责删
}
```

当你写：

```cpp
struct N {
    std::unique_ptr<N> next;
};
```

编译器生成的 `~N()` 调用 `next.~unique_ptr<N>()`，而这个析构函数内部会 `delete` 它持有的对象。于是：编译器负责调用成员的析构函数；`unique_ptr` 负责在它自己的析构函数里 delete。删资源这件事不是编译器干的，是 `unique_ptr` 这个类型自己干的。

## 五、 对比表

| 成员类型                | 编译器生成的析构做什么 | 资源释放了吗                 |
| ----------------------- | ---------------------- | ---------------------------- |
| `int val;`              | 析构 int，无操作       | 没有资源                     |
| `N* next;`              | 析构指针变量，无操作   | ❌ 不释放，需手写 delete      |
| `std::string s;`        | 调用 `~string()`       | ✅ string 内部自己 delete     |
| `std::vector<int> v;`   | 调用 `~vector()`       | ✅ vector 内部自己 delete     |
| `std::unique_ptr<N> p;` | 调用 `~unique_ptr()`   | ✅ unique_ptr 内部自己 delete |
| `std::shared_ptr<N> p;` | 调用 `~shared_ptr()`   | ✅ 引用计数减到 0 时 delete   |

规律：凡是把资源封装进类型、在析构里写了解放逻辑的，编译器自动调用就够；裸指针没封装，就必须你自己写。

## 六、 重点注意事项

- `delete curr` 会调用析构函数，但不会自动删除 `next`，除非你自己在 `~N()` 里写了 `delete next`。
- 如果 `~N()` 里写了 `delete next`，那么 `Clear` 里就不要再循环 `delete` 了，否则会重复释放。推荐让节点析构不删除 `next`，由链表类统一循环删除。
- 手写析构函数体不会阻止成员自动析构。成员析构总是自动发生，顺序是声明逆序。
- 你的 `{ ++g_frees; }` 只是在成员析构之前加一次计数，这正是你想要的。
- 真正会出问题的是在函数体里手动删 `next` 指向的对象，那样会和 `unique_ptr` 的自动删除冲突。
- 长链表用 `unique_ptr` 递归析构时，也可能栈溢出。