
普通数组：固定大小，没有内置功能，不安全
vector： 

# 常用成员函数速查表

## 📋 基本信息
- **头文件**：`#include <vector>`
- **声明示例**：`std::vector<int> v;`
- **本质**：自动管理堆内存的动态数组，大小可变，元素连续存储。

---

## 1. 构造与赋值

| 函数 | 作用 | 示例 |
|------|------|------|
| `vector()` | 默认构造，空容器 | `vector<int> v;` |
| `vector(size_t n)` | 创建 n 个默认值元素 | `vector<int> v(10);` // 10个0 |
| `vector(size_t n, const T& val)` | 创建 n 个 val | `vector<int> v(5, 42);` // 5个42 |
| `vector(const vector& other)` | 拷贝构造 | `vector<int> v2(v1);` |
| `vector(initializer_list<T> il)` | 列表初始化 | `vector<int> v{1,2,3};` |
| `operator=` | 赋值替换内容 | `v1 = v2;` |
| `assign(size_t n, const T& val)` | 重新分配 n 个 val | `v.assign(3, 99);` |
| `assign(InputIt first, InputIt last)` | 用迭代器范围赋值 | `v.assign(v2.begin(), v2.end());` |

---

## 2. 容量相关

| 函数 | 返回值 | 作用 |
|------|--------|------|
| `size()` | `size_t` | 当前元素个数 |
| `empty()` | `bool` | 是否为空（size == 0） |
| `capacity()` | `size_t` | 已分配内存最多能存多少元素 |
| `reserve(size_t n)` | `void` | 预分配至少 n 个元素的空间，避免反复扩容 |
| `shrink_to_fit()` | `void` | 释放多余内存，使 capacity == size |
| `max_size()` | `size_t` | 理论最大元素数（通常很大） |

---

## 3. 元素访问

| 函数 | 返回值 | 说明 |
|------|--------|------|
| `operator[](size_t i)` | `T&` | 无越界检查，直接索引 |
| `at(size_t i)` | `T&` | **有越界检查**，越界抛 `std::out_of_range` |
| `front()` | `T&` | 第一个元素（等价于 v[0]） |
| `back()` | `T&` | 最后一个元素（等价于 v[v.size()-1]） |
| `data()` | `T*` | 返回底层数组的首地址指针 |

---

## 4. 添加与删除

| 函数 | 返回值/参数 | 作用 |
|------|-------------|------|
| `push_back(const T& val)` | `void` | 尾部添加元素（拷贝） |
| `push_back(T&& val)` | `void` | 尾部添加元素（移动） |
| `emplace_back(Args&&... args)` | `void` | 尾部**原地构造**元素，避免拷贝/移动 |
| `pop_back()` | `void` | 移除最后一个元素（不返回） |
| `insert(iterator pos, const T& val)` | `iterator` | 在 pos 前插入 val，返回新元素位置 |
| `insert(iterator pos, size_t n, const T& val)` | `iterator` | 插入 n 个 val |
| `emplace(iterator pos, Args&&... args)` | `iterator` | 在 pos 前原地构造新元素 |
| `erase(iterator pos)` | `iterator` | 删除 pos 处元素，返回下一个元素位置 |
| `erase(iterator first, iterator last)` | `iterator` | 删除 [first, last) 范围元素 |
| `clear()` | `void` | 清空所有元素（size=0，capacity不变） |
| `resize(size_t n)` | `void` | 改变 size，新增元素默认值 |
| `resize(size_t n, const T& val)` | `void` | 改变 size，新增元素为 val |

---

## 5. 遍历与迭代器

| 函数 | 返回值 | 作用 |
|------|--------|------|
| `begin()` | `iterator` | 指向第一个元素的迭代器 |
| `end()` | `iterator` | 指向**尾后**的迭代器（非元素） |
| `rbegin()` | `reverse_iterator` | 反向开始（最后一个元素） |
| `rend()` | `reverse_iterator` | 反向结束（第一个元素之前） |
| `cbegin()` | `const_iterator` | 只读 begin |
| `cend()` | `const_iterator` | 只读 end |

**典型遍历**：
```cpp
for (auto it = v.begin(); it != v.end(); ++it) { ... }       // 迭代器
for (auto& val : v) { ... }                                  // 范围for