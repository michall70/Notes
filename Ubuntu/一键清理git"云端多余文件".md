放在仓库根目录

```bash
#!/bin/bash
# 一键清理"云端多余文件":
# 把被 .gitignore 排除但仍被 git 追踪的文件从索引移除并推送到远程。
# 只影响 git 追踪, 不删除磁盘文件。
set -e
cd "$(dirname "$0")"

echo "==> 将被移除追踪的文件:"
git ls-files -ci --exclude-standard
echo ""

echo "==> [1/3] 从索引移除(磁盘文件保留)"
git ls-files -ci --exclude-standard -z | xargs -0 git rm --cached
echo ""

echo "==> [2/3] 提交删除"
git commit -m "Remove generated/ignored files from tracking" || echo "(无变更可提交)"
echo ""

echo "==> [3/3] 推送到远程"
git push origin "$(git branch --show-current)"
echo ""
echo "完成。远程仓库中这些文件已被删除。"

```