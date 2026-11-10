

> 架构：**服务器 srv1 / srv2 跑 Isaac Sim（无头）→ WebRTC 推流 → 本机客户端显示**
> 本机与服务器在同一内网（实测 ping 2~3ms 直连），**不需要 SSH 隧道**，也不依赖公网。
>
> 📌 **两台服务器都装了完整环境、互为备用**：`srv1 = 10.10.223.11`，`srv2 = 10.10.223.10`
> （srv2 那份于 2026-09-21 从 srv1 复制）。某台显卡被占满时换另一台即可，
> **两台只有 Server IP 不同**（端口 `49100` / `47998` 相同）。

---

## 0. 一分钟速查

| 步骤        | 在哪执行        | 命令                                                      |
| --------- | ----------- | ------------------------------------------------------- |
| 1. 启动仿真服务 | 🖥️ **服务器** | `isaac-ctl start`                                       |
| 2. 查看服务状态 | 🖥️ **服务器** | `isaac-ctl status`                                      |
| 3. 打开客户端  | 💻 **本机**   | `~/opt/start-webrtc-client.sh`                          |
| 4. 填写连接参数 | 💻 **本机**   | Server 填**你启动的那台 IP** / Signal `49100` / Stream `47998` |
| 5. 停止仿真服务 | 🖥️ **服务器** | `isaac-ctl stop`                                        |
|           |             | `isaac-ctl start N` 指定卡号                                |

**连接参数（两台只有 Server 不同）：**

```
Server : 10.10.223.11  (srv1)    或    10.10.223.10  (srv2)
Signal : 49100      (TCP)     ← 两台相同
Stream : 47998      (UDP)     ← 两台相同
```

---

## 1. 环境信息

### 🖥️ 服务器 srv1 与 srv2（两台配置相同，各一份完整环境）

| 项目        | 值                                                                      |
| --------- | ---------------------------------------------------------------------- |
| 地址 / 账号   | `srv1` = `10.10.223.11`；`srv2` = `10.10.223.10`；账号均为 `tianhaojin`      |
| 系统        | Ubuntu 22.04.5 LTS，112 核，252GB 内存                                      |
| GPU       | 每台 RTX 5880 Ada ×2（各 48GB，算力 8.9）                                      |
| 驱动        | 595.71.05（两台相同）                                                        |
| 显卡约定      | 先 `isaac-ctl gpu` 看占用，**选空闲的卡**；`isaac-ctl start N` 指定卡号；**两台都满就换另一台** |
| conda     | `~/miniforge3`（conda 26.7.2）                                           |
| 环境名       | `isaacsim`（Python 3.12.14）                                             |
| Isaac Sim | 6.1.0.0 ｜ PyTorch 2.11.0+cu130｜每台约 27GB                                |
| 控制脚本      | `~/bin/isaac-ctl`（推荐）                                                  |
| 启动脚本      | `~/start_isaac_streaming.sh`（底层，一般不用直接调）                               |
| 运行日志      | `~/isaac_streaming.log`                                                |

### 💻 本机

| 项目 | 值 |
|---|---|
| 系统 | Ubuntu 24.04.4 LTS，IP `10.16.50.80` |
| 客户端 | Isaac Sim WebRTC Streaming Client 2.0.0（官方原生客户端） |
| 安装位置 | `~/opt/isaacsim-webrtc-client/`（用户级解包，未改动系统目录） |
| 启动脚本 | `~/opt/start-webrtc-client.sh` |
| 桌面入口 | 应用菜单搜 “Isaac Sim WebRTC Streaming Client” |

---

## 2. 启动流程

### 第 1 步：在服务器上启动 Isaac Sim 流式服务

> 🖥️ **【在服务器 srv1 或 srv2 上执行（二选一）】**

```bash
ssh tianhaojin@10.10.223.11     # srv1
# 或
ssh tianhaojin@10.10.223.10     # srv2

isaac-ctl start                 # 默认用 GPU 0 启动，并自动等待就绪
isaac-ctl start 1               # 如需指定 GPU 编号（先确认空闲！）
```

