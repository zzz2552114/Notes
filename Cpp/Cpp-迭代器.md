# C++ 迭代器笔记：从指针到自定义双向链表迭代器

先说本质：迭代器就是“指向容器里某个元素的对象”，用起来像指针。

它把“怎么遍历容器”和“算法怎么处理元素”拆开了。核心操作就两个：`*it` 拿当前位置的值，`++it` 走到下一个位置。

C 数组里的 `int*` 就是最朴素的迭代器：`*iter` 取值，`iter++` 前进。

STL 里的惯例是：`begin()` 指向第一个元素，`end()` 指向最后一个元素后面的“尾后”位置。所以区间是左闭右开 `[begin, end)`。循环总写 `it != end()`，而不是 `<=`，也不能解引用 `end()`。

遍历时推荐写前缀 `++it`，因为后缀 `it++` 要先复制一个旧迭代器再返回，泛型代码里可能多一次拷贝。范围 for 底层也是找 `begin()` / `end()`

再拆文件里的实现。`Node` 有 `next_`、`prev_`、`value_`。

`DLLIterator` 内部就一个 `Node* curr_`，构造函数接一个节点指针。重点操作符：

```cpp
DLLIterator& operator++() {        // 前缀 ++it
  curr_ = curr_->next_;
  return *this;
}

DLLIterator operator++(int) {      // 后缀 it++
  DLLIterator temp = *this;
  ++*this;
  return temp;
}
```

前缀返回递增后的自己，而且返回引用；后缀返回递增前的旧副本。那个 `int` 参数只是占位，用来区分前缀和后缀。

`==` 和 `!=` 比较的是 `curr_`，也就是看两个迭代器是不是指向同一个节点。

`operator*` 返回 `curr_->value_`。但这里有个细节：它返回的是 `int` 拷贝，所以只能读，不能写。

比如 `*iter = 10;` 是不行的。真实工程里通常返回 `int&` 或 `const int&`，这样迭代器才能既读又改。

`Begin()` 返回 `DLLIterator(head_)`，`End()` 返回 `DLLIterator(nullptr)`。

`End()` 是哨兵，表示“最后一个元素之后”，不能解引用，也不能对它 `++`，因为 `curr_` 是空指针，`curr_->next_` 会炸。遍历写法就是：

```cpp
for (DLLIterator iter = dll.Begin(); iter != dll.End(); ++iter) {
  std::cout << *iter << " ";
}
```

前后缀两个循环输出结果一样，但前缀更常见，也更省。



`DLL` 本身是头插，`InsertAtHead` 是 O(1)，更新 `head_`、旧头的 `prev_` 和 `size_`。

析构函数从头遍历 `delete`，这个没问题。但有个坑：`DLL` 没有禁用拷贝构造和拷贝赋值，默认浅拷贝会让两个 `DLL` 共享同一串节点，析构时 double free。

工程里要么删掉拷贝，要么实现深拷贝，按 Rule of Three / Five 来。

如果想让这个迭代器更像 STL 迭代器，还缺不少东西：

小写 `begin()` / `end()`、const 版本、`operator--`、`operator->`、`*` 返回引用、默认构造、类型别名 `value_type`、`difference_type`、`pointer`、`reference`、`iterator_category`，以及单独的 `const_iterator`。

标准算法会通过这些 traits 判断迭代器能力。当前 `Begin` / `End` 大写不符合 STL 惯例，范围 for 也不会认，除非加小写 `begin` / `end` 成员或自由函数。

最后记几个坑：

- 尾后迭代器不能碰；

- 迭代器失效规则随容器不同，`vector` 扩容后旧迭代器可能全废，`list` / DLL 删除节点后指向该节点的迭代器废；
- 自定义迭代器要注意节点生命周期，别返回局部节点指针；`++it` 通常比 `it++` 更省。



## 1. 指针就是最朴素的迭代器

```cpp
int arr[10] = {};
int* iter = arr;

int zero_elem = *iter; // 解引用，取当前位置的值
++iter;                // 前进到下一个元素
int first_elem = *iter;
```

对应的 `begin/end` 区间写法：

```cpp
int* begin = arr;
int* end = arr + 10; // 尾后位置，不能解引用

for (int* it = begin; it != end; ++it) {
  std::cout << *it << ' ';
}
```

泛型算法里只依赖这些操作：

```cpp
template <class It>
void print_all(It first, It last) {
  for (; first != last; ++first) {
    std::cout << *first << ' ';
  }
}
```

核心就三件事：`*it` 取值，`++it` 前进，`it != last` 判断结束。

---

## 2. 原文件里的 DLLIterator：先能跑，但偏只读

原文件的结构：

