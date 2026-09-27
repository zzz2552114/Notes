# `<signal.h>` 预定义宏常量全解

`<signal.h>` 里的宏可以分成六大类：**信号编号宏、特殊处理函数指针宏、屏蔽字操作码宏、sigaction 行为标志宏、实时信号范围宏、辅助常量宏**。下面逐个讲透。

------

## 一、信号编号宏（Signal Number Macros）

这些是最常见的，每个宏展开为一个正整数，代表一个具体信号。POSIX 标准定义了约 20 多个，Linux 上实际有 30+ 个（编号 131 为标准信号，3464 为实时信号）。

### 1.1 进程终止类

| 宏        | 编号(Linux) | 触发方式               | 默认动作         | 说明                                                         |
| --------- | ----------- | ---------------------- | ---------------- | ------------------------------------------------------------ |
| `SIGTERM` | 15          | `kill` 命令默认发送    | 终止             | **优雅终止**请求，进程可捕获、可忽略、可清理后退出           |
| `SIGINT`  | 2           | 终端按 Ctrl+C          | 终止             | 中断信号，面向前台进程组                                     |
| `SIGKILL` | 9           | `kill -9`              | 终止（不可捕获） | **强制杀死**，不能被捕获、阻塞或忽略，内核直接收尸           |
| `SIGHUP`  | 1           | 终端断开 / `kill -HUP` | 终止             | 挂起信号，常用于让守护进程**重新加载配置**（如 nginx -s reload） |
| `SIGQUIT` | 3           | 终端按 Ctrl+\          | 终止 + core dump | 退出信号，产生核心转储供调试                                 |

**为什么这么设计？** `SIGTERM` 和 `SIGKILL` 的区分是关键：`SIGTERM` 给进程"留遗言"的机会（写日志、释放资源、保存状态），`SIGKILL` 是最后的杀手锏——当进程已经卡死、连信号处理函数都进不去时，内核直接强制终止。这就是为什么运维上总是先 `kill <pid>`（发 SIGTERM），等几秒没反应才 `kill -9 <pid>`。

**例子：**

```
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

void handle_term(int sig) {
    printf("收到 SIGTERM，正在优雅退出...\n");
    // 清理资源、保存状态
    _exit(0);
}

int main() {
    signal(SIGTERM, handle_term);  // 捕获 SIGTERM
    // signal(SIGKILL, handle_term); // ❌ 编译可能过，但运行时无效，SIGKILL 不能捕获
    while (1) pause();
    return 0;
}
```

### 1.2 内存错误类（产生 core dump）

| 宏        | 编号 | 触发                                              | 默认动作  |
| --------- | ---- | ------------------------------------------------- | --------- |
| `SIGSEGV` | 11   | 段错误（非法内存访问/解引用空指针/栈溢出）        | 终止+core |
| `SIGBUS`  | 7    | 总线错误（未对齐访问/不存在的物理地址）           | 终止+core |
| `SIGILL`  | 4    | 执行了非法指令（损坏的二进制/栈溢出跳到垃圾数据） | 终止+core |
| `SIGFPE`  | 8    | 浮点异常（除零/整数溢出）                         | 终止+core |
| `SIGABRT` | 6    | 调用 `abort()`                                    | 终止+core |

**设计意图：** 这些信号默认都产生 core dump，因为它们代表程序内部的严重 bug，开发者需要事后用 gdb 分析核心转储文件来定位崩溃现场。

**注意：** `SIGSEGV` 虽然可以被捕获，但**在信号处理函数里做复杂操作极其危险**——因为触发 SIGSEGV 时栈可能已经损坏，处理函数本身也可能再次触发 SIGSEGV 导致无限递归。常见做法是捕获后只做最小记录（如写一个标记文件）然后立即 `_exit`。

### 1.3 定时器与子进程类

| 宏        | 编号 | 触发                                    | 默认动作         |
| --------- | ---- | --------------------------------------- | ---------------- |
| `SIGALRM` | 14   | `alarm()` / `setitimer()` 到期          | 终止             |
| `SIGCHLD` | 17   | 子进程终止/停止/继续                    | 忽略             |
| `SIGCONT` | 18   | 进程被 `fg`/`kill -CONT` 恢复           | 继续（若已停止） |
| `SIGSTOP` | 19   | `kill -STOP` / Ctrl+Z（实际发 SIGTSTP） | 停止（不可捕获） |
| `SIGTSTP` | 20   | 终端按 Ctrl+Z                           | 停止             |
| `SIGTTIN` | 21   | 后台进程试图读终端                      | 停止             |
| `SIGTTOU` | 22   | 后台进程试图写终端                      | 停止             |

