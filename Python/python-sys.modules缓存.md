## 一、`sys.modules` 到底是什么

`sys.modules` 是一个**普通字典**（准确说是 `dict`，键是字符串，值是 module 对象）。它回答的问题是：**“这个解释器进程到目前为止，已经成功导入过哪些模块？每个模块名对应哪个模块对象？”**

- **键（key）**：模块的**完整点分名称字符串**，比如 `'myproject'`、`'myproject.core'`、`'myproject.core.parser'`。
- **值（value）**：那个模块对应的 **module 对象**。

它不是“某个文件里的内容”，而是一张**全局注册表**。导入系统的第一件事就是查这张表：名字在表里，直接返回对应的 module 对象，整个导入过程立刻结束；名字不在，才去 `sys.path` 里搜索、加载、执行，成功后**把新模块对象挂进这张表**。

**关键区分**：`sys.modules` 只挂**模块对象**，不挂模块里的函数、类、变量。模块里的成员在**模块对象自己的 `__dict__`** 里，不在 `sys.modules` 里。这是两套完全不同的存储。

---

## 二、什么东西会被挂进去

### 2.1 每一个“被成功导入的模块/包”，无论类型

只要导入成功，就会在 `sys.modules` 里出现一个条目。包括：

| 类型              | 例子键名                              | 说明                                   |
| ----------------- | ------------------------------------- | -------------------------------------- |
| 内置模块          | `'sys'`、`'builtins'`、`'_io'`        | 解释器启动时就已存在，C 写的           |
| 标准库 `.py` 模块 | `'os'`、`'json'`、`'collections.abc'` | 标准库里的模块和包                     |
| 第三方库          | `'numpy'`、`'numpy.linalg'`           | site-packages 里装的                   |
| 你自己的包        | `'myproject'`、`'myproject.core'`     | 有 `__init__.py` 的目录                |
| 你自己的模块      | `'myproject.core.parser'`             | 普通 `.py` 文件                        |
| 命名空间包        | `'some.ns.pkg'`                       | 没有 `__init__.py` 的目录，也会挂      |
| C 扩展模块        | `'math'`、`'_socket'`                 | `.so` / `.pyd` 文件                    |
| 包和子包          | 各自独立条目                          | 父包、子包、子模块是**三个不同的 key** |

**每个条目对应一个独立的 module 对象**。父包和子模块在 `sys.modules` 里是**平级的两个键**，不是嵌套结构。`sys.modules['myproject.core.parser']` 和 `sys.modules['myproject.core']` 是两个互不包含的条目，只是 `myproject.core.__dict__['parser']` 这个属性**指向**了同一个 `parser` 模块对象。

### 2.2 什么**不会**被挂进去

- **模块里的函数、类、变量**：比如 `parser.py` 里的 `parse` 函数，它在 `myproject.core.parser.__dict__['parse']` 里，**不在** `sys.modules` 里。
- **导入失败时残留的半成品**：Python 3 在导入失败时会**清理**掉已经挂进去的部分条目（早期 Python 2 行为不同，会残留）。所以一次失败的 import 通常不会在 `sys.modules` 里留下残缺条目。
- **只被“提及”但没有真正导入的模块**：比如你在注释里写了 `import foo`，或者字符串里出现了模块名，都不会挂。
- **父包未触发时，子包也不会自动挂**：只 `import myproject` 不会让 `'myproject.core'` 出现在 `sys.modules` 里。

---

## 三、用你的 `myproject` 一步一步走

目录树：

```
myproject/
├── pyproject.toml
├── src/
│   └── myproject/                ← 常规包
│       ├── __init__.py
│       ├── core/                 ← 常规子包
│       │   ├── __init__.py
│       │   ├── parser.py
│       │   └── utils.py
│       └── main.py
```

假设在项目根目录（`src/` 的父目录），用 `python -c "..."` 或交互式环境，且 `sys.path` 里包含 `src/`。

### 第 0 步：导入之前

```python
import sys

# 只保留我们自己关心的键，过滤掉解释器自带的一堆
mine = [k for k in sys.modules if k.startswith('myproject')]
print(mine)   # []
```

此时 `sys.modules` 里**没有任何**以 `myproject` 开头的键。解释器启动时已经挂了一堆内置和标准库模块（`sys`、`builtins`、`_frozen_importlib` 等），但你的包一个都没有。

