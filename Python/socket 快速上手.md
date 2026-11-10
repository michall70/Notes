# 参考 · socket 快速上手

> **这一页只讲"有哪些 API 可以用"，具体怎么拼是你的活。**
> 目标：把点云从一个 Python 进程，搬到另一个 Python 进程。

---

## 1. 协议（已经定死，两边必须一模一样）

```
┌──────────────┬────────────────────────────┐
│   4 字节      │         L 个字节            │
│  L（小端）     │         payload            │
└──────────────┴────────────────────────────┘
       ↑
   "说明书"：我后面有 L 个字节
```

| 项         | 值                                               |
| --------- | ----------------------------------------------- |
| 地址        | `127.0.0.1:9999`                                |
| 传输        | TCP                                             |
| `L`       | `uint32`，**小端**，= payload 的字节数                  |
| `payload` | `float32`，x y z 三列**连续**排列                      |
| 点数        | `L / 12`（1 个 float32 = 4 字节，1 个点 = 3 个 float32） |

**谁当服务端**：算法侧（receiver）当**服务端**，**必须先启动**。
**谁当客户端**：ROS 侧（sender）当**客户端**，后启动。

---

## 2. API 清单

### 2.1 造 socket / 收尾（两边一样）

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)    # IPv4 + TCP
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  # 可选：重启时不用等 TIME_WAIT
sock.close()                                                # 用完关掉
```

### 2.2 服务端（receiver）

```python
sock.bind(("127.0.0.1", 9999))    # 占住这个门牌号。已被占用会抛 OSError
sock.listen(1)                    # 开始听。1 = 允许 1 条连接排队等着被 accept。不阻塞

conn, addr = sock.accept()        # 【阻塞】等一个客人来
                                  # 返回：conn = 专用 socket；addr = 对端地址，如 ('127.0.0.1', 50310)
```

- `conn` 是**专线**，之后所有收发都用它，**它知道对面是谁**
- `sock` 只是个"门"，accept 完了就可以不管它了

### 2.3 客户端（sender）

```python
sock.connect(("127.0.0.1", 9999))   # 连过去
                                    # 对面没人监听 -> ConnectionRefusedError
sock.sendall(...)
sock.recv(...)
```

### 2.4 收发

```python
sock.sendall(b"...")        # 发 bytes。内部保证全部发完（不要用 send，它可能只发一部分）

chunk = sock.recv(4096)     # 【最多】收 4096 字节 —— 可能更少，也可能阻塞不动
                            # 返回 b"" 表示对面把连接关了
```

> ⚠️ `recv(n)` **不保证**给你 n 个字节。要"收满 n 字节"得自己写循环。

### 2.5 超时（调试时很有用）

```python
sock.settimeout(5.0)        # 之后阻塞操作超过 5 秒，抛 socket.timeout
```

### 2.6 整数 <-> 字节

```python
b = (1440000).to_bytes(4, "little")      # 整数 -> 4 字节
n = int.from_bytes(b, "little")          # 4 字节 -> 整数
```

### 2.7 numpy

```python
arr = np.load("0000.npy")                    # 先 print(arr.shape, arr.dtype) 看清楚它是啥
print(arr.shape, arr.dtype)                  #   ← 这行别省

payload = arr.astype("<f4").tobytes()        # -> bytes（具体该转成什么形状，按协议来）
pts = np.frombuffer(data, dtype="<f4")       # bytes -> 一维数组
pts = pts.reshape(-1, 3)                     # 一维 -> (N, 3)
```

- `"<f4"` = 小端 float32（`<` 小端，`>` 大端）
- `.tobytes()` 要求数组在内存里**连续**；从切片来的数组可能不连续，必要时先 `.copy()`

### 2.8 看一眼地址

```python
sock.getsockname()   # 我这边是谁
sock.getpeername()   # 对面是谁（listen 的那个 socket 上没有对端，会报错）
```

---

## 3. 骨架（TODO 留给你）

### `bridge/cloud_receiver.py` —— 算法侧，服务端

用 **conda 的 pyslam 环境**跑。

```python
import socket
import pathlib
import numpy as np

