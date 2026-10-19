可以，最推荐的是：**保留 `public` 的 fetch，只把它的 push URL 改成一个无效地址**。这样你还能 `git fetch public` 拉 CMU 上游更新，但 `git push public` 会直接失败。

```bash
git remote set-url --push public no_push
```

然后检查：

```bash
git remote -v
```

应该看到类似：

```text
origin  git@github.com:zzz2552114/bustub.git (fetch)
origin  git@github.com:zzz2552114/bustub.git (push)
public  https://github.com/cmu-db/bustub.git (fetch)
public  no_push (push)
```

以后如果你手滑执行：

```bash
git push public
```

会报错，类似：

```text
fatal: 'no_push' does not appear to be a git repository
```

不会推到 `cmu-db/bustub`。

---

如果你以后想恢复 `public` 的 push URL：

```bash
git remote set-url --push public https://github.com/cmu-db/bustub.git
```

如果你完全不需要 fetch 上游，也可以直接删掉这个 remote：

```bash
git remote remove public
```

---

另外建议顺手检查一下默认 push 目标，防止 `git push` 默认跑去 `public`：

```bash
git config --get remote.pushDefault
git branch -vv
```

如果 `remote.pushDefault` 输出是 `public`，改成：

```bash
git config remote.pushDefault origin
```

如果当前分支 upstream 指向 `public/...`，也可以改到你的 fork：

```bash
git branch --set-upstream-to=origin/master
```

不过只要执行了：

```bash
git remote set-url --push public no_push
```

即使默认 push 目标误指到 `public`，也推不上去。