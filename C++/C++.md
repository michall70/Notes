### 类的继承
public
protected
private
```c++
class Frame {
private:
    int secret_id;      // 只能自己用，儿子都不能直接碰
protected:
    int width, height;  // 自己和儿子可以用，但外人不行
public:
    void display() {    // 谁都能用
        // ...
    }
};

class VideoFrame : public Frame {
public:
    void showInfo() {
        // width = 1920;   ✅ 可以，因为 width 是 protected
        // secret_id = 5;  ❌ 不行！secret_id 是 private，子类不能直接访问
    }
};

int main() {
    VideoFrame vf;
    vf.display();   // ✅ public，类外部可以调用
    // vf.width = 100;  // ❌ protected，类外部不能访问
    // vf.secret_id = 1; // ❌ private，更不能访问
}
```


###  `.` `->`
```c++
class Camera {
public:
    void start() { /*...*/ }
    int id;
};

int main() {
    // 1. 对象本身（在栈上）
    Camera cam;          // cam 是一个对象，不是指针
    cam.start();         // 用 . 调用方法
    cam.id = 1;          // 用 . 访问成员变量

    // 2. 指向对象的指针
    Camera* pCam = &cam; // pCam 是一个指针
    pCam->start();       // 用 -> 调用方法
    pCam->id = 2;        // 用 -> 访问成员变量

    // 或者用 (*pCam). 也可以，但麻烦
    (*pCam).start();     // 等价于 pCam->start()
}
```


### 类的链式调用
```cpp
std::string filePath = "/home/michall/AAAProjects/RGBD/projects/testproject/data/1280_800_30fps.bag";

std::string fileName = fs::path(filePath).stem().string();
```
等价于：
```cpp
fs::path(filePath)          // 1. 创建一个临时的 path 对象（没有名字）
            .filename()     // 2. 对这个临时对象调用 filename()，返回一个新的临时 path 对象
            .string()       // 3. 再调用 string()，返回 std::string
// 即为：
fs::path p(filePath);               // 有名对象
std::string name = p.stem().string(); // 分两步
```


### 格式化写入
```cpp
char buf[32];
snprintf(buf, sizeof(buf), "%04d", saveIndex);
cout << std::string(buf);
```
#### 第1行：`char buf[32];`

- **语法**：声明一个**字符数组**（也叫 C 风格字符串缓冲区）。
- **`char`**：元素类型，每个元素存放一个 ASCII 字符。
- **`buf`**：数组名，可以当成指向数组首地址的指针。
- **`[32]`**：数组大小，能存 31 个有效字符 + 1 个字符串结束符 `'\0'`。
- **作用**：预留一小块内存，用来临时存放拼接好的字符串。这里 32 个字节足够存放最多 4 位数字 + 可能的其他字符，非常安全。

#### 第2行：`snprintf(buf, sizeof(buf), "%04d", saveIndex);`

这是核心，用来把整数 `saveIndex` 格式化成“4位补零”的字符串。
- **`snprintf`**：来自 C 标准库 `<cstdio>` 的安全格式化输出函数。相比古老的 `sprintf`，它多一个 `size` 参数，**绝不会写越界**。
- **函数签名**：`int snprintf(char* str, size_t size, const char* format, ...);`
    - `str`：输出字符串存放的目标缓冲区。
    - `size`：缓冲区最大可写入的字节数（包括结尾 `'\0'`）。写满则截断，防止溢出。
    - `format`：格式化控制字符串，后面跟可变参数（要填入的值）。
    - 返回值：实际想写入的字符数（不包含 `'\0'`），若超过 `size-1` 会被截断。

