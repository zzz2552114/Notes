C++ 文件操作的核心是 `<fstream>` 里的三个类：

```cpp
std::ifstream  // 输入文件流，读文件
std::ofstream  // 输出文件流，写文件
std::fstream   // 读写文件流，可读可写
```

它们把文件抽象成“流”，让你能像用 `std::cin` / `std::cout` 一样，用 `>>`、`<<`、`getline`、`read`、`write` 来操作文件。下面从“为什么”到“怎么写”再到“常见做法”系统讲。

---

## 1. 为什么用 C++ 流操作文件？

### 1.1 数据需要持久化

内存里的变量、容器在程序结束后就没了。文件可以把数据保存到磁盘，下次程序启动再读回来。

### 1.2 统一抽象

C++ 把“输入”抽象成 `std::istream`，把“输出”抽象成 `std::ostream`：

- 键盘输入：`std::cin`
- 屏幕输出：`std::cout`
- 字符串输入输出：`std::istringstream` / `std::ostringstream`
- 文件输入输出：`std::ifstream` / `std::ofstream`

因此文件操作可以复用你已经熟悉的 `>>`、`<<`、`getline`、状态检查等机制。

### 1.3 RAII 自动管理资源

文件流对象在析构时会自动关闭文件。即使中途 `return` 或抛异常，也能关闭，避免文件描述符泄漏。

```cpp
{
    std::ofstream out("a.txt");
    out << "hello\n";
} // 离开作用域，out 析构，自动 close
```

### 1.4 状态机式错误处理

流内部维护状态位：

- `goodbit`：一切正常
- `eofbit`：到达文件末尾
- `failbit`：格式错误、打开失败等
- `badbit`：严重 I/O 错误，如磁盘故障

你可以通过 `if (!in)`、`in.fail()`、`in.bad()` 等检查。

---

## 2. 核心头文件

```cpp
#include <fstream>   // ifstream, ofstream, fstream
#include <iostream>  // cin, cout, cerr
#include <string>    // std::string, std::getline
#include <vector>    // 二进制缓冲区
#include <iterator>  // istreambuf_iterator
#include <limits>    // numeric_limits
#include <sstream>   // stringstream，常用于解析
#include <iomanip>   // std::quoted 等
#include <filesystem> // C++17，目录/路径操作
```

---

## 3. 打开文件与打开模式

### 3.1 最基本写法

```cpp
std::ifstream in("data.txt");   // 默认 ios::in，文本读
std::ofstream out("out.txt");   // 默认 ios::out，文本写，通常截断
std::fstream fs("db.bin", std::ios::in | std::ios::out | std::ios::binary);
```

### 3.2 打开模式

| 模式               | 含义                                               |
| ------------------ | -------------------------------------------------- |
| `std::ios::in`     | 读                                                 |
| `std::ios::out`    | 写。单独使用时通常会截断文件                       |
| `std::ios::app`    | 追加。每次写之前定位到文件末尾                     |
| `std::ios::ate`    | 打开后立即定位到文件末尾，但之后可以 `seek` 到别处 |
| `std::ios::trunc`  | 截断文件，清空内容                                 |
| `std::ios::binary` | 二进制模式，不做换行转换                           |

常见组合：

```cpp
// 读文本
std::ifstream in("a.txt", std::ios::in);

// 写文本，清空或创建
std::ofstream out("a.txt", std::ios::out | std::ios::trunc);

// 追加文本
std::ofstream log("log.txt", std::ios::out | std::ios::app);

// 二进制读
std::ifstream bin("a.bin", std::ios::in | std::ios::binary);

// 二进制写
std::ofstream bout("a.bin", std::ios::out | std::ios::binary);

// 读写已有文件，不截断
std::fstream fs("a.bin", std::ios::in | std::ios::out | std::ios::binary);

// 读写并清空/创建
std::fstream fs2("a.bin", std::ios::in | std::ios::out | std::ios::trunc | std::ios::binary);
```

注意：

- `std::ofstream out("a.txt");` 默认会创建文件，如果文件已存在通常截断。
- `std::ios::app` 是追加，不会清空。
- `std::ios::ate` 只是打开后到末尾，之后仍可 `seekp` 到前面写。
- `std::ios::binary` 在 Unix/Linux 上通常和文本模式差别不大，但在 Windows 上很重要：文本模式会把 `\n` 转成 `\r\n`，读时再转回来；二进制模式原样读写。

### 3.3 检查是否打开成功

```cpp
std::ifstream in("data.txt");
if (!in) {
    std::cerr << "打开失败\n";
    return;
}
```

`!in` 检查流是否处于失败状态。也可以：

```cpp
if (!in.is_open()) { ... }
```

但 `is_open()` 只表示文件是否成功打开，不表示后续读写一定成功。

