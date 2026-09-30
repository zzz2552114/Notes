Q:

那我eval里，如果是前台工作，那我不解除信号屏蔽，在sugsuspend里面弄，如果是后台工作我就解除，不也行吗？哦是不是因为，后台运行的时候可以同时布置前台进程，这个时候解除还是不解除就会出问题。 第二个问题，sigsuspend放在while里面，是不是为了防止除了sigchld以外的东西无意间唤醒sigsuspend？

A:

## 问题一：前台不解除屏蔽、后台解除，行不行？

**行，这其实就是课本 8.5.7 的正规写法之一。** 你抓到的关键点是对的：掩码是进程级的，不是"只对前台子进程生效"。

完整时序应该是这样（你脑子里那版）：

```
eval:
  block SIGCHLD（存 prev）
  fork
  parent: addjob
  if (bg)  { unblock(prev); 打印 "[1] 1234 &" }
  else     { waitfg(pid); unblock(prev); }   // 前台：waitfg 期间保持阻塞
```

waitfg 里：

```
while (fgpid(jobs) == pid)
    sigsuspend(&empty);   // empty = 全解阻塞
```

这里**为什么传 empty 也不会 lost-wakeup**：因为进入 while 做 `fgpid(jobs)==pid` 判断时，SIGCHLD 仍然是被 eval 挡住的。子进程此刻退出，SIGCHLD 只是 pending；等 `sigsuspend(&empty)` 这一步原子地把掩码换成空、去入睡时，那个 pending 的信号立刻递达，handler 跑完，sigsuspend 返回。判断和入睡之间没有缝隙。这正是"先 block、再用 sigsuspend 原子放开"的精髓。

**你自己猜的那个原因——后台和前台并发——确实是关键，但要把它说精确：**

- 在前台 waitfg 期间，**后台子进程完全可能同时退出**。它的 SIGCHLD 因为你挡着而 pending，等 sigsuspend 开闸那一刻，handler 一次 `waitpid` 循环把前台、后台的僵尸全部收掉。这本身没问题。
- **真正的坑是：waitfg 返回之后，eval 必须把掩码恢复（unblock(prev)）。** 如果你只在后台分支里 unblock、忘了前台分支 waitfg 之后也要 unblock，那从这个前台命令之后，shell 就永远挡着 SIGCHLD 了——之后所有后台子进程退出都不会被 reap，堆成僵尸。这就是"解除还是不解除会出问题"的精确含义：不是前后台冲突，是**前台路径容易漏掉收尾的那一下 unblock**。

所以两种合法写法并存：

1. **你这版**：eval 里前台不 unblock，waitfg 用 empty-mask sigsuspend，waitfg 返回后再补一次 unblock。waitfg 代码少，但有个容易忘的收尾。
2. **更常见的一版**：eval 里**前后台都**在 addjob 后立刻 unblock，然后 waitfg 自己内部重新 `sigprocmask(SIG_BLOCK, &chld, &prev)`、再 `sigsuspend(&prev)`、恢复。掩码状态干净，但 waitfg 多几行。

writeup 为什么都不推，而是推 sleep 轮询？因为上面这两版**只要掩码窗口写错一个字节就静默 hang**（比如你 eval 挡着、waitfg 里又挡一次，那 `prev` 里已经带着 SIGCHLD 被挡，你把 `prev` 传给 sigsuspend 就等于永远不开闸）。在一个已经够绕的 lab 里，它选了不依赖你把这两处窗口同时写对的版本。

## 问题二：while 套 sigsuspend，是不是为了防非 SIGCHLD 信号误唤醒？

方向对了一半，但机制要说清楚：**你"防不住"误唤醒，sigsuspend 一旦被任何已捕获信号打断就返回 EINTR，它不告诉你为什么返回。** while 不是过滤器，是**事后复查**。

具体到这个 lab，sigsuspend 返回了，但前台 job 还在跑的情况至少有三种：

1. **退出的是后台子进程，不是前台那个。** 后台 child 退出 → SIGCHLD → handler 跑 → reap 掉后台那个 → sigsuspend 返回。但 `fgpid(jobs)` 还是你在等的前台 pid，循环必须继续。
2. **其他被捕获的信号**（Ctrl-Z/SIGTSTP 转发、终端产生的信号等）打断了 sigsuspend。
3. 就算是前台 child 触发的 SIGCHLD，handler 也是在 sigsuspend 返回**之前**跑完的——job 链表已经更新——但你仍然得重新读一遍 `fgpid(jobs)` 来确认，因为 sigsuspend 本身不给你这个信息。

这就是系统编程里通用的"围绕阻塞调用套循环"习惯（`read()`/EINTR、`pthread_cond_wait`、`pause()` 全都一样）：**返回值只表示"被打断了"，不表示"条件满足了"，条件要自己再查一遍。**

所以准确说法是：while 不是"不让别的信号唤醒你"（做不到），而是"不管被谁唤醒，都重新问一遍前台 job 是不是真的没了"。非 SIGCHLD 信号只是唤醒源之一，更常见的误唤醒反而是**后台 child 的 SIGCHLD**——那也是 SIGCHLD，但不是你在等的那个。