启动脚本会自动：检查是否已在运行 → 检查目标 GPU 是否被占用 → 拉起进程 → **轮询等待直到 `app ready`**（首次约 3 分钟，有 shader 缓存后约 70 秒）。

**就绪标志：**

```
[isaac-ctl] ✅ 已就绪！信令端口 49100 监听中（用时 70s）
     连接参数: Server=<本机IP>  Signal=49100  Stream=47998
```

> `<本机IP>` 由脚本自动探测：srv1 上显示 `10.10.223.11`，srv2 上显示 `10.10.223.10`。

### 第 2 步：在本机打开客户端并连接

> 💻 **【在本机上执行】**

```bash
~/opt/start-webrtc-client.sh
```

也可以在图形界面里直接打开（应用菜单搜索 “Isaac Sim WebRTC Streaming Client”）。

打开后填写：

| 输入框 | 填写内容 |
|---|---|
| **Server** | 你启动的那台：`10.10.223.11`（srv1）或 `10.10.223.10`（srv2） |
| **Signal Port** | `49100` |
| **Stream Port** | `47998` |
| **Resolution** | 按需选择（默认 1280x720；带宽充足可选 1920x1080） |

点击 **Connect**，正常约 5 秒内出画面。

---

## 3. 一键启停脚本 `isaac-ctl`（推荐）

> 🖥️ **【以下命令全部在服务器 srv1 或 srv2 上执行（二选一）】**

服务器上已安装控制脚本 `~/bin/isaac-ctl`，并已加入 `PATH`（新开 shell 可直接使用；若提示找不到命令，先执行 `source ~/.bashrc`）。

### 命令一览

| 命令 | 作用 |
|---|---|
| `isaac-ctl start [GPU]` | 启动服务并等待就绪（默认 GPU 0） |
| `isaac-ctl stop [秒数]` | 停止服务（先优雅 SIGTERM，默认 30s 超时后强制 SIGKILL） |
| `isaac-ctl restart [GPU]` | 重启（可顺便换卡） |
| `isaac-ctl status` | 查看进程 / 端口 / GPU / 日志状态 |
| `isaac-ctl logs` | 实时跟踪日志（Ctrl+C 退出） |
| `isaac-ctl gpu` | 查看 GPU 占用与使用者 |
| `isaac-ctl wait [秒数]` | 仅等待服务就绪 |

### 常用示例

```bash
# 启动（会自动等就绪）
isaac-ctl start

# 看状态（推荐常看）
isaac-ctl status

# 跑任务前先看显卡是谁在用
isaac-ctl gpu

# 重启并换到 GPU 1
isaac-ctl restart 1

# 停止并释放 GPU
isaac-ctl stop
```

### `isaac-ctl status` 输出示例

```
==================== Isaac Sim Streaming 状态 ====================
  服务状态 : ● 运行中
  进程 PID : 2121113
  运行时长 : 01:25
  内存占用 : 7.9 GB
------------------------------------------------------------------
  TCP 49100 (信令) : 监听中
  UDP 47998 (媒体) : 待客户端连接（正常）
------------------------------------------------------------------
  GPU 状态:
    0, NVIDIA RTX 5880 Ada Generation, 2339 MiB, 16 %
    1, NVIDIA RTX 5880 Ada Generation, 22856 MiB, 99 %
  GPU 上的进程:
    PID 2121113  用户 tianhaojin   显存 1995 MiB   .../isaacsim
    PID 1768790  用户 其他用户      显存 21946 MiB  python
```

> 💡 **注意**：`UDP 47998` 只在**有客户端连进来时**才会绑定，平时显示“待客户端连接（正常）”属于正常现象，不是故障。判断服务是否就绪，看 **TCP 49100** 和日志里的 `app ready`。

### 脚本的安全性设计

