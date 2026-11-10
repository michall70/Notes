

> 部署日期：2026-09-24　状态：**5 个服务全部运行中，空闲显存 0**
> 服务端文档：srv14 上的 `~/ai/README.md`（更详细的技术细节）

---

## 一、一句话上手

**所有模型统一走一个入口**，OpenAI 兼容格式：

| 项目  | 值                          |
| --- | -------------------------- |
| 地址  | `http://10.10.220.70:4000` |
| 密钥  | `sk-srv14-local`           |

```bash
curl http://10.10.220.70:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-srv14-local" \
  -H 'Content-Type: application/json' \
  -d '{"model":"gemma-4-12b","messages":[{"role":"user","content":"你好"}]}'
```

任何支持"自定义 OpenAI 接口"的应用（Continue、Dify、Open WebUI、LangChain…）直接填上面这两个值就能接入。

---

## 二、可用的 5 个模型

| 模型名             | 能力               | 显存     | 备注           |
| --------------- | ---------------- | ------ | ------------ |
| `gemma-4-12b`   | 文本 / 图像 / **音频** | 10 GB  | 综合最强         |
| `minicpm-v-4.6` | 文本 / 图像          | 5 GB   | 1.3B，极快      |
| `soprano-80m`   | 语音合成             | 0（CPU） | 32kHz，最快     |
| `moss-tts-nano` | 语音合成             | 0（CPU） | 48kHz，18 种音色 |
| `audio8-tts`    | 语音合成             | 0（CPU） | 44.1kHz，音色克隆 |

---

## 三、调用示例

### 1. 纯文本

```bash
curl http://10.10.220.70:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-srv14-local" -H 'Content-Type: application/json' \
  -d '{"model":"gemma-4-12b","messages":[{"role":"user","content":"用一句话解释什么是显存"}]}'
```

### 2. 图像理解（两个模型都支持）

```bash
# 把图片转 base64 塞进 image_url
IMG=$(base64 -w0 /path/to/photo.jpg)

curl http://10.10.220.70:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-srv14-local" -H 'Content-Type: application/json' \
  -d "{\"model\":\"gemma-4-12b\",\"messages\":[{\"role\":\"user\",\"content\":[
        {\"type\":\"text\",\"text\":\"描述这张图\"},
        {\"type\":\"image_url\",\"image_url\":{\"url\":\"data:image/jpeg;base64,$IMG\"}}]}]}"
```

### 3. 音频理解（**只有 `gemma-4-12b` 支持**）

```bash
AUD=$(base64 -w0 /path/to/audio.wav)

curl http://10.10.220.70:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-srv14-local" -H 'Content-Type: application/json' \
  -d "{\"model\":\"gemma-4-12b\",\"messages\":[{\"role\":\"user\",\"content\":[
        {\"type\":\"text\",\"text\":\"这段音频说了什么\"},
        {\"type\":\"input_audio\",\"input_audio\":{\"data\":\"$AUD\",\"format\":\"wav\"}}]}]}"
```

### 4. 语音合成

```bash
curl http://10.10.220.70:4000/v1/audio/speech \
  -H "Authorization: Bearer sk-srv14-local" -H 'Content-Type: application/json' \
  -d '{"model":"soprano-80m","input":"要合成的文本","voice":"alloy"}' \
  -o out.wav
```

- `moss-tts-nano` 的 voice 可填 `Junhao` / `Zhiming` / `Weiguo` / `Trump` / `Ava` 等
- `audio8-tts` 的 voice 用 `default`

### Python 示例

```python
from openai import OpenAI

client = OpenAI(base_url="http://10.10.220.70:4000/v1", api_key="sk-srv14-local")

r = client.chat.completions.create(
    model="gemma-4-12b",
    messages=[{"role": "user", "content": "你好"}],
)
print(r.choices[0].message.content)
```

---

## 四、⚠️ 明天测试前必读的三件事

### 1. 第一次调用会慢几秒

模型**空闲 5 分钟后会自动卸载**，把显存还给同事。所以：

- **第一次调用**：需要几秒加载模型（冷启动）
- **之后连续调用**：很快（Gemma 约 1 秒，MiniCPM 约 0.4 秒）

这是刻意设计的 —— 不用的时候显存占用是 **0**。

### 2. thinking 已默认关闭，但可以开

