C++ 里 `new` 速查：

```cpp
// 1. 单个对象
T* p = new T;          // 默认初始化：类调默认构造；int 等未初始化
T* p = new T(args);    // 调用构造函数
T* p = new T{args};    // 列表初始化，C++11
delete p;              // 释放单个对象

// 2. 内置类型初始化
int* a = new int;      // 未初始化
int* b = new int();    // 初始化为 0
int* c = new int(5);   // 初始化为 5

// 3. 数组
T* arr = new T[n];         // n 个元素，默认初始化
T* arr = new T[n]();       // 值初始化，内置类型通常为 0
int* arr = new int[3]{1,2,3};
delete[] arr;              // 必须 delete[]

// 4. new 失败不抛异常，返回 nullptr
#include <new>
T* p = new (std::nothrow) T;
if (!p) { /* 分配失败 */ }
delete p;

// 5. 定位 new：在已有内存上构造对象
#include <new>
alignas(T) unsigned char buf[sizeof(T)];
T* p = new (buf) T(args);
p->~T();   // 手动析构；不能用 delete p
```

关键规则：

- `new` 对应 `delete`
- `new[]` 对应 `delete[]`
- 不能混用，否则 UB
- `new` 失败默认抛 `std::bad_alloc`
- `delete nullptr` 安全
- 不要 `free` 掉 `new` 的指针，也不要 `delete` 掉 `malloc` 的指针
- 定位 new 不分配内存，不负责释放内存

现代 C++ 更推荐：

```cpp
auto p = std::make_unique<T>(args);  // C++14
auto q = std::make_shared<T>(args);
```

避免裸 `new/delete`，优先用智能指针和容器。