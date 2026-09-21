别慌，核心就一句话：

> **C++ 模板的尖括号里，除了可以放“类型”，还可以放“编译期就定死的值”。这个值会成为模板实例类型的一部分。**

你贴的三段代码分别展示：

1. 函数模板：从数组类型里推导出长度 `N`。
2. 类模板：把整数作为模板参数，生成带编译期常量的类型。
3. 类模板：把长度作为模板参数，生成固定大小数组类型。

下面逐段拆。

------

## 1. 基础：模板参数有两种

平时最常见的是类型模板参数：

```cpp
template <typename T>
void f(T x);
```

这里 `T` 是“类型”，调用时可以是 `int`、`double`、`std::string`。

但模板参数也可以是非类型模板参数，也就是“编译期常量值”：

```cpp
template <int N>
struct A {};

template <size_t N>
struct B {};
```

这里 `N` 不是类型，而是一个值。比如：

```cpp
A<5>   // N = 5
A<42>  // N = 42
B<10>  // N = 10
```

关键点：

- 这个值必须在编译期确定。
- 不能是运行期变量。
- 不同值会生成不同类型。
- `A<5>` 和 `A<6>` 是两个不同的类型。

例如这样不行：

```cpp
int n;
std::cin >> n;
A<n> a; // 错误：n 是运行期变量，不是编译期常量
```

这样才行：

```cpp
constexpr int n = 5;
A<n> a; // 可以，n 是编译期常量
```

------

## 2. 第一段：`ArrayLen`

```cpp
template <typename T, size_t N>
constexpr size_t ArrayLen(const T (&)[N]) {
    return N;
}
```

拆开看：

```cpp
template <typename T, size_t N>
```

这里有两个模板参数：

- `typename T`：类型参数，表示数组元素类型。
- `size_t N`：非类型参数，表示数组长度，类型是 `size_t`。

然后函数参数是：

```cpp
const T (&)[N]
```

这是一个“数组引用”：

- `T[N]` 是数组类型。
- `T (&)[N]` 是对这个数组的引用。
- `const T (&)[N]` 是对“元素为 const T、长度为 N 的数组”的引用。

注意括号不能少：

```cpp
const T (&)[N]
```

如果写成：

```cpp
const T &[N]
```

那是“引用数组”，语法不合法。

### 为什么必须用数组引用？

因为数组在 C++ 里很容易“退化”成指针。

比如：

```cpp
int arr[4];
int* p = arr; // arr 退化成指向首元素的指针
```

指针 `int*` 里没有“长度 4”这个信息。

如果你把函数写成：

```cpp
template <typename T, size_t N>
size_t ArrayLen(const T* p) {
    return N; // N 没法从 p 推导出来
}
```

调用：

```cpp
int arr[4];
ArrayLen(arr);
```

编译器只能从 `arr` 推导出 `T = int`，但推导不出 `N`，因为 `const T*` 里根本没有数组长度。

而写成数组引用：

```cpp
const T (&)[N]
```

数组不会退化，长度 `N` 被保留在类型里。

### 推导过程

假设：

```cpp
int arr[4] = {1, 2, 3, 4};
```

调用：

```cpp
ArrayLen(arr);
```

编译器拿实参类型 `int[4]` 去匹配形参 `const T (&)[N]`：

- 实参类型：`int[4]`
- 形参模式：`const T (&)[N]`
- 匹配结果：`T = int`，`N = 4`

于是编译器实例化出类似：

```cpp
constexpr size_t ArrayLen<int, 4>(const int (&)[4]) {
    return 4;
}
```

所以返回 `4`。

再比如：

```cpp
double d[7];
ArrayLen(d); // T = double, N = 7，返回 7
```

`constexpr` 的意思是：这个函数在条件满足时可以在编译期求值。这个例子里最核心的不是 `constexpr`，而是 `N` 是从数组类型里推导出来的编译期常量。