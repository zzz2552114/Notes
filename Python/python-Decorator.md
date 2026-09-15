# Decorator

## 1. 装饰器简介

装饰器本质上是一个**可调用对象**，它接收一个函数或类作为参数，并返回一个新的函数或可调用对象。

基本语法：

```python
@decorator
def func():
    ...
```

等价于：

```python
func = decorator(func)
```

也就是说，`@` 只是一个语法糖。装饰器在**函数定义时**就会生效，而不是在函数调用时才生效。

装饰器常见用途：

- 日志打印
- 性能计时
- 权限校验
- 缓存
- 参数检查
- 函数注册

---

## 2. 最简单的装饰器：`simple_wrapper`

```python
def simple_wrapper(func):
    def inner():
        print("Before calling")
        print(func())
        print("After calling")
    return inner


def hello():
    return "Hi!"
```

### 原理解释

`simple_wrapper(func)` 这个函数的目的，是返回一个函数，让一个变量可以绑定这个返回的函数。

如果不用 `simple_wrapper`，直接写 `inner`，会有问题：

1. `hello = inner(hello)` 是获取确定的返回值，不是装饰。
2. `hello = inner` 又没有传入 `func` 参数。

所以需要 `def simple_wrapper`，在它里面再定义一个 `inner`，然后返回内部的 `inner`。而 `inner` 里调用了传给 `simple_wrapper` 的函数。

这样就实现了对 `hello` 的装饰。

### 手工装饰

```python
# 手工装饰
hello = simple_wrapper(hello)
# hello()
```

此时 `hello` 已经被替换成 `simple_wrapper` 内部的 `inner`。

如果调用 `hello()`，会输出：

```text
Before calling
Hi!
After calling
```

注意：因为 `inner` 没有 `return`，所以 `hello()` 的返回值是 `None`。

### 使用 `@` 语法糖

```python
@simple_wrapper
def hey():
    return "Hey!"
# hey()
```

这实际上等价于：

```python
def hey():
    return "Hey!"

hey = simple_wrapper(hey)
```

也就是把 `hey` 变成装饰器里面那个函数，然后 `hey` 本身会被传到装饰器里作为参数。

---

## 3. 带参数的装饰器

### 3.1 手动版：`wrapper(times, func)`

```python
def wrapper(times, func):
    def inner(*args, **kwargs):
        # 注意：
        # 在函数定义中，*args 表示把位置参数打包成 args 元组
        # **kwargs 表示把关键字参数打包成 kwargs 字典
        # 在调用 func 时，*args 和 **kwargs 又用于解包
        print("Before calling")
        for i in range(times):
            print(f'that is {i}')
        result = func(*args, **kwargs)
        print("After calling")
        return result
    return inner


def f(a, b):
    return f'a+b = {a+b}'


f = wrapper(8, f)
# print(f(7, 8))
```

如果取消注释 `print(f(7, 8))`，会先打印 `Before calling`，再打印 `that is 0` 到 `that is 7`，然后调用原来的 `f(7, 8)`，再打印 `After calling`，最后输出：

```text
a+b = 15
```

### 3.2 装饰器版：三层函数

如果要用 `@` 语法糖实现带参数的装饰器，就必须使用三层函数：

```python
def outter_wrapper(times):
    def inner_wrapper(func):
        def inner(*args, **kwargs):
            print("Before calling")
            for i in range(times):
                print(f'that is {i}')
            result = func(*args, **kwargs)
            print("After calling")
            return result
        return inner
    return inner_wrapper


@outter_wrapper(times=3)
def f2(a, b):
    return f'a+b = {a+b}'


print(f2(10, 20))
```

### 语法糖 `@` 在底层做了什么？

当你写 `@outter_wrapper(times=3)` 时，Python 解释器**强制拆成了两步**：

- **第一步（求值）**：执行 `outter_wrapper(times=3)`，此时只传入了 `times`，**必须返回一个东西**。
- **第二步（装饰）**：把第一步返回的东西当成“装饰器”，再去调用它，并把 `f2` 传进去，即 `上一步的返回值(f2)`。

输出结果：

```text
Before calling
that is 0
that is 1
that is 2
After calling
a+b = 30
```

三层函数的分工：

1. 第一层：接收装饰器参数，例如 `times`
2. 第二层：接收被装饰的函数，例如 `func`
3. 第三层：真正替代原函数，接收原函数的参数并执行包装逻辑

---

## 4. 多层装饰器

```python
def deco1(func):
    print("deco1 包装")
    def wrapper1(*args, **kwargs):
        print("wrapper1 进入")
        result = func(*args, **kwargs)
        print("wrapper1 退出")
        return result
    return wrapper1


def deco2(func):
    print("deco2 包装")
    def wrapper2(*args, **kwargs):
        print("wrapper2 进入")
        result = func(*args, **kwargs)
        print("wrapper2 退出")
        return result
    return wrapper2


@deco1
@deco2
def target():
    print("执行 target")


target()
```

### 装饰顺序

```python
@deco1
@deco2
def target():
    ...
```

等价于：

```python
target = deco1(deco2(target))
```

也就是说：

1. 先执行 `deco2(target)`，打印 `deco2 包装`，返回 `wrapper2`
2. 再执行 `deco1(wrapper2)`，打印 `deco1 包装`，返回 `wrapper1`

所以 `target` 先变成内部有原 `target` 的 `wrapper2`，再变成内部有 `wrapper2` 的 `wrapper1`。

### 调用顺序

调用 `target()` 时，实际调用的是最外层的 `wrapper1`。

输出顺序为：

```text
deco2 包装
deco1 包装
wrapper1 进入
wrapper2 进入
执行 target
wrapper2 退出
wrapper1 退出
```

规律：

- 包装阶段：从下到上
- 调用阶段：从上到下
- 退出阶段：从下到上

---

## 5. 类装饰器

装饰器不仅可以是函数，也可以是类。只要类实现了 `__call__`，它的实例就可以像函数一样被调用。

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"第 {self.count} 次调用")
        return self.func(*args, **kwargs)


@CountCalls
def hi():
    print("Hi")


hi()
hi()
```

这里：

```python
@CountCalls
def hi():
    print("Hi")
```

等价于：

```python
def hi():
    print("Hi")

hi = CountCalls(hi)
```

所以 `hi` 不再是原来的函数，而是一个 `CountCalls` 实例。由于 `CountCalls` 实现了 `__call__`，所以 `hi()` 仍然可以像函数一样调用。

输出结果：

```text
第 1 次调用
Hi
第 2 次调用
Hi
```

---

## 6. 小结

1. 装饰器本质：
   ```python
   func = decorator(func)
   ```

2. `@decorator` 只是语法糖：
   ```python
   @decorator
   def func():
       ...
   ```
   等价于：
   ```python
   func = decorator(func)
   ```

3. 带参数的装饰器需要三层函数：
   ```python
   def outer(参数):
       def decorator(func):
           def wrapper(*args, **kwargs):
               ...
           return wrapper
       return decorator
   ```

4. 多层装饰器：
   - 包装时从下到上
   - 调用时从上到下
   - 退出时从下到上

5. 类装饰器：
   - 类需要实现 `__init__`
   - 类需要实现 `__call__`
   - 装饰后变量绑定的是类实例

6. 可选补充：实际开发中，函数装饰器内部常用 `functools.wraps` 保留原函数的元信息：

   ```python
   from functools import wraps
   
   def deco(func):
       @wraps(func)
       def wrapper(*args, **kwargs):
           return func(*args, **kwargs)
       return wrapper
   ```

   这样可以保留原函数的 `__name__`、`__doc__` 等属性，避免被装饰后信息丢失。