- **不会误杀别人的进程**：只匹配自己的 `isaacsim.exp.full.streaming` 进程。
- **启动前会检查 GPU**：若目标卡已用显存 > 2000 MiB，会列出占用进程并暂停 5 秒让你确认（可 Ctrl+C 取消）。
- **优雅停止**：先发 SIGTERM 等它自己退出（实测约 2 秒），超时才强制杀。
- **登出后仍存活**：用 `setsid + nohup` 完全脱离 SSH 会话，断开终端服务不会被杀。

---

## 4. systemd 用户服务（备选方案）

> 🖥️ **【在服务器 srv1 或 srv2 上执行（任选一台）】**

unit 文件已创建：`~/.config/systemd/user/isaac-streaming.service`（当前为 `disabled`）。

```bash
# 启用开机自启（登录后自动拉起）
systemctl --user enable isaac-streaming
systemctl --user start  isaac-streaming
systemctl --user status isaac-streaming

# 停止 / 禁用
systemctl --user stop    isaac-streaming
systemctl --user disable isaac-streaming

# 看日志
journalctl --user -u isaac-streaming -f
```

### ⚠️ 使用 systemd 方案的两个限制

1. **没有 root 权限**，无法安装系统级服务（`/etc/systemd/system` 不可写），只能用用户级 `systemctl --user`。
2. **`Linger=no`**：你**完全登出**（关掉所有 SSH 会话）后，systemd 会回收用户级服务。想让它像真正的守护进程一样常驻，需要管理员执行一次：
   ```bash
   sudo loginctl enable-linger tianhaojin
   ```
   **在拿到这个权限之前，推荐用第 3 章的 `isaac-ctl`**——它用 `setsid` 脱离会话，登出后依然存活，不依赖 systemd。

> 两种方式**不要同时用**，否则会争抢 49100 端口。

---

## 5. 停止 / 重启

### 🖥️ 服务器端（停止仿真）

```bash
isaac-ctl stop              # 推荐
isaac-ctl stop 60           # 自定义优雅停止超时为 60 秒

# 若脚本不可用时的兜底命令
pkill -f 'isaacsim.exp.full.streaming'
```

### 💻 本机（关闭客户端）

```bash
# 直接关窗口即可；或
pkill -f 'isaacsim-webrtc-streaming-client'
```

> ⚠️ **关闭客户端不会停止服务器上的 Isaac Sim**——流式服务会继续跑并占着 GPU。不用时请务必到服务器执行 `isaac-ctl stop`。

---

## 6. 服务器上的日常操作

> 🖥️ **【以下全部在服务器 srv1 或 srv2 上执行（二选一）】**

```bash
# 激活 Isaac Sim 环境（跑 python 脚本、调试时用）
source ~/miniforge3/etc/profile.d/conda.sh
conda activate isaacsim

# 跟踪服务日志
isaac-ctl logs
# 或
tail -f ~/isaac_streaming.log

# 查看 GPU 占用（等价于 isaac-ctl gpu）
nvidia-smi
```

### 换用另一张 GPU

服务器有 2 张卡，占用情况随时会变（可能有人在训练）。**先看再选**：

```bash
isaac-ctl gpu            # 看两张卡分别是谁在用

isaac-ctl restart 0      # 选 GPU 0
isaac-ctl restart 1      # 选 GPU 1（先确认这张卡空闲！）
```

> **本机两张卡都紧张时**：换另一台服务器（srv1 ↔ srv2）。客户端只需把 **Server** 改成另一台的 IP，
> 端口 `49100` / `47998` 不变。

---

## 7. 重要注意事项

### ⚠️ 1. 客户端目前以 `--no-sandbox` 运行

因为系统级安装需要 sudo 密码，客户端被解包到用户目录，无法给 Electron 的 `chrome-sandbox` 设置 SUID 位，所以只能靠 `--no-sandbox` 绕过。

