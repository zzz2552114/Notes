# CSAPP Shell Lab 骨架代码详解

下面我按代码出现的顺序，把每个 C 库函数、系统调用和自定义函数都讲清楚。**只讲已给出的代码和通用 C/POSIX 知识，不涉及你需要补全的部分。**

------

## 一、头文件（#include）

```
#include <stdio.h>      // 标准输入输出
#include <stdlib.h>     // 通用工具：exit、atoi、malloc 等
#include <unistd.h>     // POSIX 系统调用：fork、exec、pipe、dup2 等
#include <string.h>     // 字符串操作：strcpy、strlen、strchr 等
#include <ctype.h>      // 字符分类：isspace、isdigit 等（本文件实际未直接用）
#include <signal.h>     // 信号相关：sigaction、kill、信号常量
#include <sys/types.h>  // 基本系统数据类型：pid_t、size_t 等
#include <sys/wait.h>   // 等待子进程：waitpid、WIFEXITED 等宏
#include <errno.h>      // 错误码变量 errno
```

每个头文件提供一组声明。比如 `pid_t` 类型在 `<sys/types.h>` 里定义（本质是 `int` 的 typedef），`sigaction` 结构体在 `<signal.h>` 里。

------

## 二、宏定义（#define）

```
#define MAXLINE 1024
#define MAXARGS 128
#define MAXJOBS 16
#define MAXJID 1<<16
```

`#define` 是**预处理宏**，编译前做纯文本替换。`MAXJID 1<<16` 会被替换成 `1<<16`，即 `65536`。注意宏没有类型检查，就是字面替换。

```
#define UNDEF 0
#define FG 1
#define BG 2
#define ST 3
```

这四个是作业状态的枚举式常量。用宏而不是 `enum` 是 CSAPP 代码的老风格。

------

## 三、全局变量

```
extern char **environ;
```

`environ` 是 C 库（libc）中定义的全局变量，指向环境变量字符串数组。每个元素形如 `"PATH=/usr/bin"`，以 `NULL` 结尾。`extern` 表示"这个变量在别处定义，这里只是声明"。`execve` 系列函数会用到它来传递环境。

```
char prompt[] = "tsh> ";
int verbose = 0;
int nextjid = 1;
char sbuf[MAXLINE];
```

- `prompt[]`：字符数组，初始化为 `"tsh> "`（末尾自动加 `\0`）。
- `verbose`：标志位，`-v` 参数开启后打印额外诊断信息。
- `nextjid`：下一个要分配的作业 ID，从 1 开始。
- `sbuf`：通用字符串拼接缓冲区。

### struct job_t

```
struct job_t {
    pid_t pid;          // 进程 ID
    int jid;            // 作业 ID（shell 自己分配的编号）
    int state;          // UNDEF / FG / BG / ST
    char cmdline[MAXLINE]; // 对应的命令行字符串
};
struct job_t jobs[MAXJOBS]; // 长度 16 的作业数组
```

`pid_t` 是 `int` 的 typedef，专门表示进程 ID。`jobs` 是全局数组，所有函数共享同一份作业表。

------

## 四、main 函数逐行讲解

### 4.1 dup2 —— 重定向文件描述符

```
dup2(1, 2);
```

**函数原型**：`int dup2(int oldfd, int newfd);`

**原理**：Linux 中每个进程有一张文件描述符表，下标 0/1/2 约定为 stdin/stdout/stderr。`dup2(1, 2)` 让描述符 2（stderr）指向描述符 1（stdout）所指向的同一个文件/管道。之后写入 stderr 的内容会出现在 stdout 上。

**返回值**：成功返回新的描述符（这里是 2），失败返回 -1 并设置 `errno`。

**为什么这么做**：自动测试驱动程序（sdriver.pl）通过管道只捕获 stdout，把 stderr 合并过去就能拿到全部输出。

### 4.2 getopt —— 解析命令行参数

```
while ((c = getopt(argc, argv, "hvp")) != EOF) {
```

**函数原型**：`int getopt(int argc, char *const argv[], const char *optstring);`

**原理**：遍历 `argv`，按 `optstring` 中声明的选项逐个解析。`"hvp"` 表示支持 `-h`、`-v`、`-p` 三个不带参数的选项。每调用一次返回一个选项字符；全部解析完返回 `-1`（即 `EOF`）。

**全局变量**：

