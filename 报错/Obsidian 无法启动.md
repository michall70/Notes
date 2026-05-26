---
tags: [troubleshooting, obsidian, linux]
---

# Obsidian 无法启动 - 速查

## AppImage 版 (Ubuntu)

### 症状
`chmod +x` 后双击 / 运行无反应

### 关键步骤
1. **安装 libfuse2**  
   `sudo apt install libfuse2`
2. **重新赋予权限**  
   `chmod +x Obsidian-*.AppImage`
3. **测试运行**  
   `./Obsidian-*.AppImage`

### 其他可能修复
- **沙盒问题** (Ubuntu 24.04+)：  
  `./Obsidian.AppImage --no-sandbox`
- **改用 deb 包**（推荐）：  
  从官网下载 `.deb`，直接安装
