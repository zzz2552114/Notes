# 0. 总览：智能指针解决什么问题？

裸指针 `T*` 的问题：

```cpp
Point* p = new Point(1, 2);
// ...
delete p; // 必须手动释放
```

问题：

1. **忘记 `delete`**：内存泄漏。
2. **重复 `delete`**：二次释放，未定义行为。
3. **异常安全差**：`new` 之后、`delete` 之前抛异常，资源泄漏。
4. **所有权不清晰**：谁负责释放？函数返回指针时尤其明显。
5. **悬垂指针**：对象已释放，指针还在用。
6. **生命周期难以管理**：多个对象共享资源时，谁最后释放？

C++ 的解决办法是 **RAII**：  
资源在对象构造时获取，在对象析构时自动释放。智能指针就是管理动态分配对象生命周期的 RAII 类。

C++11 起 `<memory>` 提供：

- `std::unique_ptr`：独占所有权。
- `std::shared_ptr`：共享所有权，引用计数。
- `std::weak_ptr`：弱引用，不增加强引用计数，配合 `shared_ptr` 打破循环引用。

旧的 `std::auto_ptr` 已被废弃，C++17 移除。

---

# 1. `std::unique_ptr`：独占所有权

## 1.1 核心语义

`unique_ptr` 表示：

> 同一时间，只有一个 `unique_ptr` 拥有某个对象。  
> 它不能复制，只能移动。

也就是说：

```cpp
std::unique_ptr<Point> p1 = std::make_unique<Point>(1, 2);
// std::unique_ptr<Point> p2 = p1; // 错误：不能复制
std::unique_ptr<Point> p2 = std::move(p1); // 正确：所有权转移
// 此后 p1 为空，p2 拥有对象
```

## 1.2 原理

简化理解：

```cpp
template<class T, class Deleter = std::default_delete<T>>
class unique_ptr {
  T* ptr;
  Deleter deleter;
public:
  ~unique_ptr() {
    if (ptr) deleter(ptr);
  }

  unique_ptr(const unique_ptr&) = delete;
  unique_ptr& operator=(const unique_ptr&) = delete;

  unique_ptr(unique_ptr&& other) noexcept
    : ptr(other.ptr), deleter(std::move(other.deleter)) {
    other.ptr = nullptr;
  }

  // ...
};
```

特点：

- 默认删除器 `std::default_delete<T>` 调用 `delete ptr;`
- 对数组特化 `unique_ptr<T[]>` 调用 `delete[] ptr;`
- 无状态删除器通常不增加对象大小，因此 `unique_ptr` 通常和裸指针一样大。
- 删除器类型是 `unique_ptr` 类型的一部分：

```cpp
std::unique_ptr<T, D>
```

所以不同删除器会产生不同类型。

## 1.3 常见创建方式

C++14 起推荐：

```cpp
auto p = std::make_unique<Point>(2, 3);
auto arr = std::make_unique<Point[]>(5);
```

C++11 没有 `make_unique`，可以：

```cpp
std::unique_ptr<Point> p(new Point(2, 3));
std::unique_ptr<Point[]> arr(new Point[5]);
```

自定义删除器：

```cpp
auto file_deleter = [](FILE* f) {
  if (f) std::fclose(f);
};

std::unique_ptr<FILE, decltype(file_deleter)> fp(
    std::fopen("data.txt", "r"),
    file_deleter);
```

## 1.4 常用成员函数

```cpp
std::unique_ptr<Point> p = std::make_unique<Point>(1, 2);

p.get();        // 返回裸指针，不释放所有权
p.release();    // 放弃所有权，返回裸指针，p 变空；需要自己 delete
p.reset();      // 释放当前对象，p 变空
p.reset(new Point(3, 4)); // 释放旧对象，接管新对象
p.swap(other);  // 交换
if (p) { ... }  // 是否非空
p->GetX();      // 像指针一样使用
(*p).GetY();
```

注意：

- `release()` 不会释放对象，只是放弃管理。
- `reset()` 会释放当前管理的对象。
- `get()` 只是观察，不要用它去 `delete`，也不要用它创建另一个智能指针。

## 1.5 函数传参

### 不转移所有权，只观察

```cpp
void print(const Point& p);
void print(Point* p); // 不管理生命周期
```

### 通过引用修改 `unique_ptr` 本身

