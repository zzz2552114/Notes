# CSAPP / CMU 15-213：异常控制流与上下文切换

## 0. 先抓住整章的主线

平时我们想象程序都是这样执行的：

```text
PC = A
↓
A 的下一条指令
↓
下一条
↓
下一条
↓
下一条
```

也就是：

> CPU 按照程序自己的控制流，一条一条往下执行。

但真实系统里，控制流经常会被“打断”：

```text
程序正在执行
    ↓
突然发生某个事件
    ↓
CPU 不再执行原来的下一条指令
    ↓
转去执行一段特殊代码
    ↓
处理完以后
    ↓
可能回来继续原来的程序
也可能换成另一个执行流
```

这就是 **Exceptional Control Flow（ECF，异常控制流）**。

这一章真正想解决的问题其实是：

```text
为什么程序的控制流会突然改变？
        ↓
谁让它改变？
        ↓
CPU 怎么知道去哪里处理？
        ↓
处理完以后怎么回来？
        ↓
为什么一个 CPU 可以让多个程序轮流运行？
        ↓
上下文切换到底是怎么回事？
```

CSAPP 把异常看成一种系统级的控制转移机制，并进一步利用它建立进程、并发执行和上下文切换的概念。


# 1. 什么叫 Control Flow？

先别急着想“异常”。

**Control Flow 就是程序执行指令时，PC/RIP 的变化轨迹。**

比如：

```c
int main() {
    foo();
    printf("hello\n");
}
```

正常情况下：

```text
main
 ↓
foo
 ↓
return
 ↓
printf
 ↓
...
```

CPU 每执行一条指令，PC 就按照正常规则变化。

最普通的情况：

```text
PC → PC + 指令长度
```

遇到：

```text
call
jmp
ret
```

PC 会按照这些指令指定的方式跳转。

这种就是正常控制流。

---

# 2. Exceptional Control Flow 是什么？

异常控制流的核心就是：

> **控制流发生了正常程序逻辑之外的转移。**

例如：

```text
程序 A 正在运行

↓ 突然发生 timer interrupt

CPU
↓
kernel exception handler
```

或者：

```text
程序执行了一条 system call

↓
进入 kernel

↓
处理完

↓
继续执行用户程序
```

或者：

```text
程序访问一个不存在的虚拟内存页

↓
page fault

↓
kernel 处理缺页

↓
程序重新执行刚才那条指令
```

所以“异常”不要理解成：

> “程序出 bug 了。”

这是最容易产生的误解。

异常只是：

> **CPU 的正常控制流因为某个特殊事件而发生了改变。**

有些异常是完全正常的操作系统机制，比如系统调用、timer interrupt。


# 3. Exception 的四种类型

CSAPP 把异常按照两个维度组织：

```text
                   同步 / 异步
                       │
             ┌─────────┴─────────┐
             │                   │
          同步发生             异步发生
             │                   │
        Trap/Fault/Abort       Interrupt
```

四种：

```text
Interrupt   中断
Trap        陷阱
Fault       故障
Abort       终止
```

这里一定要记住：

**同步 / 异步描述的是“异常什么时候发生”；  
Trap/Fault/Abort/Interrupt 描述的是“异常属于什么类型”。**

---

# 4. Interrupt：异步中断

Interrupt 的关键词：

> **来自处理器外部，而且与当前执行的哪一条指令没有直接关系。**

例如：

```text
CPU 正在运行程序 A

            ↓

网卡：
“数据到了！”

磁盘：
“IO完成了！”

定时器：
“时间到了！”
```

这些都可能产生 interrupt。

例如 timer interrupt：

```text
A 正在运行

A
A
A
A
A

↓ timer interrupt

Kernel
```

为什么叫“异步”？

因为：

```text
A 当前正在执行哪条指令
```

并不是 interrupt 来源决定的。

定时器不会说：

> “等 CPU 执行到某一条特定指令时我再响。”

它是由外部硬件事件触发。

---

# 5. Trap：主动进入异常处理

Trap 是：

> **由当前程序主动、有意地触发的同步异常。**

最重要的例子：

```text
system call
```

比如用户程序想：

```c
read(fd, buf, 100);
```

用户程序本身没有权限直接操作所有底层资源，所以需要进入 kernel：

```text
User mode
   ↓
system call / trap
   ↓
Kernel mode
```

处理完成：

```text
Kernel
   ↓
返回用户程序
```

所以：

