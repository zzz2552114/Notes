对代码的解释

```cpp
template<typename T,size_t N>
size_t ArrayLen(const T (&arr)[N]){
    return N;
}
```

## 0. 参数名

## 参数名字怎么写？

你原来写的是无名参数：

```cpp
const T (&)[N]
```

如果要给参数起名字，必须写成：

```cpp
const T (&arr)[N]
```

完整定义：

```cpp
template<typename T, size_t N>
size_t ArrayLen(const T (&arr)[N]) {
    return N;
}
```

这里 `arr` 是参数名。

注意不能写成：

```cpp
const T &arr[N]   // 错误
```

因为 `[]` 优先级高于 `&`，`const T &arr[N]` 会被理解成“数组元素是引用”，而引用数组不合法。

必须加括号：

```cpp
const T (&arr)[N]
```

读作：

> `arr` 是一个引用，引用到长度为 `N` 的 `const T` 数组。

因为函数体里只返回 `N`，根本没用到 `arr`，所以也可以省略参数名，避免未使用参数警告：

```cpp
template<typename T, size_t N>
constexpr size_t ArrayLen(const T (&)[N]) {
    return N;
}
```

## 1. 调用时发生了什么？

代码：

```cpp
template<typename T, size_t N>
size_t ArrayLen(const T (&arr)[N]) {
    return N;
}
```

调用：

```cpp
int a[10];
size_t n = ArrayLen(a);
```

编译器推导：

- 实参 `a` 的类型是 `int[10]`；
- 形参类型是 `const T (&)[N]`；
- 匹配后得到：
  - `T = int`
  - `N = 10`

于是隐式实例化出一个函数，类似：

```cpp
size_t ArrayLen<int, 10>(const int (&arr)[10]) {
    return 10;
}
```

所以 `ArrayLen(a)` 返回 `10`。

如果你再写：

```cpp
int b[10];
size_t m = ArrayLen(b);
```

`b` 也是 `int[10]`，所以它调用的是**同一个实例** `ArrayLen<int, 10>`。

如果你写：

```cpp
int c[5];
size_t k = ArrayLen(c);
```

`c` 是 `int[5]`，编译器会生成另一个实例：

```cpp
size_t ArrayLen<int, 5>(const int (&arr)[5]) {
    return 5;
}
```

所以模板并没有“只能用于一个数组”。它是为每一种 `(T, N)` 组合生成一个专用函数。相同类型和长度的数组共享同一个实例。

---

## 2. 为什么模板参数里必须有 `N`？

因为 `N` 出现在形参类型里：

```cpp
const T (&)[N]
```

这里的 `N` 是数组长度。数组长度是**数组类型的一部分**。

在 C++ 里：

```cpp
int[10]
int[5]
double[10]
```

是不同类型。

所以：

- `T` 表示元素类型；
- `N` 表示数组长度；
- 它们都必须先声明，编译器才能用实参去推导。

如果写成：

```cpp
template<typename T>
size_t ArrayLen(const T (&)[N]); // 错误
```

`N` 没有声明，编译器不知道 `N` 是什么，直接报错。

也不能写成：

```cpp
template<typename T, typename N>
size_t ArrayLen(const T (&)[N]); // 错误
```

因为 `typename N` 表示 `N` 是一个类型，但数组长度需要的是一个编译期整数值，所以必须是非类型模板参数，通常是：

```cpp
std::size_t N
```

---

## 3. 这个模板有什么用？

它的核心作用是：**安全地获取原生数组长度，并且保留数组类型，不让数组退化成指针。**

普通函数参数如果写成：

```cpp
void f(int arr[10]);
```

其实等价于：

```cpp
void f(int* arr);
```

数组会退化成指针，长度信息丢失。

而：

```cpp
template<typename T, size_t N>
size_t ArrayLen(const T (&arr)[N]);
```

参数是“数组的引用”，数组不会退化成指针，所以 `N` 能被保留下来。

用途举例：

### 1. 获取数组长度

```cpp
int a[10];
size_t n = ArrayLen(a); // n = 10
```

### 2. 用于模板中确定长度

