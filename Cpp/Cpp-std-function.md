一句话先总结：

```cpp
std::vector<std::function<size_t(const KeyType &)>>
```

表示：

> 一个动态数组 `vector`，里面每个元素都是一个“函数包装器” `std::function`；  
> 这些函数都接受一个 `const KeyType&` 参数，并返回一个 `size_t`。  
> 换句话说：**一组“输入 KeyType 常量引用，输出 size_t”的可调用对象。**

---

## 1. 逐字逐句拆开看

原类型：

```cpp
std::vector<std::function<size_t(const KeyType &)>>
```

可以拆成：

| 片段            | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| `std::vector`   | 标准库的动态数组模板，头文件 `<vector>`                      |
| `< ... >`       | `vector` 的模板参数：它里面存什么类型                        |
| `std::function` | 标准库的通用函数包装器，头文件 `<functional>`                |
| `< ... >`       | `function` 的模板参数：它包装的函数签名是什么                |
| `size_t`        | 返回类型，通常是 `std::size_t`，无符号整数，表示大小、计数、哈希值等 |
| `(`             | 函数参数列表开始                                             |
| `const`         | 表示参数只读，函数内部不能修改它                             |
| `KeyType`       | 一个占位类型，通常是模板参数或 `using` 出来的类型别名        |
| `&`             | 左值引用，避免拷贝                                           |
| `)`             | 函数参数列表结束                                             |
| `>`             | 结束 `std::function` 的模板参数                              |
| `>`             | 结束 `std::vector` 的模板参数                                |

所以整体读法是：

```cpp
std::vector<
    std::function<
        size_t(const KeyType&)
    >
>
```

即：

> `vector` 的元素类型是 `std::function<size_t(const KeyType&)>`。

---

## 2. `std::function<size_t(const KeyType&)>` 到底是什么？

`std::function` 的模板参数写的是“函数类型”：

```cpp
返回类型(参数类型列表)
```

所以：

```cpp
std::function<size_t(const KeyType&)>
```

表示一个可调用对象，调用它时：

```cpp
size_t result = f(key);
```

其中：

- `f` 是 `std::function` 对象；
- `key` 是 `const KeyType&`；
- `result` 是 `size_t`。

它不是函数指针，但可以装：

- 普通函数；
- 函数指针；
- lambda；
- 仿函数，也就是重载了 `operator()` 的类对象；
- `std::bind` 的结果；
- 其他满足签名的可调用对象。

例如：

```cpp
std::size_t f(const std::string& s) {
    return s.size();
}
```

这个函数可以放进：

```cpp
std::function<std::size_t(const std::string&)> fn = f;
```

调用：

```cpp
std::cout << fn("hello"); // 输出 5
```

---

## 3. 如果 `KeyType` 是 `std::string`

那么：

```cpp
std::vector<std::function<size_t(const KeyType &)>>
```

就变成：

```cpp
std::vector<std::function<std::size_t(const std::string&)>>
```

意思是：

> 一个数组，里面存若干个函数；每个函数接受 `const std::string&`，返回 `std::size_t`。

可以先用别名简化：

```cpp
using KeyType = std::string;
using KeyFunc = std::function<std::size_t(const KeyType&)>;
using KeyFuncList = std::vector<KeyFunc>;
```

然后：

```cpp
KeyFuncList funcs;
```

这就和你问的类型完全一样。

---

## 4. 完整例子：一组哈希/计算函数

```cpp
#include <vector>
#include <functional>
#include <string>
#include <iostream>
#include <cstddef>

using KeyType = std::string;
using KeyFunc = std::function<std::size_t(const KeyType&)>;

// 1. 普通函数
std::size_t hash_length(const KeyType& s) {
    return s.size();
}

// 2. 仿函数
struct HashFirstChar {
    std::size_t operator()(const KeyType& s) const {
        return s.empty() ? 0 : static_cast<std::size_t>(s.front());
    }
};

int main() {
    std::vector<KeyFunc> funcs;

    // 放普通函数
    funcs.push_back(hash_length);

    // 放仿函数
    funcs.push_back(HashFirstChar{});

    // 放 lambda
    funcs.push_back([](const KeyType& s) {
        std::size_t h = 0;
        for (unsigned char c : s) {
            h = h * 131 + c;
        }
        return h;
    });

    // 放标准哈希
    funcs.emplace_back([](const KeyType& s) -> std::size_t {
        return std::hash<KeyType>{}(s);
    });

    KeyType key = "hello";

    for (std::size_t i = 0; i < funcs.size(); ++i) {
        if (funcs[i]) { // 检查非空
            std::cout << "func " << i << " -> " << funcs[i](key) << '\n';
        }
    }
}
```

这里：

```cpp
std::vector<KeyFunc> funcs;
```

等价于：

```cpp
std::vector<std::function<std::size_t(const std::string&)>> funcs;
```

`funcs[0]` 是一个 `std::function` 对象，不是函数本身。  
`funcs[0](key)` 才是调用它。

---

## 5. 为什么需要 `std::function`？

因为 `vector` 要求所有元素类型相同。

但 lambda 的类型是编译器生成的、彼此不同的类型：

```cpp
auto l1 = [](const std::string& s) { return s.size(); };
auto l2 = [](const std::string& s) { return std::hash<std::string>{}(s); };
```

`l1` 和 `l2` 类型不同，不能直接放进同一个 `vector`。