```text
Trap
=
程序故意让 CPU 进入异常处理路径
```

系统调用是最经典的 trap 场景。

注意：

**进入 kernel 不等于 context switch。**

例如：

```text
A user
 ↓
system call
 ↓
A kernel
 ↓
A user
```

这里一直都是 A，只是：

```text
user mode → kernel mode → user mode
```

没有换成 B。

这是后面理解 context switch 最重要的区分之一。

---

# 6. Fault：执行某条指令时出了一个可以处理的问题

Fault 是：

> **同步发生，并且通常有可能被修复，然后重新执行引发 fault 的指令。**

最经典：

```text
page fault
```

假设程序执行：

```text
指令 I：
访问虚拟地址 VA
```

CPU 发现：

```text
这个页面当前没有映射到物理内存
```

于是：

```text
I
↓
page fault
↓
进入 kernel
↓
kernel 处理
↓
问题解决
↓
重新执行 I
```

注意这里非常关键：

> **Fault 返回到的是产生 fault 的那条指令。**

因为那条指令之前没有成功完成。

例如：

```text
I1
I2
I3  ← page fault
I4
```

修复以后：

```text
I1
I2
I3  ← 重新执行
I4
```

而不是直接：

```text
I4
```

这和 Trap 的返回位置形成一个很重要的区别。

---

# 7. Abort：严重到无法恢复

Abort 也是同步异常，但通常表示：

> **发生了无法恢复的严重错误。**

典型理解：

```text
硬件检测到严重问题
↓
无法可靠地继续执行
↓
停止当前程序/系统处理
```

和 fault 最重要的区别：

```text
Fault：
有可能修复
→ 处理以后可能重新执行原指令

Abort：
严重且不可恢复
→ 通常不会回到原来的程序继续执行
```

所以可以这样记：

```text
Trap：
“我故意来找 kernel。”

Fault：
“这条指令出了问题，修好以后再试一次。”

Abort：
“情况太严重了，没法继续。”
```

---

# 8. 四种异常一张表记住

| 类型 | 同步？ | 典型来源 | 处理后通常去哪 |
|---|---|---|---|
| Interrupt | 异步 | timer、磁盘、网卡 | 继续原程序的后续执行 |
| Trap | 同步 | system call | 通常返回下一条指令 |
| Fault | 同步 | page fault | 通常重新执行当前指令 |
| Abort | 同步 | 严重硬件/机器错误 | 通常不再返回原程序 |

“通常”很重要，因为这里是从控制流机制角度总结，不要把它理解成每一种实现绝对只能有一种行为。CSAPP 对四类异常的划分以及 handler 返回位置就是这一节的核心。


# 9. CPU 发生 Exception 后，到底做了什么？

这是异常机制真正的核心。

假设：

```text
用户程序正在运行
```

突然：

```text
Exception
```

CPU 不会简单地说：

```text
“好，我去执行 handler。”
```

它需要知道至少：

```text
发生的是什么异常？
以后从哪里回来？
当前执行状态是什么？
应该使用什么权限？
```

所以硬件会进行一系列状态保存和控制转移。

概念上可以理解为：

```text
当前执行状态
    ↓
保存必要的返回信息
    ↓
确定 exception number
    ↓
找到对应 handler
    ↓
进入 handler
```

---

# 10. Exception Number 与 Exception Table

CPU 怎么知道：

> “发生的是哪一种异常，该跳到哪段代码？”

靠的是异常编号以及异常表。

可以把它想象成：

```text
exception number
      ↓
Exception Table
      ↓
handler address
```

例如概念上：

```text
0 → handler_0
1 → handler_1
2 → handler_2
...
14 → page_fault_handler
...
```

所以：

```text
发生异常
  ↓
得到 exception number
  ↓
查表
  ↓
找到 handler
  ↓
跳过去执行
```

CSAPP 对这个表称为 **exception table**，每一种异常对应一个 handler 的入口地址。

注意：

> **exception table 不是普通 C 数组给用户程序查的东西。**

它属于系统/硬件约定下的异常处理机制，用户程序不能随便修改。


# 11. Handler 是什么？

Handler 就是：

> **处理某种异常的代码。**

例如：

```text
page fault
    ↓
page fault handler
```

```text
system call
    ↓
system call handler
```

```text
timer interrupt
    ↓
timer interrupt handler
```

handler 本身是 kernel 代码的一部分。

因此：

```text
用户程序
   ↓
异常
   ↓
kernel handler
```

