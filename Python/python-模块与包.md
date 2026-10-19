这份笔记把前面几轮对话的内容完整沉淀下来，按“本质 → 导入机制 → 包的结构 → 作用域与可见性 → 运行与打包 → 陷阱对照”的顺序组织，每个概念都配具体代码和你那份 `myproject` 目录树来解释。

---

## 一、模块与包：本质区别

### 1.1 模块就是一个 `.py` 文件

在 Python 中，**一个 `.py` 文件就是一个模块**。文件与模块基本一一对应。当你 `import` 一个模块时，Python 会执行这个文件的顶层代码，然后创建一个 **module 对象**，这个对象拥有**自己的独立命名空间**——一个字典，存放该文件顶层定义的所有函数、类、变量。

```python
# myproject/core/utils.py
VERSION = "utils-2.0"

def helper_function(x):
    return x * 2
```

`import myproject.core.utils` 之后，`myproject.core.utils` 就是一个 module 对象，它的 `__dict__` 里装着 `VERSION`、`helper_function` 等键。你写 `myproject.core.utils.helper_function(3)`，本质上就是连续三次属性查找，最后拿到那个函数对象。

### 1.2 包就是一个包含 `__init__.py` 的目录

**包是一种特殊的模块**：它对应一个**目录**，而不是单个 `.py` 文件。但“包”在 Python 里有精确含义：**包含 `__init__.py` 的目录是常规包（regular package）**。

包本身也是一个 module 对象。`import myproject` 得到的 `myproject` 是一个 module，只不过它多了一个 `__path__` 属性，指向那个目录，告诉导入系统“这个模块下面还可以找子模块”。

以你那份目录树为例，完整的模块名（点分名称）和类型如下：

| 文件/目录                      | 完整模块名              | 类型             |
| ------------------------------ | ----------------------- | ---------------- |
| `src/myproject/`               | `myproject`             | 包（也是模块）   |
| `src/myproject/core/`          | `myproject.core`        | 子包（也是模块） |
| `src/myproject/core/parser.py` | `myproject.core.parser` | 模块（叶子）     |
| `src/myproject/core/utils.py`  | `myproject.core.utils`  | 模块（叶子）     |
| `src/myproject/main.py`        | `myproject.main`        | 模块（叶子）     |

**关键点**：`myproject` 这个“包”和 `myproject.main` 这个“模块”是**两个不同的 module 对象**，各有独立命名空间。`myproject/__init__.py` 里定义的东西属于 `myproject` 的命名空间；`main.py` 里定义的东西属于 `myproject.main` 的命名空间，互不干扰。

### 1.3 “同一个包内有多个模块”是什么意思

`myproject/core/` 下面有 `parser.py` 和 `utils.py`，含义是：**这个包（目录）里放了两个独立的 `.py` 文件，每个文件是一个独立模块，各自有独立的模块命名空间。**

```python
# parser.py
VERSION = "parser-1.0"      # 属于 myproject.core.parser 的命名空间

# utils.py
VERSION = "utils-2.0"       # 属于 myproject.core.utils 的命名空间
```

两个 `VERSION` 是两个不同的变量，互不覆盖。这和 C++ 里两个 `.cpp` 文件各自定义 `static int VERSION` 达到的文件级隔离类似。包（目录）提供**分组**，模块（文件）提供**隔离**。拆成多个模块的理由也和组织 C++ 代码一样：按职责分文件、让依赖关系通过 `import` 显式化、方便单独测试。

---

## 二、`__init__.py` 的作用

### 2.1 机制层面：标记“这是常规包”

`__init__.py` 的存在让导入系统把这个目录当作**常规包**处理，而不是命名空间包。这不是可有可无的装饰，它决定了包的语义（下一节详细讲不写它会怎样）。

### 2.2 执行层面：包被第一次导入时执行一次