如果希望恢复沙箱（更安全），执行一次系统级安装即可：

> 💻 **【在本机上执行】**

```bash
sudo dpkg -i /tmp/isaacsim-webrtc-client.deb
```

安装后可直接用 `/usr/bin/isaacsim-webrtc-streaming-client`（无需 `--no-sandbox`）。安装包 MD5 已校验为官方值 `07bd252432fb92b93bdb33b337455827`。

### ⚠️ 2. 不要用 `CUDA_VISIBLE_DEVICES` 给 Isaac Sim 指定显卡

Isaac Sim 的 RTX 渲染器与 `CUDA_VISIBLE_DEVICES` **不兼容**（官方日志明确警告会导致崩溃或异常）。启动脚本已处理：**改用 `--/renderer/activeGpu=N` 参数选卡**。如果你自己写启动命令，请沿用这个方式。

### ✅ 3. 42 个扩展加载失败 —— 已修复（2026-09-20）

之前启动日志里有 **42 个扩展** `Failed to startup python extension`（`isaacsim.core.api`、`isaacsim.core.prims`、
`isaacsim.sensors.*`、`isaacsim.robot.*` 等）。根因是 **`charset_normalizer` 的 `.py`/`.so` 混用造成二进制 ABI 冲突**，
已在 2026-09-20 修复：现在启动日志 **0 个加载失败、0 条 `[Error]`**。

`ROS2 Bridge` 也已通过 Isaac Sim **内置的 ROS 2 Humble 运行时**启用（**不需要系统装 ROS**）。

> 根因分析、修复步骤、备份/回滚方式详见本文 **第 10 章**。

### ⚠️ 4. 共享服务器规范

- **srv1 与 srv2 都是**多人共享（LDAP 集中认证），常年有人在跑任务，多人同时使用。
  两台可各跑一个 Isaac Sim 实例；某台紧张就**换另一台**。
- **跑任何 GPU 任务前先 `isaac-ctl gpu` 确认卡是否空闲**，别抢占别人的进程。
- **禁止 kill 别人的进程**。
- 磁盘 `/home` 是 **7.3T 全员共享、无配额**，请随手清理：
  ```bash
  ~/miniforge3/envs/isaacsim/bin/pip cache purge   # 清理 pip 缓存，可释放约 12GB
  ```
- 环境全部装在 `~/` 下，不污染系统，也不需要 sudo。

---

## 8. 故障排查

| 现象 | 原因 / 处理 |
|---|---|
| `isaac-ctl: command not found` | 新开 shell 或 `source ~/.bashrc`；脚本在 `~/bin/isaac-ctl` |
| 客户端一直转圈、连不上 | 服务器侧没到 `app ready`。查 `isaac-ctl status` 与 `tail ~/isaac_streaming.log`，确认 TCP 49100 在监听 |
| 客户端显示黑屏 | 等 Isaac Sim 完全加载（看 `app ready`）；确认 Resolution 与服务器视口匹配；重连一次 |
| `status` 里 UDP 显示“待客户端连接” | **正常现象**，客户端连上后才会绑定 |
| 服务器报 `CXXABI_1.3.15 not found` | conda 的 libstdc++ 未生效。启动脚本已通过 `LD_LIBRARY_PATH=$CONDA_PREFIX/lib` 修复 |
| `Skipping NVIDIA graphics device` | 误设了 `CUDA_VISIBLE_DEVICES`，改用 `--/renderer/activeGpu=N` |
| 日志刷 `charset_normalizer.md ... size changed`，或大量 `ImportError ... (unknown location)` / `KeyError: 'isaacsim.xxx'` | `charset_normalizer` 的 `.py`/`.so` ABI 冲突引起的**级联失败**（一个扩展 import 失败，后续扩展跟着失败）。修复见第 10 章 |
| `ROS2 Bridge startup failed` | 启动脚本已用 Isaac Sim **内置 Humble** 运行时（自动设 `ROS_DISTRO` / `RMW_IMPLEMENTATION` / `LD_LIBRARY_PATH`）。若复现，请确认走的是 `isaac-ctl` / `start_isaac_streaming.sh`，而非自写启动命令 |
| 日志刷 `NVST Error: NVST_R_BUSY` | 客户端异常断开残留的流会话，`isaac-ctl restart` 即可清掉 |
| 端口被占用 | 换端口需**两端同时改**：服务端加 `--/exts/omni.kit.livestream.app/primaryStream/signalPort=新端口`，客户端填相同端口 |
| 客户端报 SUID sandbox 错误 | 运行时加 `--no-sandbox`（启动脚本已包含） |