这也是为什么用户程序不能随便“处理”这些硬件级异常。

---

# 12. User Mode 和 Kernel Mode

异常机制和权限机制是绑在一起的。

CPU 通常区分：

```text
User mode
Kernel mode
```

用户程序：

```text
User mode
```

操作系统核心代码：

```text
Kernel mode
```

User mode 权限更低，例如不能随便：

```text
修改页表
操作所有硬件
执行特权指令
修改关键 CPU 控制状态
```

所以：

```text
User program
     ↓
System call / Exception / Interrupt
     ↓
Kernel mode
```

异常机制提供了一个**受控的入口**。

程序不是：

```text
“我想执行 kernel 代码，所以直接跳进去。”
```

而是：

```text
程序触发合法入口
      ↓
CPU 按硬件规定进入 kernel
      ↓
执行指定 handler
```

这正是保护机制的重要组成部分。

---

# 13. 异常处理和 Context Switch 的关系

这是本节最容易混的地方。

一定区分：

```text
Exception
```

和：

```text
Context Switch
```

它们不是一回事。

例如：

```text
A
↓
timer interrupt
↓
kernel
↓
A
```

发生了 exception，但：

```text
没有 context switch
```

因为最后还是 A。

也可以：

```text
A
↓
timer interrupt
↓
kernel
↓
scheduler
↓
B
```

这里才发生：

```text
context switch
```

所以：

> **异常是控制流进入 kernel 的机制；context switch 是把 CPU 的执行环境从一个执行流切到另一个执行流的过程。**

二者经常连起来，但概念上必须分开。

---

# 14. Process 到底是什么？

进入进程以后，CSAPP 开始换一个角度。

一个 process 可以先粗略理解成：

> **一个正在运行的程序实例，以及操作系统为了让它运行而维护的执行环境。**

比如你打开：

```text
./a.out
```

操作系统不是简单地：

```text
把 a.out 文件放进内存
然后 CPU 跑
```

而是建立一个 process：

```text
Process A
├── 程序代码
├── 数据
├── 堆
├── 用户栈
├── 寄存器状态
├── 地址空间
└── kernel 为它维护的各种状态
```

进程的重要抽象是：

> **它让程序感觉自己独占了一台机器。**

---

# 15. Process 最重要的两个“幻觉”

### 幻觉一：我独占 CPU

实际上：

```text
A
B
C
A
B
C
...
```

CPU 在不断切换。

但 A 感觉：

```text
“CPU 就是我的。”
```

### 幻觉二：我拥有自己的内存

A 有：

```text
A 的虚拟地址空间
```

B 有：

```text
B 的虚拟地址空间
```

因此两个程序可以都有：

```text
0x400000
```

却不意味着它们访问的是同一份物理内存。

所以 process 提供的是一个很重要的抽象：

> **独立的控制流 + 独立的地址空间。**

CSAPP 也把“进程让程序产生独占处理器的错觉”作为 process 抽象的核心，并用 logical control flow 描述这种现象。


# 16. Logical Control Flow

这也是 CSAPP 一个很漂亮的概念。

物理 CPU 只有一条控制流：

```text
CPU：
A1 A2 A3 B1 B2 B3 C1 C2 A4 A5 ...
```

但从每个进程自己的角度看：

```text
A：
A1 A2 A3      A4 A5

B：
       B1 B2 B3

C：
                C1 C2
```

所以：

> **每个进程都有自己的 logical control flow。**

进程的执行时间可能被切成很多片：

```text
A：████       ████        ███
B：    ██████      ████
C：          ███
```

每一段连续运行的时间可以理解为一个 **time slice**。

多个 logical flows 在时间上重叠，就是 concurrency；单核机器也可以有 concurrent flows，因为这里说的是“执行区间发生重叠”，而不是“同一瞬间真的同时执行”。


# 17. Concurrency 和 Parallelism 不要混

### Concurrency

多个执行流：

```text
在时间上发生重叠
```

例如单核：

```text
A A A B B B A A B
```

A 和 B 是 concurrent。

### Parallelism

多个执行流：

```text
同一时刻真正运行在不同 CPU core 上
```

例如：

```text
Core 0 → A
Core 1 → B
```

所以：

```text
单核：
Concurrency ✓
Parallelism ✗

多核：
Concurrency ✓
Parallelism ✓
```

Context switch 在单核 CPU 上尤其直观，但多核系统中的每个 CPU core 仍然可能发生调度和上下文切换。


