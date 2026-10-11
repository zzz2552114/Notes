# 标准 I/O 与 UNIX I/O 在信号处理中的异步安全性详解

## 一、先搞清楚：这里的"异步"到底指什么？

这里的"异步"不是 AJAX 那种异步，也不是 async/await，而是指**信号（signal）的到达时机是异步的、不可预测的**。

具体来说：

- 你的主程序正在正常执行，可能执行到**任意一条机器指令**的中间。
- 此时一个信号递达（比如你按了 Ctrl+C 产生 SIGINT，或者子进程退出产生 SIGCHLD，或者 alarm 定时器到点产生 SIGALRM）。
- 内核会**强行打断**主程序的执行流，把 CPU 控制权转交给你注册的信号处理函数（signal handler）。
- 信号处理函数执行完毕后，内核再把控制权**交还**给主程序，主程序从刚才被打断的地方继续执行，仿佛什么都没发生过。

```
主程序执行流:  ────A────────B────────C────►
                     │
              信号在此刻到达
                     │
                     ▼
              信号处理函数执行
                     │
                     └────────► 回到B点继续
```

关键点：**信号处理函数可以在主程序执行到任何位置时被插入执行**。主程序此刻可能正处于某个函数调用的中间、可能正持有某把锁、可能正在修改某个全局数据结构的半途中。这就是"异步"的含义——它不是同步地在你代码的某个确定点发生，而是随时可能发生。

------

## 二、什么是"异步信号安全"（async-signal-safe）？

### 2.1 定义

一个函数被称为**异步信号安全**的，当且仅当：**它可以在信号处理函数中被安全地调用，无论信号在主程序执行到哪一步时到达，都不会导致数据损坏、死锁、未定义行为或程序崩溃。**

换句话说，异步信号安全 = 这个函数能承受"在自己执行到一半时被打断，然后打断者又来调用自己"这种极端情况。

### 2.2 异步信号安全的函数必须满足的条件

一个函数要做到异步信号安全，必须同时满足：

1. **可重入（reentrant）**：函数不依赖任何全局或静态的可变状态。如果它必须使用全局数据，那它必须能在"自己被打断、别人又改了这份数据、再回来继续"的情况下仍然正确。
2. **不调用任何异步信号不安全的函数**：安全是传染的——你调用了不安全的函数，你自己也就不安全了。
3. **不持有任何可能导致死锁的锁**：如果函数内部要加锁，而信号处理函数中也调用同一个函数，就可能在主程序持有锁时被信号打断，然后信号处理函数试图获取同一把锁，造成**自死锁**。
4. **对 errno 的处理要小心**：信号处理函数可能修改 errno，破坏主程序的 errno 值。安全的做法是信号处理函数入口保存 errno，出口恢复。

### 2.3 POSIX 规定的异步信号安全函数列表

POSIX.1-2008 明确列出了一组保证异步信号安全的函数，包括但不限于：

- `_Exit()`、`_exit()`、`abort()`、`accept()`、`access()`
- `alarm()`、`bind()`、`cfgetispeed()`、`cfgetospeed()`
- `chdir()`、`chmod()`、`chown()`、`clock_gettime()`
- `close()`、`connect()`、`creat()`、`dup()`、`dup2()`
- `execle()`、`execve()`、`fchmod()`、`fchown()`、`fcntl()`
- `fdatasync()`、`fork()`、`fpathconf()`、`fstat()`、`fsync()`
- `ftruncate()`、`getegid()`、`geteuid()`、`getgid()`、`getgroups()`
- `getpeername()`、`getpgrp()`、`getpid()`、`getppid()`
- `getsockname()`、`getsockopt()`、`getuid()`、`kill()`
- `link()`、`listen()`、`lseek()`、`lstat()`、`mkdir()`
- `mkfifo()`、`open()`、`pathconf()`、`pause()`、`pipe()`
- `poll()`、`posix_trace_event()`、`pselect()`、`raise()`
- `read()`、`readlink()`、`recv()`、`recvfrom()`、`recvmsg()`
- `rename()`、`rmdir()`、`select()`、`sem_post()`、`send()`
- `sendmsg()`、`sendto()`、`setgid()`、`setpgid()`、`setsid()`
- `setsockopt()`、`setuid()`、`shutdown()`、`sigaction()`
- `sigaddset()`、`sigdelset()`、`sigemptyset()`、`sigfillset()`
- `sigismember()`、`signal()`、`sigpause()`、`sigpending()`
- `sigprocmask()`、`sigqueue()`、`sigset()`、`sigsuspend()`
- `sleep()`、`sockatmark()`、`socket()`、`socketpair()`
- `stat()`、`symlink()`、`sysconf()`、`tcdrain()`、`tcflow()`
- `tcflush()`、`tcgetattr()`、`tcgetpgrp()`、`tcsendbreak()`
- `tcsetattr()`、`tcsetpgrp()`、`time()`、`timer_getoverrun()`
- `timer_gettime()`、`timer_settime()`、`times()`、`umask()`
- `uname()`、`unlink()`、`utime()`、`wait()`、`waitpid()`
- `write()`

