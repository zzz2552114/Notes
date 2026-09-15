# C++ 左值、右值、std::move 

## 1. 左值引用和右值引用到底啥区别？

核心就两点：**能绑定什么**，以及**表达什么意图**。

- **左值引用 `T&`**：绑定左值。变量、数组元素、解引用结果这些。它是原对象的别名，通常不转移所有权。
- **右值引用 `T&&`**：绑定右值。临时对象、字面量、`std::move(x)` 的结果。它表示“这个对象可以被移动/掏空”。
- **`const T&` 是个例外**：它可以绑定右值，所以常用来做只读参数。

还有一个特别容易忘的点：

> **具名的右值引用变量本身是左值。**

比如函数参数 `std::vector<int>&& vec` 里的 `vec`，它是一个左值表达式。所以你能写 `vec.push_back(3)`；如果想再把它当右值移动，必须写 `std::move(vec)`。



### 另外一个重点是拷贝构造还是移动构造！！！

- `std::vector<int> b = a;` → `a` 是左值 → 调用**拷贝构造函数** → `b` 拿到一份拷贝，`a` 原封不动。
- `std::vector<int> b = std::move(a);` → `std::move(a)` 是右值 → 调用**移动构造函数** → `b` 偷走 `a` 的资源，`a` 被掏空。

---

## 2. std::move 到底干了啥？

`std::move` 本身**不移动任何东西**。它就是个类型转换：

```cpp
std::move(x) // 大致等价于 static_cast<T&&>(x)
```

它把表达式转成右值引用，从而让编译器优先选择移动构造/移动赋值。**真正的资源转移发生在移动构造/移动赋值里。**

一句话：

> `std::move` 只是发出“可以移动”的许可，不负责搬东西。

---

## 3. 两个函数对比

你之前给了这两个：

```cpp
void move_add_three_and_print(std::vector<int> &&vec) {
  std::vector<int> vec1 = std::move(vec);
  vec1.push_back(3);
  for (const int &item : vec1) {
    std::cout << item << " ";
  }
  std::cout << "\n";
}

void add_three_and_print(std::vector<int> &&vec) {
  vec.push_back(3);
  for (const int &item : vec) {
    std::cout << item << " ";
  }
  std::cout << "\n";
}
```

区别不在参数类型，两个都是 `std::vector<int>&&`。区别在**函数体里有没有用 `std::move` 触发移动**。

### move_add_three_and_print

- 参数 `vec` 是右值引用，但它本身只是引用。
- `std::move(vec)` 把 `vec` 转成右值。
- `std::vector<int> vec1 = std::move(vec);` 调用**移动构造函数**。
- `vec1` 抢走了 `vec` 所管理的动态数组资源。
- 原来的 `vec` 仍然存在，但处于“有效但未指定状态”，通常为空。
- 然后 `vec1.push_back(3)`，打印的是 `vec1`。

所以：

```cpp
std::vector<int> v{1, 2};
move_add_three_and_print(std::move(v));
```

会打印 `1 2 3`，但调用后 `v` 通常已经空了。你注释里说“不会获取 vector 的所有权”其实不准确，`vec1` 通过移动构造获取了原本 `vec` 所管理的资源所有权。

### add_three_and_print

- 参数 `vec` 仍然是右值引用。
- 但函数体内没有 `std::move(vec)`，也没有移动构造。
- `vec.push_back(3)` 直接修改 `vec` 所绑定的那个原对象。
- 打印的也是这个原对象。
- 没有发生资源转移。

所以：

```cpp
std::vector<int> v{1, 2};
add_three_and_print(std::move(v));
```

会打印 `1 2 3`，调用后 `v` 仍然是 `{1, 2, 3}`，数据还在，只是被追加了 3。

### 总结表

| 函数                       | 是否移动                    | 操作对象                    | 调用者传入的 vector 状态 |
| -------------------------- | --------------------------- | --------------------------- | ------------------------ |
| `move_add_three_and_print` | 是，`vec1 = std::move(vec)` | 操作移动后的 `vec1`         | 原对象被掏空，通常为空   |
| `add_three_and_print`      | 否                          | 直接操作 `vec` 绑定的原对象 | 原对象保留并追加 3       |

---

## 4. 参数可以写左值引用吗？

可以。比如：