```cpp
void maybe_reset(std::unique_ptr<Point>& p) {
  p.reset(new Point(10, 20));
}
```

引用传递不会转移所有权，但函数可以修改 `p`。

### 转移所有权：按值传递 + `std::move`

```cpp
void take(std::unique_ptr<Point> p) {
  // p 现在是函数内的唯一所有者
}

auto p = std::make_unique<Point>(1, 2);
take(std::move(p)); // 所有权转移进函数
// p 现在为空
```

### 右值引用参数

```cpp
void maybe_take(std::unique_ptr<Point>&& p) {
  // p 是右值引用，绑定到外部对象
  // 如果函数内不 std::move(p)，外部对象仍然拥有所有权
  auto q = std::move(p); // 这里才真正转移
}
```

右值引用参数本身不自动转移所有权。

## 1.6 `unique_ptr` 与 `shared_ptr` 转换

`unique_ptr` 可以移动给 `shared_ptr`：

```cpp
std::unique_ptr<Point> up = std::make_unique<Point>(1, 2);
std::shared_ptr<Point> sp = std::move(up);
// up 变空，sp 接管对象
```

反过来不行：

```cpp
std::shared_ptr<Point> sp = std::make_shared<Point>(1, 2);
// std::unique_ptr<Point> up = sp; // 错误
```

因为 `shared_ptr` 可能还有别的共享者，不能强行变成独占。

## 1.7 注意事项

1. 优先用 `std::make_unique`。
2. `unique_ptr` 不能复制，容器中存它要用移动。
3. 返回 `unique_ptr` 是安全且常见的工厂模式。
4. 多态基类若通过 `unique_ptr<Base>` 删除 `Derived`，基类应有虚析构：

```cpp
struct Base {
  virtual ~Base() = default;
};

std::unique_ptr<Base> p = std::make_unique<Derived>();
```

否则 `delete Base*` 行为未定义。

5. `unique_ptr<T[]>` 用于数组；不要用 `unique_ptr<T>` 管理 `new T[]`。

---

# 2. `std::shared_ptr`：共享所有权

## 2.1 核心语义

`shared_ptr` 表示：

> 多个 `shared_ptr` 可以共同拥有同一个对象。  
> 内部使用引用计数，最后一个强引用销毁时，对象被释放。

```cpp
auto s1 = std::make_shared<Point>(1, 2);
auto s2 = s1; // 复制，引用计数增加
```

`s1` 和 `s2` 指向同一个 `Point`，`use_count()` 为 2。

## 2.2 原理：控制块 + 引用计数

一个 `shared_ptr` 通常包含：

1. 指向对象的指针 `T* ptr`；
2. 指向控制块的指针 `control_block*`。

控制块通常包含：

- 强引用计数：多少个 `shared_ptr` 拥有对象。
- 弱引用计数：多少个 `weak_ptr` 观察对象。
- 删除器 deleter。
- 分配器 allocator。
- 可能直接包含对象，若使用 `make_shared`。

生命周期规则：

- 复制 `shared_ptr`：强引用计数 +1。
- 移动 `shared_ptr`：强引用计数不变，源置空。
- 销毁 `shared_ptr`：强引用计数 -1。
- 强引用计数变为 0：销毁被管理对象。
- 弱引用计数也为 0：释放控制块。

`make_shared<T>(args...)` 通常一次分配同时容纳控制块和 `T` 对象，效率高、异常安全。  
`shared_ptr<T>(new T(...))` 通常需要两次分配：一次 `new T`，一次控制块。

## 2.3 常见创建方式

推荐：

```cpp
auto s1 = std::make_shared<Point>();
auto s2 = std::make_shared<Point>(2, 3);
```

从裸指针：

```cpp
std::shared_ptr<Point> s(new Point(2, 3));
```

但不要这样：

```cpp
Point* raw = new Point(2, 3);
std::shared_ptr<Point> a(raw);
std::shared_ptr<Point> b(raw); // 灾难：两个独立控制块，会二次释放
```

自定义删除器：

```cpp
std::shared_ptr<FILE> fp(
    std::fopen("data.txt", "r"),
    [](FILE* f) { if (f) std::fclose(f); });
```

## 2.4 复制、移动、`use_count`