**SIGCHLD 的设计非常精妙：** 子进程退出时内核自动给父进程发 SIGCHLD，父进程在处理函数里调用 `waitpid()` 回收子进程，避免产生僵尸进程。如果父进程把 SIGCHLD 设为 `SIG_IGN`（显式忽略），Linux 内核会**自动回收**子进程，连 wait 都不用调——这是一个特殊语义，不是所有 Unix 都支持。

**例子：用 SIGCHLD 回收子进程**

```
#include <signal.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>

void reap_child(int sig) {
    int saved_errno = errno;  // 保存 errno，信号处理函数不应破坏它
    while (waitpid(-1, NULL, WNOHANG) > 0)  // 循环回收，因为多个子进程可能同时退出
        ;
    errno = saved_errno;
}

int main() {
    struct sigaction sa = {0};
    sa.sa_handler = reap_child;
    sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;  // SA_NOCLDSTOP: 子进程停止时不发信号
    sigemptyset(&sa.sa_mask);
    sigaction(SIGCHLD, &sa, NULL);

    if (fork() == 0) { _exit(0); }  // 子进程立即退出
    // 父进程继续，SIGCHLD 会触发回收
    pause();
    return 0;
}
```

### 1.4 其他常用信号

| 宏                | 编号 | 触发                              | 默认动作  | 用途                                     |
| ----------------- | ---- | --------------------------------- | --------- | ---------------------------------------- |
| `SIGPIPE`         | 13   | 向已关闭读端的管道写数据          | 终止      | 网络编程中常见：对端已关闭连接还在 write |
| `SIGUSR1`         | 10   | 用户自定义                        | 终止      | 进程间自定义通信                         |
| `SIGUSR2`         | 12   | 用户自定义                        | 终止      | 同上，第二个自定义信号                   |
| `SIGPOLL`/`SIGIO` | 29   | 异步 I/O 事件就绪                 | 终止      | 驱动层通知                               |
| `SIGSYS`          | 31   | 非法系统调用                      | 终止+core | seccomp 过滤触发                         |
| `SIGURG`          | 23   | socket 收到带外数据               | 忽略      | TCP 紧急数据                             |
| `SIGXCPU`         | 24   | 超过 CPU 时间限制                 | 终止+core | setrlimit 触发                           |
| `SIGXFSZ`         | 25   | 超过文件大小限制                  | 终止+core | setrlimit 触发                           |
| `SIGVTALRM`       | 26   | 虚拟定时器到期（仅用户态时间）    | 终止      | setitimer(ITIMER_VIRTUAL)                |
| `SIGPROF`         | 27   | 性能分析定时器到期（用户+内核态） | 终止      | setitimer(ITIMER_PROF)                   |
| `SIGWINCH`        | 28   | 终端窗口大小改变                  | 忽略      | 交互式程序重新排版                       |

**SIGPIPE 的经典坑：** 写网络服务时，如果对端已经关闭连接，你继续 `write()` 会触发 SIGPIPE，默认动作是直接终止进程——很多服务莫名其妙死掉就是这个原因。通常在程序启动时加上 `signal(SIGPIPE, SIG_IGN);`，让 write 直接返回 `EPIPE` 错误码，由业务逻辑处理。

------

## 二、特殊信号处理函数指针宏

这些宏的类型是 `void (*)(int)`（函数指针），作为 `signal()` 或 `sigaction.sa_handler` 的特殊取值。

| 宏         | 值                  | 含义                                                         |
| ---------- | ------------------- | ------------------------------------------------------------ |
| `SIG_DFL`  | `(void (*)(int))0`  | 恢复**默认动作**（终止/忽略/停止/core，取决于具体信号）      |
| `SIG_IGN`  | `(void (*)(int))1`  | **忽略**该信号（来了直接丢弃）                               |
| `SIG_ERR`  | `(void (*)(int))-1` | `signal()` 调用**失败**的返回值                              |
| `SIG_HOLD` | `(void (*)(int))2`  | System V 扩展，**阻塞**该信号（POSIX 不推荐，用 sigprocmask 代替） |