当 `myproject` 这个包**第一次**被任何形式的 import 触及时，Python 会**先执行 `myproject/__init__.py`**，创建 module 对象放进 `sys.modules['myproject']`，然后才继续导入子模块。`core` 子包同理：第一次触及 `myproject.core` 时执行 `myproject/core/__init__.py`。

这个执行**只发生一次**。之后所有 import 都直接拿 `sys.modules` 里的缓存对象。

### 2.3 `__init__.py` 里可以放什么

**① 空文件（最常见）**。作用纯粹是“标记这是常规包”，绝大多数项目的最初版本就是这样。

**② 包的文档字符串和版本号。**

```python
# myproject/__init__.py
"""myproject: 一个示例包。"""
__version__ = "0.1.0"
```

外部可以 `import myproject; print(myproject.__version__)`。

**③ re-export：从子模块“提升”名称，定义包的公开接口。** 这是 `__init__.py` 最实用的用途，相当于 C++ 头文件里决定“对外暴露哪些符号”：

```python
# myproject/__init__.py
from .core.parser import parse
from .core.utils import helper_function

__all__ = ["parse", "helper_function"]
```

执行后，别人写 `import myproject` 时，`myproject.parse` 和 `myproject.helper_function` 直接可用，不需要写完整路径。包对外呈现一个**扁平、稳定的接口**，内部文件重组不影响使用者。

**④ `__all__` 列表。** 控制 `from myproject import *` 时导出哪些名称。没有 `__all__` 时，`import *` 会导出所有不以 `_` 开头的顶层名称；设置了 `__all__` 就只导出列表里的名称。

**⑤ 轻量的包级初始化。** 比如配置日志。但**不要**在里面做耗时操作（读大文件、连数据库、发网络请求），也不要 `print` 调试信息——`__init__.py` 会被所有导入者触发，副作用会影响整个程序。

### 2.4 不写 `__init__.py` 会怎样

**在 Python 3.3+ 中，不写 `__init__.py` 不会让你立刻报错。** `import myproject.core.parser` 依然能成功运行。但此时 `core` 的身份变了：它不再是常规包，而变成了**命名空间包（namespace package）**。

用一个比喻来区分：**常规包**像一栋**有实体门牌、有管理员的房子**——`__init__.py` 是管理员，房子第一次被访问时执行初始化，并且有明确的物理地址（`__file__` 指向 `__init__.py` 的真实路径）。**命名空间包**像一个**虚拟社区**——没有管理员，没有实体门牌，导入系统只是把 `sys.path` 上所有叫这个名字的目录合并成一个逻辑包，`__file__` 是 `None`，社区没有单一物理地址。

具体差异可以用代码直接观察到：

```python
# 有 __init__.py 时
print(myproject.core.__file__)          # .../src/myproject/core/__init__.py
print(myproject.core.__spec__.origin)   # 真实路径

# 删掉 __init__.py 后
print(myproject.core.__file__)          # None
print(myproject.core.__spec__.origin)   # 'namespace'
```

`type` 看起来一样（都是 `module`），但 `__file__` 从真实路径变成了 `None`，`__spec__.origin` 从路径变成了字符串 `'namespace'`。这会在四个地方埋下隐患：

**坑一：依赖 `__file__` 的代码会崩。** `os.path.dirname(myproject.core.__file__)` 会抛 `TypeError`，因为参数是 `None`。

**坑二：`__init__.py` 里的包级初始化、re-export、`__all__` 全部消失。** `from myproject.core import parse` 会失败，因为 `core` 的命名空间里没有 `parse`，导入系统会尝试把 `parse` 当作子模块名去找 `core/parse.py`，也找不到。

**坑三：打包工具默认不认它。** `setuptools` 的 `find_packages()` 只找有 `__init__.py` 的目录。删掉 `core/__init__.py` 后，`core` 及其下的模块**不会被打进发行包**。本地能跑，`pip install` 后 `ModuleNotFoundError`。要让它被打包，必须改用 `find_namespace_packages()` 或显式配置。