- `optarg`：如果选项带参数（如 `"o:"` 中冒号表示带参），指向参数字符串。
- `optind`：下一个待处理参数的索引。

**返回值**：选项字符（如 `'h'`），解析完返回 `-1`，遇到未知选项返回 `'?'`。

```
case 'h': usage();    break;
case 'v': verbose = 1; break;
case 'p': emit_prompt = 0; break;
```

`usage()` 打印帮助并 `exit(1)`。注意 `exit(1)` 直接终止整个进程，不会返回。

### 4.3 Signal —— 安装信号处理器（自定义封装）

```
Signal(SIGINT,  sigint_handler);
Signal(SIGTSTP, sigtstp_handler);
Signal(SIGCHLD, sigchld_handler);
Signal(SIGQUIT, sigquit_handler);
```

这是自定义函数，底层调用 `sigaction`。下面在"其他辅助例程"一节详细讲。这里先理解：它告诉内核"当收到 XX 信号时，调用 XX 函数"。

**信号常量含义**：

| 常量      | 编号 | 触发方式         | 默认行为         |
| --------- | ---- | ---------------- | ---------------- |
| `SIGINT`  | 2    | Ctrl-C           | 终止进程         |
| `SIGTSTP` | 20   | Ctrl-Z           | 停止（挂起）进程 |
| `SIGCHLD` | 17   | 子进程终止或停止 | 忽略             |
| `SIGQUIT` | 3    | Ctrl-\           | 终止并转储核心   |

### 4.4 主循环

```
while (1) {
    if (emit_prompt) {
        printf("%s", prompt);
        fflush(stdout);
    }
```

**printf**：`int printf(const char *format, ...);` 格式化输出到 stdout。这里 `%s` 输出字符串 `"tsh> "`。

**fflush**：`int fflush(FILE *stream);` 强制把流缓冲区里的数据立即写出。printf 默认是行缓冲（遇到 `\n` 才刷新），但 prompt 没有换行，所以必须手动 `fflush` 才能立刻显示。参数 `stdout` 是标准输出流指针。返回 0 成功，`EOF` 失败。

```
if ((fgets(cmdline, MAXLINE, stdin) == NULL) && ferror(stdin))
    app_error("fgets error");
```

**fgets**：`char *fgets(char *s, int size, FILE *stream);`

- 从 `stream` 读最多 `size-1` 个字符到 `s`，末尾自动加 `\0`。
- 遇到换行符 `\n` 也停止，并把 `\n` 存入缓冲区。
- **返回值**：成功返回 `s`（即缓冲区指针）；读到文件末尾（EOF）或出错返回 `NULL`。

**ferror**：`int ferror(FILE *stream);`

- 测试流的**错误标志位**。如果之前的 I/O 操作出错，返回非零；否则返回 0。
- 注意：fgets 返回 NULL 有两种可能——读到 EOF，或者出错。`ferror` 用来区分这两种情况。
- 这行代码的逻辑：fgets 返回 NULL **并且** ferror 为真 → 真的出错了，调用 `app_error`。如果只是 EOF，ferror 返回 0，不进入这个 if。

```
if (feof(stdin)) {
    fflush(stdout);
    exit(0);
}
```

**feof**：`int feof(FILE *stream);`

- 测试流的 **EOF 标志位**。如果之前的读操作遇到了文件结束，返回非零；否则返回 0。
- 当用户按 Ctrl-D 时，stdin 到达 EOF，fgets 返回 NULL，feof 返回真，shell 干净退出。

**ferror vs feof 的区别**：

- `ferror`：检查是否**出错**（如设备故障、读权限问题）。
- `feof`：检查是否到达**文件末尾**（正常结束）。
- 两者检查的是流结构体中不同的标志位，互不冲突。一个 fgets 返回 NULL 后，必须用这两个函数判断原因。

**exit**：`void exit(int status);` 立即终止当前进程，`status` 是退出码（0 表示成功，非 0 表示异常）。退出前会刷新所有流缓冲区、调用 `atexit` 注册的函数。

```
eval(cmdline);
fflush(stdout);
fflush(stdout);
```

调用 `eval` 解释命令行（这是你要实现的函数，骨架里只有 `return;`）。后面两次 `fflush` 是原版代码的冗余写法，确保输出全部刷出。

------

## 五、parseline

函数原型：

```c
int parseline(const char *cmdline, char **argv)
```