Gemma 4 默认会"先思考再回答"，问三个字能烧掉 272 个 token。已默认关掉。

**如果要更强的推理能力**，调用时加上：

```json
{"model": "gemma-4-12b", "reasoning_effort": "high", "...": "..."}
```

### 3. 如果服务器重启过，服务不会自动起

这台机器没有配开机自启（需要管理员权限）。**重启后要手动执行**：

```bash
ssh srv14
~/ai/bin/start-ollama.sh && ~/ai/bin/start-tts.sh && ~/ai/bin/start-gateway.sh
```

只有 SSH 断开是**不影响**的（进程用 nohup 托管）。

---

## 五、状态检查

```bash
ssh srv14 '~/ai/bin/status.sh'
```

输出示例：

```
服务           端口   进程     健康检查
litellm          4000     running    OK
ollama           11500    running    OK
soprano          9001     running    OK
moss             9002     running    OK
audio8           9003     running    OK

--- Ollama 已加载模型 (空闲 5 分钟后自动卸载) ---
  (无模型驻留, 显存已释放)

index, name, memory.used, memory.total
0, NVIDIA RTX PRO 5000 Blackwell, 24 MiB, 48935 MiB
1, NVIDIA RTX PRO 5000 Blackwell, 242 MiB, 48935 MiB
```

**显存 24/242 MiB 是正常的**（只是桌面），说明模型已卸载、没占资源。
调用模型时会临时涨到约 21 GB，5 分钟后自动回落。

---

## 六、资源占用（为什么不影响别人）

| | 值 |
|---|---|
| 空闲显存 | **≈ 0** |
| 使用时显存 | ≤ 21 GB |
| 占用哪张卡 | **只有 GPU1**（GPU0 完全不碰） |
| 磁盘 | `~/ai` 共 19 GB |

⚠️ **不要动共享的 Ollama 服务**：系统另有一个共享 Ollama 跑在 `0.0.0.0:11434`（管理员配的，里面有别人的 `deepseek-r1:32b`）。
本套服务用的是**独立的私有实例**（端口 `11500`、独立模型目录、仅本机监听），两者互不干扰。

如果你要用 `ollama` 命令，**必须显式指定端口**，否则会连到别人的服务上：

```bash
OLLAMA_HOST=127.0.0.1:11500 ~/ai/tools/ollama/bin/ollama list
```

---

## 七、常用运维命令

```bash
~/ai/bin/status.sh          # 健康总览 + 已加载模型 + 显存
~/ai/bin/start-ollama.sh    # 起私有 Ollama
~/ai/bin/start-tts.sh       # 起 3 个 TTS
~/ai/bin/start-gateway.sh   # 起 LiteLLM 网关
~/ai/bin/stop-all.sh        # 全部停止（立刻释放所有显存）
```

日志目录：`~/ai/logs/`

---

## 八、出问题时的排查顺序

1. **连不上 4000 端口** → `ssh srv14 '~/ai/bin/status.sh'` 看网关是否 running
2. **网关 OK 但某模型报错** → 看 `~/ai/logs/litellm.log`
3. **回答为空** → 大概率是 thinking 把 token 预算吃完了，加 `"reasoning_effort":"none"`，或把 `max_tokens` 调大
4. **全部 500** → 大概率服务器重启过，按第四节第 3 条重新拉起服务
5. **要彻底重启** → `~/ai/bin/stop-all.sh` 然后按顺序重新 start

---

## 附：模型是怎么部署的（备份信息）

- 私有 Ollama v0.34.4：`~/ai/tools/ollama/bin/ollama`
- Ollama 模型库：`~/ai/models-ollama/`（gemma4 12B Q4_K_M 7.6GB + minicpm-v-4.6 1.6GB）
- 量化 GGUF 源文件：`~/ai/models-gguf/MiniCPM-V-4.6/`
- TTS 权重与服务代码：`~/ai/models/`、`~/ai/services/`
- 网关配置：`~/ai/etc/litellm.yaml`
- 启动脚本：`~/ai/bin/`

**为什么用 Ollama 而不是 vLLM**：vLLM 为高并发设计，启动就预分配 85% 显存（两个实例常驻 78.7GB）。
本场景是单人低频使用，用不到并发能力。改用 Ollama 后空闲显存为 0，代价是单流速度从 47 tok/s 降到约 27 tok/s。