**逐参数详解：**
1. **`buf`**：前面声明的字符数组，格式化后的字符串就存放在这里。
2. **`sizeof(buf)`**：编译时计算的 `buf` 数组总字节数。这里 `char` 是 1 字节，所以 `sizeof(buf) = 32`。这样即使格式化出的字符串不小心超过 31 个字符（几乎不可能，但安全），也不会写坏内存。
3. **`"%04d"`**：**格式化字符串**，规则是：
    - **`%d`**：表示以十进制整数形式输出。
    - **`0`**：标志，表示**用字符 '0' 填充**空位，而不是默认的空格。
    - **`4`**：最小字段宽度。如果数字不足 4 位，左侧补零凑够 4 位。如果数字超过 4 位，完整输出，不会截断。


### 计时器
```cpp
auto t0 = std::chrono::steady_clock::now();
//--------- processing ------------
std::vector<cv::KeyPoint> keypoints;
cv::Mat descriptors;
orb->detectAndCompute(leftImg, cv::noArray(), keypoints, descriptors);
//---------    done    ------------
auto t1 = std::chrono::steady_clock::now();
auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
```
时间间隔：`elapsed`

### 指针* 引用&
#### 指针`*`
```cpp
int a = 15;
//等价
int* p = &a;
int *p = &a;
int * p = &a;
```
int 让编译器知道指向的是一个int类型的数据
`void*` 万能地址条，只存地址
```cpp
int a = 42;
int* p1 = &a;       // p1 知道指向 int
void* p2 = &a;      // p2 也指向 a，但"忘了"a 是 int

*p1 = 100;          // ✅ 可以，编译器知道你在操作一个 int
*p2 = 100;          // ❌ 编译错误！编译器不知道*p2是什么类型
// error: 'void*' is not a pointer-to-object type
```

```cpp
//转换数据类型 用()
double pi = 3.14159;

// C 风格强转
int a = (int)pi;          // a = 3

// static_cast：<目标类型>(表达式)
int b = static_cast<int>(pi);  // b = 3
```
同样对指针使用
```cpp
cv::Mat leftImg(height, width, CV_8UC1, (void *)left->getData());
//                                     ^^^^^^^^
// 函数内部会进行再次转换指针类型
```

#### 引用`&`
**房间的别名/花名**，从出生起就绑定这间房，不能改
```cpp
//等价
void draw(cv::Mat &img)   // & 靠着变量名
void draw(cv::Mat& img)   // & 靠着类型
void draw(cv::Mat & img)  // 两边都有空格
```

语法糖
```cpp
// 用指针（要一直写 *）
void addOne(int* p) {
    *p = *p + 1;     // 累，容易忘写 *
}
addOne(&a);          // 调用时还要加 &

// 用引用（干净多了）
void addOne(int& r) {
    r = r + 1;       // 像操作普通变量一样
}
addOne(a);           // 调用时也简单，直接传 a
```
此处在函数内修改，会影响外部变量。

```cpp
int a = 42;
int& r = a;
int* p = &a;
std::cout << p;   // 输出 a 的地址，比如 0x7fff1234
std::cout << &a;  // 输出 a 的地址，和 p 一样
std::cout << &r;  // 输出 a 的地址！和 &a 完全一样，r 没有自己的地址
std::cout << &p;  // 输出 p 自己的地址！p 也是一个变量，占内存
```

###  `static_cast` — 带检查的强制类型转换

`static_cast` 是 C++ 里四种类型转换操作符之一，用来在**编译时**把一个类型转成另一个类型。你可以把它理解为 C 风格强转 `(int)x` 的"升级版"。
前台检查不过，编译就报错，不让你在运行时出神秘 bug。
```cpp
double pi = 3.14159;

// C 风格强转
int a = (int)pi;          // a = 3
// static_cast：<目标类型>(表达式)
int b = static_cast<int>(pi);  // b = 3

// 你把指针随便乱转，C 风格不吭声
int* p = (int*)&pi;           // ✅ 编译通过，但运行时可能一团糟
// static_cast 直接拒绝这种不相关的指针转换
int* q = static_cast<int*>(&pi);  // ❌ 编译错误！
// error: invalid 'static_cast' from type 'double*' to type 'int*'
```