---

## 9. 关键文件清单

| 位置 | 文件 | 作用 |
|---|---|---|
| 🖥️ 服务器 | `~/bin/isaac-ctl` | **一键启停控制脚本（日常主要用它）** |
| 🖥️ 服务器 | `~/start_isaac_streaming.sh` | 底层启动脚本（被 isaac-ctl 调用） |
| 🖥️ 服务器 | `~/.config/systemd/user/isaac-streaming.service` | systemd 用户服务（备选，未启用） |
| 🖥️ 服务器 | `~/isaac_streaming.log` | 运行日志 |
| 🖥️ 服务器 | `~/miniforge3/envs/isaacsim/` | Isaac Sim 环境（约 27GB） |
| 🖥️ 服务器 | `~/README.md` | **服务器侧家目录导览**（共享账号给项目组看的入口） |
| 🖥️ 服务器 | `~/Isaac-Sim-远程操作说明.md` | **服务器侧完整操作说明**（面向项目组成员） |
| 🖥️ 服务器 | `~/isaac/logs/`、`~/isaac/backups/` | 流追踪/安装日志归档、修复前备份（确认无误可删） |
| 🖥️ 服务器 | `~/isaac/migrate/` | **迁移工具包**：`MIGRATION.md` + 安装/修复/验收脚本（换服务器时用） |
| 💻 本机 | `~/opt/start-webrtc-client.sh` | 客户端启动脚本 |
| 💻 本机 | `~/opt/isaacsim-webrtc-client/` | 客户端程序（用户级安装） |
| 💻 本机 | `~/.local/share/applications/isaacsim-webrtc-client.desktop` | 应用菜单入口 |

---

## 10. 已修复：42 个扩展加载失败（根因与修复）

> 记录时间：2026-09-20 ｜ 修复后：`Failed to startup python extension` = **0**，启动日志 `[Error]` = **0**

### 现象

启动日志出现 42 个扩展启动失败，错误高度一致：

```
ImportError: cannot import name 'Articulation' from 'isaacsim.core.experimental.prims' (unknown location)
ImportError: cannot import name 'Prim'        from 'isaacsim.core.experimental.prims' (unknown location)
KeyError: 'isaacsim.sensors.rtx'
AttributeError: module 'isaacsim.robot_motion.experimental.motion_generation' has no attribute ...
```

`(unknown location)` 很容易被误判成"命名空间包没注册"。实际上它是**第一个扩展 import 失败后留下的残缺模块**，
后续扩展再去 `from ... import ...` 就报 `(unknown location)` —— 本质是**级联错误**，真正的第一现场只有一个。

### 真正的根因

第一现场是 `charset_normalizer` 的二进制 ABI 冲突：

```
ValueError: charset_normalizer.md.MessDetectorPlugin size changed, may indicate binary incompatibility.
            Expected 24 from C header, got 16 from PyObject
```

触发链：

```
isaacsim.core.experimental.prims → scipy.spatial → ... → numpy.f2py → import charset_normalizer
```

Isaac Sim 进程里同时存在 **5 份 `charset_normalizer`**，且版本不一致；更关键的是，
Kit 的 **fast importer** 会优先缓存 `.py` 而不是编译好的 `.so`
（`isaacsim/kit/kernel/py/omni/ext/_impl/fast_importer.py` 中 `MODULES_EXTS = (".py", *EXTENSION_SUFFIXES)`）。