```cpp
template<typename T, size_t N>
void print(const T (&arr)[N]) {
    for (size_t i = 0; i < N; ++i) {
        std::cout << arr[i] << ' ';
    }
}
```

这里 `N` 是编译期常量，循环边界明确，编译器还能优化。

### 3. 用于编译期检查

```cpp
constexpr int a[10] = {};
static_assert(ArrayLen(a) == 10);
```

### 4. 避免 `sizeof(arr) / sizeof(arr[0])` 的误用

`sizeof` 对数组有效，但如果数组退化成指针，就会算错：

```cpp
void f(int* p) {
    // sizeof(p) / sizeof(p[0]) 是指针大小除以元素大小，错误
}
```

而 `ArrayLen` 只接受真正的数组引用，传指针会编译错误，所以更安全。

C++17 之后，标准库已经提供了 `std::size`，对原生数组也能返回长度：

```cpp
int a[10];
size_t n = std::size(a); // 10
```

所以自己写 `ArrayLen` 通常用于学习模板推导，或者特殊需求。

---

## 4. 参数名字怎么写？

你原来写的是无名参数：

```cpp
const T (&)[N]
```

如果要给参数起名字，必须写成：

```cpp
const T (&arr)[N]
```

完整定义：

```cpp
template<typename T, size_t N>
size_t ArrayLen(const T (&arr)[N]) {
    return N;
}
```

这里 `arr` 是参数名。

注意不能写成：

```cpp
const T &arr[N]   // 错误
```

因为 `[]` 优先级高于 `&`，`const T &arr[N]` 会被理解成“数组元素是引用”，而引用数组不合法。

必须加括号：

```cpp
const T (&arr)[N]
```

读作：

> `arr` 是一个引用，引用到长度为 `N` 的 `const T` 数组。

因为函数体里只返回 `N`，根本没用到 `arr`，所以也可以省略参数名，避免未使用参数警告：

```cpp
template<typename T, size_t N>
constexpr size_t ArrayLen(const T (&)[N]) {
    return N;
}
```

---

## 5. 有开销吗？

分两部分看。

### 编译期开销

每遇到一个新的 `(T, N)` 组合，编译器会实例化一个函数。

例如：

```cpp
int a[10];
int b[10];
double c[10];
int d[5];
```

会生成：

```cpp
ArrayLen<int, 10>
ArrayLen<double, 10>
ArrayLen<int, 5>
```

`a` 和 `b` 共享 `ArrayLen<int, 10>`。

这些实例函数体都只有：

```cpp
return N;
```

所以编译期实例化开销很小，但确实存在。如果程序里有大量不同长度和类型的原生数组，会生成多个小函数。

### 运行时开销

通常**没有运行时开销**。

原因：

- 函数体不访问数组内容；
- 不复制数组；
- 参数是引用，只传地址；
- 返回值 `N` 是编译期常量；
- `constexpr` 函数隐式 `inline`；
- 优化后，`ArrayLen(a)` 往往直接被替换成常量 `10`，连函数调用都没有。

即使不优化，调用也只是一个极小的函数，返回一个立即数。开销可以忽略。

只有一种情况可能产生实际函数符号：你取了函数地址，例如：

```cpp
auto p = &ArrayLen<int, 10>;
```

这时编译器必须生成这个函数实体。但函数体仍然只是 `return 10;`，非常小。

---

## 6. 总结

你的理解基本正确，但更准确的说法是：

> `ArrayLen` 是一个函数模板。传入原生数组时，编译器根据数组类型 `T[N]` 推导出 `T` 和 `N`，隐式实例化一个专用函数。这个函数返回 `N`。  
> 它锁定的不是某个具体数组变量，而是数组类型。相同 `(T, N)` 的数组共享同一个实例。

模板参数 `N` 必须写，因为数组长度 `N` 出现在形参类型 `const T (&)[N]` 中，是类型的一部分。

它的用途是：安全获取原生数组长度，防止数组退化为指针，并可在模板和编译期上下文中使用。

运行时开销通常为零；编译期有轻微的模板实例化开销。实际项目中也可以直接用 C++17 的 `std::size`。