## 以后怎么做（就三条）

1. **换 example 之前先重启 Isaac** —— `~/bin/isaac-ctl restart`，然后客户端重新 Connect 再 load。这一个动作就能规避闪退。
2. **别用 Stop → Play 来"重跑一遍"** —— 要重来就用 `isaac-ctl restart`，或在面板里重新按 **LOAD**（不是 Play）。
3. **客户端闪退先看服务端日志** —— `tail -50 ~/isaac_streaming.log`；客户端只是下游受害者。