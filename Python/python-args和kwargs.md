## `*args` 和 `**kwargs` —— 万能参数的艺术

在 Python 中，`*args` 和 `**kwargs` 是用于函数定义时的特殊语法，可以让函数接受任意数量的参数。它们同样可以用在函数调用中，用于解包参数。下面逐字逐句剖析。

### 1. `*args`：接收任意数量的位置参数

当你在函数定义中写 `*args` 时，它的含义是：**把调用时传入的、所有未被显式形参捕获的“位置参数”打包成一个元组，并把这个元组赋值给变量 `args`**。

这里的 `args` 只是一个命名习惯，真正的魔法在于前面的星号 `*`。你可以写成 `*other`，但强烈建议使用 `*args`。

**最简单的例子：**

python

```
def print_args(*args):
    print(type(args))   # 看看 args 是什么类型
    for i, arg in enumerate(args):
        print(f"第{i}个参数: {arg}")

print_args("hello", 42, [1,2,3])
```



**逐句解释执行过程：**

- 调用 `print_args("hello", 42, [1,2,3])` 传入了三个位置参数。
- 在函数内部，`*args` 将所有位置参数打包成一个元组 `("hello", 42, [1,2,3])`，`args` 变量就指向这个元组。
- `print(type(args))` 输出 `<class 'tuple'>`。
- 循环遍历元组，依次打印每个元素。

**如果函数既有普通形参，又有 `\*args`：**

python

```
def mixed(first, *args):
    print(f"第一个参数: {first}")
    print(f"剩余参数: {args}")

mixed("a", "b", "c", "d")
```



- 调用时，实参 `"a"` 被匹配给形参 `first`，剩下的 `"b"`, `"c"`, `"d"` 被打包成元组 `("b", "c", "d")` 赋给 `args`。

- 输出：

  text

  ```
  第一个参数: a
  剩余参数: ('b', 'c', 'd')
  ```

  

### 2. `**kwargs`：接收任意数量的关键字参数

`**kwargs` 在函数定义中的意思是：**把调用时传入的、所有未被显式形参捕获的“关键字参数”打包成一个字典，并把这个字典赋值给变量 `kwargs`**。

注意是**两个星号**。`kwargs` 也是习惯命名（keyword arguments 的缩写），可改为其他名字。

**简单例子：**

python

```
def print_kwargs(**kwargs):
    print(type(kwargs))
    for key, value in kwargs.items():
        print(f"{key} = {value}")

print_kwargs(name="Alice", age=30, city="Shanghai")
```



- 调用时传入了三个关键字参数：`name="Alice"`, `age=30`, `city="Shanghai"`。
- 函数定义中的 `**kwargs` 将这些参数打包成字典：`{"name": "Alice", "age": 30, "city": "Shanghai"}`。
- 循环打印键值对。

**混合使用普通参数、`\*args` 和 `\**kwargs`：**

python

```
def full_func(a, b, *args, option=True, **kwargs):
    print(f"a={a}, b={b}")
    print(f"args={args}")
    print(f"option={option}")
    print(f"kwargs={kwargs}")

full_func(1, 2, 3, 4, option=False, x=10, y=20)
```



- `a=1, b=2` 按位置匹配。
- `3, 4` 是额外位置参数，打包给 `args`。
- `option` 是默认参数，通过关键字指定 `option=False` 被捕获，不会进入 `kwargs`。
- `x=10, y=20` 是未匹配的关键字参数，打包进 `kwargs`。
- 顺序规则：定义时，`*args` 必须在普通位置参数之后，`**kwargs` 必须在最后；中间可以有仅关键字参数（如 `option` 前的 `*args` 使其成为仅关键字参数）。

### 3. 在函数调用中使用 `*` 和 `**` 进行解包

除了定义，星号还能在函数调用时**解包可迭代对象和字典**。

python

```
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))   # 等价于 add(1, 2, 3)

params = {"a": 5, "b": 6, "c": 7}
print(add(**params)) # 等价于 add(a=5, b=6, c=7)
```



混合解包：

python

```
data = (2, 3)
print(add(1, *data))   # add(1,2,3)
```



### 4. 为什么在装饰器中必须用 `*args, **kwargs`？

装饰器要包装一个未知的函数，这个函数可能有任意数量、任意类型的参数。为了让包装器能原样把参数传递进去，我们必须用 `*args, **kwargs` 全部接收，然后再用 `*args, **kwargs` 全部传递。这正是万能参数的典型应用场景，也是我们下一步详细解构装饰器的基础。