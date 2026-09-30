下面按 **参数、返回值、status 解析、options、典型用法、坑** 详细展开。

## 1. 函数原型与基本语义

```c
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *status, int options);
```

作用：等待某个/某类子进程发生状态变化，并回收其终止状态。

一次 `waitpid` 只处理一个状态变化。如果有多个子进程退出，要循环调用。

`wait(&status)` 基本等价于：

```c
waitpid(-1, &status, 0);
```

但 `waitpid` 更灵活：可以指定 PID、进程组、非阻塞、报告停止/继续。

---

## 2. `pid` 参数：决定“等谁”

这是 `waitpid` 最核心的参数。

| `pid` 取值 | 等待对象                               |
| ---------- | -------------------------------------- |
| `> 0`      | 等待 PID 等于 `pid` 的那个子进程       |
| `0`        | 等待与调用进程同一个进程组的任意子进程 |
| `-1`       | 等待任意子进程，等价于 `wait()`        |
| `< -1`     | 等待进程组 ID 等于 `-pid` 的任意子进程 |

注意几个坑：

- `pid == -1` 不是“进程组 1”，而是“任意子进程”。
- `pid == 0` 不是等待 PID 为 0 的进程，而是等待同进程组的子进程。
- `pid < -1` 时，例如 `waitpid(-1234, ...)`，表示等待进程组 ID 为 `1234` 的任意子进程。
- 只能等待调用者的直接子进程，不能等待任意进程，也不能等待孙子进程。
- 如果指定 `pid > 0`，但那个进程不是你的子进程，或者已经被回收，会返回 `-1`，`errno = ECHILD`。

例如：

```c
pid_t child = fork();
if (child == 0) {
    _exit(0);
}

int status;
pid_t ret = waitpid(child, &status, 0);  // 精准等待这个子进程
```

---

## 3. `status` 参数：不能直接当退出码用

`status` 是一个输出参数，内核把子进程状态写进去。

可以传 `NULL`，表示不关心状态，只回收：

```c
waitpid(child, NULL, 0);
```

如果传了非空指针，必须用宏解析，不能直接比较：

```c
if (WIFEXITED(status)) {
    printf("正常退出，退出码 = %d\n", WEXITSTATUS(status));
}
```

常见宏：

| 宏                     | 含义                    | 配套宏                                  |
| ---------------------- | ----------------------- | --------------------------------------- |
| `WIFEXITED(status)`    | 子进程正常退出          | `WEXITSTATUS(status)`                   |
| `WIFSIGNALED(status)`  | 子进程被信号终止        | `WTERMSIG(status)`、`WCOREDUMP(status)` |
| `WIFSTOPPED(status)`   | 子进程被暂停            | `WSTOPSIG(status)`                      |
| `WIFCONTINUED(status)` | 子进程被 `SIGCONT` 恢复 | 无                                      |

解析模板：

```c
if (WIFEXITED(status)) {
    printf("exited, code = %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("killed by signal %d\n", WTERMSIG(status));
#ifdef WCOREDUMP
    if (WCOREDUMP(status)) {
        printf("core dumped\n");
    }
#endif
} else if (WIFSTOPPED(status)) {
    printf("stopped by signal %d\n", WSTOPSIG(status));
} else if (WIFCONTINUED(status)) {
    printf("continued\n");
}
```

注意：

- `WEXITSTATUS` 只在 `WIFEXITED` 为真时有效。
- `WTERMSIG` 只在 `WIFSIGNALED` 为真时有效。
- `WSTOPSIG` 只在 `WIFSTOPPED` 为真时有效。
- `WCOREDUMP` 不是 POSIX 标准，但 Linux/BSD 通常可用，最好条件编译。
- 退出码只有低 8 位有效，`exit(256)` 实际看到的是 0。
- 不要写 `if (status == 3)`，因为正常退出码 3 时，`status` 通常不是 3，而是类似 `0x0300`。