- `cmdline`：输入命令行，通常由 `fgets` 读入，末尾带 `\n`。
- `argv`：输出参数数组，函数把每个参数的指针放进去，最后以 `NULL` 结尾。
- 返回值：`0` 表示前台运行，`1` 表示后台运行；但空行也返回 `1`，所以调用者通常还要看 `argv[0]` 是否为 `NULL`。

---

## 1. 变量说明

```c
static char array[MAXLINE];
char *buf = array;
char *delim;
int argc;
int bg;
```

- `array`：保存 `cmdline` 的副本。必须是 `static`，因为 `argv` 中的指针会指向它，函数返回后不能失效。
- `buf`：遍历命令行的指针。
- `delim`：指向当前参数结束的分隔符。
- `argc`：参数个数。
- `bg`：是否为后台任务。

注意：`static` 导致这个函数不可重入，多次调用会覆盖上一次的解析结果。

---

## 2. 复制并处理末尾换行

```c
strcpy(buf, cmdline);
buf[strlen(buf)-1] = ' ';
```

先把输入复制到静态数组 `array`。  
然后假设最后一个字符是 `'\n'`，把它替换成空格，方便后面统一按空格分割。

例如：

```text
"ls -l\n"
```

变成：

```text
"ls -l "
```

这里代码假设 `cmdline` 非空且以 `\n` 结尾。如果传入空字符串，`strlen(buf)-1` 会下溢，造成越界写，这是潜在问题。

---

## 3. 跳过前导空格

```c
while (*buf && (*buf == ' '))
    buf++;
```

如果命令开头有多个空格，直接跳过，让 `buf` 指向第一个有效字符。

---

## 4. 开始构建 argv

```c
argc = 0;
if (*buf == '\'') {
    buf++;
    delim = strchr(buf, '\'');
}
else {
    delim = strchr(buf, ' ');
}
```

这里判断第一个参数是否以单引号 `'` 开头。

- 如果以单引号开头，说明这个参数内部可能包含空格，例如 `'my file'`。  
  所以跳过起始单引号，然后寻找下一个单引号作为参数结束位置。
- 否则，普通参数按空格结束，寻找下一个空格。

`strchr` 用来查找字符第一次出现的位置。

---

## 5. 主循环：逐个切分参数

```c
while (delim) {
    argv[argc++] = buf;
    *delim = '\0';
    buf = delim + 1;
    while (*buf && (*buf == ' '))
        buf++;

    if (*buf == '\'') {
        buf++;
        delim = strchr(buf, '\'');
        // strchr是找到从第一个参数往后的第一个对应char
    }
    else {
        delim = strchr(buf, ' ');
    }
}
```

每次循环完成一个参数的切分：

1. `argv[argc++] = buf;`  
   当前参数从 `buf` 开始。

2. `*delim = '\0';`  
   把分隔符位置改成字符串结束符，这样当前参数就被截断成一个独立 C 字符串。

3. `buf = delim + 1;`  
   `buf` 移到分隔符后面，准备处理下一个参数。

4. 跳过连续空格。

5. 判断下一个参数是否以单引号开头：
   - 是，则跳过起始单引号，并寻找下一个单引号作为结束。
   - 否，则继续寻找空格作为结束。

循环直到找不到分隔符。

最后：

```c
argv[argc] = NULL;
```

让 `argv` 以 `NULL` 结尾，这是 `execvp` 等函数要求的格式。

---

## 6. 空行处理

```c
if (argc == 0)
    return 1;
```

如果输入是空行或只有空格，`argc` 仍为 0，直接返回 `1`，让上层忽略这一行。

---

## 7. 判断后台运行

```c
if ((bg = (*argv[argc-1] == '&')) != 0) {
    argv[--argc] = NULL;
}
return bg;
```

检查最后一个参数是否以 `&` 开头。

例如：

```text
ls -l &
```

解析后 `argv` 可能是：

```c
{"ls", "-l", "&", NULL}
```

最后一个参数是 `"&"`，于是：

- `bg = 1`
- `argc--`，去掉 `&`
- `argv[argc] = NULL`，重新设置结尾

最终变成：

```c
{"ls", "-l", NULL}
```

返回 `bg`，即 `1`，表示后台运行。

如果最后一个参数不是 `&` 开头，则 `bg = 0`，返回 `0`，表示前台运行。

---

## 8. 示例

输入：

```text
ls -l 'my file' &
```

处理过程大致为：

