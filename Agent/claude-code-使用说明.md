# Claude Code 使用说明

## 环境配置

本机已通过 litellm 代理将 Claude Code 接入 DeepSeek V4 API。

### 启动流程

```bash
# 1. 启动 litellm 代理（每次使用前需运行）
bash ~/start-claude-code.sh

# 2. 运行 Claude Code
claude
```

环境变量已写入 `~/.bashrc`（`source ~/.bashrc` 或重开终端即可生效）：
- `ANTHROPIC_BASE_URL=http://localhost:4000`
- `ANTHROPIC_API_KEY=REDACTED_REVOKED_KEY`

---

## 模型切换

| 模式 | DeepSeek 模型 | 命令 |
|------|-------------|------|
| ⚡ Flash 快速模式 | deepseek-v4-flash | `claude --model claude-sonnet` |
| 🔬 Pro 专家/专业模式 | deepseek-v4-pro | `claude --model claude-opus-4-8`（默认） |

交互式会话中用 `/model` 命令切换：
```
/model claude-sonnet   → 切到 Flash
/model claude-opus-4-8 → 切到 Pro
```

---

## 常用命令

### 启动方式

```bash
claude                         # 交互式会话
claude --print "问题"          # 单次问答（非交互）
claude --model claude-sonnet   # 使用 Flash 模型
claude --model claude-opus-4-8 # 使用 Pro 模型
claude -c                      # 继续上次会话
```

### 交互式命令

| 命令 | 功能 |
|------|------|
| `/help` | 查看帮助 |
| `/model <名称>` | 切换模型 |
| `/effort low/medium/high/max` | 设置思考深度 |
| `/compact` | 压缩对话历史 |
| `/clear` | 清屏 |
| `/resume` | 恢复历史会话 |
| `/bug` | 报告问题 |

---

## 使用示例

### 代码相关

```bash
# 解释代码
claude --print "解释当前目录下的 main.py"

# 查找 bug
claude --print "检查 src/ 目录下的代码问题"

# 重构
claude --print "重构 utils.js 中的 fetchData 函数"
```

### Git 相关

```bash
claude --print "分析最近的 git 提交"
claude --print "帮我写 commit message"
```

### 文件操作

Claude Code 会自动读取项目结构，可直接要求它：
- "创建文件 src/utils/helper.ts"
- "修改 config.json 中的端口为 8080"
- "删除 dist/ 目录"

---

## 注意事项

1. **每次使用前必须启动 litellm 代理**：`bash ~/start-claude-code.sh`
2. 代理进程若中断，Claude Code 会报连接错误，重启代理即可
3. DeepSeek V4 模型按 token 计费，注意控制使用量
4. 项目根目录可创建 `CLAUDE.md` 文件，写入项目说明供 Claude Code 参考