```cpp
void move_add_three_and_print(std::vector<int>& vec) {
  std::vector<int> vec1 = std::move(vec);
  vec1.push_back(3);
  // ...
}
```

这样**能编译**，而且 `std::move(vec)` 仍然有效。因为 `std::move` 不要求 `vec` 必须是右值引用，它只是把表达式 `vec` 强制转换成右值引用。

但参数类型从 `std::vector<int>&&` 改成 `std::vector<int>&` 后，接口含义和调用方式会变：

- 只能接受非 const 左值。
- 函数内部仍然可以 `std::move(vec)` 把调用者的对象移动走。
- 调用者可能以为只是传了个引用，结果原对象被掏空，副作用更隐蔽。

### 参数写法对比

| 参数写法                  | 能接左值 | 能接右值 | 内部 `std::move(vec)` 效果     |
| ------------------------- | -------: | -------: | ------------------------------ |
| `std::vector<int>&&`      |       否 |       是 | 移动绑定对象                   |
| `std::vector<int>&`       |       是 |       否 | 移动调用者传入的左值对象       |
| `const std::vector<int>&` |       是 |       是 | 拷贝，不移动                   |
| `std::vector<int>` 按值   |       是 |       是 | 操作局部副本，不影响左值调用者 |

注意：按值传参 `std::vector<int> vec`，传左值会拷贝，传右值会移动。不是“一定拷贝”。

---

## 5. 右值引用不是移动，是“允许移动”的类型标记

更准确的说法：

> 右值引用把“这个对象是右值，通常可以被安全掏空”这件事编码进了类型系统，让编译器能优先选择移动构造/移动赋值。

但注意：

- 右值引用参数本身不会自动移动。
- 具名的右值引用变量 `vec` 本身是左值，想再移动还要写 `std::move(vec)`。
- 函数也可以不移动它，比如 `add_three_and_print` 就只是修改它。

真正转移所有权的是**移动构造/移动赋值**，不是右值引用本身。

```cpp
std::vector<int> a{1, 2};
std::vector<int> b = std::move(a);
```

这里：

- `std::move(a)`：只是把 `a` 转成右值引用，相当于一个“许可证”。
- `std::vector<int> b = ...`：真正调用移动构造函数。
- 移动构造函数里才发生资源所有权转移。
- 移动后 `a` 通常为空，但仍然有效。

---

## 6. 移动完之后原对象还有内容吗？

看这个：

```cpp
std::vector<int> int_array = {1, 2, 3, 4};
std::vector<int> stealing_ints = std::move(int_array);
```

这里调用的是 `std::vector` 的**移动构造函数**。结果是：

- `stealing_ints` 接管了原来 `int_array` 管理的动态数组；
- `stealing_ints` 里是 `{1, 2, 3, 4}`；
- `int_array` 变成空 vector，`int_array.size() == 0`，`int_array.empty() == true`。

可以验证：

```cpp
std::cout << "stealing_ints: ";
for (int x : stealing_ints) std::cout << x << " ";
std::cout << "\n";

std::cout << "int_array.size() = " << int_array.size() << "\n";
```

典型输出：

```text
stealing_ints: 1 2 3 4 
int_array.size() = 0
```

`int_array` 这个对象本身仍然存在，仍然有效，但资源已经被偷走了。你可以继续使用它，比如重新 `push_back`、赋值、`clear`，但不要再指望它里面还有 `{1, 2, 3, 4}`。

对于 `std::vector` 的移动构造，标准保证移动后源对象是空的。

---

## 7. 引用绑定：右值引用变量 vs 左值引用变量

看这两行：

```cpp
std::vector<int>&& rvalue_stealing_ints = std::move(stealing_ints);
std::vector<int>&  lvalue_stealing_ints = stealing_ints;
```

### 实际运行过程：完全一样

这两行在运行时做的事情**完全相同**：

- 不构造新对象
- 不移动
- 不拷贝
- 只是让一个引用变量绑定到 `stealing_ints` 这个已有对象

它们指向的是同一个 vector 对象，同一个地址，同一块资源。`&rvalue_stealing_ints == &stealing_ints`，`&lvalue_stealing_ints == &stealing_ints`，都成立。

所以如果你只是：

```cpp
std::cout << rvalue_stealing_ints[1]; // 2
std::cout << lvalue_stealing_ints[1]; // 2
```

