# C 语言函数指针与回调函数详细笔记

> 一句话总纲：**函数指针是“保存函数入口地址的变量”；回调函数是“把函数指针交给别人，让别人在合适时机反过来调用你”的编程模式。**
>
> 记忆口诀：**函数指针存地址，签名一致是铁律；注册调用带上下文，空指针和生命周期要管住。**

---

## 一、函数指针是什么

### 1.1 函数在程序中的本质

C 程序编译后，每个函数都有一段机器指令代码，这段代码在程序运行时位于内存的代码段，有一个入口地址。函数名在表达式中通常会被转换为“指向该函数的指针”。

例如：

```c
int add(int a, int b) {
    return a + b;
}
```

`add` 是函数指示符。在大多数表达式中，`add` 会退化为指向函数的指针，类型是：

```c
int (*)(int, int)
```

即“指向一个参数为 `(int, int)`、返回 `int` 的函数的指针”。

### 1.2 函数指针变量

函数指针变量保存函数入口地址：

```c
int (*fp)(int, int);   // fp 是函数指针变量
fp = add;              // 等价于 fp = &add;
int r = fp(3, 4);      // 等价于 r = (*fp)(3, 4);
```

注意：

- `fp` 是变量，可以改指向别的同签名函数。
- `add` 和 `&add` 在赋值时通常等价。
- `fp(3,4)` 和 `(*fp)(3,4)` 等价。
- 函数指针的类型必须与目标函数“兼容”：返回类型、参数类型、参数个数、调用约定要匹配。

### 1.3 函数指针与指针函数的区别

这是最常混淆的两个概念：

```c
int *f(int x);          // 指针函数：返回 int* 的函数
int (*fp)(int x);       // 函数指针：指向“参数 int、返回 int”的函数的指针
```

再看：

```c
int (*a[5])(int);       // 函数指针数组：a 是数组，5 个元素，每个是函数指针
int (*a)[5];            // 数组指针：a 是指针，指向“含 5 个 int 的数组”
```

记忆：**看括号，`(*名字)` 先说明它是指针；再看右边是不是 `()`，是 `()` 就是指向函数。**

---

## 二、函数指针的由来与为什么需要它

C 语言没有类、没有虚函数、没有 lambda、没有闭包。为了实现“通用算法”和“库与用户代码解耦”，C 使用函数指针。

典型需求：

1. **通用排序**：`qsort` 不知道元素类型，也不知道怎么比较，于是要求用户传入比较函数指针。
2. **事件通知**：库检测到事件后，调用用户注册的函数。
3. **驱动/插件接口**：结构体里放一组函数指针，形成操作表 `ops`。
4. **状态机/命令分派**：用函数指针数组代替长 `switch`。
5. **异步完成通知**：线程、定时器、I/O 完成后调用回调。
6. **模拟多态**：结构体保存函数指针，不同对象填不同函数，实现类似“接口”。

核心思想：**调用者不知道具体函数是谁，只知道函数签名；通过地址间接调用，实现解耦和扩展。**

---

## 三、原理：为什么函数指针能调用函数

函数指针变量中保存的是函数代码的入口地址。调用 `fp(args)` 时，CPU 跳转到 `fp` 保存的地址执行，并按函数签名传递参数、接收返回值。

伪流程：

```text
fp 变量 --> 保存 add 的入口地址
fp(3,4) --> 跳转到 add 的入口 --> 执行 add --> 返回结果
```

函数指针与普通数据指针的区别：

- 数据指针指向数据对象。
- 函数指针指向函数代码。
- C 标准**不保证**函数指针能无损转换为 `void*`，也不保证二者大小相同。
- 不要把函数指针当普通整数随意运算；函数指针加减、解引用取数据不是标准用法。

---

## 四、函数指针的基本语法

### 4.1 声明、赋值、调用

```c
#include <stdio.h>

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

int main(void) {
    int (*fp)(int, int);   // 声明函数指针

    fp = add;              // 赋值：fp = &add; 也可以
    printf("%d\n", fp(5, 3));       // 8
    printf("%d\n", (*fp)(5, 3));    // 8，等价写法

    fp = sub;
    printf("%d\n", fp(5, 3));       // 2
    return 0;
}
```

### 4.2 用 typedef 简化

复杂声明建议用 `typedef`：

```c
typedef int (*binop_t)(int, int);

binop_t fp = add;
int r = fp(1, 2);
```

注意区别：

```c
typedef int (*binop_t)(int, int);  // binop_t 是“函数指针类型”
typedef int binop_func(int, int);  // binop_func 是“函数类型”
```

