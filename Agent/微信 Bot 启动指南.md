
项目目录：`~/AAAProjects/dsh-wechat`

---
## 一、开机后怎么启动
先启动dsh：
``` bash
npx @deepseek-ai/dsh web
```

前台运行，关掉终端就停：

```bash
cd ~/AAAProjects/dsh-wechat
./start.sh dsh
```

推荐后台常驻：

```bash
cd ~/AAAProjects/dsh-wechat
setsid nohup ./start.sh dsh > /tmp/wx-bridge.log 2>&1 < /dev/null &
```

或用 tmux（方便随时看日志）：

```bash
tmux new -s wx
cd ~/AAAProjects/dsh-wechat && ./start.sh dsh
# 脱离：Ctrl+B 然后 D；重新进入：tmux attach -t wx
```

启动成功后日志里会出现 `[weixin] monitor started`，之后在微信里发消息即可。

---

## 二、日常命令

| 命令 | 作用 |
|------|------|
| `./start.sh` | 启动（默认 agent=dsh） |
| `./status.sh` | 查看登录状态、进程、最近日志 |
| `./stop.sh` | 停止桥接 |
| `tail -f /tmp/wx-bridge.log` | 实时看日志（后台启动方式） |
| `./login.sh` | 重新扫码登录 |
| `./logout.sh` | 退出登录、清除本地凭据 |
| `./accounts.sh` | 查看已登录账号 |

微信里发 `/clear` 可以重置当前对话的上下文。

---

## 三、开机自启（systemd user 服务，可选）

仓库里已经准备好 unit 文件：`systemd/dsh-wechat.service`

安装：

```bash
mkdir -p ~/.config/systemd/user
cp ~/AAAProjects/dsh-wechat/systemd/dsh-wechat.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now dsh-wechat

# 让服务在未登录桌面时也能自启（需要一次权限确认）
loginctl enable-linger $USER
```

管理：

```bash
systemctl --user status dsh-wechat     # 状态
systemctl --user restart dsh-wechat    # 重启
systemctl --user stop dsh-wechat       # 停止
journalctl --user -u dsh-wechat -f     # 日志（也写进 bridge.log）
```

> 注意：unit 里的 `node` / `dsh` 路径是按当前机器写死的（nvm 和 npx 缓存）。
> 如果升级了 Node 或清了 npx 缓存，需要同步改 unit 里的 `PATH` 和 `DSH_BIN`。

---

## 四、注意事项

1. **登录凭据会持久保存**在 `~/.openclaw/openclaw-weixin/`，正常重启电脑后直接 `./start.sh` 即可，不用重新扫码。

2. **token 可能失效**：日志出现 `errcode -14`、`session expired` 或提示重新登录时，跑一次 `./login.sh` 重新扫码即可。

3. **不要重复启动同一账号**：同一账号跑多个桥接进程会争抢消息长轮询，导致重复回复或丢消息。启动前先用 `./status.sh` 确认。

4. **网络要求**：
   - 要能访问 `https://ilinkai.weixin.qq.com`（微信侧）。
   - `dsh` 要能访问 DeepSeek API。

5. **dsh 依赖 npx 缓存**：`start.sh` 默认依次找 `$DSH_BIN` → PATH 里的 `dsh` → `npx @deepseek-ai/dsh`。
   如果哪天启动报找不到 dsh，重新装一下：`npx -y @deepseek-ai/dsh@0.1.5-rc.2 --version`。

6. **安全**：桥接会把 dsh 的权限请求**自动选第一个选项（通常是允许）**，也就是 agent 能在工作目录里自主执行命令。
   默认工作目录是 `~/AAAProjects/dsh-wechat/workspace`。
   建议按需限制：
   ```bash
   WX_WORKSPACE=~/某个安全目录 ./start.sh dsh
   ```
   **不要**把工作目录设成 `~`、`/` 等敏感范围。

7. **上下文与进程**：桥接进程重启后，微信侧每个会话的 ACP session 会重建（对话上下文丢失），
   但 `workspace/` 里的文件是持久保存的。

8. **日志清理**：后台方式日志写 `/tmp/wx-bridge.log`（重启会丢）；systemd 方式写 `bridge.log`（会一直追加），
   太大时手动清：`: > ~/AAAProjects/dsh-wechat/bridge.log`。

---

## 五、当前配置速览

- 微信账号：`f2e51bd6ed23-im-bot`
- 桥接链路：微信 → weixin-agent-sdk → weixin-acp → ACP → `dsh --profile acp` → DeepSeek
- 默认工作目录：`~/AAAProjects/dsh-wechat/workspace`
- 启动 agent：`dsh --profile acp`（可用 `./start.sh codex` 等切换，需对应 agent 可用）