```cpp
auto s3 = std::make_shared<Point>(2, 3);

std::cout << s3.use_count(); // 1

auto s4 = s3;                // 复制构造
std::cout << s3.use_count(); // 2

std::shared_ptr<Point> s5(s4); // 复制构造
std::cout << s3.use_count();   // 3

auto s6 = std::move(s5);       // 移动构造
// s5 变空，s6 接管；强引用计数仍然是 3
std::cout << s3.use_count();   // 3
```

关键点：

- 复制增加计数。
- 移动不增加计数，源变空。
- `use_count()` 主要用于调试，不要用它做业务逻辑判断。多线程下它只是近似值。

## 2.5 修改共享对象

因为多个 `shared_ptr` 指向同一对象：

```cpp
s3->SetX(445);
std::cout << s4->GetX(); // 445
std::cout << s5->GetX(); // 445
```

只要它们共享同一个控制块，就共享同一个对象。

## 2.6 函数传参：结合你给出的文件

你的文件里有三个函数：

```cpp
void modify_ptr_via_ref(std::shared_ptr<Point>& point) {
  point->SetX(15);
}

void modify_ptr_via_rvalue_ref(std::shared_ptr<Point>&& point) {
  point->SetY(645);
}

void copy_shared_ptr_in_function(std::shared_ptr<Point> point) {
  std::cout << point.use_count() << std::endl;
}
```

### 引用传递

```cpp
modify_ptr_via_ref(s2);
```

- 不复制 `shared_ptr`。
- 不增加 `use_count`。
- 函数内 `point` 是 `s2` 的别名。
- 修改 `point->SetX(15)` 就是修改 `s2` 指向的对象。

### 右值引用传递

```cpp
modify_ptr_via_rvalue_ref(std::move(s2));
```

这里要特别小心：

- `std::move(s2)` 只是把 `s2` 转换成右值引用。
- 函数参数是 `std::shared_ptr<Point>&&`，这是引用，不是新对象。
- 绑定右值引用本身**不会移动**，也不会增加引用计数。
- 函数内 `point` 仍然引用 `s2` 所管理的对象。
- 函数内 `point->SetY(645)` 修改的是同一个 `Point`。
- 函数结束后 `s2` 仍然有效，仍拥有对象。

所以运行后：

```cpp
s2->GetX() == 15
s2->GetY() == 645
```

### 按值传递

```cpp
copy_shared_ptr_in_function(s2);
```

- 按值传递会复制一个 `shared_ptr`。
- 函数内 `use_count` 增加 1。
- 如果调用前 `s2.use_count() == 1`，函数内是 2。
- 函数返回后，参数副本析构，`use_count` 回到 1。
- 按值传递不会影响外部 `s2` 的所有权，只是临时共享。

## 2.7 你的文件逐段解释

```cpp
std::shared_ptr<Point> s1; // 空 shared_ptr
std::shared_ptr<Point> s2 = std::make_shared<Point>(); // 默认构造 Point
std::shared_ptr<Point> s3 = std::make_shared<Point>(2, 3); // 自定义构造
```

`s1` 为空，`s2`、`s3` 非空。

```cpp
std::cout << (s1 ? "not empty" : "empty");
```

`shared_ptr` 可以像 bool 一样判断是否为空。

```cpp
std::shared_ptr<Point> s4 = s3; // 复制构造，use_count 从 1 到 2
std::shared_ptr<Point> s5(s4);  // 复制构造，use_count 到 3
```

`s3`、`s4`、`s5` 共享同一个 `Point`。

```cpp
s3->SetX(445);
```

因为共享，`s4`、`s5` 看到的 `x` 也是 445。

```cpp
std::shared_ptr<Point> s6 = std::move(s5);
```

移动后：

- `s5` 为空。
- `s6` 接管 `s5` 原来的共享所有权。
- 强引用计数仍然是 3：`s3`、`s4`、`s6`。

```cpp
modify_ptr_via_ref(s2);
modify_ptr_via_rvalue_ref(std::move(s2));
```

修改 `s2` 指向对象，`s2` 本身没有被移动走。

```cpp
copy_shared_ptr_in_function(s2);
```

函数内计数为 2，函数外恢复为 1。

---

# 3. `weak_ptr`：配合 `shared_ptr` 的弱引用

`weak_ptr` 不拥有对象，不增加强引用计数。

```cpp
std::shared_ptr<Point> sp = std::make_shared<Point>(1, 2);
std::weak_ptr<Point> wp = sp;

std::cout << sp.use_count(); // 1
```