日常写回调接口，几乎都用“函数指针类型”的 typedef。

### 4.3 函数指针作为参数

```c
int apply(int a, int b, int (*op)(int, int)) {
    return op(a, b);
}

int main(void) {
    int r = apply(3, 4, add);  // 7
    return 0;
}
```

用 typedef：

```c
typedef int (*binop_t)(int, int);

int apply(int a, int b, binop_t op) {
    return op(a, b);
}
```

### 4.4 函数指针作为返回值

直接写法较绕：

```c
int (*choose(int id))(int, int) {
    return id == 0 ? add : sub;
}
```

用 typedef 清晰：

```c
typedef int (*binop_t)(int, int);

binop_t choose(int id) {
    return id == 0 ? add : sub;
}
```

### 4.5 函数指针数组

```c
typedef int (*binop_t)(int, int);

binop_t ops[] = { add, sub };
int r = ops[0](10, 2);  // add
```

常用于命令分派、状态机、操作表。

### 4.6 结构体成员函数指针

```c
typedef int (*binop_t)(int, int);

struct Calculator {
    binop_t op;
};

int main(void) {
    struct Calculator c = { add };
    int r = c.op(2, 3);  // 5
    return 0;
}
```

这是 C 实现“接口/操作表”的常见方式：

```c
struct file_ops {
    int (*open)(const char *path);
    int (*read)(void *buf, int n);
    int (*close)(void);
};
```

---

## 五、回调函数是什么

### 5.1 定义

**回调函数**：由用户编写、但由库/框架/其他模块在合适时机调用的函数。实现方式通常是把函数指针注册给调用方。

关系：

- **函数指针**：C 语言的一种类型/变量。
- **回调函数**：一种设计模式，通常用函数指针实现。
- 回调函数本身是普通函数，不是新语法。

### 5.2 最小回调模型

```c
#include <stdio.h>

typedef void (*event_cb_t)(int event, void *user_data);

struct EventSource {
    event_cb_t cb;
    void *user_data;
};

void event_source_init(struct EventSource *src) {
    src->cb = NULL;
    src->user_data = NULL;
}

void event_source_register(struct EventSource *src,
                           event_cb_t cb,
                           void *user_data) {
    src->cb = cb;
    src->user_data = user_data;
}

void event_source_fire(struct EventSource *src, int event) {
    if (src->cb != NULL) {
        src->cb(event, src->user_data);
    }
}

/* 用户回调 */
static void on_event(int event, void *user_data) {
    int *counter = (int *)user_data;
    (*counter)++;
    printf("event=%d counter=%d\n", event, *counter);
}

int main(void) {
    struct EventSource src;
    int counter = 0;

    event_source_init(&src);
    event_source_register(&src, on_event, &counter);

    event_source_fire(&src, 1);
    event_source_fire(&src, 2);

    return 0;
}
```

这里：

- `event_cb_t` 是回调函数指针类型。
- `on_event` 是回调函数。
- `event_source_register` 是“注册”。
- `event_source_fire` 是“触发/调用”。
- `user_data` 是上下文参数，用来模拟闭包，保存用户状态。

### 5.3 为什么回调要带 `void *user_data`

C 没有闭包。回调函数通常无法直接访问调用者的局部变量。通过 `void *user_data` 把上下文传进去：

```c
typedef void (*cb_t)(int event, void *ctx);

void register_cb(cb_t cb, void *ctx);
```

回调中再转回真实类型：

```c
static void my_cb(int event, void *ctx) {
    struct MyState *s = (struct MyState *)ctx;
    s->count++;
}
```

注意：`ctx` 的生命周期必须覆盖回调可能发生的整个时期。不能把局部变量地址交给异步回调后函数就返回。

---

## 六、回调的完整设计模式：注册—触发—注销

一个健壮的回调接口通常包括：

```c
typedef void (*event_cb_t)(int event, void *user_data);

struct EventSource {
    event_cb_t cb;
    void *user_data;
};

void event_register(struct EventSource *src, event_cb_t cb, void *user_data);
void event_unregister(struct EventSource *src);
void event_fire(struct EventSource *src, int event);
```

实现：

```c
void event_register(struct EventSource *src, event_cb_t cb, void *user_data) {
    src->cb = cb;
    src->user_data = user_data;
}

void event_unregister(struct EventSource *src) {
    src->cb = NULL;
    src->user_data = NULL;
}

void event_fire(struct EventSource *src, int event) {
    if (src->cb) {
        src->cb(event, src->user_data);
    }
}
```