# 18. 为什么需要 Context Switch？

假设：

```text
A 正在运行
```

这时：

```text
A 需要等待磁盘
```

如果不切换：

```text
A：
“我等磁盘……”

CPU：
“那我也等。”

```

CPU 就浪费了。

所以：

```text
A
↓
blocked
↓
scheduler
↓
B
```

或者即使 A 没有阻塞：

```text
A 已经运行了一段时间
↓
timer interrupt
↓
scheduler
↓
B
```

因此 context switch 是操作系统实现：

```text
multitasking
```

的重要基础。CSAPP 对这一过程的总图就是：进程轮流取得 CPU，某个进程被暂时 preempt 后，其他进程执行，再恢复原来的 logical flow。


# 19. Context 到底是什么？

一句话：

> **Context 就是让一个执行流以后能够继续运行所需要保存的执行状态。**

最重要的内容包括：

```text
RIP / PC
RSP
通用寄存器
状态寄存器
其他相关 CPU 状态
栈状态
地址空间相关状态
以及 kernel 为这个进程维护的相关信息
```

不要把 context 理解成：

> “一个 C struct，把 CPU 的所有东西一股脑复制进去。”

真正的实现取决于：

```text
CPU architecture
OS
当前发生切换的位置
具体调度实现
```

CSAPP 讲的是这个概念层面的 context；实际操作系统可能把这些状态分散保存在不同的 kernel data structures 中。CSAPP 也明确提到 context 包含重新启动被抢占进程所需要的状态，而不仅仅是几个寄存器。


# 20. Context Switch 的本质

现在可以定义得非常准确：

```text
旧执行流
   ↓
保存它的 context
   ↓
选择新的执行流
   ↓
恢复新的 context
   ↓
继续运行新的执行流
```

比如：

```text
A
↓
Save A
↓
Load B
↓
B
```

所以：

> **Context switch 不是“复制整个进程”。**

尤其不是：

```text
把 A 的全部内存复制走
↓
把 B 的全部内存复制回来
```

进程的代码、数据、堆等地址空间本来就在那里。

切换关注的是：

```text
“CPU 怎么从 A 的执行现场变成 B 的执行现场？”
```

而不是搬运整个进程。

---

# 21. 最典型的 Context Switch：A 阻塞，B 接着跑

这是 CSAPP 里最应该真正搞懂的一条时序。

假设：

```c
read(fd, buf, n);
```

A 执行：

```text
A user mode
   ↓
read()
   ↓
system call
   ↓
进入 kernel
```

现在 kernel 发现：

```text
需要等待 IO
```

A 暂时不能继续。

于是：

```text
A → blocked
```

接下来 scheduler 找：

```text
哪个执行流现在可以运行？
```

发现：

```text
B → ready
```

于是：

```text
保存 A context
      ↓
恢复 B context
      ↓
B 开始执行
```

完整画出来：

```text
              Process A
                  │
                  │ read()
                  ↓
             Kernel mode
                  │
                  │ 等待 I/O
                  ↓
              A blocked
                  │
                  ↓
              Scheduler
                  │
          ┌───────┴───────┐
          │               │
       Save A          Load B
          │               │
          └───────┬───────┘
                  ↓
              Process B
```

这就是典型的 context switch。

---

# 22. 保存 A 到底意味着什么？

假设 A 暂停时：

```text
RIP = 某个位置
RSP = A 的栈顶
RAX = 某个值
RBX = 某个值
...
```

这些状态必须以某种方式保存。

否则以后：

```text
A 又回来
```

CPU 就不知道：

```text
“我原来执行到哪里？”
“我的栈在哪里？”
“寄存器原来是什么？”
```

尤其最关键的是：

```text
RIP
```

因为 RIP 决定：

> **以后从哪条指令继续执行。**

以及：

```text
RSP
```

因为它决定：

> **当前栈在哪里。**

所以一个非常好的直觉是：

```text
Context
=
“暂停录像”
```

恢复 context：

```text
“把录像里 CPU 当时的执行现场重新摆回来”
```

当然实际实现比录像复杂得多，但这个直觉是对的。

---

# 23. User Stack 和 Kernel Stack

这块特别容易在第一次学的时候混掉。

一个进程涉及至少两种重要的栈：

```text
User Stack
Kernel Stack
```

### User Stack

用户程序自己的：

```text
main()
foo()
bar()
```

函数调用、局部变量等。