### 3.4 关闭文件

文件流析构会自动关闭：

```cpp
{
    std::ofstream out("a.txt");
    out << "hello\n";
} // 自动 close
```

也可以手动关闭并检查错误：

```cpp
out.close();
if (!out) {
    std::cerr << "关闭/写入过程中出错\n";
}
```

写入错误有时在 `close()` 或 `flush()` 时才暴露，所以重要文件建议手动 `close()` 后检查。

---

## 4. 文本写入

```cpp
#include <fstream>
#include <iostream>

int main() {
    std::ofstream out("demo.txt");
    if (!out) {
        std::cerr << "打开失败\n";
        return 1;
    }

    out << "name: " << "Alice" << '\n';
    out << "age: " << 30 << '\n';
    out.put('!');          // 写一个字符
    out << std::flush;     // 刷新缓冲区

    if (!out) {
        std::cerr << "写入失败\n";
        return 1;
    }
    return 0;
}
```

关键点：

- `out << x` 是格式化输出，像 `cout` 一样。
- `out.put(ch)` 写单个字符。
- `'\n'` 只换行，不刷新。
- `std::endl` 换行并刷新，频繁使用会降低性能。
- `std::flush` 只刷新。
- 写入可能先进入缓冲区，`flush`、`close`、程序正常结束会刷新。
- 如果程序崩溃，未刷新数据可能丢失。

---

## 5. 文本读取

### 5.1 格式化读取 `>>`

适合读取按空白分隔的 token：

```cpp
std::ifstream in("data.txt");
std::string name;
int age;

while (in >> name >> age) {
    std::cout << name << " " << age << '\n';
}
```

`>>` 会跳过空格、制表符、换行，并按目标类型解析。例如 `int` 会解析整数。

如果格式不对，会设置 `failbit`。

### 5.2 逐行读取 `std::getline`

最常用的大文件读取方式：

```cpp
std::ifstream in("data.txt");
if (!in) { /* ... */ }

std::string line;
while (std::getline(in, line)) {
    std::cout << line << '\n';
}
```

`getline` 读取一行，不包含换行符，并把换行符从流中丢弃。

常见坑：`>>` 和 `getline` 混用。

```cpp
int age;
std::string line;

in >> age;                 // 读完后，换行符还留在流里
std::getline(in, line);    // 会读到一个空行
```

解决办法：

```cpp
in >> age;
in.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
std::getline(in, line);
```

### 5.3 读取整个文件到 `std::string`

```cpp
#include <fstream>
#include <string>
#include <iterator>

std::ifstream in("data.txt", std::ios::binary);
if (!in) { /* ... */ }

std::string content(
    (std::istreambuf_iterator<char>(in)),
    std::istreambuf_iterator<char>()
);
```

`std::istreambuf_iterator` 会逐字符读取，直到 EOF。用 `std::ios::binary` 可以保留原始换行，不进行文本转换。

也可以用 `seekg` 获取大小后一次性读：

```cpp
std::ifstream in("data.txt", std::ios::binary);
in.seekg(0, std::ios::end);
std::streamsize size = in.tellg();
in.seekg(0, std::ios::beg);

std::string content;
content.resize(static_cast<size_t>(size));
in.read(content.data(), size);
```

注意检查 `size` 是否有效，`tellg()` 失败可能返回 `-1`。

### 5.4 逐字符读取

```cpp
char ch;
while (in.get(ch)) {
    // 处理 ch
}
```

`in.get(ch)` 不跳过空白，适合需要保留所有字符的场景。

---

## 6. 二进制读写

二进制模式适合图片、压缩包、自定义格式、序列化数据。

### 6.1 写二进制

```cpp
std::ofstream out("a.bin", std::ios::binary);
if (!out) { /* ... */ }

int x = 42;
double y = 3.14;

out.write(reinterpret_cast<const char*>(&x), sizeof(x));
out.write(reinterpret_cast<const char*>(&y), sizeof(y));
```

### 6.2 读二进制

```cpp
std::ifstream in("a.bin", std::ios::binary);
if (!in) { /* ... */ }

int x = 0;
double y = 0.0;

in.read(reinterpret_cast<char*>(&x), sizeof(x));
in.read(reinterpret_cast<char*>(&y), sizeof(y));
```

### 6.3 `read` / `write` 的参数

```cpp
istream& read(char* buffer, std::streamsize count);
ostream& write(const char* buffer, std::streamsize count);
```

- 需要 `char*`，所以常用 `reinterpret_cast`。
- `count` 是字节数，类型是 `std::streamsize`。
- `gcount()` 返回上一次非格式化读取实际读了多少字节。

### 6.4 二进制复制文件

高效写法：

