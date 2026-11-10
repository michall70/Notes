
好，那我从零讲。你后面写这个节点，其实只用得着三件事：**造一个地址**、**往那儿写**、**从那儿读**。

先讲地址。

---

## 一、pathlib：把"路径"当成一个对象

以前大家写路径是拼字符串：

```python
path = "/home/michall/go2_ws/bridge/" + name + ".npy"
```

这样拼容易出错，斜杠漏一个就废了。`pathlib` 的想法是：**路径本身也可以是一个对象，它知道自己是什么。**

```python
from pathlib import Path

d = Path("/home/michall/AAAProjects/go2_ws/bridge/clouds")
```

就这一行，`d` 就是一个路径对象。

**拼下一层**，用 `/` —— 对，就是除号：

```python
f = d / "000.npy"          # /home/.../clouds/000.npy
```

这个 `/` 挺好用，你想拼几层就拼几层，斜杠它自己会加。

---

**它身上有几个属性，是拿来"切"自己的：**

```python
Path("/a/b/x.npy").name      # 'x.npy'      文件名（带后缀）
Path("/a/b/x.npy").stem      # 'x'          文件名（不带后缀）
Path("/a/b/x.npy").suffix    # '.npy'       后缀
Path("/a/b/x.npy").parent    # '/a/b'       上一层
```

你存文件的时候会用上 `stem`，比如想从 `000.npy` 反推出它对应第几帧。

---

**建目录：**

```python
d.mkdir(parents=True, exist_ok=True)
```

两个参数都得说一句。

`parents=True`：如果 `/a/b/c` 里的 `b` 也不存在，一路帮你建出来。不加就报错。

`exist_ok=True`：目录已经在了也别报错，当无事发生。

（这俩不加的话，目录已存在会给你一个 `FileExistsError`。）

---

**列目录里有什么：**

```python
sorted(x.name for x in d.glob("*.npy"))
```

`glob("*.npy")` 就是"这个目录下所有名字以 `.npy` 结尾的"。拿到的是路径对象，取 `.name` 就是文件名。

---

## ⚠️ 这里有个坑，你现在就会踩

```python
Path("~/AAAProjects/go2_ws/bridge/clouds").exists()   # False ！
```

**`~` 它不认。** `~` 是 shell 的东西，Python 不知道那是"我的家目录"。

三种解法，挑一个：

```python
Path("~/a/b").expanduser()      # 手动展开
Path.home() / "a/b"             # 从家目录开始拼
Path("/home/michall/a/b")       # 直接写死全路径
```

我建议你用 `Path.home()` 那个，又短又清楚。

---

## 二、numpy 存文件

就两个函数：

```python
np.save(path, array)      # 存
array = np.load(path)     # 读
```

**它的好处是：存进去什么样，读回来就什么样。**

形状、数据类型全都记着。你存 `(123397, 4) float32`，读回来就是 `(123397, 4) float32`，不用你自己转。

---

**但是有个非常烦人的不对称，我特意去你机器上验了一遍：**

```python
np.save(d / "frame", arr)      # 存的时候：我帮你自动补上 .npy ✅
                               # → 生成 frame.npy

np.load(d / "frame")           # 读的时候：我不补 ❌
                               # → FileNotFoundError
```

**存会自动补后缀，读不会。**

所以规矩就是：**你自己老老实实把 `.npy` 写上**，别指望它。

```python
np.save(d / "000.npy", arr)
```

---

## ⚠️ 第二个坑：文件名的排序

这个坑你今天可能遇不到，但**下节课一定会遇到**，先给你打个预防针。

假设你存了 12 帧，名字是 `0.npy` 到 `11.npy`。你 `sorted()` 一下，猜猜什么顺序？

按字符串排的，不是按数字。

> `'0.npy'` → `'1.npy'` → **`'10.npy'`** → `'11.npy'` → `'2.npy'` → ...

**第 10 帧跑到第 2 帧前面去了。** 拿这个顺序去跑 KISS-ICP，轨迹直接乱掉。

办法就是**补零**，让所有名字一样长：

```python
f"{i:03d}.npy"     # 0 → 000.npy,  1 → 001.npy,  10 → 010.npy
```

这样字符串顺序就等于数字顺序了。`03d` 的意思是"当三位数的整数，不够就补 0"。

**记住了：凡是文件名带序号，一律补零。**

---

## 三、送你一个彩蛋

我拆了一个 `.npy` 文件给你看：

```
前 6 字节:   \x93NUMPY        ← 魔法数，证明"我是个 npy"
接着:        版本号、头部长度
前 128 字节: {'descr': '<f4', 'shape': (5, 4), ...}   ← 说明书！
第 128 字节之后: 真正的数据
```

**看明白了吗？**

> `.npy 文件 = 128 字节的说明书 + 光溜溜的数据`

**这是我们这几课第五次看到同一个套路了。**

CDR 消息配 `msgtype`、mcap 配 `metadata.yaml`、`PointCloud2.data` 配 `fields`、rosbag 目录配 `metadata.yaml`……

现在 `.npy` 也配了个头。

**把"是什么"和"怎么解释"分开存，是这些工具的共同长相。**

---

API 就这些，够你用了。现在你去写那个节点吧，**卡住了再来找我。**