### Kernel Stack

进程进入 kernel 后，kernel 执行过程中使用的栈。

可以想象成：

```text
Process A

User space
────────────
User stack
────────────

Kernel space
────────────
Kernel stack
────────────
```

所以：

```text
A user mode
```

进入：

```text
A kernel mode
```

并不是“同一套用户栈继续随便干”。

kernel 有自己的受保护执行环境。

---

# 24. Context Switch 和 Mode Switch

这是必须分清的两个概念。

### Mode Switch

```text
User mode
↓
Kernel mode
```

或者：

```text
Kernel mode
↓
User mode
```

发生的是：

> 权限级别改变。

### Context Switch

```text
A
↓
B
```

发生的是：

> 执行环境/执行流改变。

所以：

```text
A user
 ↓
A kernel
```

是：

```text
Mode switch
```

但不是必然的：

```text
Context switch
```

而：

```text
A kernel
 ↓
B kernel
```

则是：

```text
Context switch
```

同时整个过程可能仍然处于 kernel mode。

这点非常重要。

---

# 25. Scheduler 到底干什么？

Scheduler 不负责“保存所有东西”。

它最核心的任务是：

> **决定接下来哪个可运行的执行流使用 CPU。**

概念上：

```text
当前：
A

可运行：
B
C
D
```

scheduler：

```text
“下一位是谁？”
```

选出：

```text
B
```

然后才进入实际的：

```text
context switch
```

所以：

```text
Scheduler
=
做选择

Context switch
=
完成执行环境的切换
```

两者相关，但不是同一个概念。

---

# 26. Timer Interrupt 为什么特别重要？

如果只有：

```text
进程主动调用系统调用
```

操作系统就很难阻止某个进程一直霸占 CPU。

比如：

```c
while (1) {
    // 什么都不干
}
```

它可以一直运行。

于是硬件 timer 周期性产生：

```text
Timer Interrupt
```

例如：

```text
A A A A A A A
      ↓
Timer interrupt
      ↓
Kernel
      ↓
Scheduler
      ↓
B
```

于是即使 A：

> “我根本不想让出 CPU。”

kernel 也可以获得执行机会，然后决定：

```text
A 暂停
B 上来
```

这就是抢占式 multitasking 的关键基础。

---

# 27. Context Switch 不一定发生在 System Call 里

一个非常容易出现的错误推理：

```text
system call
↓
context switch
```

不是必然。

正确的是：

```text
system call
↓
进入 kernel
↓
kernel 可以继续让 A 跑
```

也可以：

```text
system call
↓
A 阻塞
↓
scheduler
↓
B
```

类似地：

```text
interrupt
```

也不必然导致：

```text
A → B
```

可能：

```text
A
↓
interrupt
↓
kernel
↓
A
```

因此一定要把逻辑拆成：

```text
Exception
    ↓
进入 Kernel
    ↓
Kernel 是否需要调度？
    ↓
否 → 回原来的执行流
是 → Context Switch
```

---

# 28. Context Switch 和 “暂停一个程序”到底是什么关系？

从程序的角度：

```text
A：
“我刚刚还在执行。”
```

下一瞬间：

```text
A：
“怎么突然不动了？”
```

其实：

```text
A 的 context 已保存
A 被标记为暂时不运行
CPU 去运行 B
```

等 A 再次获得 CPU：

```text
恢复 A context
↓
A 从之前的位置继续
```

所以对 A 来说：

```text
执行
↓
时间突然过去了一段
↓
继续执行
```

它自己通常感觉不到中间 CPU 去执行过多少别的东西。

CSAPP 所说的“process 看起来拥有独占处理器”的幻觉，本质上就是通过这种逻辑控制流 + 调度实现的。


# 29. 一个完整的故事：A → B → A

把前面的东西全部串起来。

### 第一步

```text
A user mode
```

### 第二步

A 调用：

```text
read()
```

发生：

```text
Trap / system call
```

于是：

```text
A user
↓
A kernel
```

### 第三步

kernel 发现 IO 还没完成：

```text
A → blocked
```

### 第四步

scheduler 选择 B：

```text
Save A
Load B
```

### 第五步

```text
B kernel/user execution
```

B 运行。

### 第六步

设备完成 IO：

```text
device
↓
interrupt
↓
kernel
```

此时 kernel 得知：

```text
A 等待的数据好了
```

于是 A 可以重新变成：

```text
ready
```

之后某次调度：