### 第 1 步：`import myproject`

```python
import myproject

print([k for k in sys.modules if k.startswith('myproject')])
# ['myproject']
```

发生了什么：

1. 导入系统查 `sys.modules['myproject']`，没有。
2. 在 `sys.path` 里找到 `src/myproject/` 目录，发现 `__init__.py`，按常规包处理。
3. **执行 `myproject/__init__.py`** 的全部顶层代码。
4. 创建 module 对象 `M_myproject`，**挂进 `sys.modules['myproject'] = M_myproject`**。
5. 在当前作用域绑定名字 `myproject` → `M_myproject`。

此时 `sys.modules` 里只有 `'myproject'` 一个键。`core` 和 `parser` 都**没有**被挂——因为 `import myproject` 只要求加载 `myproject` 本身，没有要求加载子包。

### 第 2 步：`import myproject.core`

```python
import myproject.core

print([k for k in sys.modules if k.startswith('myproject')])
# ['myproject', 'myproject.core']
```

发生了什么：

1. 查 `sys.modules['myproject.core']`，没有。
2. 先确保 `myproject` 已加载（已在表里，跳过）。
3. 在 `M_myproject.__path__` 指向的 `src/myproject/` 下找 `core`，发现 `core/__init__.py`，按常规包处理。
4. **执行 `core/__init__.py`** 的顶层代码。
5. 创建 module 对象 `M_core`，**挂进 `sys.modules['myproject.core'] = M_core`**。
6. **在父包对象上设置属性**：`M_myproject.core = M_core`（这就是上一轮讲的“副作用”）。
7. 在当前作用域绑定名字 `myproject` → `M_myproject`（`import myproject.core` 绑定的是最左边的 `myproject`）。

现在 `sys.modules` 里有两个 `myproject` 开头的键：`'myproject'` 和 `'myproject.core'`。`M_myproject.__dict__` 里多了一个 `'core'` 键，指向 `M_core`。

### 第 3 步：`import myproject.core.parser`

```python
import myproject.core.parser

print([k for k in sys.modules if k.startswith('myproject')])
# ['myproject', 'myproject.core', 'myproject.core.parser']
```

发生了什么：

1. 查 `sys.modules['myproject.core.parser']`，没有。
2. 先确保 `myproject`、`myproject.core` 已加载（都在表里，跳过）。
3. 在 `M_core.__path__` 指向的 `src/myproject/core/` 下找到 `parser.py`。
4. **执行 `parser.py`** 的顶层代码（定义 `VERSION`、`parse` 等）。
5. 创建 module 对象 `M_parser`，**挂进 `sys.modules['myproject.core.parser'] = M_parser`**。
6. **在父包对象上设置属性**：`M_core.parser = M_parser`。
7. 在当前作用域绑定名字 `myproject` → `M_myproject`。

现在表里有三个条目。`M_core.__dict__` 里多了 `'parser'`，指向 `M_parser`。

### 第 4 步：验证三个条目是三个不同对象

```python
import sys

m_root   = sys.modules['myproject']
m_core   = sys.modules['myproject.core']
m_parser = sys.modules['myproject.core.parser']

print(m_root is m_core)        # False
print(m_core is m_parser)      # False
print(m_root.core is m_core)   # True
print(m_core.parser is m_parser)  # True
```

**`sys.modules` 里是三个平级的键，三个独立的 module 对象**；`m_root.core` 这个属性和 `sys.modules['myproject.core']` 指向**同一个对象**。前者是“属性引用”，后者是“注册表条目”，两条路径通向同一个东西。

### 第 5 步：`from ... import ...` 不会新增 `sys.modules` 条目

```python
from myproject.core.parser import parse

print([k for k in sys.modules if k.startswith('myproject')])
# ['myproject', 'myproject.core', 'myproject.core.parser']   —— 没变
```

`from ... import ...` 只改变**当前作用域里绑定的名字**（多了 `parse` → 函数对象），不会往 `sys.modules` 里加新条目。`parse` 是函数，不是模块，永远不会成为 `sys.modules` 的键。

### 第 6 步：重复 import 不会重复挂

```python
import myproject.core.parser   # 再导入一次

print([k for k in sys.modules if k.startswith('myproject')])
# 仍然是三个，顺序也不变
```