```cpp
bool copyFile(const std::string& src, const std::string& dst) {
    std::ifstream in(src, std::ios::binary);
    std::ofstream out(dst, std::ios::binary);
    if (!in || !out) return false;

    out << in.rdbuf();  // 把输入流缓冲区全部写给输出流
    return !out.bad();
}
```

缓冲循环写法：

```cpp
std::ifstream in(src, std::ios::binary);
std::ofstream out(dst, std::ios::binary);
if (!in || !out) return false;

char buf[8192];
while (in) {
    in.read(buf, sizeof(buf));
    std::streamsize n = in.gcount();
    if (n > 0) {
        out.write(buf, n);
        if (!out) return false;
    }
}
return true;
```

### 6.5 直接写结构体的警告

可以这样写 POD 结构体：

```cpp
struct Record {
    int id;
    double score;
};

Record r{1, 99.5};
out.write(reinterpret_cast<const char*>(&r), sizeof(r));
```

但跨平台/跨版本不安全，因为：

- 结构体可能有填充字节。
- 字节序可能不同。
- `int`、`double` 大小可能不同。
- 不能包含指针、`std::string`、虚函数等。
- 版本升级后字段变化会读错。

正式做法是逐字段序列化，并统一字节序、版本号、长度。例如：

```cpp
uint32_t id = 1;
double score = 99.5;

out.write(reinterpret_cast<const char*>(&id), sizeof(id));
out.write(reinterpret_cast<const char*>(&score), sizeof(score));
```

或者使用 JSON、protobuf、FlatBuffers 等库。

---

## 7. 错误处理与状态位

流状态：

```cpp
in.good();  // 一切正常
in.eof();   // 到达末尾
in.fail();  // 格式错误、打开失败等
in.bad();   // 严重错误
in.clear(); // 清除状态位
```

常见正确循环：

```cpp
std::string line;
while (std::getline(in, line)) {
    // 正常处理
}
if (in.bad()) {
    std::cerr << "读取发生严重错误\n";
}
```

不要这样写：

```cpp
while (!in.eof()) {   // 错误做法
    std::getline(in, line);
    // 可能多处理一次
}
```

因为 `eof()` 是在尝试读取之后才设置的，容易重复处理最后一行。

异常方式：

```cpp
std::ifstream in;
in.exceptions(std::ifstream::failbit | std::ifstream::badbit);

try {
    in.open("data.txt");
    std::string line;
    std::getline(in, line);
} catch (const std::ios_base::failure& e) {
    std::cerr << "IO 异常: " << e.what() << '\n';
}
```

注意：如果设置了 `failbit`，读到 EOF 时 `getline` 可能抛异常。所以很多场景只对 `badbit` 开异常，或者只在 `open` 时临时开。

---

## 8. 文件位置与随机访问

`seekg` / `tellg` 用于读位置，`seekp` / `tellp` 用于写位置。

```cpp
std::ifstream in("data.bin", std::ios::binary);

in.seekg(0, std::ios::end);
std::streampos endPos = in.tellg();
in.seekg(0, std::ios::beg);

std::cout << "size = " << endPos << '\n';
```

定位：

```cpp
in.seekg(100, std::ios::beg);  // 从开头偏移 100
in.seekg(-10, std::ios::cur);  // 从当前位置向前 10
in.seekg(-4, std::ios::end);   // 从末尾向前 4
```

写定位：

```cpp
std::fstream fs("db.bin", std::ios::in | std::ios::out | std::ios::binary);
fs.seekp(0, std::ios::beg);
fs.write(...);
```

注意：`fstream` 读写切换时，最好在读写之间调用 `seekg` / `seekp` 或 `flush`，避免缓冲区位置混乱。

---

## 9. 常见做法

### 9.1 逐行读大文件

```cpp
std::ifstream in("big.txt");
std::string line;
while (std::getline(in, line)) {
    // 处理一行
}
```

不要一次性读入超大文件，除非你确定内存足够。

### 9.2 追加日志

```cpp
std::ofstream log("app.log", std::ios::app);
if (!log) return;

log << "2026-10-10 12:00:00 INFO started\n";
log.flush();
```

日志建议定期 `flush`，但不要每行都用 `std::endl`，否则性能差。

### 9.3 解析 CSV 或配置

```cpp
std::ifstream in("config.txt");
std::string line;

while (std::getline(in, line)) {
    std::istringstream iss(line);
    std::string key;
    std::string value;

    if (std::getline(iss, key, '=') && std::getline(iss, value)) {
        // key = value
    }
}
```

### 9.4 写文本时用 `'\n'` 而不是频繁 `std::endl`

```cpp
out << "a\n";
out << "b\n";
out << std::flush;  // 最后刷新一次
```

### 9.5 处理 Windows 换行

