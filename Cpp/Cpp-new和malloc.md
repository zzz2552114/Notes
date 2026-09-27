一句话总结：**`malloc/free` 只负责“原始内存”的分配和释放；`new/delete` 除了分配/释放内存，还负责调用对象的构造/析构函数。**  
所以 C++ 里管理类对象，通常必须用 `new/delete`。

## 1. 主要区别表

| 对比点    | `new/delete`                 | `malloc/free`         |
| --------- | ---------------------------- | --------------------- |
| 本质      | C++ 运算符表达式             | C 标准库函数          |
| 返回类型  | `T*`，类型安全               | `void*`，C++ 中要强转 |
| 大小计算  | 自动推导                     | 手动 `sizeof`         |
| 构造/析构 | 调用构造、析构               | 不调用                |
| 初始化    | `new T()` 可值初始化         | 返回未初始化内存      |
| 失败行为  | 默认抛 `std::bad_alloc`      | 返回 `NULL`           |
| 数组      | `new[]/delete[]`             | 手动算字节，`free`    |
| 释放配对  | `delete` / `delete[]`        | `free`                |
| 可重载    | `operator new/delete` 可重载 | 不可重载              |
| 混用      | UB 未定义行为                | UB 未定义行为         |

核心流程：

```cpp
// new Foo(42) 大致做两件事：
// 1. 调用 operator new 分配 sizeof(Foo) 字节
// 2. 在这块内存上调用 Foo::Foo(42)

// delete p 大致做两件事：
// 1. 调用 p->~Foo()
// 2. 调用 operator delete 释放内存
```

而：

```cpp
// malloc(sizeof(Foo)) 只分配内存，不构造对象
// free(p) 只释放内存，不析构对象
```

---

## 2. 用类对象例子看区别

```cpp
#include <iostream>
#include <cstdlib>
#include <new>

struct Foo {
    int x;
    Foo(int v = 0) : x(v) {
        std::cout << "Foo(" << x << ") 构造\n";
    }
    ~Foo() {
        std::cout << "Foo(" << x << ") 析构\n";
    }
};

int main() {
    std::cout << "--- malloc/free ---\n";
    Foo* a = (Foo*)std::malloc(sizeof(Foo));
    // a 指向一块原始内存，不是已构造的 Foo 对象
    // a->x = 1;  // 危险：对象未构造，使用是 UB
    std::free(a);  // 只释放内存，不会调用析构

    std::cout << "--- new/delete ---\n";
    Foo* b = new Foo(42);  // 分配 + 构造
    std::cout << b->x << '\n';
    delete b;              // 析构 + 释放

    std::cout << "--- new[]/delete[] ---\n";
    Foo* arr = new Foo[3]; // 调用 3 次默认构造
    delete[] arr;          // 调用 3 次析构
}
```

输出重点：

```text
--- malloc/free ---
（没有任何构造/析构输出）
--- new/delete ---
Foo(42) 构造
42
Foo(42) 析构
--- new[]/delete[] ---
Foo(0) 构造
Foo(0) 构造
Foo(0) 构造
Foo(0) 析构
Foo(0) 析构
Foo(0) 析构
```

可以看到：

- `malloc` 不会调用 `Foo` 构造函数。
- `free` 不会调用 `Foo` 析构函数。
- `new` 会调用构造函数。
- `delete` 会调用析构函数。
- `new[]` 会为每个元素调用构造，`delete[]` 会为每个元素调用析构。

如果 `Foo` 内部管理了资源，比如：

```cpp
struct Buffer {
    char* data;
    Buffer() { data = new char[100]; }
    ~Buffer() { delete[] data; }
};
```

用 `malloc` 创建 `Buffer`，然后 `free`，会导致：

- `data` 没有被初始化；
- `Buffer` 构造函数没调用，内部资源没分配；
- `Buffer` 析构函数没调用，内部资源没释放。

所以对于非平凡类型，`malloc/free` 基本不能直接拿来创建对象。

---

## 3. 内置类型例子

对于 `int` 这种简单类型，差别看起来小，但语义仍不同：

```cpp
int* p = (int*)std::malloc(sizeof(int)); // 未初始化
*p = 10;
std::free(p);

int* q = new int;    // 未初始化
int* r = new int();  // 初始化为 0
int* s = new int(5); // 初始化为 5
delete q;
delete r;
delete s;
```

`malloc` 需要手动写 `sizeof(int)`，还要强转。  
`new int(5)` 自动知道类型，并直接初始化。

---

## 4. 失败处理区别

```cpp
Foo* p = new Foo(1); // 分配失败默认抛 std::bad_alloc

Foo* q = (Foo*)std::malloc(sizeof(Foo));
if (q == nullptr) {
    // malloc 失败返回 NULL
}

Foo* r = new (std::nothrow) Foo(1);
if (r == nullptr) {
    // nothrow new 分配失败返回 nullptr
}
```

注意：`new (std::nothrow)` 只影响分配失败，如果构造函数本身抛异常，异常仍会传播。

另外，`new` 如果构造函数抛异常，已经分配的内存会自动释放，不容易泄漏。`malloc` 没有构造过程，也就没有这个自动处理。

---

## 5. 为什么不能混用？

下面都是未定义行为：

```cpp
Foo* p = new Foo(1);
// free(p);   // UB：不调用析构，且 new/free 不配对

Foo* q = (Foo*)std::malloc(sizeof(Foo));
// delete q;  // UB：delete 会调用析构，但对象根本没构造
```

即使某些简单类型上“看起来能跑”，也不能依赖。  
因为 `new` 和 `malloc` 可能来自不同分配器，`delete` 还需要调用析构，`new[]` 还可能保存数组长度信息，`free` 不知道这些。

---

## 6. `malloc` 和 `new` 也能配合：定位 new

`malloc` 可以分配原始内存，然后用定位 new 在上面构造对象：

```cpp
void* raw = std::malloc(sizeof(Foo));
Foo* p = new (raw) Foo(7); // 在 raw 上构造，不分配内存

p->~Foo();                 // 手动析构
std::free(raw);            // 释放原始内存
```

这说明：

- `malloc` 负责原始内存；
- 定位 `new` 负责在已有内存上构造对象；
- 析构要手动调用；
- 最后用 `free` 释放原始内存。

---

## 7. 结论

- `malloc/free`：管字节，管原始内存，不管对象生命周期。
- `new/delete`：管对象，分配内存 + 构造对象，析构对象 + 释放内存。
- `new[]` 必须配 `delete[]`。
- `malloc` 必须配 `free`。
- 不要混用。
- C++ 中优先使用 `std::make_unique`、`std::make_shared`、`std::vector` 等，避免裸 `new/delete`，更不要用 `malloc/free` 管理类对象。

记忆口诀：**`malloc/free` 管内存，`new/delete` 管对象。**