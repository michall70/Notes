# 参考 · pydantic 使用指南

> 适用：pydantic v2（KISS-ICP 1.3.0 用的就是它）
> 验证环境：`~/anaconda3/envs/pyslam`

---

## 1. 一句话

> **pydantic 是"格式检查器"**：你先把"我要什么格式"写成类，它负责在你填值时**逐项检查、自动补默认值、类型不对就当场报警**。

它**不是**语言特性，就是一个普通的第三方库。没有魔法。

---

## 2. 最小用法：三步

```python
from pydantic import BaseModel

class LidarConfig(BaseModel):          # ① 继承 BaseModel
    sensor_name: str                   # ② 声明字段
    max_range: float = 30.0
    deskew: bool = False

c = LidarConfig(sensor_name="utlidar") # ③ 造对象，它会自动检查
print(c)              # sensor_name='utlidar' max_range=30.0 deskew=False
print(c.max_range)    # 30.0
print(c.model_dump()) # {'sensor_name': 'utlidar', 'max_range': 30.0, 'deskew': False}
```

**关键点**：你省掉的 `__init__` 和 `self.xxx = xxx`，是 pydantic 替你生成的。

---

## 3. 字段声明规则

| 写法 | 含义 |
|---|---|
| `name: str` | **必填**（不给默认值就是必填） |
| `name: str = "abc"` | 可不填，默认 `"abc"` |
| `name: str \| None = None` | 可不填；填了必须是 str 或 None |

```python
LidarConfig()      # -> ValidationError: sensor_name  Field required
```

### 常用类型

```python
x: int
x: float
x: bool
x: str
x: list[int]
x: dict[str, float]
x: int | None = None            # Python 3.10+ 写法
x: Optional[int] = None         # 老写法，等价（需 from typing import Optional）
```

---

## 4. 加范围限制（`Field`）

```python
from pydantic import Field

class LidarConfig(BaseModel):
    max_range: float = Field(default=30.0, gt=0, le=200)
    port: int = Field(default=9999, ge=1024, le=65535)
```

| 参数 | 含义 |
|---|---|
| `gt` | 大于 |
| `ge` | 大于等于 |
| `lt` | 小于 |
| `le` | 小于等于 |

```python
LidarConfig(max_range=500)
# -> ValidationError: Input should be less than or equal to 200
```

还可以：`min_length` / `max_length`（字符串、列表长度）、`pattern`（正则）。

---

## 5. ⚠️ 赋值时默认**不**校验（最大的坑）

```python
class Loose(BaseModel):
    max_range: float = 30.0

l = Loose()
l.max_range = "我是一个字符串"      # 不报错！！！
```

要补上，加**一行**：

```python
from pydantic import ConfigDict

class Strict(BaseModel):
    model_config = ConfigDict(validate_assignment=True)   # ← 就这一行

    max_range: float = 30.0
```

```python
s = Strict()
s.max_range = "错的"        # -> ValidationError
print(s.max_range)          # 30.0  （旧值没被打坏，它拒绝了这次赋值）
```

**记住：构造会校验，赋值默认不校验。**

---

## 6. 嵌套模型

```python
class RegistrationConfig(BaseModel):
    max_num_iterations: int = 500
    convergence_criterion: float = 0.0001


class KISSConfig(BaseModel):
    out_dir: str = "results"
    registration: RegistrationConfig = RegistrationConfig()   # ← 值本身是个模型
```

访问：`cfg.registration.max_num_iterations`

**KISS-ICP 的 config 就是这么组织的**：4 个子组 + `out_dir`。

---

## 7. 常用的三个方法

```python
cfg = LidarConfig(sensor_name="utlidar")

cfg.model_dump()                 # -> 变成普通字典
cfg.model_dump_json()            # -> 变成 JSON 字符串
LidarConfig.model_validate({...}) # -> 从字典造对象（也会校验）
```

**看一个模型有哪些字段**：

```python
LidarConfig.model_fields
# {'sensor_name': FieldInfo(annotation=str, required=True), ...}
```

这是"审接口"最有用的一招。

---

## 8. 从环境变量读（`BaseSettings`）

KISS-ICP 的 `KISSConfig` 继承的是 `BaseSettings` 而不是 `BaseModel`：

```python
class KISSConfig(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="kiss_icp_")
```

意思是：可以用环境变量覆盖配置，前缀是 `kiss_icp_`：

```bash
export kiss_icp_data__max_range=30.0     # 双下划线表示嵌套
```

**一般用不到，知道有这回事就行。**

---

## 9. 什么时候该用 pydantic

| 场景 | 该用吗 |
|---|---|
| 从 **yaml / json / 环境变量** 读配置 | ✅ 非常合适 |
| 从 **socket / 网络** 收到数据要检查 | ✅ 合适 |
| 命令行参数 | ✅ 合适（比 argparse 省事） |
| 项目内部函数之间传的参数 | ❌ 别用，手写 `if` 更简单 |
| 只是检查一个变量是不是数字 | ❌ 别用，`isinstance` 三行搞定 |

**pydantic 的价值在"声明一次，处处生效"** —— 你有 10 个配置项、要在 5 个地方用，它才划算。

---

## 10. pydantic ≠ 静态类型检查

这是两件事，别混：

| | pydantic | mypy / Pylance / pyright |
|---|---|---|
| **什么时候检查** | **运行**时（造对象那一刻） | **写代码时**，根本不运行 |
| **管什么范围** | 只有继承 `BaseModel` 的类 | **整个项目**，所有类型注解 |
| **管普通函数吗** | ❌ 管不到 | ✅ 管得到 |
| **报错时机** | 程序跑到那儿才炸 | 编辑器里就标红 |
| **需要装吗** | 要装 pydantic | mypy 要装；**Pylance 在 VSCode 里默认就有** |

**你的 VSCode 里那个 Pylance，就是静态检查器。** 之前 `import rclpy` 下面那条波浪线，就是它在工作。

---

## 11. 命令速查

```bash
# 检查单个文件
mypy your_script.py

# 检查整个目录（会递归）
mypy your_package/

# 库没有类型标注时，别让 mypy 抱怨
mypy --ignore-missing-imports your_script.py

# 慢慢严格起来的第一步
mypy --check-untyped-defs your_script.py
```

pydantic 自带 print 检查结果：直接 `python your_script.py` 跑到报错就行。

---

## 12. 一句话总结

> **pydantic = 运行时的"守门员"，只管它接管的那几个类。**
> **mypy / Pylance = 编译期的"校对员"，管你整个项目的注解，但一行代码都不运行。**
>
> **两者互补，不互相替代。**