**为什么用函数指针而不是普通整数？** 因为 `signal()` 的第二个参数和返回值类型都是 `void (*)(int)`，用特殊地址值（0、1、-1）编码特殊语义，既不与真实函数地址冲突（函数地址不可能是 0、1、-1），又不需要额外的返回参数。

**例子：**

```
#include <signal.h>

int main() {
    // 1. 忽略 SIGINT（Ctrl+C 不再终止程序）
    signal(SIGINT, SIG_IGN);

    // 2. 恢复默认行为
    signal(SIGINT, SIG_DFL);

    // 3. 检查 signal 是否调用成功
    if (signal(SIGTERM, my_handler) == SIG_ERR) {
        perror("signal failed");
    }
    return 0;
}
```

**注意 SIG_IGN 的继承性：** `fork()` 创建子进程时，子进程**继承**父进程的信号处理设置。但 `exec()` 执行新程序时，被设为 `SIG_IGN` 的信号会**继续保持忽略**，而被捕获的信号会**重置为 SIG_DFL**（因为原处理函数的代码在新程序的地址空间里不存在了）。这就是为什么很多 shell 启动后台进程时会先把 SIGINT 和 SIGQUIT 设为 SIG_IGN——这样后台进程就不会被 Ctrl+C 影响。

------

## 三、信号屏蔽字操作码宏（sigprocmask / pthread_sigmask 的 how 参数）

| 宏            | 含义     | 集合运算                            |
| ------------- | -------- | ----------------------------------- |
| `SIG_BLOCK`   | 追加屏蔽 | `new_mask = old_mask ∪ set`         |
| `SIG_UNBLOCK` | 解除屏蔽 | `new_mask = old_mask \ set`（差集） |
| `SIG_SETMASK` | 直接赋值 | `new_mask = set`                    |

这三个上一轮已经详细讲过，这里补充一个关键点：

**为什么需要三个操作而不是只有 SETMASK？** 因为你通常**不知道**当前屏蔽字里已经有什么。如果只有 SETMASK，你想"临时屏蔽 SIGINT"就必须先读出现有屏蔽字、或上 SIGINT、再写回去——需要两次系统调用。SIG_BLOCK 把"读-改-写"原子化在一次系统调用里，避免了竞态条件。

**完整的临界区模式（上一轮的代码）：**

```
sigset_t mask, prev_mask;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);

sigprocmask(SIG_BLOCK, &mask, &prev_mask);  // 追加屏蔽，保存旧值
/* 临界区：不会被 SIGINT 打断 */
sigprocmask(SIG_SETMASK, &prev_mask, NULL);  // 精确恢复
```

------

## 四、sigaction 的 sa_flags 行为标志宏

`struct sigaction` 里有一个 `int sa_flags` 字段，用按位或（`|`）组合以下宏，精细控制信号递送的行为。这是 `<signal.h>` 里最容易被忽略但最强大的一组宏。

| 宏             | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| `SA_RESTART`   | 被信号打断的系统调用**自动重启**（而不是返回 EINTR）         |
| `SA_SIGINFO`   | 使用 `sa_sigaction`（三参数处理函数）而非 `sa_handler`，可获取信号来源详细信息 |
| `SA_NODEFER`   | 处理信号期间**不自动屏蔽**该信号本身（允许递归嵌套）         |
| `SA_RESETHAND` | 信号递送一次后，处理函数**重置为 SIG_DFL**（System V 语义）  |
| `SA_ONSTACK`   | 在 `sigaltstack()` 指定的**替代栈**上执行处理函数（用于栈溢出时的 SIGSEGV 处理） |
| `SA_NOCLDSTOP` | 仅对 SIGCHLD 有效：子进程**停止**时不发信号，只在**终止**时发 |
| `SA_NOCLDWAIT` | 仅对 SIGCHLD 有效：子进程终止后**不变成僵尸**，内核自动回收  |
| `SA_INTERRUPT` | 与 SA_RESTART 相反：被打断的系统调用**不重启**（默认行为，几乎不用显式写） |

### 4.1 SA_RESTART —— 最常用的标志

**问题背景：** 当一个慢速系统调用（`read`、`write`、`wait`、`select`、`pause`、`sleep` 等）正在阻塞时，如果来了一个信号，系统调用会**被打断**并返回 `-1`，errno 设为 `EINTR`。这意味着你的代码必须到处写：