`weak_ptr` 不能直接 `->` 或 `*`。要使用：

```cpp
if (auto locked = wp.lock()) {
  std::cout << locked->GetX();
} else {
  std::cout << "对象已释放";
}
```

`lock()`：

- 如果对象还活着，返回一个 `shared_ptr`，强引用计数临时 +1。
- 如果对象已释放，返回空 `shared_ptr`。

`weak_ptr` 的典型用途：

1. 打破循环引用。
2. 观察者模式。
3. 缓存，不阻止对象释放。
4. 避免悬垂指针。

---

# 4. 稍复杂例子

## 4.1 循环引用问题

错误示例：

```cpp
struct BadNode {
  std::shared_ptr<BadNode> next;
  std::shared_ptr<BadNode> prev;
};

auto a = std::make_shared<BadNode>();
auto b = std::make_shared<BadNode>();

a->next = b;
b->prev = a;
```

此时：

- `a` 被外部 `a` 和 `b->prev` 持有。
- `b` 被外部 `b` 和 `a->next` 持有。

外部 `a`、`b` 销毁后，它们各自仍被对方持有，引用计数不为 0，内存泄漏。

解决：把其中一个改成 `weak_ptr`。

```cpp
struct GoodNode {
  std::shared_ptr<GoodNode> next;
  std::weak_ptr<GoodNode> prev;
};

auto a = std::make_shared<GoodNode>();
auto b = std::make_shared<GoodNode>();

a->next = b;
b->prev = a; // weak_ptr 不增加强引用计数
```

这样外部 `a`、`b` 销毁后，对象可以正常释放。

## 4.2 `enable_shared_from_this`

如果类对象已经被 `shared_ptr` 管理，而类内部想返回一个指向自己的 `shared_ptr`，不能直接：

```cpp
std::shared_ptr<Widget> get() {
  return std::shared_ptr<Widget>(this); // 错误：会产生第二个控制块
}
```

正确做法：

```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
  std::shared_ptr<Widget> get() {
    return shared_from_this();
  }
};

auto w = std::make_shared<Widget>();
auto w2 = w->get(); // 正确，共享同一个控制块
```

注意：

- 对象必须已经被 `shared_ptr` 管理。
- 不要在构造函数里调用 `shared_from_this()`。
- 否则可能抛 `std::bad_weak_ptr`。

## 4.3 工厂：`unique_ptr` 转 `shared_ptr`

```cpp
std::unique_ptr<Point> create_point() {
  return std::make_unique<Point>(1, 2);
}

std::shared_ptr<Point> sp = create_point();
// unique_ptr 被移动进 shared_ptr
```

这是合法且常见的：先独占创建，再根据需要共享。

## 4.4 自定义删除器

`unique_ptr`：

```cpp
auto deleter = [](FILE* f) {
  if (f) std::fclose(f);
};

std::unique_ptr<FILE, decltype(deleter)> fp(
    std::fopen("a.txt", "r"),
    deleter);
```

`shared_ptr`：

```cpp
std::shared_ptr<FILE> fp(
    std::fopen("a.txt", "r"),
    [](FILE* f) { if (f) std::fclose(f); });
```

区别：

- `unique_ptr` 的删除器是类型的一部分。
- `shared_ptr` 的删除器在控制块里，不改变 `shared_ptr<T>` 类型。
- `make_shared` 不支持自定义删除器。

## 4.5 多态

```cpp
struct Base {
  virtual ~Base() = default;
  virtual void foo() = 0;
};

struct Derived : Base {
  void foo() override {}
};

std::unique_ptr<Base> up = std::make_unique<Derived>();
std::shared_ptr<Base> sp = std::make_shared<Derived>();
```

建议基类析构函数为 `virtual`。  
尤其是：

```cpp
std::unique_ptr<Base> p(new Derived);
std::shared_ptr<Base> q(new Derived);
```

这种写法中删除器可能按 `Base*` 删除，若 `Base` 析构非虚，行为未定义。

而：

```cpp
std::shared_ptr<Base> q = std::make_shared<Derived>();
```

因为控制块记录的是 `Derived` 的删除方式，通常更安全，但仍建议虚析构。

## 4.6 别名构造 `aliasing constructor`

```cpp
struct Foo {
  int x;
};

auto f = std::make_shared<Foo>();
std::shared_ptr<int> px(f, &f->x);
```

