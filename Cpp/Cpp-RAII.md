# RAII

RAII = **Resource Acquisition Is Initialization（资源获取即初始化）**。

一句话理解：

> **让对象负责资源的获取和释放：对象构造时获取资源，对象析构时释放资源。**

所以核心就是：

```text
构造对象 → 获取资源 → 使用资源 → 对象析构 → 释放资源
```

这里的“资源”不只是内存，也可以是文件、socket、锁等。

**析构**可以理解为：**对象生命周期结束时的自动清理过程**。在 C++ 里，具体由**析构函数（destructor）**来完成。

- **构造**：对象诞生时调用构造函数，通常用来初始化、申请资源。
- **析构**：对象死亡时调用析构函数，通常用来释放资源、做清理。

------

## 1. 为什么需要 RAII？

手动管理资源很容易忘记释放：

```cpp
int *p = new int(42);

// 使用 p

delete p;
```

如果中间：

```cpp
return;
```

或者抛异常：

```cpp
throw ...;
```

就可能根本执行不到 `delete`，造成资源泄漏。

RAII 就是把这件事交给对象的生命周期。

------

## 2. 看这个例子

```cpp
class IntPtrManager {
public:
    // 构造函数
    IntPtrManager(int val) {
        ptr_ = new int;
        *ptr_ = val;
    }
	
    // 析构函数
    ~IntPtrManager() {
        if (ptr_) {
            delete ptr_;
        }
    }

private:
    int *ptr_;
};
```

比如：

```cpp
IntPtrManager a(445);
```

构造 `a` 的时候：

```cpp
ptr_ = new int;
*ptr_ = 445;
```

相当于：

```text
创建 a
 ↓
申请堆内存
 ↓
a 开始管理这块内存
```

当 `a` 离开作用域：

```cpp
{
    IntPtrManager a(445);
}
```

自动调用：

```cpp
~IntPtrManager()
```

然后：

```cpp
delete ptr_;
```

所以：

```text
a 活着  →  资源活着
a 死了  →  资源释放
```

这就是 RAII。

------

## 3. 为什么这个类不能随便拷贝？

假设允许：

```cpp
IntPtrManager a(445);
IntPtrManager b = a;
```

默认拷贝可能变成：

```text
a.ptr_ ───┐
          ├──→ [445]
b.ptr_ ───┘
```

两个对象管理同一块内存。

最后：

```text
b 析构 → delete
a 析构 → 再 delete
```

就可能出现 **double free**。

所以代码里直接禁止拷贝：

```cpp
IntPtrManager(const IntPtrManager &) = delete;
IntPtrManager &operator=(const IntPtrManager &) = delete;
```

意思就是：

> 一个资源只能由一个对象负责。

------

## 4. 不能拷贝，那怎么把资源交给另一个对象？

可以 **move**。

```cpp
IntPtrManager a(445);
IntPtrManager b(std::move(a));
```

移动构造：

```cpp
IntPtrManager(IntPtrManager&& other) {
    ptr_ = other.ptr_;
    // 这里由于 ptr_ 是 int* 没有移动构造设计
    // 所以不用 ptr_ = std::move(other.ptr_) 虽然也可以
    // 但是注意，vector 和 string 里面内置了 移动构造的设计，
    // 所以 a = move(b) 之后，b 就自动变成未定义了，但是这里没有，所以还要手动变成 nullptr
    other.ptr_ = nullptr;
}
```

移动前：

```text
a ───→ [445]
```

移动后：

```text
a ───→ nullptr

b ───→ [445]
```

注意：

> **move 主要是转移资源的所有权，不是把 `[445]` 这块内存复制一份。**

`std::move(a)` 本身也不是“真的执行移动”，它主要是把 `a` 转成可以进行移动操作的右值，从而让移动构造函数/移动赋值被调用。

------

## 5. moved-from 对象怎么理解？

移动之后：

```cpp
IntPtrManager b(std::move(a));
```

`a` **没有被销毁**，它还是一个活着的对象，只不过这里我们主动把：

```cpp
a.ptr_ = nullptr;
```

所以这个例子里 `a` 已经不再拥有原来的资源。

因此此时再：

```cpp
a.GetVal();
```

会因为：

```cpp
*nullptr
```

产生未定义行为。

但 `a` 本身并没有消失。

------

## 6. 移动赋值和移动构造不一样

移动构造：

```cpp
IntPtrManager b(std::move(a));
```

是在**创建 b**。

移动赋值：

```cpp
b = std::move(a);
```

是在**一个已经存在的 b 上接管资源**。

所以移动赋值要先把 `b` 原来拥有的资源释放掉：

```cpp
IntPtrManager &operator=(IntPtrManager &&other) {
    if (ptr_ == other.ptr_) {
        return *this;
    }

    if (ptr_) {
        delete ptr_;
    }

    ptr_ = other.ptr_;
    other.ptr_ = nullptr;

    return *this;
}
```

核心过程：

```text
b 原来的资源
     ↓
   delete
     ↓
b 接管 a 的资源
     ↓
a.ptr_ = nullptr
```

------

## 7. 为什么 `operator=` 返回 `IntPtrManager&`？

最后：

```cpp
return *this;
```

`this` 是指向当前对象的指针，所以：

```cpp
*this
```

就是“当前对象自己”。

这里返回的是：

```cpp
IntPtrManager&
```

也就是当前对象的**引用**。

这样：

```cpp
b = std::move(a);
```

这个赋值表达式执行完之后，结果仍然可以理解成：

```text
b
```

所以赋值可以继续参与表达式。

普通的：

```cpp
a = b = c;
```

本质上是：

```cpp
a = (b = c);
```

先执行：

```cpp
b = c
```

然后 `operator=` 返回 `b`，再执行：

```cpp
a = b
```

所以链式赋值能够成立。

这里的 `return *this` 可以直接理解成：

> “我已经赋值完了，现在把我自己返回出去。”



### 如果这个地方不返回引用而是 `IntPtrManager`

那么在返回的时候会新构造一个临时对象，而不是返回等号左边本身，那么在链式赋值的时候就会导致

`a = b = c` 先运算 `b = c` ，此时 `b = c，c = nullptr`

然后返回临时变量 `t`，此时 `a = t`，就不会把 `b` 变成 `null`

------

## 8. RAII 最后记这几句话就够了

### 核心

> **对象负责资源。**

### 生命周期

```text
构造
 ↓
获取资源

对象存活
 ↓
使用资源

析构
 ↓
释放资源
```

### 这段代码的设计

```text
IntPtrManager
├── new int              → 获取资源
├── ~IntPtrManager()     → 释放资源
├── 禁止 copy            → 防止两个对象同时拥有一个资源
└── 支持 move            → 转移资源所有权
```

现代 C++ 里，这种思想最典型的实现就是：

```cpp
std::unique_ptr
```

所以你以后看到 `unique_ptr`、`lock_guard`、`fstream` 之类的东西，都可以想到：

> **“哦，这东西本质上是在用 RAII 管资源。”**