注意看：`read()`、`write()`、`open()`、`close()` 都在列表里——这就是 **UNIX I/O**。而 `printf()`、`fprintf()`、`fgets()`、`fread()`、`fwrite()`、`malloc()`、`free()` 都**不在**列表里——这就是**标准 I/O** 和内存分配函数。

------

## 三、为什么标准 I/O（stdio）是异步信号不安全的？

标准 I/O 指的是 `<stdio.h>` 里的那套函数：`printf`、`fprintf`、`scanf`、`fgets`、`fread`、`fwrite`、`putchar`、`getchar` 等等。它们不安全，根源有三个：

### 3.1 根源一：标准 I/O 使用用户态缓冲区，缓冲区有复杂的内部状态

标准 I/O 不是直接调用 `write()` 把数据写给内核，而是先把数据写到**用户态的缓冲区**（`FILE` 结构体内部的一块内存），等缓冲区满了、或者遇到换行符（行缓冲）、或者程序显式 `fflush()` 时，才一次性调用 `write()` 把缓冲区内容刷给内核。

这个 `FILE` 结构体里维护着大量可变状态：

```
// 简化的 FILE 结构体概念（实际 glibc 实现更复杂）
struct _IO_FILE {
    int _flags;           // 文件状态标志
    char *_IO_read_ptr;   // 读指针：当前读到哪了
    char *_IO_read_end;   // 读结束位置
    char *_IO_read_base;  // 读缓冲区基址
    char *_IO_write_base; // 写缓冲区基址
    char *_IO_write_ptr;  // 写指针：当前写到哪了
    char *_IO_write_end;  // 写结束位置
    char *_IO_buf_base;   // 缓冲区基址
    char *_IO_buf_end;    // 缓冲区结束位置
    int _fileno;          // 文件描述符
    // ... 还有锁、宽字符状态、编码状态等等
};
```

当你调用 `printf("hello")` 时，它内部要做的事情大致是：

1. 格式化字符串，生成 `"hello"`。
2. 检查 `stdout` 的缓冲区状态：缓冲区还有多少空间？当前写指针在哪？
3. 把 `"hello"` 拷贝到缓冲区中 `_IO_write_ptr` 指向的位置。
4. 更新 `_IO_write_ptr`，往前移动 5 个字节。
5. 检查是否需要 flush（比如缓冲区满了、或者是行缓冲且遇到了 `\n`）。
6. 如果需要 flush，调用 `write()` 系统调用，然后重置缓冲区指针。

**问题来了**：假设主程序正在执行 `printf`，刚刚完成第 3 步（把数据拷贝进缓冲区），但还没来得及执行第 4 步（更新写指针）——就在这两条指令之间，信号来了！

```
主程序 printf 内部:
  步骤3: 把 "hello" 拷贝到缓冲区  ← 信号在此刻到达
  步骤4: 更新 _IO_write_ptr
  步骤5: 检查是否需要 flush
```

信号处理函数里也调用了 `printf("world")`。它看到 `_IO_write_ptr` 还没更新（还指向旧位置），于是把 `"world"` 也拷贝到了同一个位置——**覆盖了刚才的 "hello"**。然后信号处理函数的 printf 更新了写指针、可能还 flush 了缓冲区。等信号处理函数返回，主程序从步骤 4 继续执行，它又把写指针往前移动 5 个字节（但这 5 个字节的内容其实已经被 "world" 覆盖了）。

结果：缓冲区里的数据乱了，`"hello"` 丢失了，输出可能是 `"world"` 后面跟着一堆垃圾，或者直接段错误。

这就是**不可重入**的典型表现：函数依赖全局/静态的可变状态（缓冲区指针），在状态修改的中途被打断，打断者又修改了同一份状态，回来后状态已经不一致了。

### 3.2 根源二：标准 I/O 内部使用锁，可能导致自死锁

现代的 glibc 标准 I/O 实现是**线程安全**的。为了支持多线程环境下多个线程同时操作同一个 `FILE*`，glibc 在每个 `FILE` 结构体里加了一把锁（递归互斥锁 `recursive mutex`）。