**坑四：意外的跨目录合并风险。** 命名空间包的设计目的是：如果 `sys.path` 上有多个位置都存在 `myproject/core/` 目录（且都没有 `__init__.py`），导入系统会把它们**合并成一个逻辑上的 `myproject.core`**。对普通项目，这是纯粹的意外风险——测试目录或旧版本目录可能被混进来。

**结论：项目内部的子包，永远写一个 `__init__.py`，空文件也行。** 命名空间包是为“大型组织把同一个命名空间拆到多个独立发行版里”这种特殊场景设计的，不是拿来组织普通项目内部代码的。

---

## 三、导入的底层机制

### 3.1 `import` 不是 `#include`

这是从 C++ 背景过来最需要扭转的一点。`#include` 是**文本复制**：把头文件内容逐字复制到当前编译单元。`import` 是**运行时加载模块对象**：它不复制代码文本，而是在运行时找到模块、执行一次、创建 module 对象，然后在当前作用域中绑定一个指向该对象的名称。

### 3.2 `sys.path`：模块搜索路径

当 `import` 一个模块时，Python 按顺序在 `sys.path` 这个列表里查找。`sys.path` 的初始化顺序大致是：

1. **脚本所在目录**（如果直接运行 `.py` 文件），或者**当前目录**（交互式、`-c`、`-m`）。
2. `PYTHONPATH` 环境变量指定的目录。
3. 标准库目录。
4. `site-packages` 第三方库目录。

所以 `sys.path` 决定了“Python 去哪里找模块”。如果 `import` 失败，第一件事应该是打印 `sys.path`，看目标目录在不在里面。

### 3.3 `import myproject.core.parser` 到底做了什么

导入系统按顺序执行：

**第 1 步：加载 `myproject`。** 在 `sys.path` 里找到 `myproject/` 目录，确认是包，执行 `myproject/__init__.py`，创建 module 对象 `M_myproject`，放进 `sys.modules['myproject']`。

**第 2 步：加载 `myproject.core`。** 在 `M_myproject.__path__` 指向的目录里找到 `core/`，执行 `core/__init__.py`，创建 `M_core`，放进 `sys.modules['myproject.core']`。**然后，导入系统在父包对象 `M_myproject` 上设置属性 `core`，值为 `M_core`。** 等价于悄悄执行了：

```python
M_myproject.core = M_core
```

**第 3 步：加载 `myproject.core.parser`。** 在 `M_core.__path__` 里找到 `parser.py`，执行它，创建 `M_parser`，放进 `sys.modules['myproject.core.parser']`。**然后，在父包对象 `M_core` 上设置属性 `parser`，值为 `M_parser`。** 等价于悄悄执行了：

```python
M_core.parser = M_parser
```

**第 4 步：绑定顶层名字。** 因为语句是 `import myproject.core.parser`，Python 在当前作用域里绑定的是**最左边的名字** `myproject`，指向 `M_myproject`。

### 3.4 那个“副作用”到底是什么

“父包对象上出现与子模块同名的属性”这个说法，指的就是上面第 2 步和第 3 步里导入系统**悄悄执行的赋值**。你只写了 `import myproject.core.parser`，本意是“把 parser 模块加载进来”，但导入系统**额外修改了 `myproject` 和 `myproject.core` 这两个对象的属性字典**——这就是“副作用”的含义：修改父对象的状态，并不是你这条语句字面上要求的东西。

为什么导入系统要这么做？因为 `import myproject.core.parser` 之后，你最自然的访问方式就是写 `myproject.core.parser.parse(...)`。如果不把 `core` 和 `parser` 挂到父对象上，这条属性链就会在某处 `AttributeError`。导入系统预先挂好属性，正是为了让这条链畅通。

你可以亲手验证：

```python
import sys
import myproject.core.parser

print('core' in myproject.__dict__)            # True
print('parser' in myproject.core.__dict__)     # True
print(myproject.core.parser is sys.modules['myproject.core.parser'])  # True
```