```text
B
↓
scheduler
↓
Save B
↓
Load A
↓
A
```

### 第七步

A 恢复之后：

```text
A 的寄存器恢复
A 的栈恢复
A 的执行位置恢复
```

于是继续原来的控制流。

这就是：

```text
A
 ↓
system call
 ↓
kernel
 ↓
A blocked
 ↓
context switch
 ↓
B
 ↓
interrupt
 ↓
kernel
 ↓
A ready
 ↓
context switch
 ↓
A
```

这条时序基本就是这一节最值得掌握的“主线”。

---

# 30. 最容易错的几个地方

## ① Exception ≠ Error

异常不等于 bug。

```text
System call
Timer interrupt
```

都是正常系统机制。

---

## ② Enter Kernel ≠ Context Switch

```text
A user
↓
A kernel
```

可以完全没有切换进程。

---

## ③ Context Switch ≠ 把整个进程复制一遍

主要是切换：

```text
执行状态
寄存器
栈
地址空间相关状态
kernel 管理状态
```

而不是复制全部代码和数据。

---

## ④ Scheduler ≠ Context Switch

```text
Scheduler：
“让谁跑？”

Context switch：
“把 CPU 真正切到他身上。”
```

---

## ⑤ Interrupt ≠ Context Switch

```text
Interrupt
↓
Kernel
↓
可能继续 A
```

也可能：

```text
Interrupt
↓
Kernel
↓
Scheduler
↓
B
```

---

## ⑥ 单核也存在 Concurrency

```text
A A B B A B
```

虽然同一瞬间 CPU 只有一个执行流，但 A、B 的生命周期发生了时间上的重叠，所以它们是 concurrent。

---

## ⑦ Context 不只是 RIP

至少应该想到：

```text
RIP
RSP
registers
flags / machine state
stack
address-space-related state
kernel-maintained state
```

不要把：

```text
context = PC
```

理解成这么简单。

---

# 31. 最后整理成一张“脑内地图”

```text
                    Exceptional Control Flow
                              │
               ┌──────────────┴──────────────┐
               │                             │
          正常控制流被打断                  为什么？
               │
               ↓
           Exception
               │
       ┌───────┼────────┬────────┐
       ↓       ↓        ↓        ↓
 Interrupt   Trap     Fault    Abort
       │       │        │        │
       │       │        │        └─ 通常无法恢复
       │       │        └─ 修复后重试当前指令
       │       └─ system call
       └─ timer/device
               │
               ↓
           Kernel Handler
               │
               ↓
        是否需要换执行流？
          /           \
        否              是
        │                │
        ↓                ↓
      回去          Scheduler
                         │
                         ↓
                  Context Switch
                    /          \
               Save A         Load B
                    \          /
                         ↓
                         B
```

而 **Process** 可以放在这张图旁边理解：

```text
Process
=
程序的执行实例
+
独立的地址空间
+
自己的 logical control flow
+
kernel 为其维护的执行状态

Context Switch
=
把 CPU 从 Process A 的执行现场
切换到 Process B 的执行现场
```

---

# 32. 这一节真正应该记住的 10 句话

1. **ECF 的本质是：控制流偏离正常顺序。**

2. **Exception 是 CPU 从当前控制流转入特殊处理代码的一套机制。**

3. **Interrupt 是异步的；Trap、Fault、Abort 是同步的。**

4. **Trap 最典型的用途是 system call。**

5. **Fault 往往可以修复，修复后通常重新执行产生 fault 的指令。**

6. **进入 kernel 不等于发生 context switch。**

7. **Process 提供了“我拥有自己的 CPU 和地址空间”的抽象。**

8. **多个 process 可以通过时间片轮流执行，在单核上也形成 concurrency。**

9. **Scheduler 决定“下一个是谁”，context switch 完成“从这个执行现场换到那个执行现场”。**

10. **Context switch 的核心就是 Save old context + Restore new context，然后让新的执行流继续。**

这也是为什么 CMU 的课程会把 **exceptions、processes、concurrent execution、context switch、user/kernel mode** 放在同一组内容里：它们其实是一条连续的机制链，而不是五个互不相关的知识点。

注：这里主要按 CSAPP/CMU 15-213 的概念层次整理；真正落到 x86-64/Linux 时，异常入口保存哪些状态、内核栈具体长什么样、`syscall`/`iretq`/页表切换等会有更具体的硬件和实现细节，不能机械地把这份概念模型当成 Linux 源码实现。