每次调用 `printf`、`fputc` 等函数时，内部会先执行：

```
_flockfile(fp);   // 加锁
// ... 操作缓冲区 ...
_funlockfile(fp); // 解锁
```

现在考虑这个场景：

```
主程序:
  printf("hello")
    → _flockfile(stdout) 成功，持有锁
    → 正在操作缓冲区...
    ← 信号在此刻到达！

信号处理函数:
  printf("world")
    → _flockfile(stdout)  ← 试图获取同一把锁！
    → 但锁被主程序持有，而且主程序被信号打断了，
      永远不会执行到 _funlockfile，因为它在等信号处理函数返回。
    → 死锁！程序永远卡住。
```

这就是**自死锁（self-deadlock）**：同一个执行流（其实是同一个线程）在持有锁的情况下被信号打断，然后信号处理函数又试图获取同一把锁。虽然 glibc 的 stdio 锁是递归锁（同一个线程可以重复加锁），但问题在于——**信号处理函数运行在被中断的线程的上下文中**，从内核角度看确实是同一个线程，递归锁理论上允许重复获取。但 glibc 的实现中，信号处理函数中调用 stdio 函数时，锁的状态可能已经被破坏（比如锁的所有者字段、递归计数处于不一致的中间状态），仍然可能出问题。而且即使不死锁，缓冲区状态的问题（根源一）依然存在。

更关键的是：**POSIX 标准并没有要求 stdio 的锁能在信号处理函数中安全使用**。即使某个实现碰巧不死锁，也不代表它是异步信号安全的。

### 3.3 根源三：标准 I/O 可能间接调用 malloc/free，而 malloc/free 不是异步信号安全的

标准 I/O 在某些情况下会动态分配内存。比如：

- 第一次使用某个 `FILE*` 时，可能需要分配缓冲区（`setvbuf` 或自动分配）。
- `open_memstream()`、`fopencookie()` 等函数会动态分配。
- 宽字符函数（`wprintf` 等）可能分配转换缓冲区。
- `glibc` 的 `printf` 内部在处理某些格式时可能使用 `malloc`。

而 `malloc` 和 `free` 是**出了名的异步信号不安全**。它们维护着全局的堆管理数据结构（空闲链表、bin 数组等），并且内部有锁。

考虑这个场景：

```
主程序:
  malloc(100)
    → 获取 malloc 全局锁
    → 正在修改空闲链表（已经把一个块从链表中摘下，
      但还没把剩余块链回去）← 信号在此刻到达！

信号处理函数:
  printf("...")
    → 内部调用 malloc(...)
      → 试图获取 malloc 全局锁 → 死锁
      → 或者锁碰巧可重入 → 看到半修改的空闲链表 → 内存损坏
```

`malloc` 的全局数据结构处于不一致的中间状态，信号处理函数中的 `malloc`/`free` 基于这个损坏的状态继续操作，会导致**堆损坏（heap corruption）**，程序可能在之后任意时刻崩溃，而且极难调试。

### 3.4 一个具体的不安全示例

```
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void handler(int sig) {
    // 不安全！在信号处理函数中调用 printf
    printf("Caught signal %d\n", sig);
}

int main() {
    signal(SIGINT, handler);
    while (1) {
        printf("Main loop working...\n");
        // 主程序可能正在 printf 的中间被 SIGINT 打断
        sleep(1);
    }
    return 0;
}
```

这段代码在大多数时候可能看起来能跑，但它是**未定义行为**。在高负载下、或者信号在"恰好那个时机"到达时，就可能出现输出错乱、死锁或崩溃。CSAPP 教材强调的就是：**不能因为它大多数时候能跑就认为它是安全的。**

------

## 四、为什么 UNIX I/O 是异步信号安全的？

UNIX I/O 指的是直接使用系统调用的那套 I/O：`open()`、`read()`、`write()`、`close()`、`lseek()` 等，定义在 `<unistd.h>` 和 `<fcntl.h>` 中。

### 4.1 UNIX I/O 不使用用户态缓冲区

`write(fd, buf, count)` 做的事情很简单：

1. 把 `buf` 中的 `count` 字节数据，通过系统调用陷入内核。
2. 内核把数据拷贝到内核缓冲区（或直接写到设备）。
3. 返回实际写入的字节数。

整个过程中，**用户态没有任何需要维护的可变状态**。没有写指针、没有缓冲区满不满的判断、没有刷新逻辑。每次 `write()` 调用都是独立的、自包含的。

即使主程序正在执行 `write()` 的中间被信号打断：