```
while ((n = read(fd, buf, size)) == -1 && errno == EINTR)
    ;  // 被信号打断，重试
```

**SA_RESTART 的作用：** 设置后，内核会在信号处理函数返回后**自动重启**被打断的系统调用，对你的代码透明。这样 `read()` 就不会因为一个信号而返回 EINTR。

**例子：**

```
struct sigaction sa = {0};
sa.sa_handler = handler;
sa.sa_flags = SA_RESTART;          // 关键：自动重启系统调用
sigemptyset(&sa.sa_mask);
sigaction(SIGINT, &sa, NULL);

char buf[1024];
read(STDIN_FILENO, buf, sizeof(buf));  // 被 SIGINT 打断后会自动重试，不会返回 EINTR
```

**注意：** 不是所有系统调用都支持自动重启。`sleep()`、`msgrcv()`、某些 `semop()` 等即使设了 SA_RESTART 也会返回 EINTR。`accept()`、`recv()`、`send()` 在 Linux 上支持重启。

### 4.2 SA_SIGINFO —— 获取信号的"元数据"

默认的 `sa_handler` 只有一个参数（信号编号）。设置 `SA_SIGINFO` 后，改用 `sa_sigaction` 字段，签名为：

```
void (*sa_sigaction)(int sig, siginfo_t *info, void *ucontext);
```

- `sig`：信号编号
- `info`：`siginfo_t` 结构体，包含**谁发的信号**（`si_pid`、`si_uid`）、**为什么发**（`si_code`）、**附带数据**（`si_value`）等
- `ucontext`：信号发生时的 CPU 寄存器上下文（`ucontext_t`），可用于调试或从错误中恢复

**例子：区分 SIGSEGV 的原因**

```
void segv_handler(int sig, siginfo_t *info, void *ucontext) {
    printf("SIGSEGV at address: %p\n", info->si_addr);
    if (info->si_code == SEGV_MAPERR)
        printf("原因：访问了未映射的地址\n");
    else if (info->si_code == SEGV_ACCERR)
        printf("原因：访问了无权限的地址（如写只读页）\n");
    _exit(1);
}

struct sigaction sa = {0};
sa.sa_sigaction = segv_handler;   // 注意用 sa_sigaction，不是 sa_handler
sa.sa_flags = SA_SIGINFO | SA_ONSTACK;  // SA_ONSTACK 配合替代栈，防止栈溢出时处理函数也崩
sigemptyset(&sa.sa_mask);
sigaction(SIGSEGV, &sa, NULL);
```

### 4.3 SA_NODEFER —— 允许信号递归嵌套

**默认行为：** 当一个信号的处理函数正在执行时，内核会**自动屏蔽**该信号本身（防止处理函数被同一个信号再次打断，导致栈无限增长）。处理函数返回后自动解除。

**SA_NODEFER 的作用：** 取消这个自动屏蔽，允许在处理函数执行期间再次收到并处理同一个信号——即**递归嵌套**。

**为什么默认要屏蔽？** 想象你的 SIGINT 处理函数里有 `printf`，如果不屏蔽，处理函数执行到一半又来一个 SIGINT，又进处理函数，又 `printf`……栈会迅速爆炸。所以默认屏蔽是安全设计。

**什么时候用 SA_NODEFER？** 极少数场景，比如你需要在处理函数里 `sigsuspend` 等待另一个信号，同时不希望屏蔽当前信号。绝大多数情况**不要用**。

### 4.4 SA_RESETHAND —— 一次性信号处理

设置后，信号处理函数**只执行一次**，执行完后该信号的处理方式自动重置为 `SIG_DFL`。这是 System V 的历史行为（早期 Unix 的 `signal()` 就是这样的），POSIX 标准化后 `signal()` 的行为由实现决定，所以用 `sigaction` + `SA_RESETHAND` 可以显式复现这种"一次性"语义。

### 4.5 SA_ONSTACK —— 替代栈上处理信号

**经典应用：处理栈溢出导致的 SIGSEGV。** 如果程序因为无限递归导致栈溢出，触发 SIGSEGV 时，正常的栈已经满了，信号处理函数也需要用栈——结果处理函数一进来就再次触发 SIGSEGV，死循环。