设计时必须写清楚：

- 回调何时调用？
- 调用多少次？
- 在哪个线程调用？
- 是否允许 `cb == NULL`？
- 回调是否可重入？
- 回调能否阻塞？
- 回调返回值含义？
- `user_data` 谁拥有、何时释放？
- 注销后是否保证不再调用？
- 注销时如果回调正在执行怎么办？

---

## 七、常用用法与代码示例

### 7.1 `qsort` 比较器回调

```c
#include <stdio.h>
#include <stdlib.h>

static int cmp_int(const void *a, const void *b) {
    int x = *(const int *)a;
    int y = *(const int *)b;
    if (x < y) return -1;
    if (x > y) return 1;
    return 0;
}

int main(void) {
    int arr[] = {3, 1, 4, 2};
    size_t n = sizeof(arr) / sizeof(arr[0]);
    qsort(arr, n, sizeof(arr[0]), cmp_int);
    for (size_t i = 0; i < n; i++) printf("%d ", arr[i]);
    return 0;
}
```

注意：比较器不要写成 `return x - y;`，因为整数相减可能溢出。应返回负数/0/正数。

### 7.2 通用遍历回调

```c
typedef void (*visit_fn)(int value, void *ctx);

void for_each(int *arr, size_t n, visit_fn visit, void *ctx) {
    for (size_t i = 0; i < n; i++) {
        visit(arr[i], ctx);
    }
}
```

### 7.3 命令分派表

```c
typedef void (*cmd_fn)(const char *arg);

static void cmd_help(const char *arg) { (void)arg; puts("help"); }
static void cmd_quit(const char *arg) { (void)arg; puts("quit"); }

struct Command {
    const char *name;
    cmd_fn fn;
};

static struct Command cmds[] = {
    {"help", cmd_help},
    {"quit", cmd_quit},
};

void dispatch(const char *name) {
    for (size_t i = 0; i < sizeof(cmds)/sizeof(cmds[0]); i++) {
        if (strcmp(cmds[i].name, name) == 0) {
            cmds[i].fn(NULL);
            return;
        }
    }
}
```

### 7.4 状态机

```c
typedef void (*state_fn)(void);

static void state_idle(void) { /* ... */ }
static void state_run(void)  { /* ... */ }

state_fn states[] = { state_idle, state_run };
states[0]();
```

### 7.5 驱动/插件操作表

```c
struct device_ops {
    int (*init)(void);
    int (*read)(void *buf, int len);
    void (*deinit)(void);
};

struct device {
    const char *name;
    const struct device_ops *ops;
};
```

### 7.6 线程入口函数

```c
#include <pthread.h>

static void *worker(void *arg) {
    int *p = (int *)arg;
    /* ... */
    return NULL;
}

pthread_create(&tid, NULL, worker, &data);
```

`pthread_create` 要求函数类型为 `void *(*)(void *)`。

### 7.7 `atexit` 与 `signal`

```c
void cleanup(void);
atexit(cleanup);

void handler(int sig);
signal(SIGINT, handler);
```

信号处理函数中只能调用 async-signal-safe 函数，通常只设置 `volatile sig_atomic_t` 标志，不要 `printf`、`malloc`、加锁。

---

## 八、函数指针声明解读技巧

复杂声明可用“右左法则”或 typedef 拆解：

```c
void (*signal(int sig, void (*func)(int)))(int);
```

拆解：

```c
typedef void (*sighandler_t)(int);
sighandler_t signal(int sig, sighandler_t func);
```

含义：`signal` 是函数，参数为 `int` 和函数指针，返回函数指针。

再看：

```c
int (*fp)(int, int);          // fp 是指向函数的指针
int *fp(int, int);            // fp 是返回 int* 的函数
int (*arr[10])(int);          // arr 是数组，元素是函数指针
int (*p)[10];                 // p 是指向含 10 个 int 的数组的指针
```

---

## 九、常见注意事项与重要事项

### 9.1 签名必须匹配

函数指针类型与目标函数必须兼容：

```c
typedef int (*cmp_t)(const void *, const void *);
int cmp_int(const void *a, const void *b);  // 匹配
```

不匹配可能导致未定义行为。不要用强制转换“糊弄”签名，除非你非常清楚调用约定和标准限制。

### 9.2 调用前检查空指针

```c
if (cb != NULL) {
    cb(event, ctx);
}
```

除非接口明确保证非空。

### 9.3 生命周期