- 如果 `write()` 已经陷入内核，内核会保证系统调用的原子性（对于普通文件的 write，POSIX 要求在某些条件下原子；对于管道和 socket，小于 PIPE_BUF 的 write 是原子的）。
- 如果 `write()` 被信号中断，它会返回 `-1` 并设置 `errno = EINTR`，主程序可以检查这个错误并重试。这不是"不安全"，而是系统调用的正常行为——它不会损坏任何数据。
- 信号处理函数中再调用 `write()`，它操作的是独立的参数，不依赖任何用户态全局状态，所以不会冲突。

### 4.2 UNIX I/O 是系统调用，由内核保证安全

`read()`、`write()` 等本质上是**系统调用**（syscall）。系统调用的执行由内核管理：

- 内核内部的数据结构（如文件表、inode 缓存）有自己的锁机制，这些锁是**针对内核设计的**，能正确处理中断和并发。
- 系统调用要么完整执行、要么返回错误（如 EINTR），不会处于"半完成"的不一致状态。
- POSIX 标准明确将 `read`、`write`、`open`、`close` 等列入异步信号安全函数列表。

### 4.3 UNIX I/O 不调用 malloc/free

`write()` 系统调用不涉及用户态的动态内存分配。它直接使用你传入的缓冲区，不会去碰全局的堆管理结构。因此不会有 malloc/free 那种堆损坏的问题。

### 4.4 一个安全的示例

```
#include <unistd.h>
#include <signal.h>
#include <string.h>

void handler(int sig) {
    // 安全！write 是异步信号安全的
    const char msg[] = "Caught signal\n";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);
}

int main() {
    signal(SIGINT, handler);
    while (1) {
        const char msg[] = "Main loop working...\n";
        write(STDOUT_FILENO, msg, sizeof(msg) - 1);
        sleep(1);
    }
    return 0;
}
```

这段代码是安全的。即使信号在主程序的 `write()` 执行中间到达，信号处理函数中的 `write()` 也不会造成数据损坏或死锁。

------

## 五、对比总结

| 维度           | 标准 I/O（printf/fread 等）                        | UNIX I/O（read/write 等）                |
| -------------- | -------------------------------------------------- | ---------------------------------------- |
| 缓冲区         | 用户态缓冲区，有复杂内部状态（读写指针、缓冲模式） | 无用户态缓冲区，直接陷入内核             |
| 锁             | 内部有锁（线程安全用），信号处理中可能自死锁       | 系统调用，内核态锁由内核管理             |
| 内存分配       | 可能间接调用 malloc/free                           | 不涉及用户态动态内存分配                 |
| 可重入性       | 不可重入（依赖 FILE 结构体的全局状态）             | 可重入（每次调用独立，无用户态全局状态） |
| POSIX 认证     | **不在**异步信号安全列表中                         | **在**异步信号安全列表中                 |
| 信号处理中使用 | 未定义行为，可能死锁/数据损坏/崩溃                 | 安全，最多返回 EINTR                     |

------

## 六、信号处理函数的正确写法原则

CSAPP 强调的信号处理函数最佳实践：

1. **保持信号处理函数尽可能简单**——最好只设置一个全局的 `volatile sig_atomic_t` 标志位，然后主程序循环检查这个标志位来做实际工作。

2. **在信号处理函数中只调用异步信号安全的函数**——比如 `write()`、`_exit()`、`sigaction()` 等。绝对不要调用 `printf`、`malloc`、`free`、`syslog` 等。

3. **保存和恢复 errno**——信号处理函数中调用的安全函数（如 `write`）可能修改 errno，会破坏主程序的 errno。做法：

   ```
   void handler(int sig) {
       int saved_errno = errno;  // 保存
       // ... 做事情 ...
       errno = saved_errno;      // 恢复
   }
   ```

4. **保护共享数据结构**——如果主程序和信号处理函数共享某个全局数据结构，主程序在操作该结构时要暂时阻塞相关信号（用 `sigprocmask`），操作完再解除。

5. **用 `sigaction` 代替 `signal`**——`signal` 的语义在不同 UNIX 实现中不一致，`sigaction` 更可靠、更可控。

------

## 七、一句话总结

> **"异步"指信号随时可能在主程序的任意指令处到达并打断执行；"异步信号安全"指函数能在这种被打断的情况下仍然正确工作。标准 I/O 因为有用户态缓冲区（不可重入）、内部锁（可能死锁）、可能调用 malloc（堆损坏），所以不安全；UNIX I/O 因为是无缓冲的系统调用、不碰用户态全局状态、不分配内存，所以安全。**