也正因为这个副作用，**只 `import myproject` 时 `myproject.core` 不一定存在**——导入系统只加载了 `myproject` 本身，没有去加载 `core` 子包，自然没挂 `core` 属性。要访问 `myproject.core`，必须先 `import myproject.core`。经典例子：

```python
import xml
xml.etree          # AttributeError

import xml.etree   # 这之后 xml.etree 才可用
```

### 3.5 三种导入写法到底绑定了什么

| 语句（在 `myproject/main.py` 里）         | 当前作用域新增的名字 | 绑定到哪个对象    |
| ----------------------------------------- | -------------------- | ----------------- |
| `import myproject.core.parser`            | `myproject`          | 最顶层包对象      |
| `from myproject.core import parser`       | `parser`             | `parser` 模块对象 |
| `from myproject.core.parser import parse` | `parse`              | `parse` 函数对象  |

**关键点**：不管用哪种写法，只要导入路径经过了 `myproject.core.parser`，导入系统都会逐级加载并挂属性。区别只在于“最后把哪个对象绑定到你当前作用域的名字上”。

第三种写法是**值绑定**：`parse` 这个名字指向的是那个函数对象。之后如果 `myproject.core.parser.parse` 被重新赋值成别的函数，你手里的 `parse` 仍指向旧函数。这和 C++ 里“拷贝了一个函数指针”类似，不是持续的“链接”。

### 3.6 `sys.modules`：模块缓存

`sys.modules` 是一个字典，缓存了所有已经被导入的模块。**在导入搜索时，第一件事就是查 `sys.modules`**：如果模块名已经在里面，直接返回缓存的对象，整个导入过程立即结束。

这保证了：**同一个模块在同一个解释器进程中只会被初始化一次**。后续所有 `import` 拿到的都是同一个 module 对象。模块在 Python 中相当于“单例”。

因此，**重复 `import` 同一个模块并不会重新执行它**。如果你修改了模块源代码，在同一个 Python 进程里再 `import` 一次，拿到的还是旧的缓存对象。要强制重新加载，需要用 `importlib.reload(module)`。但要注意：`reload` **不会重新定义已经从这个模块导入到别处的对象**——你之前 `from module import func` 拿到的 `func` 仍然指向旧函数，`reload` 不会更新它。

---

## 四、作用域与可见性

### 4.1 LEGB 规则

Python 查找名称遵循 **LEGB**：**L**ocal（当前函数）→ **E**nclosing（外层函数）→ **G**lobal（当前模块）→ **B**uilt-in（内置）。

- **局部作用域**：函数内部赋值的名称默认是局部的。
- **全局作用域**：模块顶层赋值的名称是该模块的全局名称。
- **在函数内修改全局变量**：必须显式声明 `global`。不声明的话，在函数内**赋值**会创建一个新的局部变量，遮蔽外部同名变量；只是**读取**则可以直接访问全局名称。
- **在嵌套函数中修改外层函数变量**：用 `nonlocal`。`nonlocal` 找的是**外层函数的局部变量**，不是全局变量。

和 C++ 的一个重要差异：C++ 中名称的作用域从**声明点**开始；Python 中，只要一个名称在函数体内**任何位置**被赋值，它在整个函数作用域内都被视为**局部名称**，除非用 `global` / `nonlocal` 显式声明。

### 4.2 “私有”靠约定，不靠编译器

Python **没有强制性的翻译单元可见性控制**。一个模块里定义了 `foo`，任何导入该模块的代码都可以通过 `mod.foo` 访问它。Python 的“私有”靠**命名约定**：

- **单下划线前缀 `_name`**：表示“内部使用”，是给模块作者自己用的，外部使用者“最好别碰”。这是**约定，不是强制**——解释器不会阻止 `mod._name` 的访问。
- **`__all__`**：控制 `from module import *` 时导出哪些名称，是公开接口的一部分。
- **双下划线前缀 `__name`**（不以后缀 `__` 结尾）：在**类**里触发名称改写（name mangling），变成 `_ClassName__name`，用于类内部的“私有”属性。模块级别不使用这种约定。