HOST, PORT = "127.0.0.1", 9999
SAVE_DIR = pathlib.Path("/home/michall/AAAProjects/go2_ws/bridge/from_ros")


def recv_exact(sock, n):
    """从 sock 收满 n 个字节再返回。收不满就继续收；对面关了抛 ConnectionError。"""
    # TODO ① 你来写（5 行）
    ...


def main():
    SAVE_DIR.mkdir(parents=True, exist_ok=True)

    srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind((HOST, PORT))
    srv.listen(1)
    print(f"[receiver] 蹲在 {HOST}:{PORT} 等着…（应该先启动我）")

    conn, addr = srv.accept()
    print(f"[receiver] 连上了: {addr}")

    count = 0
    while True:
        # TODO ② 收 4 字节头
        # TODO ③ 还原 L
        # TODO ④ 收 L 字节 payload
        # TODO ⑤ payload -> (N, 3) float32
        # TODO ⑥ 打印一行：第几帧 / L / N
        # TODO ⑦ 存到 SAVE_DIR / f"{count:04d}.npy"
        # TODO ⑧ count += 1
        ...


if __name__ == "__main__":
    main()
```

### `bridge/cloud_sender.py` —— 客户端（**这一步还不碰 ROS**）

先用 `bridge/*.npy` 当数据源，把 socket 打通再说。

```python
import socket
import pathlib
import time
import numpy as np

HOST, PORT = "127.0.0.1", 9999
SRC_DIR = pathlib.Path("/home/michall/AAAProjects/go2_ws/bridge")


def main():
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # TODO ① connect 过去

    frames = sorted(SRC_DIR.glob("*.npy"))
    for path in frames:
        arr = np.load(path)
        print(f"[sender] {path.name}: shape={arr.shape} dtype={arr.dtype}")
        # TODO ② arr -> 协议要的 payload（bytes）
        # TODO ③ 拼上头：L.to_bytes(4, "little") + payload
        # TODO ④ sendall 出去
        # TODO ⑤ 打印一行，sleep(1)

    # TODO ⑥ 收尾


if __name__ == "__main__":
    main()
```

---

## 4. 怎么跑

**终端 A —— 服务端，先起：**

```bash
source ~/anaconda3/etc/profile.d/conda.sh
conda activate pyslam
cd ~/AAAProjects/go2_ws/bridge
python cloud_receiver.py
```

**终端 B —— 客户端，后起：**

```bash
cd ~/AAAProjects/go2_ws/bridge
python3 cloud_sender.py
```

> 注意：服务端在 **conda pyslam** 里跑，客户端用系统 `python3` 就行 —— 这一步还不涉及 ROS。

---

## 5. 调试手段

```bash
ss -ltn | grep 9999        # 有没有人在听？（服务端起没起来）
ss -tn  | grep 9999        # 连接建上了没？
lsof -p <pid> | grep -i sock   # 看某个进程手上有哪些 socket
```

---

## 6. 撞墙对照表

| 现象 | 大概是 |
|---|---|
| `ConnectionRefusedError: [Errno 111]` | 服务端（receiver）还没启动 —— 顺序反了 |
| `TypeError: a bytes-like object is required, not 'str'` | 你 send 了字符串或 ndarray，得先 `.tobytes()` |
| 帧数看着对，但每帧点数乱跳 / 报错 | 忘了 `recv_exact`，直接用了 `recv` |
| 程序卡在 `recv` 不动 | 对面还没发；或对面已经关了而你没判断 `b""` |
| `OSError: [Errno 98] Address already in use` | 上次没关干净，加上 `SO_REUSEADDR` |
| 存出来的形状不对 | `np.frombuffer` 出来是**一维**的，要 `.reshape(-1, 3)` |
| 数字大得离谱（1440000 变成 16323840） | 字节序不一致，两边没对齐 |
| 传得通、存得对，但 KISS-ICP 读不进去 | 想想它 `read_npy` 里那行 `raise ValueError` |

---

## 7. 打通之后，下一步

把 `cloud_sender.py` 里"从 `npy` 读一帧"换成"从 ROS 订阅拿到一帧 msg"，**其余一个字不用改**。

那就是 `my_first_pkg/kitti_sub.py` 的活儿了。