1. 复制并去掉换行。
2. 解析出 `ls`。
3. 解析出 `-l`。
4. 遇到单引号，解析出 `my file`，空格保留。
5. 解析出 `&`。
6. 发现最后一个参数是 `&`，删除它，设置 `bg = 1`。

最终：

```c
argv = {"ls", "-l", "my file", NULL}
```

返回值：

```c
1
```

表示后台运行。

---

## 9. 这个实现的局限和注意点

1. **只支持单引号**，不支持双引号、转义、变量替换、管道、重定向等完整 shell 语法。
2. 单引号只支持参数开头，例如 `'hello world'`。不支持 `ab'cd'` 这种拼接。
3. 如果单引号没有闭合，后面的内容可能被丢失。
4. 只按空格分割，不处理制表符 `\t`。
5. `strcpy` 不检查长度，输入超过 `MAXLINE` 会缓冲区溢出。
6. 假设输入以 `\n` 结尾，否则会错误覆盖最后一个字符。
7. `static char array` 导致结果在下一次调用 `parseline` 时被覆盖，也不能用于多线程。
8. `&` 判断只看最后一个参数的第一个字符，所以 `&foo` 也会被误认为后台标志并被删除。

总的来说，这是一个用于教学 shell 的简化解析函数，核心功能是：**按空格分词、支持简单单引号保留空格、识别末尾 `&` 后台运行**。

### 5.4 指针操作细节

```
argv[argc++] = buf;
*delim = '\0';
buf = delim + 1;
```

- `argv[argc++] = buf`：把当前参数的起始指针存入 argv 数组。
- `*delim = '\0'`：在分隔符位置写入字符串结束符，这样 `buf` 就被"截断"成一个独立字符串。这是 C 字符串处理的经典技巧——**不复制内存，只靠插入 `\0` 来切分**。
- `buf = delim + 1`：指针移动到分隔符下一个字符，继续处理。

```
argv[argc] = NULL;
```

argv 数组以 `NULL` 结尾，这是 exec 系列函数的约定。

```
if ((bg = (*argv[argc-1] == '&')) != 0) {
    argv[--argc] = NULL;
}
```

- `*argv[argc-1]`：取最后一个参数的**第一个字符**。如果是 `'&'`，说明是后台作业。
- `bg = (...)`：把比较结果（0 或 1）赋给 bg。
- `argv[--argc] = NULL`：先把 argc 减 1，再把那个位置设为 NULL，相当于从 argv 中移除 `&`。

------

## 六、作业列表辅助函数（全部已实现）

### 6.1 clearjob

```
void clearjob(struct job_t *job) {
    job->pid = 0;
    job->jid = 0;
    job->state = UNDEF;
    job->cmdline[0] = '\0';
}
```

**作用**：把一个作业结构体重置为"空"状态。`pid = 0` 是关键标记——作业表中用 `pid == 0` 表示空槽位。`cmdline[0] = '\0'` 把字符串变成空串（不需要清空整个数组，第一个字符为 `\0` 就够了）。

**`->` 运算符**：`job->pid` 等价于 `(*job).pid`，通过结构体指针访问成员。

### 6.2 initjobs

```
void initjobs(struct job_t *jobs) {
    int i;
    for (i = 0; i < MAXJOBS; i++)
        clearjob(&jobs[i]);
}
```

**作用**：遍历整个作业数组，逐个清空。`&jobs[i]` 取第 i 个元素的地址传给 clearjob。程序启动时调用一次。

### 6.3 maxjid

```
int maxjid(struct job_t *jobs) {
    int i, max=0;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].jid > max)
            max = jobs[i].jid;
    return max;
}
```

**作用**：返回当前作业表中最大的 jid。删除作业后用来重置 `nextjid`（`nextjid = maxjid(jobs) + 1`），保证 jid 尽量紧凑不浪费。空表返回 0。

### 6.4 addjob

```
int addjob(struct job_t *jobs, pid_t pid, int state, char *cmdline) {
    int i;
    if (pid < 1) return 0;
    for (i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid == 0) {
            jobs[i].pid = pid;
            jobs[i].state = state;
            jobs[i].jid = nextjid++;
            if (nextjid > MAXJOBS) nextjid = 1;
            strcpy(jobs[i].cmdline, cmdline);
            if (verbose) {
                printf("Added job [%d] %d %s\n", jobs[i].jid, jobs[i].pid, jobs[i].cmdline);
            }
            return 1;
        }
    }
    printf("Tried to create too many jobs\n");
    return 0;
}
```