### 4.3 Python 里的“静态”

Python 没有 C++ 的 `static` 关键字，但 C++ 中 `static` 的几种典型用途有不同的对应方式：

**① 文件局部 / 翻译单元内部链接** → 用 `_` 前缀约定。`_helper` 表示“这是本模块内部的”，但解释器不强制。

**② 函数内部的静态局部变量**（C++ 中在函数内 `static int count = 0;`）→ Python 函数**没有**在调用之间保留状态的静态局部变量。替代方案有三种：

**函数属性**（最接近 C++ 的函数内静态变量）：

```python
def counter():
    counter.count += 1
    return counter.count
counter.count = 0

print(counter())  # 1
print(counter())  # 2
```

**闭包**（用 `nonlocal` 保持状态）：

```python
def make_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

c = make_counter()
print(c())  # 1
print(c())  # 2
```

闭包的好处是可以创建**多个独立的计数器**，而函数属性是全局共享的一份状态。

**类变量**：如果状态属于一个“东西”的多个实例共享，用类变量。

**③ 类级“静态”成员** → 类变量。注意：通过实例**赋值** `instance.attr = ...` 会在实例上创建新属性，而不是修改类变量；要修改类变量需显式写 `ClassName.attr = ...`。

---

## 五、运行方式与打包

### 5.1 `if __name__ == '__main__':`

当一个 `.py` 文件被**直接运行**时，它的 `__name__` 被设为 `'__main__'`；当它被 `import` 时，`__name__` 是模块名（如果是包内模块，还包括包的路径）。

因此，**只应在直接运行时执行的逻辑**（命令行解析、测试代码、入口调用）放在这个块下面：

```python
# myproject/main.py
def run():
    print("running")

if __name__ == '__main__':
    run()
```

这样，`import myproject.main` 时 `run()` 不会自动执行，只有 `python myproject/main.py` 或 `python -m myproject.main` 时才会执行。

### 5.2 `python -m` 与直接运行文件路径的区别

推荐用 **`python -m myproject.main`** 运行，而不是 `python src/myproject/main.py`。原因在于 `sys.path` 的初始化：

- `python src/myproject/main.py` 会把 `src/myproject/` 放到 `sys.path[0]`。此时 `main.py` 的包上下文是“孤立的”，绝对导入 `from myproject.core.parser import parse` 可能找不到 `myproject`（因为 `src/myproject/` 的父目录 `src/` 不在 `sys.path` 上），相对导入也容易出问题。
- `python -m myproject.main` 会把**当前目录**（通常是项目根）放到 `sys.path[0]`，并且正确设置包的上下文，使绝对导入和相对导入都按预期工作。

### 5.3 相对导入与绝对导入

**绝对导入**用完整的点分路径：

```python
from myproject.core.utils import helper_function
```

**相对导入**用 `.` 和 `..`：

```python
from .utils import helper_function      # "." 表示当前包（core）
from ..main import something           # ".." 表示上一级包（myproject）
```

相对导入是**基于当前模块的 `__name__`** 来确定它在包层次中的位置的。因此，**相对导入只能用在包内的模块中，不能用在直接被当作脚本运行的文件里**。如果你直接运行 `python parser.py`，此时 `parser.py` 的 `__name__` 是 `'__main__'`，不包含包信息，相对导入会抛 `ImportError: attempted relative import with no known parent package`。

**实践建议**：包内互相引用优先用**绝对导入**，清晰且不易出错；`__init__.py` 里的 re-export 用相对导入是标准做法。

### 5.4 `pyproject.toml` 与打包

`pyproject.toml` 是项目元数据文件，相当于 C++ 的 `CMakeLists.txt` / `Makefile`。对于 setuptools，核心配置是告诉它“去哪里找包”：

```toml
[build-system]
requires = ["setuptools>=64.0.0"]
build-backend = "setuptools.build_meta"

[project]
name = "myproject"
version = "0.1.0"

[tool.setuptools.packages.find]
where = ["src"]
namespaces = false          # 默认就是 false，只找有 __init__.py 的常规包
```