因为第 1 步就查 `sys.modules` 命中，直接返回缓存对象，**`parser.py` 的顶层代码不会再执行一次**。这就是“模块在进程内是单例”的含义。

---

## 四、几个容易混的细节

### 4.1 键是**完整点分名称**，不是短名

`sys.modules` 里存的是 `'myproject.core.parser'`，**不是** `'parser'`。短名 `parser` 永远不会出现在 `sys.modules` 里（除非你有一个顶层模块就叫 `parser.py` 且在 `sys.path` 根上）。这是为什么裸写 `import parser` 找不到包内模块——它去 `sys.path` 顶层找 `parser.py`，而不是去 `myproject/core/` 下找。

### 4.2 父包和子模块是**独立条目**，不是嵌套

`sys.modules` 是**扁平字典**，没有树结构。`'myproject'`、`'myproject.core'`、`'myproject.core.parser'` 是三个平级键。它们之间的“父子关系”体现在**属性引用**上（`M_myproject.core is M_core`），而不是体现在 `sys.modules` 的嵌套里。

### 4.3 挂载顺序：从左到右，父先子后

`import a.b.c` 触发挂载的顺序是 `'a'` → `'a.b'` → `'a.b.c'`。父包必须先在表里，子模块才能被加载，因为加载子模块需要从父包的 `__path__` 里找文件。如果中间某一级失败，Python 3 会把这一轮已经挂进去的部分清理掉。

### 4.4 只 `import myproject` 不会挂子包

```python
import sys
import myproject
print('myproject.core' in sys.modules)   # False
```

要访问 `myproject.core`，必须显式 `import myproject.core`（或 `from myproject import core`，或在 `myproject/__init__.py` 里写 `from . import core`）。这是 `xml.etree` 那个经典坑的根源。

### 4.5 `importlib.reload` 是**替换 value**，键不变

```python
import importlib
import myproject.core.parser

old = sys.modules['myproject.core.parser']
importlib.reload(old)
new = sys.modules['myproject.core.parser']

print(old is new)   # False —— reload 后是新的 module 对象
```

`reload` 会重新执行模块代码，**创建一个新的 module 对象**替换 `sys.modules` 里原来的 value，但**键名不变**。而且它**不会更新**你之前用 `from ... import name` 拿到别处的旧引用——那些引用仍然指向旧对象。

### 4.6 手动删除 `sys.modules` 条目会强制重新导入

```python
del sys.modules['myproject.core.parser']
import myproject.core.parser   # 会重新执行 parser.py
```

因为表里没了，导入系统会重新走完整流程。但要注意：父包对象上挂的 `parser` 属性**不会自动更新**，它仍然指向旧对象，除非重新导入时导入系统覆盖它（导入系统确实会重新设置父包属性，所以最终会指向新对象；但别处 `from ... import` 拿到的旧引用不会变）。这种操作主要用于调试和热重载，生产代码不要依赖。

### 4.7 和“父包属性”的区别

三个地方容易混：

| 存储位置              | 存什么                             | 例子                                   |
| --------------------- | ---------------------------------- | -------------------------------------- |
| `sys.modules`         | **模块名 → 模块对象** 的全局注册表 | `sys.modules['myproject.core.parser']` |
| 模块对象的 `__dict__` | 该模块**顶层定义的所有名称**       | `M_parser.__dict__['parse']`           |
| 父包对象的属性        | 子模块引用（导入系统自动挂的）     | `M_core.__dict__['parser']`            |

`sys.modules` 是**注册表**，`__dict__` 是**命名空间**，父包属性是**属性引用**。三者可能指向同一个对象，但是三条独立的访问路径，修改其中一条不自动同步另外两条（除了导入系统主动挂属性那一步）。

---

## 五、一句话总结

**`sys.modules` 是一张“模块名 → 模块对象”的全局字典。每一个被成功导入的模块或包——无论内置、标准库、第三方、你自己的 `.py` 文件、有 `__init__.py` 的目录、还是命名空间包——都会在导入成功时以“完整点分名称”为键挂进去，父包和子模块是平级的独立条目。模块里的函数、类、变量不会进 `sys.modules`，它们在模块对象自己的 `__dict__` 里。导入系统每次先查这张表，命中就直接返回缓存对象，所以同一个模块在进程内只执行一次、只有一个对象。**