**作用**：把一个新作业加入作业表。

- `pid < 1` 直接拒绝（合法 PID 从 1 开始，0 有特殊含义）。
- 找第一个 `pid == 0` 的空槽位填入。
- `nextjid++`：先使用当前值作为 jid，再自增。这是**后缀自增**的经典用法。
- `if (nextjid > MAXJOBS) nextjid = 1`：jid 超过 16 就回绕到 1（环形复用）。注意这里比较的是 MAXJOBS(16) 而非 MAXJID(65536)，因为作业表最多 16 个槽。
- verbose 模式打印诊断信息。
- 表满了打印错误并返回 0。

**返回值**：成功 1，失败 0。

### 6.5 deletejob

```
int deletejob(struct job_t *jobs, pid_t pid) {
    int i;
    if (pid < 1) return 0;
    for (i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid == pid) {
            clearjob(&jobs[i]);
            nextjid = maxjid(jobs)+1;
            return 1;
        }
    }
    return 0;
}
```

**作用**：按 PID 查找并删除作业。删除后调用 `clearjob` 清空槽位，并重置 `nextjid = maxjid(jobs) + 1`——这样下一个新作业的 jid 是当前最大 jid + 1，保持编号紧凑。

### 6.6 fgpid

```
pid_t fgpid(struct job_t *jobs) {
    int i;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].state == FG)
            return jobs[i].pid;
    return 0;
}
```

**作用**：返回当前前台作业的 PID。因为最多只有一个 FG 作业，找到就返回；没有则返回 0（PID 0 不是合法进程，用作"无前台"的哨兵值）。

### 6.7 getjobpid / getjobjid

```
struct job_t *getjobpid(struct job_t *jobs, pid_t pid) { ... }
struct job_t *getjobjid(struct job_t *jobs, int jid) { ... }
```

**作用**：分别按 PID 和 JID 查找作业，返回**指向该作业结构体的指针**（可以通过它修改作业状态）。找不到返回 `NULL`。这是两个几乎对称的函数，只是查找字段不同。

**返回指针而非拷贝**：返回指针意味着调用者可以直接修改 `job->state` 等字段，这是有意设计的。

### 6.8 pid2jid

```
int pid2jid(pid_t pid) {
    int i;
    if (pid < 1) return 0;
    for (i = 0; i < MAXJOBS; i++)
        if (jobs[i].pid == pid)
            return jobs[i].jid;
    return 0;
}
```

**作用**：PID → JID 的映射。注意这里直接访问全局变量 `jobs`（没有作为参数传入），和其他函数风格不一致，是原版代码如此。找不到返回 0。

### 6.9 listjobs

```
void listjobs(struct job_t *jobs) {
    int i;
    for (i = 0; i < MAXJOBS; i++) {
        if (jobs[i].pid != 0) {
            printf("[%d] (%d) ", jobs[i].jid, jobs[i].pid);
            switch (jobs[i].state) {
            case BG: printf("Running "); break;
            case FG: printf("Foreground "); break;
            case ST: printf("Stopped "); break;
            default: printf("listjobs: Internal error: job[%d].state=%d ", i, jobs[i].state);
            }
            printf("%s", jobs[i].cmdline);
        }
    }
}
```

**作用**：打印所有非空作业。输出格式：

```
[jid] (pid) Running    命令行
[jid] (pid) Stopped    命令行
```

**switch 语句**：根据 `state` 字段选择打印的状态字符串。每个 case 后必须 `break`，否则会"穿透"到下一个 case。`default` 处理意外状态值（防御性编程）。

**注意**：状态字符串后面都带空格（`"Running "`、`"Stopped "`），这是格式要求，和参考 shell 输出严格对齐。

------

## 七、其他辅助例程

### 7.1 usage

```
void usage(void) {
    printf("Usage: shell [-hvp]\n");
    printf("   -h   print this message\n");
    printf("   -v   print additional diagnostic information\n");
    printf("   -p   do not emit a command prompt\n");
    exit(1);
}
```

**作用**：打印用法说明，然后以退出码 1 终止。`void` 参数列表表示不接受任何参数（C 中 `()` 表示参数不确定，`(void)` 才明确表示无参）。

### 7.2 unix_error

```
void unix_error(char *msg) {
    fprintf(stdout, "%s: %s\n", msg, strerror(errno));
    exit(1);
}
```