```cpp
struct Node {
  Node(int val)
      : next_(nullptr), prev_(nullptr), value_(val) {}

  Node* next_;
  Node* prev_;
  int value_;
};

class DLLIterator {
 public:
  explicit DLLIterator(Node* head) : curr_(head) {}

  DLLIterator& operator++() {          // 前缀 ++it
    curr_ = curr_->next_;
    return *this;
  }

  DLLIterator operator++(int) {        // 后缀 it++
    DLLIterator temp = *this;
    ++*this;
    return temp;
  }

  bool operator==(const DLLIterator& itr) const {
    return curr_ == itr.curr_;
  }

  bool operator!=(const DLLIterator& itr) const {
    return curr_ != itr.curr_;
  }

  int operator*() {                    // 注意：返回 int 拷贝
    return curr_->value_;
  }

 private:
  Node* curr_;
};
```

这个 `operator*` 返回的是 `int` 拷贝，所以：

```cpp
*iter = 10; // 不行，编译不过
```

它只能读，不能写。真实迭代器通常返回引用。

---

## 3. 改成能读能写：`operator*` 返回引用

```cpp
class DLLIterator {
 public:
  explicit DLLIterator(Node* head) : curr_(head) {}

  DLLIterator& operator++() {
    curr_ = curr_->next_;
    return *this;
  }

  DLLIterator operator++(int) {
    DLLIterator temp = *this;
    ++*this;
    return temp;
  }

  bool operator==(const DLLIterator& itr) const {
    return curr_ == itr.curr_;
  }

  bool operator!=(const DLLIterator& itr) const {
    return curr_ != itr.curr_;
  }

  int& operator*() {               // 返回引用，可读可写
    return curr_->value_;
  }

  const int& operator*() const {   // const 迭代器对象只读
    return curr_->value_;
  }

 private:
  Node* curr_;
};
```

这样就能：

```cpp
for (DLLIterator it = dll.Begin(); it != dll.End(); ++it) {
  *it += 10;
  std::cout << *it << ' ';
}
```

---

## 4. 补双向能力：`operator--` 和 `--end()`

原文件只有 `End()` 返回 `nullptr`，所以 `--End()` 会直接崩，因为 `nullptr->prev_` 不合法。

要支持双向迭代器，需要让迭代器知道尾节点。可以让 `DLL` 维护 `tail_`，迭代器同时保存 `curr_` 和 `tail_`。

```cpp
#include <cstddef>
#include <iterator>

class DLLIterator {
 public:
  using iterator_category = std::bidirectional_iterator_tag;
  using value_type = int;
  using difference_type = std::ptrdiff_t;
  using pointer = int*;
  using reference = int&;

  DLLIterator(Node* curr, Node* tail)
      : curr_(curr), tail_(tail) {}

  reference operator*() const {
    return curr_->value_;
  }

  pointer operator->() const {
    return &curr_->value_;
  }

  DLLIterator& operator++() {
    curr_ = curr_->next_;
    return *this;
  }

  DLLIterator operator++(int) {
    DLLIterator temp = *this;
    ++*this;
    return temp;
  }

  DLLIterator& operator--() {
    if (curr_ == nullptr) {
      curr_ = tail_;          // --end() 回到最后一个元素
    } else {
      curr_ = curr_->prev_;
    }
    return *this;
  }

  DLLIterator operator--(int) {
    DLLIterator temp = *this;
    --*this;
    return temp;
  }

  bool operator==(const DLLIterator& itr) const {
    return curr_ == itr.curr_;
  }

  bool operator!=(const DLLIterator& itr) const {
    return !(*this == itr);
  }

 private:
  Node* curr_;
  Node* tail_;
};
```

对应的 `DLL`：

```cpp
class DLL {
 public:
  using iterator = DLLIterator;

  DLL() : head_(nullptr), tail_(nullptr), size_(0) {}

  ~DLL() { clear(); }

  void InsertAtHead(int val) {
    Node* new_node = new Node(val);
    new_node->next_ = head_;

    if (head_ != nullptr) {
      head_->prev_ = new_node;
    } else {
      tail_ = new_node;
    }

    head_ = new_node;
    ++size_;
  }

  iterator begin() {
    return iterator(head_, tail_);
  }

  iterator end() {
    return iterator(nullptr, tail_);
  }

 private:
  void clear() {
    Node* current = head_;
    while (current != nullptr) {
      Node* next = current->next_;
      delete current;
      current = next;
    }
    head_ = nullptr;
    tail_ = nullptr;
    size_ = 0;
  }

  Node* head_;
  Node* tail_;
  size_t size_;
};
```

这样：

```cpp
DLL dll;
dll.InsertAtHead(3);
dll.InsertAtHead(2);
dll.InsertAtHead(1);

for (auto it = dll.begin(); it != dll.end(); ++it) {
  std::cout << *it << ' ';
}

auto it = dll.end();
--it;
std::cout << *it << '\n'; // 输出最后一个元素
```

---

## 5. 更像 STL：类型别名和 `iterator_traits`

标准算法会通过 `std::iterator_traits` 问迭代器一些问题：

```cpp
using iterator_category = std::bidirectional_iterator_tag;
using value_type = int;
using difference_type = std::ptrdiff_t;
using pointer = int*;
using reference = int&;
```

