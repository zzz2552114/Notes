### 收下所有权 VS 绑定引用

因为 `std::unique_ptr` **不能拷贝，但可以移动**。  
`Take(std::unique_ptr<Widget> up)` 这个按值参数，不是拷贝，而是**移动构造**出来的。

关键点：

```cpp
std::unique_ptr<Widget> Take(std::unique_ptr<Widget> up) {
    return up;   // 返回时再移动给返回值
}
```

调用时：

```cpp
auto p = std::make_unique<Widget>(10);
auto q = Take(std::move(p));   // 必须 std::move
```

这里发生的事，约等于：

```cpp
std::unique_ptr<Widget> up = std::move(p);  // 移动构造，p 变空，up 接管对象
// 进入 Take 函数体
// return up; 约等于把 up 再移动给返回值
```

所以：

- `std::move(p)` 本身不移动，它只是把 `p` 转成右值。
- 真正移动发生在初始化形参 `up` 时：调用 `unique_ptr` 的移动构造函数。
- `up` 是函数内部的局部 `unique_ptr`，它拥有对象。
- 函数返回时，再把 `up` 移动出去。
- 调用者的 `p` 变成 `nullptr`。

你问“接收的时候也要用引用吧？”——不需要。  
如果形参写成引用，那只是**绑定**到调用者的对象，不是“收下所有权”。

对比三种写法：

### 1. 按值：收下所有权，推荐

```cpp
std::unique_ptr<Widget> Take(std::unique_ptr<Widget> up) {
    return up;
}

auto q = Take(std::move(p));  // 合法，p 变空，q 拥有对象
```

按值形参 `up` 是一个新的局部 `unique_ptr`，它通过移动构造从 `p` 那里接管对象。

### 2. 左值引用：不能接 `std::move(p)`

```cpp
std::unique_ptr<Widget> Take(std::unique_ptr<Widget>& up) {
    return std::move(up);
}
```

这个签名不能这样调用：

```cpp
Take(std::move(p));  // 错：右值不能绑定到非 const 左值引用
```

只能：

```cpp
Take(p);  // 可以，但函数里会从 p 移走
```

而且语义上不如按值清晰。

### 3. 右值引用：也能接，但只是引用

```cpp
std::unique_ptr<Widget> Take(std::unique_ptr<Widget>&& up) {
    return std::move(up);
}
```

这个可以接：

```cpp
Take(std::move(p));
```

但注意：`up` 是右值引用，它仍然只是调用者 `p` 的引用，不是新的局部对象。要转移所有权，还得显式 `std::move(up)`。

所以结论是：

> `unique_ptr` 不能拷贝，但能移动。  
> 按值参数不是拷贝，而是用 `std::move` 后的实参进行移动构造。  
> 接收所有权时，按值形参就是正确姿势；不需要写成引用。  
> 写成引用只是绑定，不是接管。