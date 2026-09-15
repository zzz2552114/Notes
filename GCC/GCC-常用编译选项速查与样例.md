# GCC/G++ 常用编译选项速查与样例

以下按功能分类整理最常用的 GCC/G++ 选项，每项附简要说明与典型用法。示例中 `gcc` 与 `g++` 在选项层面基本通用，涉及 C++ 标准库时用 `g++`。

## 一、控制编译流程：-E / -S / -c / -o

GCC 的编译过程分为四个阶段：**预处理 → 编译 → 汇编 → 链接**。`-E`、`-S`、-c 分别让编译在对应阶段停下。

- **`-o` 只负责“指定输出文件名”**，不决定编译到哪一步。
- **`-S`、`-c`、`-E` 才决定“在哪个阶段停下来”**。
- 如果输入是 `.c`，并且**没有** `-E`、`-S`、`-c`，那么默认就是：**预处理 → 编译 → 汇编 → 链接**，最终生成可执行文件。



**`-E`** 只做预处理（宏展开、头文件包含、去注释），输出 `.i` 文件，不进行编译和汇编。

```bash
gcc -E hello.c -o hello.i          # 生成预处理后的 .i 文件
```



**`-S`** 将源代码编译为汇编代码（`.s` 文件），不进行汇编和链接。

```bash
gcc -S hello.c -o hello.s
```



**`-c`** 编译并汇编，生成二进制目标文件（`.o`），不链接。这是分步编译中最常用的选项。

```bash
gcc -c main.c -o main.o
g++ -c util.cpp -o util.o
```



**`-o`** 指定输出文件的名称。不指定时，可执行文件默认名为 `a.out`。

```bash
gcc main.c -o myprogram            # 直接编译链接并指定可执行文件名
```



**分步编译再链接的典型流程**：

```bash
gcc -c main.c -o main.o
gcc -c util.c -o util.o
gcc main.o util.o -o app           # 链接阶段
```



## 二、警告选项：-Wall / -Wextra / -Werror / -w

**`-Wall`** 开启大多数常见警告。注意它并非“所有”警告，建议配合 `-Wextra` 使用。

```bash
g++ -Wall main.cpp -o main
```



**`-Wextra`** 补充 `-Wall` 未涵盖的额外警告。

```bash
g++ -Wall -Wextra main.cpp -o main
```



**`-Werror`** 将所有警告视为错误，强制在编译阶段暴露问题，适合 CI 环境或对代码质量有要求的项目。

```bash
gcc -Wall -Wextra -Werror main.c -o main
```



**`-w`** 关闭所有警告。不推荐日常使用，仅在处理第三方遗留代码等特殊场景下临时使用。

```bash
gcc -w legacy_code.c -o legacy
```



## 三、优化选项：-O0 / -O1 / -O2 / -O3 / -Os

**`-O0`** 不优化，保留原始代码结构，调试时推荐，编译速度最快。

**`-O1`** 基本优化，在编译速度与运行效率之间取得平衡。

**`-O2`** 深度优化，包含大多数安全的优化算法，是发布版本最常用的级别。

**`-O3`** 最高级别优化，可能显著增加代码体积，且某些激进优化可能改变程序行为，需充分测试。

**`-Os`** 优化代码大小，适用于存储空间受限的嵌入式环境。

```bash
gcc -O2 main.c -o main             # 发布版本常用
gcc -O0 -g main.c -o main_debug    # 调试版本
```



## 四、调试：-g

**`-g`** 在编译时生成调试信息，供 GDB 等调试器使用。若不指定，默认生成的是不含调试信息的 release 版本。

bash

```
gcc -g main.c -o main_debug
gdb ./main_debug
```



## 五、链接：-static / -shared / -l / -L

**`-static`** 强制静态链接，将库代码直接嵌入可执行文件，生成的文件不依赖外部动态库，但体积较大。

```bash
gcc main.c -static -o main_static
```



**`-shared`** 生成动态链接库（`.so` 文件），通常配合 `-fPIC` 使用。

```bash
gcc -fPIC -shared mylib.c -o libmylib.so
```



**`-l`** 链接指定的库，库名去掉 `lib` 前缀和后缀。例如链接数学库 `libm` 用 `-lm`。

bash

```
gcc main.c -lm -o main             # 链接数学库
```



**`-L`** 指定库文件的搜索路径，用于非标准路径下的库。

```bash
gcc main.c -L./libs -lmylib -o main
```



## 六、预处理相关：-I / -D

**`-I`** 指定头文件搜索路径，用于自定义头文件不在系统默认路径中的情况。

```bash
gcc -I./include main.c -o main
```



**`-D`** 在编译时定义宏，等价于在代码中 `#define`，常用于条件编译或传入编译期常量。

```bash
gcc -DDEBUG -DVERSION=\"1.0\" main.c -o main
```



## 七、C/C++ 标准：-std=

**`-std=`** 指定代码遵循的语言标准，对保证可移植性和使用新语言特性至关重要。

```bash
g++ -std=c++11 main.cpp -o main
g++ -std=c++17 -Wall -O2 main.cpp -o app
gcc -std=c99 main.c -o main
```



## 综合示例

一个兼顾警告、优化、调试信息和标准的完整编译命令：

```bash
g++ -Wall -Wextra -O2 -g -std=c++17 -I./include -L./libs -lmylib main.cpp util.cpp -o app
```



该命令的含义：开启常规警告和额外警告，使用 `-O2` 优化，生成调试信息，遵循 C++17 标准，头文件路径为 `./include`，库路径为 `./libs`，链接 `mylib` 库，最终输出可执行文件 `app`。日常开发中可在此基础上按需增删选项。