---

## 4. `options` 参数：控制阻塞与报告范围

`options` 是位掩码，可以组合。

| 选项         | 含义                                           |
| ------------ | ---------------------------------------------- |
| `0`          | 阻塞等待，只报告子进程终止                     |
| `WNOHANG`    | 非阻塞，如果没有可报告的子进程状态，立即返回 0 |
| `WUNTRACED`  | 也报告被暂停的子进程                           |
| `WCONTINUED` | 也报告被 `SIGCONT` 恢复的子进程，Linux 2.6.10+ |

可以组合：

```c
waitpid(pid, &status, WNOHANG | WUNTRACED | WCONTINUED);
```

### `WNOHANG`

非阻塞模式：

```c
pid_t ret = waitpid(child, &status, WNOHANG);

if (ret == 0) {
    // 子进程还没退出/没有状态变化
} else if (ret == child) {
    // 子进程已经退出并被回收
} else if (ret == -1) {
    // 出错
}
```

`ret == 0` 只可能出现在 `WNOHANG` 模式。阻塞模式不会返回 0。

### `WUNTRACED`

默认情况下，子进程被 `SIGSTOP`、`SIGTSTP` 等暂停，`waitpid` 不会返回。加上 `WUNTRACED` 后，暂停也会返回：

```c
pid_t ret = waitpid(child, &status, WUNTRACED);
if (WIFSTOPPED(status)) {
    printf("stopped by signal %d\n", WSTOPSIG(status));
}
```

注意：停止不是终止，子进程还在，只是暂停了。

### `WCONTINUED`

子进程被 `SIGCONT` 恢复时也报告：

```c
pid_t ret = waitpid(child, &status, WCONTINUED);
if (WIFCONTINUED(status)) {
    printf("child continued\n");
}
```

### Linux 扩展

Linux 还有一些内部扩展，如 `__WNOTHREAD`、`__WALL`、`__WCLONE`，可移植性差，普通应用不建议使用。

---

## 5. 返回值与错误

```c
pid_t waitpid(pid_t pid, int *status, int options);
```

返回值：

| 返回值 | 含义                                     |
| ------ | ---------------------------------------- |
| `> 0`  | 成功，返回发生状态变化的子进程 PID       |
| `0`    | 仅当 `WNOHANG`，表示没有可报告的状态变化 |
| `-1`   | 出错，设置 `errno`                       |

常见 `errno`：

| `errno`  | 含义                                                        |
| -------- | ----------------------------------------------------------- |
| `ECHILD` | 没有符合条件的子进程，可能已全部回收，或指定 PID 不是子进程 |
| `EINTR`  | 被信号中断                                                  |
| `EINVAL` | `options` 非法                                              |

`EINTR` 处理模板：

```c
pid_t ret;
do {
    ret = waitpid(pid, &status, 0);
} while (ret == -1 && errno == EINTR);
```

---

## 6. 典型用法

### 6.1 等待指定子进程

```c
int status;
pid_t ret;

do {
    ret = waitpid(child_pid, &status, 0);
} while (ret == -1 && errno == EINTR);

if (ret == child_pid) {
    if (WIFEXITED(status)) {
        printf("child exit code = %d\n", WEXITSTATUS(status));
    }
}
```

如果该子进程已经退出但还没被回收，`waitpid` 会立即返回。如果它还没退出，父进程会阻塞，直到它退出。

### 6.2 等待任意子进程

```c
int status;
pid_t pid;

while ((pid = waitpid(-1, &status, 0)) > 0) {
    printf("reaped child %d\n", pid);
}

if (errno == ECHILD) {
    printf("all children reaped\n");
}
```

这是“谁先死先收谁”。

### 6.3 非阻塞回收所有已退出子进程