于是出现 **编译版 `cd.so` + 纯 Python `md.py`** 的混搭：`cd.so` 用 C 头文件里的结构体大小去校验 Python 对象，
纯 Python 类的 `tp_basicsize` 是 16 → 抛 `ValueError` → `charset_normalizer` 整体 import 失败 →
经 `numpy.f2py` / `scipy` 放大成 42 个扩展的级联失败。

各份副本本来版本就乱：

| 位置（相对 `site-packages/`） | 原版本 | 说明 |
|---|---|---|
| `charset_normalizer` | 3.3.2 | conda 环境（`isaacsim-kernel` 的 pin） |
| `isaacsim/extscache/omni.services.pip_archive-*/pip_prebundle` | 3.5.1 | NVIDIA 主 prebundle |
| `isaacsim/extscache/omni.cip.pip-*/pip_prebundle` | 3.5.1 | |
| `isaacsim/extscache/omni.ai.langchain.core-*/pip_core_prebundle` | 3.4.7 | |
| `isaacsim/extscache/omni.simready.content.browser-*/pip_prebundle` | 3.4.7 | **还误带了 Windows 的 `.pyd`** |

### 修复内容（全部在 `~/` 下，未用 sudo）

1. 环境里的 `charset_normalizer` 由 3.3.2 升级到 **3.5.1**（与 NVIDIA prebundle 对齐）：
   ```bash
   ~/miniforge3/envs/isaacsim/bin/pip install "charset_normalizer==3.5.1"
   ```
2. 把 `omni.ai.langchain.core` 与 `omni.simready.content.browser` 两个 prebundle 里的
   `charset_normalizer` 3.4.7（含误带的 Windows `.pyd`）替换为 **3.5.1**，使 **5 份版本完全一致**。
3. 在全部 5 份里把纯 Python 的 `md.py` / `cd.py` 改名为 `*.py.disabled`，
   **只保留编译好的 `md.cpython-312-*.so` / `cd.cpython-312-*.so`**。
   这样无论 fast importer 先缓存到哪一份，拿到的都是**同一套编译版**，不会再出现 `.py`/`.so` 混搭。
4. 启动脚本 `~/start_isaac_streaming.sh` 增加 ROS 2 **内置 Humble** 运行时环境变量
   （`ROS_DISTRO` / `RMW_IMPLEMENTATION` / `LD_LIBRARY_PATH`），
   使 `isaacsim.ros2.bridge` 正常 startup —— **不需要系统安装 ROS 2**。

### 备份与回滚

修复前的文件都已备份在服务器家目录：

| 备份 | 内容 |
|---|---|
| `~/isaac/backups/charset_normalizer-3.3.2/` | 原 conda 环境的 `charset_normalizer` 3.3.2 |
| `~/isaac/backups/charset_normalizer-prebundles/` | langchain / simready 原 3.4.7 prebundle |
| `~/isaac/backups/charset_normalizer-pyfiles/` | 5 份被改名为 `.disabled` 的 `md.py` / `cd.py` |
| `~/isaac/backups/start_isaac_streaming.sh.bak` | 原启动脚本 |

回滚：把 `*.py.disabled` 改回 `*.py`、恢复 3.4.7 prebundle 目录，再
`pip install "charset_normalizer==3.3.2"`。
> ⚠️ 回滚后 42 个失败会**复现**，仅用于紧急对照。

> ⚠️ 若将来执行 `pip install --force-reinstall "isaacsim[all,extscache]==6.1.0.0"` 重装，
> `extscache` 下的改动会被覆盖，需要按上面第 2、3 步重做。

### 已验证