解决方案：用 `sigaltstack()` 预先分配一块**替代信号栈**（通常在堆上），设置 `SA_ONSTACK` 后，SIGSEGV 的处理函数就在这块替代栈上执行，不受原栈溢出影响。

```
// 预先分配替代栈
stack_t ss;
ss.ss_sp = malloc(SIGSTKSZ);   // SIGSTKSZ 是推荐的替代栈大小（通常 8192）
ss.ss_size = SIGSTKSZ;
ss.ss_flags = 0;
sigaltstack(&ss, NULL);

// 注册 SIGSEGV 处理时加上 SA_ONSTACK
struct sigaction sa = {0};
sa.sa_sigaction = segv_handler;
sa.sa_flags = SA_SIGINFO | SA_ONSTACK;
sigaction(SIGSEGV, &sa, NULL);
```

------

## 五、实时信号范围宏

| 宏         | 含义                                        |
| ---------- | ------------------------------------------- |
| `SIGRTMIN` | 第一个实时信号的编号（Linux 上通常是 34）   |
| `SIGRTMAX` | 最后一个实时信号的编号（Linux 上通常是 64） |

**实时信号与标准信号的核心区别：**

1. **排队**：同一个实时信号来了多次，会**排队递送多次**；标准信号的 pending 是位图，多次到达只递送一次
2. **顺序保证**：多个不同实时信号按编号从小到大递送（小编号优先级高）
3. **可携带数据**：用 `sigqueue()` 发送实时信号时，可以附带一个整数或指针（`union sigval`）
4. **没有预定义语义**：SIGRTMIN~SIGRTMAX 全部留给应用程序自定义用途

**注意：不要直接硬编码 34 或 64**，因为不同平台/架构的实时信号范围可能不同（比如有些架构 SIGRTMIN 是 35，因为 34 被 glibc 内部用作线程取消信号）。始终用 `SIGRTMIN` 和 `SIGRTMAX` 计算。

**例子：**

```
#include <signal.h>
#include <stdio.h>

int main() {
    printf("实时信号范围: %d ~ %d，共 %d 个\n",
           SIGRTMIN, SIGRTMAX, SIGRTMAX - SIGRTMIN + 1);
    // 应用通常用 SIGRTMIN + n 来分配自定义信号
    // 但注意：glibc 内部可能占用了 SIGRTMIN 开头的几个，
    // 推荐从 SIGRTMIN + 3 或更高开始用
    return 0;
}
```

------

## 六、辅助常量宏

| 宏            | 含义                                                         |
| ------------- | ------------------------------------------------------------ |
| `NSIG`        | 信号总数 + 1（Linux 上通常是 65，即 0~64）。常用于声明信号处理函数数组的大小 |
| `SIGSTKSZ`    | 替代信号栈的**推荐大小**（字节数，通常 8192）                |
| `MINSIGSTKSZ` | 替代信号栈的**最小必需大小**（通常 2048），小于这个值 `sigaltstack` 会报错 |

**NSIG 的用法：**

```
// 声明一个信号处理函数指针数组，覆盖所有可能的信号编号
void (*handlers[NSIG])(int);
```

------

## 七、设计哲学总结

把这些宏放在一起看，能体会到 Unix 信号设计的几个核心原则：

1. **不可杀死的逃生舱**：`SIGKILL` 和 `SIGSTOP` 不能被捕获/阻塞/忽略——保证系统总有办法强行控制一个失控进程
2. **原子性优先**：`SIG_BLOCK`/`SIG_UNBLOCK`/`SIG_SETMASK` 把"读-改-写"压缩成一次原子系统调用，避免竞态
3. **保存恢复模式**：`prev_mask` + `SIG_SETMASK` 的模式无处不在，本质是"不破坏调用者状态"的 RAII 思想
4. **可扩展性**：`SA_SIGINFO` 让信号从"只有一个编号"进化到"携带完整元数据"；实时信号 `SIGRTMIN`~`SIGRTMAX` 留给应用自由定义
5. **安全默认**：处理函数执行期间默认屏蔽同信号（不用 SA_NODEFER），防止递归栈溢出
6. **兼容性钩子**：`SA_RESETHAND`、`SA_NODEFER`、`SIG_HOLD` 等是为了兼容 System V / BSD 的历史行为差异

这些宏不是孤立的常量，而是一整套围绕"进程间异步通知"设计的词汇表。理解了它们，就理解了 Unix 信号机制的全部行为边界。