在 Linux 读 Windows 文本文件时，行尾可能残留 `\r`：

```cpp
if (!line.empty() && line.back() == '\r') {
    line.pop_back();
}
```

### 9.6 原子写文件

先写临时文件，再重命名：

```cpp
#include <filesystem>

namespace fs = std::filesystem;

void atomicWrite(const fs::path& target, const std::string& data) {
    fs::path tmp = target;
    tmp += ".tmp";

    {
        std::ofstream out(tmp, std::ios::binary);
        out.write(data.data(), static_cast<std::streamsize>(data.size()));
        out.flush();
        if (!out) throw std::runtime_error("write failed");
    }

    fs::rename(tmp, target); // 同一文件系统内通常是原子替换
}
```

### 9.7 使用 `std::filesystem` 做路径和目录操作

C++17 起：

```cpp
#include <filesystem>
namespace fs = std::filesystem;

if (!fs::exists("data")) {
    fs::create_directories("data");
}

fs::path p = fs::path("data") / "a.txt";
auto size = fs::file_size(p);
fs::remove(p);
fs::rename("old.txt", "new.txt");
```

### 9.8 读取带空格的字符串

输出：

```cpp
#include <iomanip>
out << std::quoted("hello world");
```

输入：

```cpp
std::string s;
in >> std::quoted(s);
```

### 9.9 封装成函数

```cpp
bool readFile(const std::string& path, std::string& out) {
    std::ifstream in(path, std::ios::binary);
    if (!in) return false;

    out.assign(
        std::istreambuf_iterator<char>(in),
        std::istreambuf_iterator<char>()
    );
    return !in.bad();
}

bool writeFile(const std::string& path, const std::string& data) {
    std::ofstream out(path, std::ios::binary);
    if (!out) return false;

    out.write(data.data(), static_cast<std::streamsize>(data.size()));
    out.close();
    return !out.fail();
}
```

---

## 10. 常见陷阱与最佳实践

1. **不检查打开是否成功**  
   文件不存在、权限不足、目录不存在都会失败。

2. **混用 `>>` 和 `getline`**  
   `>>` 会留下换行符，导致 `getline` 读到空行。用 `ignore` 清掉。

3. **用 `while (!in.eof())` 循环**  
   应该用 `while (getline(in, line))` 或 `while (in >> x)`。

4. **文本模式读写二进制**  
   图片、压缩包、加密数据必须用 `std::ios::binary`。

5. **直接读写非 POD 对象**  
   `std::string`、指针、虚类不能直接 `write`。结构体也要注意填充和字节序。

6. **忘记刷新或关闭**  
   重要写入后 `flush` 或 `close`，并检查错误。

7. **频繁 `std::endl`**  
   会频繁刷新，性能差。用 `'\n'`。

8. **路径编码问题**  
   C++ 流按字节处理路径。Windows 中文路径建议用 `std::filesystem::path` 或宽字符 API。

9. **多线程同时写同一文件**  
   需要加锁或使用专门日志库。不同流对象一般可并行，同一流对象不要并发操作。

10. **大文件一次性读入内存**  
    能逐行/分块就逐行/分块。

---

## 11. 一个完整小例子

```cpp
#include <fstream>
#include <iostream>
#include <string>

int main() {
    const std::string path = "people.txt";

    // 写
    {
        std::ofstream out(path);
        if (!out) {
            std::cerr << "打开写文件失败\n";
            return 1;
        }

        out << "Alice 30\n";
        out << "Bob 25\n";
        out << "Cindy 28\n";
    } // 自动关闭

    // 读
    std::ifstream in(path);
    if (!in) {
        std::cerr << "打开读文件失败\n";
        return 1;
    }

    std::string name;
    int age;

    while (in >> name >> age) {
        std::cout << name << " is " << age << '\n';
    }

    if (in.bad()) {
        std::cerr << "读取发生严重错误\n";
        return 1;
    }

    return 0;
}
```

---

## 12. 总结

C++ 文件操作记住这条主线：

1. 用 `<fstream>`。
2. 读用 `std::ifstream`，写用 `std::ofstream`，读写用 `std::fstream`。
3. 打开时明确模式：文本/二进制、截断/追加。
4. 打开后立即检查 `if (!in)`。
5. 文本用 `<<`、`>>`、`getline`；二进制用 `read`、`write`。
6. 用 `while (getline(...))` 或 `while (in >> x)` 循环。
7. 重要写入后 `flush` / `close` 并检查状态。
8. 大文件逐行/分块，别一次性读入。
9. 二进制序列化不要直接写复杂对象。
10. C++17 起用 `std::filesystem` 处理路径、目录、重命名、文件大小。

掌握这些，绝大多数 C++ 文件读写场景都能正确处理。