- `isaac-ctl restart 0` → `app ready`；日志中 `Failed to startup python extension` = **0**，`[Error]` = **0**
- 关键扩展全部 startup：`isaacsim.core.api-5.3.2`、`isaacsim.core.experimental.prims-1.10.8`、
  `isaacsim.sensors.rtx-15.17.5`、`isaacsim.sensors.camera-1.7.13`、`isaacsim.robot.wheeled_robots-5.2.5`、
  `isaacsim.robot_motion.motion_generation-8.2.9`、`omni.simready.content.browser-0.3.8`、`isaacsim.ros2.bridge-5.1.4`
- WebRTC 流式仍正常：客户端 `video.readyState = 4`、分辨率正常、持续播放（`currentTime` 递增）

### 排查中被否定/排除的方向（备查）

- ❌ **线程超订阅（112 核）**：不是原因。失败是**确定的 import 错误**，不是初始化超时。
- ❌ **`$ISAAC_SIM_DIR` 未设置**：那是 NVIDIA 面向**源码构建**的约定，pip 安装不依赖它，与本次问题无关。
- ❌ **命名空间包路径未注册**：`(unknown location)` 是级联假象，不是根因。
- ⚠️ **不要用 `--/app/extensions/fastImporter/enabled=false` 去"修"**：虽能消掉 `charset_normalizer` 报错，
  但会让 `pxr` / `omni.usd` / `omni.kit.usd.layers` 等一组核心扩展解析失败（变成另外 38 个失败），
  且启动变得极慢（>450s 仍未 ready）。**保持 fast importer 开启**才和官方默认一致。

---

## 11. 换服务器 / 迁移（方案 A）

工具包的**源头（canonical）**在工作区 `~/AAAProjects/go2_ws/isaac-sim/migrate/`（约 60KB，自包含）；
服务器 `~/isaac/migrate/` 与 本机 `~/isaac/migrate/`（后者软链到工作区）都能直接用。
工作区里还带了资料副本：`isaac-sim/README.md`、`isaac-sim/docs/`。

> 🤖 **AI Agent 请直接读工具包里的 `AGENTS.md`** —— 那是可执行的自动搬迁运行手册（判定 / 命令 / 验收 / 失败处理）。

**方案 A（默认）**：新服务器用户名 / 家目录路径与旧机**相同** → 整目录 `rsync`（约 27GB）。

```bash
# 在【旧服务器】上执行
cd ~/isaac/migrate
./preflight_migrate.sh <新机IP> [用户名]            # 只读预检；退出码 0 = 可行
./migrate_route_a.sh <新机IP> [用户名] --dry-run    # 预演，不做改动
./migrate_route_a.sh <新机IP> [用户名]              # 正式迁移 + 自动验收
```

脚本自动完成：预检 → rsync `~/miniforge3` → 同步脚本/文档 → 新机 `~/.bashrc` 加 PATH → 新机启动验收。

**为什么必须路径相同**：conda 环境的 `bin/*` shebang 与 `conda-meta/*.json` 的 `prefix` 写死了绝对路径。
路径不同 → 用工具包里的 `bootstrap_isaacsim.sh`（**方案 B**，自动重装 + 修复），**不要硬拷**。

**注意**：新机要求驱动 ≥ 595.58.03、家目录 ≥ 60GB 可用、**不需要 sudo/docker**；
方案 A 会把已修好的 `extscache` 改动一起带走，**无需重做 42 扩展修复**（方案 B 重装后则由 `bootstrap` 自动重做）。

---

*文档更新：2026-09-20 ｜ 安装方式：官方 Python Environment (conda + pip) + 官方 WebRTC Streaming Client 2.0.0*
*2026-09-21：srv2 部署完成（由 srv1 复制），**两台现均可用、互为备用**；两台地址与端口见第 1 章*
*已验证：`isaac-ctl restart` 全流程可用（优雅停止约 2s，重启到就绪 25~70s；srv2 首次启动约 435s 含 shader 编译）；42 个扩展加载失败已修复（0 失败 / 0 Error）；端到端视频流 `readyState=4` 正常播放*