两者没有任何运行差异。

### 区别只在类型系统和重载决议

区别体现在**编译器怎么看待这个表达式**，以及后续使用它时会发生什么：

1. **直接用它初始化新对象时**

   ```cpp
   std::vector<int> a = rvalue_stealing_ints; // 调用移动构造！因为表达式是右值
   std::vector<int> b = lvalue_stealing_ints; // 调用拷贝构造，因为表达式是左值
   ```

   这是真正有行为差异的地方。

2. **传参给重载函数时**

   ```cpp
   void f(std::vector<int>&);
   void f(std::vector<int>&&);
   
   f(rvalue_stealing_ints); // 选 f(std::vector<int>&&)
   f(lvalue_stealing_ints); // 选 f(std::vector<int>&)
   ```

3. **能否绑定到左值引用**

   ```cpp
   std::vector<int>& x = rvalue_stealing_ints; // 错误，右值引用表达式不能绑定到非 const 左值引用
   std::vector<int>& y = lvalue_stealing_ints; // OK
   ```

### 小结

|                  | 运行时有区别吗             | 类型系统/重载有区别吗 |
| ---------------- | -------------------------- | --------------------- |
| 绑定本身         | 没有，都是别名             | 有                    |
| 直接初始化新对象 | 有：右值走移动，左值走拷贝 | 有                    |
| 传参给重载函数   | 有                         | 有                    |

一句话：**运行时绑定行为一样，都是别名；区别在于编译器把它当右值还是左值，从而影响后续移动/拷贝/重载选择。**

---

## 8. 再移动一次，谁被清零？

```cpp
std::vector<int> stealing_ints = {1, 2, 3, 4};
std::vector<int>&& rvalue_stealing_ints = std::move(stealing_ints);

std::vector<int> another = std::move(rvalue_stealing_ints);
```

过程：

1. `rvalue_stealing_ints` 是 `stealing_ints` 的别名，不拥有数据。
2. `std::move(rvalue_stealing_ints)` 得到右值，指向的还是 `stealing_ints`。
3. `std::vector<int> another = ...` 调用移动构造。
4. 移动构造从 `stealing_ints` 那里偷走资源。
5. 所以 `stealing_ints` 被掏空，变成空 vector。
6. `another` 拿到 `{1, 2, 3, 4}`。

因为 `rvalue_stealing_ints` 只是 `stealing_ints` 的别名，所以：

```cpp
std::cout << stealing_ints.size();            // 0
std::cout << rvalue_stealing_ints.size();     // 0，同一个对象
std::cout << another.size();                  // 4
```

准确说法：

> 不是 `rvalue_stealing_ints` 被清零，而是它引用的对象 `stealing_ints` 被移动掏空了。`rvalue_stealing_ints` 仍然绑定着 `stealing_ints`，只是这个对象现在是空的。

引用变量本身不会“变空”或“变无效”，它只是别名。如果被引用的对象还活着，引用就继续指向它。

---

## 9. 一个类比

把 `stealing_ints` 想成一个房子，里面有家具。

- `std::vector<int>& lref = stealing_ints;` → 给这个房子贴了个左值标签。
- `std::vector<int>&& rref = std::move(stealing_ints);` → 给这个房子贴了个右值标签。

标签不改变房子，也不搬家具。

- `std::vector<int> b = lref;` → 按照左值标签的指示，**复制**了一份家具到新房子 `b`。
- `std::vector<int> a = rref;` → 按照右值标签的指示，**搬走**了原房子里的家具到新房子 `a`。

搬完之后，原来的房子 `stealing_ints` 还在，但里面空了。两个引用标签仍然贴在原房子上，指向的还是那个空房子。

---

## 10. 最后总结

- 左值引用：我引用你，通常不转移所有权。
- 右值引用：你是右值，我可以移动你，但不强制移动。
- `std::move`：把左值伪装成右值，允许移动。
- 移动构造/移动赋值：真正执行资源所有权转移。
- 具名的右值引用变量本身是左值。
- 被移动后的对象仍然“有效”，但值处于未指定状态；对 `std::vector` 来说，移动后保证为空。
- 引用变量只是别名，不拥有数据；移动后变空的是被引用的对象，不是引用本身。