有了这些，下面这种代码才能正常工作：

```cpp
#include <algorithm>
#include <iostream>

std::for_each(dll.begin(), dll.end(), [](int& x) {
  x *= 2;
});

auto it = std::find(dll.begin(), dll.end(), 2);
if (it != dll.end()) {
  std::cout << "found: " << *it << '\n';
}
```

`std::distance` 对双向迭代器是 O(n)：

```cpp
auto n = std::distance(dll.begin(), dll.end());
std::cout << n << '\n';
```

`std::advance` 也是：

```cpp
auto it = dll.begin();
std::advance(it, 2);
std::cout << *it << '\n';
```

---

## 6. 范围 for 需要小写 `begin/end`

范围 for：

```cpp
for (int v : dll) {
  std::cout << v << ' ';
}
```

等价于找 `dll.begin()` 和 `dll.end()`，所以成员函数最好小写：

```cpp
class DLL {
 public:
  using iterator = DLLIterator;

  iterator begin() { return iterator(head_, tail_); }
  iterator end()   { return iterator(nullptr, tail_); }

  // const 版本以后可以补 const_iterator
};
```

原文件里的 `Begin()` / `End()` 大写不符合 STL 惯例，范围 for 不认。

---

## 7. 拷贝和析构：别忘了 Rule of Three

原文件 `DLL` 没有禁用拷贝。默认拷贝是浅拷贝，两个 `DLL` 会共享同一串 `Node`，析构时 double free。

```cpp
class DLL {
 public:
  DLL(const DLL&) = delete;
  DLL& operator=(const DLL&) = delete;

  DLL(DLL&& other) noexcept
      : head_(other.head_), tail_(other.tail_), size_(other.size_) {
    other.head_ = nullptr;
    other.tail_ = nullptr;
    other.size_ = 0;
  }

  DLL& operator=(DLL&& other) noexcept {
    if (this != &other) {
      clear();
      head_ = other.head_;
      tail_ = other.tail_;
      size_ = other.size_;
      other.head_ = nullptr;
      other.tail_ = nullptr;
      other.size_ = 0;
    }
    return *this;
  }

  ~DLL() { clear(); }
};
```

如果不想写移动，至少把拷贝删掉：

```cpp
DLL(const DLL&) = delete;
DLL& operator=(const DLL&) = delete;
```

---

## 8. 遍历示例：前缀、后缀、修改元素

```cpp
DLL dll;
dll.InsertAtHead(6);
dll.InsertAtHead(5);
dll.InsertAtHead(4);
dll.InsertAtHead(3);
dll.InsertAtHead(2);
dll.InsertAtHead(1);

std::cout << "prefix ++:\n";
for (auto it = dll.begin(); it != dll.end(); ++it) {
  std::cout << *it << ' ';
}

std::cout << "\npostfix ++:\n";
for (auto it = dll.begin(); it != dll.end(); it++) {
  std::cout << *it << ' ';
}

std::cout << "\nmodify:\n";
for (auto it = dll.begin(); it != dll.end(); ++it) {
  *it += 100;
  std::cout << *it << ' ';
}
```

前缀 `++it` 通常比后缀 `it++` 更省，因为后缀要先复制一个旧迭代器：

```cpp
DLLIterator operator++(int) {
  DLLIterator temp = *this;
  ++*this;
  return temp;
}
```

---

## 9. 迭代器失效：不同容器不一样

DLL 头插不会让已有节点失效，但逻辑位置会变：

```cpp
DLL dll;
dll.InsertAtHead(2);

auto it = dll.begin();   // 指向 2
dll.InsertAtHead(1);     // 头插 1

std::cout << *it << '\n'; // 仍然输出 2
```

但如果实现 `Erase`，删掉某个节点后，指向它的迭代器就失效了。

`vector` 更危险：

```cpp
std::vector<int> v{1, 2, 3};
auto it = v.begin();

v.push_back(4); // 可能扩容，it 可能失效
// std::cout << *it; // 不要再解引用
```

常见规则：

```cpp
// vector：扩容后所有迭代器失效
// list / DLL：删除节点只失效指向被删节点的迭代器
// map / set：删除只失效被删元素的迭代器
// unordered_*：rehash 可能导致迭代器失效
```

---

## 10. 一句话收尾

迭代器就是容器和算法之间的桥。最小接口是：

```cpp
*it      // 解引用
++it     // 前进
it != end // 判断结束
```

DLL 迭代器的关键实现就是：

```cpp
DLLIterator& operator++() {
  curr_ = curr_->next_;
  return *this;
}

int& operator*() {
  return curr_->value_;
}

bool operator!=(const DLLIterator& itr) const {
  return curr_ != itr.curr_;
}
```

再补上 `--`、`tail_`、类型别名、小写 `begin/end`、引用返回和拷贝控制，它就更接近标准库迭代器了。