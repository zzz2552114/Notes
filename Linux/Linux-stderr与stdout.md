## stderr 与 stdout



默认情况下，`fprintf(stderr, ...)` 会显示在**运行这个程序的终端/控制台**里，和 `printf` 一样能看到，只是它走的是“标准错误”流。

`stderr` 对应文件描述符 `2`。  
`stdout` 对应 `1`，普通 `printf` 走的是 `1`。

例如：

```c
#include <stdio.h>

int main(void) {
    fprintf(stdout, "这是正常输出\n");
    fprintf(stderr, "这是错误输出\n");
    return 0;
}
```

直接运行：

```bash
./a.out
```

终端里两行都会显示。

关键区别在重定向：

```bash
./a.out > out.txt
```

只会把 `stdout` 写到 `out.txt`，`stderr` 仍然显示在终端。

```bash
./a.out 2> err.txt
```

把 `stderr` 写到 `err.txt`，然后你可以：

```bash
cat err.txt
```

或者实时看：

```bash
tail -f err.txt
```

如果想把正常输出和错误输出都存到一个文件：

```bash
./a.out > all.txt 2>&1
```

注意顺序：`> all.txt 2>&1` 表示先把 stdout 指向文件，再让 stderr 也指向同一个地方。

如果只想在终端里过滤错误：

```bash
./a.out 2>&1 | grep -i error
```

在不同环境里，`stderr` 可能显示在：

- 终端直接运行：显示在当前终端。
- IDE：通常在“终端”“控制台”“调试控制台”或“错误输出”窗口。
- systemd 服务：进 journal，可以用 `journalctl -u 服务名 -f` 看。
- Docker 容器：用 `docker logs -f 容器名` 看。
- GUI 程序：可能没有终端，`stderr` 可能被丢弃，或者进系统日志。
- 被重定向时：`stderr` 会进你指定的文件或管道，不一定还在屏幕上。

如果程序正在运行，想知道它的 `stderr` 到底去哪了，可以查：

```bash
ls -l /proc/<pid>/fd/2
```

如果显示 `2 -> /dev/pts/0`，说明在某个终端；如果显示 `2 -> /path/err.log`，说明被重定向到文件了。

另外注意：`stderr` 通常不带缓冲，`stdout` 在重定向到文件时可能是全缓冲，所以混合输出时顺序可能看起来乱。需要顺序一致时可以加：

```c
fflush(stdout);
```

---



## 关于 2>&1

`2>&1` 里的 `&1` 要整体看：

- `2>`：重定向文件描述符 `2`，也就是 `stderr`
- `&`：告诉 shell，后面跟的是**文件描述符**，不是文件名
- `1`：文件描述符 `1`，也就是 `stdout`

所以：

```bash
2>&1
```

意思是：**把 stderr 重定向到 stdout 当前指向的地方**。

在：

```bash
./a.out 2>&1 | grep -i error
```

里面，管道 `|` 已经把 `./a.out` 的 stdout 接到了 `grep` 的标准输入。然后 `2>&1` 让 stderr 也复制 stdout 的目标，所以 stderr 也进了管道，被 `grep` 过滤。

---

### 完全不写 `2>&1` 呢？

```bash
./a.out | grep -i error
```

这样只有 stdout 会进管道给 `grep`。stderr 仍然直接显示在终端里，不会被 `grep` 过滤。

如果你只想过滤正常输出，那这样可以；如果你想连错误信息一起过滤，就必须写：

```bash
./a.out 2>&1 | grep -i error
```

---

记住关键点：

```bash
2>&1
```

不是“把 2 重定向到文件 1”，而是“把 fd 2 复制成 fd 1 当前指向的目标”。`&` 不能省，省了就变成文件名了。



## 关于管道

默认可以这么理解：`A | B` 会把 **A 的 stdout 接到 B 的 stdin**。  
但严格说，管道符连接的是**文件描述符**：

- 左边命令的 **fd 1**，也就是 stdout
- 右边命令的 **fd 0**，也就是 stdin

所以底层大概是：

```text
A 的 fd 1  ->  管道  ->  B 的 fd 0
```

不是“一定只能是 stdout”，而是 **默认只自动处理 fd 1 和 fd 0**。

---

### stderr 会不会自动进管道？

不会。

---

### 右边的 stdin 一定来自管道吗？

也不一定。右边命令的 stdin 可以被重新重定向覆盖：

```bash
./a.out | grep error < other.txt
```

这里 `grep` 的 stdin 被 `other.txt` 覆盖了，它不再读管道。

左边 stdout 也可以被重定向走：

```bash
./a.out > out.txt | grep error
```