`where = ["src"]` 表示在 `src/` 下搜索包。`namespaces = false` 表示只识别常规包（有 `__init__.py`）。如果你的项目确实使用了命名空间包，需要设为 `true`，或者用 `find_namespace_packages()`。

**注意**：`find_packages()`（以及 `namespaces = false` 时的自动发现）**只找有 `__init__.py` 的目录**。这就是前面强调“项目内部子包永远写 `__init__.py`”的打包层面的理由——不写的话，子包可能在打包时被静默丢弃。

---

## 六、常见陷阱与对照总结

### 6.1 循环导入

模块 A 导入模块 B，模块 B 又导入模块 A，会形成循环。Python 的导入系统在加载一个模块时会先把它放进 `sys.modules`，然后执行它的代码。如果执行到一半时它又去导入另一个模块，而那个模块反过来导入它，就可能拿到一个**只执行了一半的模块对象**，导致 `ImportError: cannot import name ... from partially initialized module`。

**解决思路**（按推荐顺序）：

1. **重构**：把共享的代码抽到一个独立的第三方模块里，打破循环。
2. **把导入移到函数内部**：只在真正需要的时候才导入，避免在模块顶层形成环。
3. **改用 `import module` 而不是 `from module import name`**：`import module` 拿到的是模块对象引用，即使模块还在初始化中，后续访问属性时它可能已经完成了初始化。

### 6.2 核心对照表

| C++ 概念               | Python 对应                                             | 关键差异                       |
| ---------------------- | ------------------------------------------------------- | ------------------------------ |
| `.h` 头文件            | 无直接对应；`__init__.py` 的 re-export 可近似“对外接口” | 靠约定，不靠编译器强制         |
| `#include`             | `import`                                                | **文本复制 vs 运行时对象加载** |
| `.cpp` 编译单元        | `.py` 模块文件                                          | 无编译/链接阶段                |
| 目录 / 工程分组        | 包（含 `__init__.py` 的目录）                           | 包本身也是 module 对象         |
| `namespace`            | 模块命名空间                                            | 每个模块独立字典               |
| `static`（文件局部）   | `_` 前缀约定                                            | 解释器不强制                   |
| `static`（函数内静态） | 函数属性 / 闭包 / 类变量                                | 没有直接关键字                 |
| 链接器                 | `sys.modules` 缓存 + 导入系统                           | 运行时按点分名称逐级加载       |
| 外部链接的全局符号     | 模块级名称                                              | 模块单例，同一进程内唯一       |
| `int main()` 入口      | `if __name__ == '__main__':`                            | 文件可直接运行，也可被导入     |

### 6.3 最后几条实操要点

1. **每个包目录都放 `__init__.py`**，哪怕是空文件。这是常规包的标记，决定了 `__file__`、打包发现、包级初始化等一整套语义。
2. **`import` 绑定的是最左边的名字**。`import a.b.c` 后，当前作用域里只有 `a`；`a.b.c` 能走通，是因为导入系统在父包上挂了 `b` 和 `c` 属性。
3. **`from module import name` 是值绑定**，拿到的对象之后不会跟着原模块的重新赋值而更新。
4. **重复 `import` 不会重新执行模块**，因为有 `sys.modules` 缓存。需要热重载用 `importlib.reload`，但要清楚它不会更新已经导入到别处的引用。
5. **多文件项目优先用 `python -m package.module` 运行**，让包上下文和 `sys.path` 正确。
6. **包内互相引用优先绝对导入**；`__init__.py` 里的 re-export 用相对导入。
7. **`__init__.py` 的副作用要克制**：它会被所有导入者触发，只放版本号、公开接口声明和轻量初始化。
8. **项目内部子包不要省略 `__init__.py`**。命名空间包不会立刻报错，但会在 `__file__`、打包、包级初始化、跨目录合并四个地方埋下隐患。