- 回调函数本身是代码，通常一直存在；但动态库卸载后函数指针会悬空。
- `user_data` 若指向局部变量，异步回调时可能已失效。
- 注册回调后，要保证回调函数和上下文在注销前有效。
- 回调中不要释放“正在被调用者使用”的资源，尤其是正在遍历的节点，建议延迟删除。

### 9.4 线程安全与重入

- 注册/注销/触发可能并发，需要锁或原子操作。
- 调用回调时尽量不要持锁，避免回调中再次注册导致死锁。
- 可先复制 `cb` 和 `ctx` 快照，再释放锁调用。
- 回调可能重入：回调中再次触发同一事件，注意栈溢出和状态一致性。
- 注销时要处理“回调正在执行”的竞态，可用引用计数、条件变量或“标记删除 + 延迟释放”。

### 9.5 不要用局部变量地址做异步上下文

错误：

```c
void f(void) {
    int x = 0;
    register_async_cb(cb, &x);  // x 在 f 返回后失效
}
```

正确：用静态、全局、堆分配并明确释放时机。

### 9.6 回调返回值约定要明确

例如遍历回调：

```c
typedef bool (*visit_t)(void *item, void *ctx);
// 返回 true 继续，false 停止
```

错误传播：

```c
typedef int (*cb_t)(void *ctx);  // 0 成功，负值错误
```

### 9.7 函数指针与 `void *`

C 标准不保证函数指针可转换为 `void *`。`dlsym` 返回 `void *` 再转函数指针是 POSIX 常见做法，但不是纯 C 标准保证。不要假设函数指针大小等于数据指针。

### 9.8 函数指针比较

只有相同类型的函数指针之间比较才有标准意义。不要拿函数指针和整数比较。

### 9.9 性能与优化

通过函数指针调用通常是间接调用，可能影响内联和分支预测。热路径中不要滥用；但为了解耦和扩展，回调通常值得。

### 9.10 安全

函数指针可被攻击者覆盖，导致控制流劫持。对保存函数指针的结构体注意内存安全、边界检查、只读保护。

### 9.11 文档化回调契约

必须写清楚：调用时机、线程、是否可空、是否可重入、是否阻塞、返回值、错误处理、上下文所有权、注销语义。

### 9.12 常见错误清单

- 函数指针未初始化就调用。
- 回调签名写错，强转后调用。
- 回调中释放了正在遍历的对象。
- 注册后上下文变量已失效。
- 注销后仍有并发回调。
- 在信号处理函数中调用非 async-signal-safe 函数。
- `qsort` 比较器返回 `a-b` 导致溢出。
- 把“函数指针数组”和“数组指针”写混。
- 把 `typedef int (*cb)(int);` 和 `typedef int cb(int);` 混用。
- 用 `%d` 打印函数指针。

---

## 十、最小完整综合示例

```c
#include <stdio.h>
#include <stddef.h>

typedef void (*event_cb_t)(int event, void *user_data);

struct EventSource {
    event_cb_t cb;
    void *user_data;
};

static void event_register(struct EventSource *src,
                           event_cb_t cb,
                           void *user_data) {
    src->cb = cb;
    src->user_data = user_data;
}

static void event_fire(struct EventSource *src, int event) {
    if (src->cb != NULL) {
        src->cb(event, src->user_data);
    }
}

struct Counter {
    int count;
};

static void on_event(int event, void *user_data) {
    struct Counter *c = (struct Counter *)user_data;
    c->count++;
    printf("event=%d count=%d\n", event, c->count);
}

int main(void) {
    struct EventSource src = { NULL, NULL };
    struct Counter counter = { 0 };

    event_register(&src, on_event, &counter);
    event_fire(&src, 1);
    event_fire(&src, 2);

    return 0;
}
```

---

## 十一、最终总结

1. **函数指针**保存函数入口地址，类型由返回值和参数列表决定。
2. `fp = func;`、`fp(args)`、`(*fp)(args)` 是基本用法。
3. **回调函数**是函数指针的典型应用：注册函数指针，由库在合适时机调用。
4. 回调接口通常包含：回调类型、注册函数、触发函数、注销函数、`void *user_data`。
5. 最重要三件事：**签名匹配、空指针检查、生命周期管理**。
6. 并发场景还要加：**线程安全、重入保护、注销竞态处理**。
7. 设计回调接口必须写清契约：何时调用、在哪个线程、能否阻塞、能否重入、返回值含义、上下文所有权、注销保证。
8. 函数指针是 C 实现通用算法、事件驱动、插件、驱动接口、状态机和“模拟多态”的核心工具。