```c
int status;
pid_t pid;

for (;;) {
    pid = waitpid(-1, &status, WNOHANG);

    if (pid > 0) {
        // 回收了一个子进程
    } else if (pid == 0) {
        // 还有子进程，但都没有状态变化
        break;
    } else {
        if (errno == ECHILD) {
            // 没有子进程了
            break;
        }
        if (errno == EINTR) {
            continue;
        }
        perror("waitpid");
        break;
    }
}
```

这种模式常用于事件循环、服务器主循环，但不要纯忙轮询，通常配合 `SIGCHLD`、`signalfd`、`epoll` 等。

### 6.4 按指定 PID 顺序收尸

你课件里的模式：

```c
pid_t pids[N];

for (int i = 0; i < N; i++) {
    pid_t pid = fork();
    if (pid == 0) {
        // 子进程做事
        _exit(i);
    }
    pids[i] = pid;
}

for (int i = N - 1; i >= 0; i--) {
    int status;
    pid_t ret;

    do {
        ret = waitpid(pids[i], &status, 0);
    } while (ret == -1 && errno == EINTR);

    if (ret == pids[i]) {
        if (WIFEXITED(status)) {
            printf("child %d exit code = %d\n",
                   pids[i], WEXITSTATUS(status));
        }
    }
}
```

这个写法能精准等待每一个子进程，但要注意：

- `waitpid(pids[i], ..., 0)` 会阻塞，直到 `pids[i]` 这个子进程退出。
- 如果 `pids[i]` 一直不退出，父进程会卡在它上面。
- 倒序只是“收尸顺序”，不是必须。它不会改变子进程实际退出顺序。
- 如果某个子进程已经退出但还没回收，它会变成僵尸，状态仍然保留，轮到它时 `waitpid` 会立即返回。
- 如果想“谁先退出先回收”，用 `waitpid(-1, ...)` 更合适。

### 6.5 等待同进程组子进程

如果多个子进程加入了同一个进程组：

```c
waitpid(-pgid, &status, 0);
```

表示等待进程组 ID 为 `pgid` 的任意子进程。

### 6.6 等待同组任意子进程

```c
waitpid(0, &status, 0);
```

表示等待与调用进程同一个进程组的任意子进程。

---

## 7. 与 `wait` 的区别

| 对比项        | `wait`          | `waitpid`                      |
| ------------- | --------------- | ------------------------------ |
| 指定 PID      | 不支持          | 支持                           |
| 指定进程组    | 不支持          | 支持                           |
| 非阻塞        | 不支持          | 支持 `WNOHANG`                 |
| 报告暂停/继续 | 不支持          | 支持 `WUNTRACED`、`WCONTINUED` |
| 等价关系      | `wait(&status)` | `waitpid(-1, &status, 0)`      |

---

## 8. 关键注意点

1. `waitpid` 一次只回收一个子进程状态，多个子进程要循环。
2. `status` 必须用宏解析，不能直接当退出码。
3. `WNOHANG` 返回 0 不是错误，表示“目前没有状态变化”。
4. `ECHILD` 表示没有符合条件的子进程，可能已经全部回收。
5. `EINTR` 要循环重试。
6. 子进程退出后、被 `wait` 前是僵尸进程，占 PID 和进程表项。
7. `WUNTRACED` 报告的“停止”不是终止，子进程还在。
8. `pid = -1` 是任意子进程，`pid = 0` 是同进程组，`pid < -1` 是进程组 ID 为 `-pid`。
9. 高性能服务器中，`WNOHANG` 可以用，但通常配合 `SIGCHLD`、`signalfd`、`pidfd` 等，不要纯忙轮询。
10. 如果子进程已经被其他地方 `wait` 回收，再 `waitpid` 同一个 PID 会返回 `-1`，`errno = ECHILD`。

一句话总结：**`pid` 决定等谁，`status` 用宏解析，`options` 决定阻塞还是非阻塞、是否报告停止/继续；`waitpid` 一次收一个，循环才能收完所有子进程。**