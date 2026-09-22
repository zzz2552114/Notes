## const指针

```cpp
const char* const&
```

**是省略了参数名的类型**。如果补上变量名，就是：

```cpp
const char* const& c;
```

或者等价地写：

```cpp
const char* const &c;
```

空格不影响含义。它们都表示：

> `c` 是一个引用，引用到一个 const 指针，这个指针指向 const char。

---

## 拆开看

```cpp
const char* const& c
  |    |  |  |  |
  |    |  |  |  +-- c 是引用 &
  |    |  |  +----- 这个指针本身是 const
  |    |  +-------- 指针 *
  |    +----------- char
  +---------------- char 是 const
```

从 `c` 往外读：

1. `c` 是引用 `&`；
2. 引用的是一个 `const` 指针 `* const`；
3. 这个指针指向 `const char`。

所以整体是：

```cpp
const char* const& c;
```

---

## 为什么第二个 `const` 要放在 `*` 后面？

这是 C/C++ 声明语法的规则：

- `const` 在 `*` 左边：修饰“指针指向的东西”；
- `const` 在 `*` 右边：修饰“指针自己”。

例子：

```cpp
const char* p;        // p 是指针，指向 const char；p 自己可以改
char* const p;        // p 是 const 指针，指向 char；p 自己不能改
const char* const p;  // p 是 const 指针，指向 const char；都不能改
```

所以：

```cpp
const char* const
```

意思是：

- `const char`：指向的内容是 const；
- `* const`：指针自己是 const。

然后再加 `&`，就变成：

```cpp
const char* const&
```

---

## 回到模板

主模板如果是：

```cpp
template <typename T>
std::string Describe(const T& value);
```

当 `T = const char*` 时：

```cpp
const T&
```

展开：

```cpp
const (const char*) &
```

因为 `T` 是一个指针类型，所以 `const T` 表示“这个指针本身是 const”：

```cpp
const char* const
```

再加引用：

```cpp
const char* const&
```

所以：

```cpp
template <>
std::string Describe<const char*>(const char* const&);
```

这里的第二个 `const` 不是 `const char*` 自带的，而是来自主模板里的 `const T&` 中的 `const`。

---

## 如果觉得乱，用别名

```cpp
using CharPtr = const char*;          // 指向 const char 的指针
using ConstCharPtr = const CharPtr;   // const 指针，指向 const char
using ConstCharPtrRef = const CharPtr&;

// ConstCharPtrRef 展开就是：
// const char* const&
```

所以：

```cpp
const char* const& c;
```

可以理解为：

```cpp
const CharPtr& c;
```

也就是：

> `c` 是对一个 `const` 的 `CharPtr` 的引用。

---

总结一句话：

```cpp
const char* const& c;
```

读作：

> `c` 是引用，引用到一个 const 指针，该指针指向 const char。

第二个 `const` 放在 `*` 后面，是因为它修饰的是“指针自己”，而不是“指向的 char”。