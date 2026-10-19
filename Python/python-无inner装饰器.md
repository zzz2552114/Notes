Q:

```python
TOOLS = [] 

def tool(func): 
    """登记一个工具：名字取模块名，描述取 docstring"""
    name = func.__module__.rsplit(".", 1)[-1]  
    TOOLS.append((name, (func.__doc__ or "").strip(), func)) 
    return func      
```
一般来说装饰器不都有def inner吗?我没见过这样的装饰器啊。
正常来说有def inner最后返回inner，装饰器不是运行时替换而是直接替换对吧，那么相当于func直接就变成inner，那这里是什么替换呢？好奇怪啊



A;

你的疑惑很对，这确实是一个"不包装"的装饰器。关键在于**装饰器返回值是什么**，而不是有没有 `inner`。

## 先看有 `inner` 的那种

```python
def deco(func):
    def inner(*a, **kw):
        print("before")
        return func(*a, **kw)
    return inner

@deco
def foo(): ...
```

展开成：

```python
foo = deco(foo)   # foo 现在指向 inner
```

因为 `deco` 返回的是 `inner`，所以名字 `foo` 被**替换**成 `inner`。调用 `foo()` 实际上在调 `inner()`，于是能插入前后逻辑。这是"包装型"装饰器。

## 再看你贴的这种

```python
def tool(func):
    TOOLS.append((name, doc, func))   # 只做副作用
    return func                        # 原封不动返回
```

展开成：

```python
foo = tool(foo)   # tool 返回的就是 foo 本身
```

所以 `foo` **还是原来那个函数对象**，一点没变。装饰器在这里的唯一作用是在**定义时执行一次注册副作用**，把 `(名字, 描述, 函数)` 塞进 `TOOLS`。

## 回答你的几个疑问

**"装饰器不是运行时替换而是定义时替换对吧？"**
对，准确说是**定义时绑定**：`@tool` 等价于在 `def` 结束后立刻执行一次 `foo = tool(foo)`，只在模块导入时跑一次，之后每次调用 `foo()` 都跟装饰器无关。

**"相当于 func 直接变成 inner"**
那是有 `inner` 的情况。这里没有 `inner`，`tool` 返回 `func` 本身，所以 `func` 不变。

**"这里是什么替换呢？"**
**没有替换**。这叫"只带副作用的装饰器"（side-effect-only decorator），常用于注册表模式：

- Flask 的 `@app.route(...)` 返回值也是 `f` 本身，注册完路由就原样返回；
- pytest 的 `@pytest.fixture`；
- 各种插件系统的 `@register`。

## 为什么这里不需要 `inner`

因为它的目的**不是改变调用行为**，而是"登记"——登记完就该让函数保持原样，能被正常调用。如果加了 `inner` 反而多一层无意义的调用开销，还得套 `functools.wraps` 保住 `__name__`、`__doc__` 等元数据。



## 执行时机：模块顶层代码执行的那一刻

装饰器不是"调用时"执行，而是**函数定义语句执行完的瞬间**执行一次。Python 的模块顶层代码是自上而下逐行执行的，遇到 `def` 就创建函数对象，紧接着看到 `@tool` 就立刻调用 `tool(func)` 并把返回值绑定回名字。所以：

```python
# mod.py
print("A")
@tool
def foo(): ...
print("B")
```

导入 `mod` 时输出顺序是：

```
A
（执行 tool，把东西 append 进 TOOLS）
B
```

**只执行一次**，发生在模块被导入（或作为 `__main__` 运行）的阶段，之后无论 `foo()` 被调用多少次都不再碰 `tool`。

## 属于"准备阶段"吗？

算。通常说的启动流程是：

1. 解释器启动
2. **import 阶段**：导入入口模块及其依赖，各模块顶层代码执行 ← 注册就发生在这里
3. 进入 `if __name__ == "__main__":` 或框架的主循环
4. 真正调用函数

所以它确实是"运行前的准备阶段"，更准确叫 **import 期 / 模块初始化期**。它不是独立的"预处理"，就是普通代码执行的一部分——`@tool` 只是语法糖，等价于在 `def` 下面写一行 `foo = tool(foo)`。

## 一个关键推论：注册依赖"模块被导入"

因为是 import 期执行，**没被 import 的模块里的工具不会被注册**。比如：

```python
# tools/__init__.py 里没 import 子模块
# 只 import 了 tools，那 tools/a.py 里 @tool 的函数根本不会跑
```

注册表模式几乎都要配一个"自动发现"机制来兜底，例如：

- 显式在 `__init__.py` 里 `from . import a, b, c`
- 用 `pkgutil.iter_modules` / `importlib.import_module` 遍历包导入所有子模块
- 框架层（如 Flask 的 app factory）在启动时主动 import 各蓝图模块

这也是为什么你常看到注册表项目里有一堆看似"没被用到"的 import——它们的作用就是触发模块执行、完成注册。

## 想验证的话

```python
# 直接在装饰器里打日志
def tool(func):
    print(f"[register] {func.__module__}.{func.__name__}")
    TOOLS.append(...)
    return func
```

或者在入口模块第一行 `import mod` 前后各 print 一句，能看到注册发生在 `import mod` 那一行。

## 小结

- 执行者：模块顶层代码，逐行执行
- 时机：`def` 语句之后立刻执行一次，属 import 期
- 频率：每个被导入的模块里每个 `@tool` 各一次
- 前提：所在模块必须真的被导入，否则不注册——这是注册表模式最常见的坑