这里 `./a.out` 的 stdout 去了 `out.txt`，管道里没数据，`grep` 读不到东西。

---

### 能不能让别的 fd 也进管道？

可以，但需要你自己重定向。比如：

```bash
./a.out 3>&1 | some_program
```

这会让 fd 3 也复制 fd 1 当前的目标，也就是管道写端。  
但管道符本身不会自动管 fd 3，它只默认连 fd 1。

---



## 关于 fd

`fd` 就是 **file descriptor，文件描述符**。它是进程内部用来表示“我打开了某个东西”的编号，是一个非负整数。这个“东西”可以是文件、终端、管道、socket、设备等等。

### 0、1、2 是标准约定

| fd   | 名称       | 默认指向       | 说明                                    |
| ---- | ---------- | -------------- | --------------------------------------- |
| 0    | stdin      | 键盘/终端      | 标准输入                                |
| 1    | stdout     | 终端           | 标准输出，`printf` 默认走这里           |
| 2    | stderr     | 终端           | 标准错误，`fprintf(stderr, ...)` 走这里 |
| 3+   | 无固定名称 | 看程序怎么打开 | 普通文件描述符                          |

所以：

```c
fprintf(stderr, "error\n");
```

底层就是往 **fd 2** 写。

`printf("hello\n");` 底层通常是往 **fd 1** 写。

---

### fd3 是什么？

`fd3` 就是“3 号文件描述符”。它本身没有任何特殊标准含义，只是进程文件描述符表里的一个编号。

通常前三个编号 `0、1、2` 已经被标准输入、标准输出、标准错误占用。所以当一个进程再打开一个新文件、管道或 socket 时，内核常常会分配 **3** 给它。

例如：

```c
int fd = open("a.txt", O_WRONLY | O_CREAT, 0644);
printf("%d\n", fd);
```

如果这个进程的 `0、1、2` 都已经打开着，那么 `fd` 通常就是 `3`。

再打开一个文件，可能得到 `4`：

```c
int fd2 = open("b.txt", O_WRONLY | O_CREAT, 0644);
printf("%d\n", fd2); // 通常 4
```

但这不是绝对的。内核一般分配“当前最小可用的 fd”。如果 `1` 被关掉了，新打开的文件可能拿到 `1`。

---

### 查看一个进程的 fd

Linux 下可以看：

```bash
ls -l /proc/<pid>/fd
```

例如：

```text
0 -> /dev/pts/0
1 -> /dev/pts/0
2 -> /dev/pts/0
3 -> /tmp/log.txt
```

也可以用：

```bash
lsof -p <pid>
```

看当前打开的文件描述符。

---

### 其他要点

- `0、1、2` 只是 POSIX 标准约定，程序可以关闭或重定向它们。
- `3` 及以上没有统一标准，谁打开谁决定。
- 新 fd 通常取最小可用整数。
- 管道符 `|` 默认只连左边 fd 1 和右边 fd 0，不自动管 fd 3。

所以，`fd3` 不是什么特殊东西，就是“第 4 个文件描述符槽位”。其他数字 `4、5、6...` 也一样，都是普通 fd。



## 仅 grep stderr 的写法

你想只让 **stderr** 进管道给 `grep`，stdout 丢掉，正确写法是：

```bash
./a.out 2>&1 1>/dev/null | grep -i error
```

注意顺序：**`2>&1` 必须写在 `1>/dev/null` 前面**。

---

### 为什么是这个顺序？

管道 `|` 默认先把左边命令的 **stdout（fd 1）** 接到 `grep` 的 stdin。  
所以一开始：

```text
./a.out 的 fd 1 -> 管道
./a.out 的 fd 2 -> 终端
```

然后从左到右执行重定向：

1. `2>&1`  
   把 fd 2 复制成 fd 1 当前的目标，也就是**管道**。  
   现在：

   ```text
   fd 1 -> 管道
   fd 2 -> 管道
   ```

2. `1>/dev/null`  
   把 fd 1 改成 `/dev/null`，也就是丢掉 stdout。  
   现在：

   ```text
   fd 1 -> /dev/null
   fd 2 -> 管道
   ```

所以最后只有 **stderr** 进了管道，`grep` 只能看到错误输出。

---

### 验证一下

假设程序输出：

```c
fprintf(stdout, "stdout hello\n");
fprintf(stderr, "stderr error\n");
```

运行：

```bash
./a.out 2>&1 1>/dev/null | grep -i error
```

输出：

```text
stderr error
```

如果写成：

```bash
./a.out 1>/dev/null 2>&1 | grep -i error
```

则没有任何输出，因为 stderr 也被重定向到 `/dev/null` 了。