而 `std::function` 可以把它们统一成同一种类型：

```cpp
std::vector<std::function<std::size_t(const std::string&)>> funcs;

funcs.push_back(l1);
funcs.push_back(l2);
```

这就是所谓的“类型擦除”。

---

## 6. 它和函数指针数组有什么区别？

也可以写：

```cpp
std::vector<std::size_t(*)(const std::string&)> funcs;
```

但函数指针只能存：

- 普通函数；
- 无捕获 lambda，且能转换成函数指针的。

不能存：

- 有捕获的 lambda；
- 带状态的仿函数；
- `std::bind` 结果。

而：

```cpp
std::vector<std::function<std::size_t(const std::string&)>> funcs;
```

可以存上面所有东西。

代价是：

- `std::function` 有类型擦除开销；
- 调用通常比直接函数调用慢一点；
- 可能发生堆分配；
- 对象更大。

如果只是无状态函数指针，函数指针数组更轻。  
如果需要统一存各种可调用对象，`std::function` 很方便。

---

## 7. 常见用途

这种类型经常表示：

> “一组函数，每个函数都根据 key 算出一个 size_t。”

例如：

- 多个哈希函数集合，用于 Bloom Filter、布谷鸟哈希；
- 多个策略函数，根据 key 计算不同索引；
- 回调列表，每个回调处理 key 并返回编号；
- 函数表，运行时动态添加、删除、遍历。

比如 Bloom Filter 里可能这样：

```cpp
class BloomFilter {
    std::vector<std::function<std::size_t(const std::string&)>> hashes_;
    std::size_t bits_;
public:
    void add(const std::string& key) {
        for (auto& h : hashes_) {
            if (h) {
                std::size_t idx = h(key) % bits_;
                // set_bit(idx);
            }
        }
    }
};
```

---

## 8. 如果 `KeyType` 是模板参数

它经常出现在模板类里：

```cpp
template <typename KeyType>
class MultiHash {
public:
    using HashFn = std::function<std::size_t(const KeyType&)>;

    void add(HashFn fn) {
        funcs_.push_back(std::move(fn));
    }

    std::size_t run(std::size_t i, const KeyType& key) const {
        return funcs_.at(i)(key);
    }

private:
    std::vector<HashFn> funcs_;
};
```

使用时：

```cpp
MultiHash<std::string> mh;

mh.add([](const std::string& s) {
    return s.size();
});

mh.add([](const std::string& s) {
    return std::hash<std::string>{}(s);
});

std::cout << mh.run(0, "hello") << '\n';
```

当 `KeyType = std::string` 时，成员类型就是：

```cpp
std::vector<std::function<std::size_t(const std::string&)>>
```

---

## 9. 重要注意点

### 9.1 `KeyType` 不是标准类型

`KeyType` 不是 C++ 标准库里的类型。  
它通常是：

```cpp
template <typename KeyType>
...
```

里的模板参数，或者：

```cpp
using KeyType = std::string;
```

如果你直接写 `KeyType` 但没有定义，会编译错误。

---

### 9.2 `size_t` 建议写 `std::size_t`

你写：

```cpp
size_t
```

可能来自全局命名空间。更标准的是：

```cpp
std::size_t
```

头文件：

```cpp
#include <cstddef>
```

---

### 9.3 `std::function` 可以为空

```cpp
std::function<std::size_t(const std::string&)> f;
```

此时 `f` 是空的。调用它会抛：

```cpp
std::bad_function_call
```

所以调用前最好检查：

```cpp
if (f) {
    std::size_t h = f(key);
}
```

---

### 9.4 注意 lambda 捕获生命周期

安全：

```cpp
int salt = 42;
funcs.push_back([salt](const std::string& s) {
    return s.size() + salt;
});
```

危险：

```cpp
std::function<std::size_t(const std::string&)> make() {
    int x = 1;
    return [&x](const std::string& s) {
        return s.size() + x; // x 已经销毁，悬空引用
    };
}
```

---

### 9.5 `std::function` 要求可拷贝，C++23 前不能直接存 move-only 捕获

C++11 到 C++20 的 `std::function` 要求包装的可调用对象可拷贝。  
如果 lambda 捕获了 `std::unique_ptr`，不能直接放进去。  
C++23 有 `std::move_only_function` 可以支持 move-only 可调用对象。

---

### 9.6 两个右尖括号

```cpp
std::vector<std::function<size_t(const KeyType &)>>
```

最后两个 `>>` 是关闭两个模板。  
C++11 以后可以直接写 `>>`。  
C++03 需要写成：

```cpp
std::vector<std::function<size_t(const KeyType &)> >
```

但 `std::function` 本身就是 C++11 的，所以通常直接写 `>>` 即可。

---

## 10. 最终总结

```cpp
std::vector<std::function<size_t(const KeyType &)>>
```

就是：

> 一个 `std::vector`，元素是 `std::function`；  
> 每个 `std::function` 包装一个可调用对象；  
> 该可调用对象接受 `const KeyType&`，返回 `size_t`。

用别名写更清楚：

```cpp
using KeyType = std::string;
using KeyFunc = std::function<std::size_t(const KeyType&)>;
using KeyFuncList = std::vector<KeyFunc>;

KeyFuncList funcs;
```

你可以把它理解成：

> “一列函数”，每个函数都吃一个 `const KeyType&`，吐一个 `size_t`。