**fprintf**：`int fprintf(FILE *stream, const char *format, ...);` 与 printf 类似，但可以指定输出流。这里输出到 stdout（因为之前 dup2 把 stderr 合并了）。

**strerror**：`char *strerror(int errnum);` 把错误码 `errnum` 翻译成人类可读的字符串。比如 `errno == 2` 时返回 `"No such file or directory"`。

**errno**：`<errno.h>` 中定义的全局整数变量。系统调用失败时会被设置为具体错误码。它是一个**线程局部存储**的变量（现代实现），每个线程独立。

**输出示例**：如果 fork 失败（errno = EAGAIN = 11），调用 `unix_error("fork error")` 会打印：

```
fork error: Resource temporarily unavailable
```

### 7.3 app_error

```
void app_error(char *msg) {
    fprintf(stdout, "%s\n", msg);
    exit(1);
}
```

**作用**：应用层错误，只打印消息不查 errno。比如 fgets 出错时调用。

### 7.4 Signal（重点）

```
handler_t *Signal(int signum, handler_t *handler) {
    struct sigaction action, old_action;
    action.sa_handler = handler;
    sigemptyset(&action.sa_mask);
    action.sa_flags = SA_RESTART;
    if (sigaction(signum, &action, &old_action) < 0)
        unix_error("Signal error");
    return (old_action.sa_handler);
}
```

这里 `sigaction(...)` 是函数调用，`sigaction` 是函数名，不是结构体

但你看到代码里又有：

```c
struct sigaction action, old_action;
```

这里 `struct sigaction` 才是**结构体类型**。

所以关键点是：

> **C 语言允许函数名和结构体标签同名。它们在不同的命名空间里，互不冲突。**

---

## 1. 两个 `sigaction` 分别是什么？

### 结构体类型

```c
struct sigaction action, old_action;
```

这里：

- `struct sigaction` 是结构体类型，名字叫 `sigaction`。
- `action` 和 `old_action` 是这个结构体类型的变量。
- 注意：必须写 `struct` 关键字，因为这里没有 `typedef`。

它大致长这样：

```c
struct sigaction {
    void (*sa_handler)(int);
    sigset_t sa_mask;
    int sa_flags;
    void (*sa_sigaction)(int, siginfo_t *, void *);
    void (*sa_restorer)(void);
};
```

### 函数

```c
if (sigaction(signum, &action, &old_action) < 0)
```

这里：

- `sigaction` 是一个**系统调用函数**。
- 它的原型在 `<signal.h>` 中，大概是：

```c
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```

- 第一个参数 `signum`：信号编号。
- 第二个参数 `&action`：指向新的 `struct sigaction`，表示要安装的新动作。
- 第三个参数 `&old_action`：指向旧的 `struct sigaction`，用来保存旧动作。
- 返回值是 `int`：成功返回 `0`，失败返回 `-1`。

所以：

```c
sigaction(signum, &action, &old_action) < 0
```

就是调用这个函数，判断返回值是否小于 0，即是否失败。

---

## 2. 为什么可以同名？

C 语言里有多个**命名空间**：

- 普通标识符命名空间：变量名、函数名、`typedef` 名等。
- 标签命名空间：`struct`、`union`、`enum` 的标签名。
- 成员命名空间：结构体成员名。
- 标签名等。

`struct sigaction` 中的 `sigaction` 是**结构体标签**，属于标签命名空间。  
函数 `sigaction` 是**函数名**，属于普通标识符命名空间。

所以它们可以同时存在，不会冲突。

编译器怎么区分？

- 看到 `struct sigaction`，它去标签命名空间找结构体。
- 看到 `sigaction(...)`，它去普通标识符命名空间找函数。

因此：

```c
struct sigaction action;          // 结构体类型
sigaction(SIGINT, &action, NULL); // 函数调用
```

完全合法。

---

## 3. 一句话总结

- `struct sigaction` 是**结构体类型**。
- `sigaction(...)` 是**函数调用**。
- 在这个 `if` 里，`sigaction` 是**函数**，返回 `int`，所以可以写 `< 0`。
- C 语言允许结构体标签和函数名同名，因为它们在不同的命名空间。

所以你的疑问可以这样记：

> **`struct sigaction` 是类型；`sigaction()` 是函数。名字一样，但不是一个东西。**



这是对 `sigaction` 系统调用的封装。逐行讲：

**typedef void handler_t(int);**：定义了一个函数指针类型 `handler_t`，表示"接受一个 int 参数、返回 void 的函数"。`handler_t *` 就是指向这种函数的指针。