`px` 和 `f` 共享所有权，但 `px.get()` 返回 `&f->x`。  
`use_count` 会增加，`f` 活着时 `px` 就有效。

---

# 5. `unique_ptr` vs `shared_ptr` vs `weak_ptr` 对比

| 特性             | `unique_ptr`   | `shared_ptr`          | `weak_ptr`         |
| ---------------- | -------------- | --------------------- | ------------------ |
| 所有权           | 独占           | 共享                  | 不拥有             |
| 可复制           | 否             | 是                    | 是                 |
| 可移动           | 是             | 是                    | 是                 |
| 引用计数         | 无             | 强引用计数            | 弱引用计数         |
| 最后一个释放对象 | 自己析构时     | 最后一个强引用        | 不释放对象         |
| 大小             | 通常一个指针   | 通常两个指针 + 控制块 | 类似 shared_ptr    |
| 删除器           | 类型一部分     | 控制块中，类型擦除    | 无                 |
| 线程安全         | 无特殊保证     | 控制块计数原子安全    | 控制块计数原子安全 |
| 典型用途         | 独占资源、工厂 | 共享生命周期          | 打破循环、观察     |
| 性能             | 几乎零开销     | 原子计数、控制块开销  | 轻量观察           |

选择原则：

- 默认优先 `unique_ptr`。
- 确实需要多个所有者共享生命周期时，用 `shared_ptr`。
- 需要观察但不拥有，用 `weak_ptr`。
- 不要为了“方便”滥用 `shared_ptr`。

---

# 6. 重要注意事项清单

## 6.1 通用

1. 优先 `make_unique`、`make_shared`。
2. 不要用同一个裸指针初始化多个 `shared_ptr`：

```cpp
Point* raw = new Point;
std::shared_ptr<Point> a(raw);
std::shared_ptr<Point> b(raw); // 错误
```

3. 不要 `delete sp.get();`。
4. 不要用 `get()` 创建另一个智能指针。
5. 移动后源智能指针为空，不要再解引用。
6. 多态基类建议虚析构。
7. 明确所有权：函数参数用 `T*`、`T&` 表示不拥有；用 `unique_ptr` 表示转移；用 `shared_ptr` 表示共享。

## 6.2 `unique_ptr`

1. 不能复制，只能移动。
2. `release()` 返回裸指针并放弃所有权，需要手动释放。
3. `reset()` 释放当前对象。
4. 数组用 `unique_ptr<T[]>`。
5. 删除器类型是类型的一部分。

## 6.3 `shared_ptr`

1. 复制增加引用计数，移动不增加。
2. `use_count()` 只用于调试，不要用于逻辑判断。
3. 循环引用会导致内存泄漏，用 `weak_ptr` 打破。
4. 控制块引用计数线程安全，但被管理对象本身不自动线程安全。
5. 同一个 `shared_ptr` 实例的并发读写需要同步。
6. `make_shared` 效率高，但若长期存在 `weak_ptr`，对象内存可能延迟释放。
7. `shared_ptr` 没有 `release()`。
8. 不要从 `this` 裸指针创建 `shared_ptr`，用 `enable_shared_from_this`。
9. 自定义删除器不改变 `shared_ptr<T>` 类型。
10. C++17 起 `shared_ptr<T[]>` 支持数组；C++20 起 `make_shared<T[]>` 支持数组。

## 6.4 `weak_ptr`

1. 不增加强引用计数。
2. 不能直接解引用，必须 `lock()`。
3. `lock()` 后要检查是否为空。
4. 适合观察者、缓存、打破循环引用。
5. `expired()` 判断是否过期，但多线程下仍需 `lock()` 后检查。

---

# 7. 一句话总结

- `unique_ptr`：独占所有权，轻量、零开销首选。
- `shared_ptr`：共享所有权，引用计数，最后一个强引用释放对象。
- `weak_ptr`：弱引用，不拥有对象，用来观察和打破循环引用。
- 默认用 `unique_ptr`；确实需要共享时再用 `shared_ptr`；需要观察时用 `weak_ptr`。
- 你给出的文件主要演示了 `shared_ptr` 的复制、移动、`use_count`、共享修改、引用/右值引用/按值传参。关键点是：复制增加计数，移动转移所有权且源为空，右值引用参数本身不移动，按值传参会临时增加计数。