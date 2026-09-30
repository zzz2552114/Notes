不会。`fork` 之后，父子进程有**各自独立的信号掩码副本**，不是共享的。

- 父进程调用 `sigprocmask(SIG_BLOCK, ...)` 阻塞 `SIGCHLD`，改变的是**父进程自己的掩码**。
- 子进程继承的是父进程当时掩码的**副本**。子进程之后调用 `sigprocmask` 解除阻塞，只改变**子进程自己的掩码**。
- 父进程的掩码不会因为子进程的操作而改变。

更关键的是：`SIGCHLD` 是内核发给**父进程**的，不是发给子进程的。父进程是否收到、是否立即递送 `SIGCHLD`，只取决于**父进程自己**有没有阻塞 `SIGCHLD`。

所以：

```c
// 父进程
sigprocmask(SIG_BLOCK, &mask, &old);
pid = fork();
if (pid == 0) {
    // 子进程解除阻塞：只影响子进程自己
    sigprocmask(SIG_SETMASK, &old, NULL);
    execve(...);
}
// 父进程仍然阻塞着 SIGCHLD
addjob(...);
// 父进程自己解除阻塞，这时才可能处理 SIGCHLD
sigprocmask(SIG_SETMASK, &old, NULL);
```

子进程先解除还是后解除，都不会让父进程提前收到 `SIGCHLD`。父进程的 `addjob` 临界区仍然安全。子进程之所以要解除，是为了自己 `execve` 后新程序不被继承的阻塞掩码影响。