**struct sigaction**：信号处理动作结构体，关键字段：

- `sa_handler`：信号处理函数指针（也可以是 `SIG_DFL` 默认行为或 `SIG_IGN` 忽略）。
- `sa_mask`：信号掩码集合，表示**在执行该处理函数期间，额外阻塞哪些信号**。
- `sa_flags`：标志位，控制行为细节。

**sigemptyset**：`int sigemptyset(sigset_t *set);` 把信号集 `set` 初始化为空集（不包含任何信号）。这里表示处理函数执行期间不额外阻塞其他信号（但当前正在处理的信号本身默认会被自动阻塞，除非设置 SA_NODEFER）。

**SA_RESTART**：标志位，表示如果信号打断了某个慢速系统调用（如 read、wait），内核会**自动重启**该系统调用而不是让它返回 EINTR 错误。这能简化很多代码。

**sigaction**：`int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);`

- `signum`：要设置的信号编号。
- `act`：新的处理动作。
- `oldact`：输出参数，返回旧的处理动作（可以用来恢复）。
- 返回 0 成功，-1 失败。

**返回旧处理器**：`return (old_action.sa_handler);` 返回之前的处理函数指针。这个设计允许临时替换信号处理器后再恢复。

### 7.5 sigquit_handler

```
void sigquit_handler(int sig) {
    printf("Terminating after receipt of SIGQUIT signal\n");
    exit(1);
}
```

**作用**：收到 SIGQUIT 信号时打印消息并退出。测试驱动程序用 SIGQUIT 来干净地终止 shell。参数 `sig` 是信号编号（这里一定是 SIGQUIT = 3），虽然没用到，但信号处理函数的签名必须是 `void handler(int)`。

------

## 八、关键概念补充

### 8.1 信号（Signal）基础

信号是 Unix 中进程间通信的一种机制，本质是**软件中断**。内核在以下时刻给进程递送信号：

- 键盘事件（Ctrl-C → SIGINT，Ctrl-Z → SIGTSTP）
- 子进程状态变化（→ SIGCHLD）
- 显式调用 `kill` 系统调用

信号处理的三种方式：

1. **默认行为**（`SIG_DFL`）：大多数信号是终止进程。
2. **忽略**（`SIG_IGN`）：信号被丢弃。
3. **捕获**：执行用户注册的处理函数。

**信号递送（delivery）与阻塞（blocking）**：信号产生后，内核先把它标记为"待处理（pending）"，在合适的时机递送给进程。如果该信号被阻塞，它会一直 pending，直到解除阻塞。`sigprocmask` 可以设置进程的信号掩码（阻塞哪些信号）。

**信号不排队**：同一种信号如果在阻塞期间产生多次，解除阻塞后通常只递送一次（POSIX 标准信号不排队）。这就是为什么处理函数里要用循环收完所有子进程。

### 8.2 进程组（Process Group）

每个进程属于一个进程组，有一个进程组 ID（pgid）。`setpgid(pid, pgid)` 可以设置进程的组 ID。`setpgid(0, 0)` 让调用进程创建一个新进程组，自己成为组长（pgid = 自己的 pid）。

**为什么重要**：终端的 Ctrl-C/Ctrl-Z 会把信号发送给**前台进程组**的所有进程。如果 shell 的子进程和 shell 在同一个组，按 Ctrl-C 会同时杀掉 shell 和子进程。让每个子进程独立成组，shell 就能选择性地把信号转发给前台作业。

`kill(-pid, sig)` 中负号表示把信号发给 pid 所在的**整个进程组**，而不是单个进程。

### 8.3 僵尸进程（Zombie）

子进程终止后，内核保留它的退出状态和一些资源，等待父进程来"收尸"（reap）。在父进程调用 `wait` 或 `waitpid` 之前，这个已终止但未被回收的子进程就是僵尸进程。

**waitpid**：`pid_t waitpid(pid_t pid, int *status, int options);`

- `pid`：等待哪个子进程。`-1` 表示任意子进程。
- `status`：输出参数，存放子进程的退出状态信息。
- `options`：`WNOHANG`（不阻塞，没有就返回 0）、`WUNTRACED`（也报告已停止的子进程）。
- 返回值：>0 是回收的子进程 PID；0 表示 WNOHANG 下没有可回收的；-1 表示出错（如没有子进程，errno = ECHILD）。

