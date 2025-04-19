---
title: '维护与性能优化'
---

# Git 维护与性能优化

随着项目增长，Git 仓库可能会变得臃肿，影响性能。以下是一些维护和性能优化技巧：

## 垃圾收集与压缩

```bash
# 手动运行垃圾收集，移除不需要的文件和优化本地仓库
git gc

# 更积极的优化（较慢但更彻底）
git gc --aggressive

# 检查仓库大小和内容
git count-objects -v
```

## 修剪远程跟踪分支

```bash
# 删除对应远程仓库已经不存在的远程跟踪分支
git remote prune origin

# 在 fetch 时自动执行修剪
git fetch --prune
```

## 压缩/修复仓库

```bash
# 检查仓库完整性
git fsck --full

# 在仓库损坏时尝试修复
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

## 减小仓库体积

```bash
# 查找大文件 (示例，可能需要调整)
git rev-list --objects --all | git cat-file --batch-check='%(objectname) %(objecttype) %(objectsize) %(rest)' | sed -n 's/^blob //p' | sort --numeric-sort --key=2 | tail -n 10

# 使用过滤分支移除大文件（谨慎使用，会重写历史！）
# 推荐使用 BFG Repo-Cleaner 或 git-filter-repo 等更安全的工具
# git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch path/to/large/file' --prune-empty --tag-name-filter cat -- --all
```

## 配置优化

```bash
# 启用并行索引预载（多核CPU优化）
git config --global core.preloadindex true

# 启用文件系统缓存
git config --global core.fscache true

# 放宽时间戳分辨率检查 (可能提高某些文件系统下的性能)
git config --global core.trustctime false

# 加速状态检查（适用于文件数量极多的仓库）
git config --global feature.manyFiles true
```

## 分支管理优化

```bash
# 删除已合并的本地分支 (保留 main/master 和 develop)
git branch --merged | grep -vE "(^\*|main|master|develop)" | xargs git branch -d

# 查找并清理过时的远程跟踪分支 (需要确认远程分支确实已删除)
# git branch -r --merged origin/main | grep -v main | sed 's/origin\///' | xargs -I% git push origin :%
```

## 配置钩子使仓库保持健康

```bash
# 创建 post-merge 钩子自动清理已合并分支
cat > .git/hooks/post-merge << 'EOF'
#!/bin/sh
git branch --merged | grep -vE "(^\*|main|master|develop)" | xargs git branch -d
EOF
chmod +x .git/hooks/post-merge
```

定期维护 Git 仓库，特别是大型或历史悠久的项目，有助于保持良好的性能和响应速度。