**status 解析宏**（在 `<sys/wait.h>` 中）：

- `WIFEXITED(status)`：子进程正常退出（调用 exit 或从 main return）→ 非零。
- `WEXITSTATUS(status)`：正常退出时的退出码（仅当 WIFEXITED 为真时有效）。
- `WIFSIGNALED(status)`：子进程被信号终止 → 非零。
- `WTERMSIG(status)`：终止子进程的信号编号。
- `WIFSTOPPED(status)`：子进程被停止（未终止）→ 非零。
- `WSTOPSIG(status)`：导致停止的信号编号。

### 8.4 fork 与 exec

**fork**：`pid_t fork(void);` 创建子进程。调用一次返回两次：

- 父进程中返回子进程的 PID（>0）。
- 子进程中返回 0。
- 失败返回 -1。

子进程获得父进程地址空间的**拷贝**（写时复制 COW），包括文件描述符表、信号处理器设置等。

**execve**：`int execve(const char *filename, char *const argv[], char *const envp[]);` 用新程序替换当前进程的地址空间。成功后不返回（原来的代码被覆盖）；失败返回 -1。`execvp` 是它的封装，会自动在 PATH 中查找可执行文件。

fork + exec 是 Unix 创建新程序的标准模式：先 fork 出子进程，再在子进程里 exec 新程序。

### 8.5 文件描述符与 stdin/stdout/stderr

每个进程默认打开三个描述符：

- 0 → stdin（标准输入）
- 1 → stdout（标准输出）
- 2 → stderr（标准错误）

它们都指向终端设备（或被重定向到管道/文件）。`dup2` 可以复制描述符，实现重定向。

### 8.6 缓冲区与 fflush

C 标准库的 FILE 流有缓冲区：

- **全缓冲**：缓冲区满才刷新（普通文件默认）。
- **行缓冲**：遇到换行符刷新（终端 stdout 默认）。
- **无缓冲**：立即写入（stderr 默认）。

当 stdout 被重定向到管道时，它变成**全缓冲**，这就是为什么代码里频繁 `fflush(stdout)`——确保输出立即到达管道另一端的测试驱动程序。

------

## 九、函数指针与 Signal 的类型

```
typedef void handler_t(int);
handler_t *Signal(int signum, handler_t *handler);
```

`typedef void handler_t(int);` 定义了一个**函数类型**（不是函数指针类型）。`handler_t *` 才是函数指针。这种写法比直接写 `void (*handler)(int)` 更清晰。

Signal 的参数和返回值都是 `handler_t *`，即"指向返回 void、接受 int 的函数的指针"。这是 C 中处理回调函数的标准模式。

------

## 十、代码中值得注意的 C 语言细节

1. **`static char array[MAXLINE]`**：parseline 中的 `array` 是 static 局部变量，存储在全局数据区而非栈上，函数返回后内容保留。这就是为什么 argv 中的指针在 parseline 返回后仍然有效——它们指向这个 static 数组。但副作用是：**再次调用 parseline 会覆盖上一次的内容**。
2. **`buf[strlen(buf)-1] = ' '` 的潜在风险**：如果 cmdline 是空字符串（`buf[0] == '\0'`），`strlen` 返回 0，`buf[-1]` 是越界访问。但实际中 fgets 至少会读入 `\n`，所以不会出现空串。这是依赖输入约定的写法。
3. **`extern char \**environ` vs `envp`**：main 的第三个参数 `envp` 和全局 `environ` 指向同一个环境变量数组。用 `environ` 更方便（不需要在函数间传递 envp）。
4. **`exit(0)` vs `return 0`**：在 main 中两者效果类似（return 会隐式调用 exit），但在其他函数中 `exit` 直接终止整个进程，`return` 只是从当前函数返回。
5. **`SA_RESTART 的局限**：不是所有系统调用都能被自动重启，比如 `sleep`、`pause`、某些 socket 操作不会被重启。但本 lab 中主要涉及的是 fgets/waitpid，SA_RESTART 足够用。
6. **信号处理函数中的 printf**：严格来说，`printf` 不是异步信号安全的（async-signal-safe）函数，在信号处理器中调用理论上可能导致死锁或数据损坏。但 CSAPP lab 的测试场景简单，教材示例也这么写，可以接受。生产代码中应该用 `write` 系统调用代替。

------

以上就是骨架代码中所有已给出部分的详细讲解。如果你对某个具体函数或概念还想再深入，随时问我。