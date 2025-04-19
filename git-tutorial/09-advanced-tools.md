---
title: '高级工具'
---

# Git 高级工具：Stash, Bisect, Worktree, Aliases, Attributes

除了历史管理，Git 还提供了一系列高级工具来优化工作流程、诊断问题和自定义行为。

## 储藏 (Stashing): 临时保存工作现场

当你正在处理某个任务，但需要紧急切换到另一个分支（例如修复 Bug），而当前的工作尚未完成，不适合提交时，`git stash` 可以将你**工作目录**和**暂存区**的未提交更改（已跟踪文件）暂时保存起来，让工作区恢复到 `HEAD` 提交时的干净状态。

**核心场景:**

1.  **紧急切换任务:** 正在开发 `feature-X`，突然需要去 `main` 修复 Bug。
    ```bash
    # 在 feature-X 分支
    git stash save "WIP: implementing X part 1" # 保存当前进度
    git switch main
    # ... 修复 bug, commit, push ...
    git switch feature-X
    git stash pop # 恢复之前保存的进度
    ```
2.  **拉取更新遇阻:** `git pull` 时提示本地有未提交的更改会与远程冲突。
    ```bash
    git stash # 储藏本地更改
    git pull # 拉取并合并远程更新
    git stash pop # 尝试重新应用本地更改，并解决可能出现的冲突
    ```

**常用操作:**

*   储藏当前更改: `git stash` 或 `git stash save "描述信息"`
*   查看储藏列表: `git stash list` (显示如 `stash@{0}`, `stash@{1}`)
*   应用**最近的**储藏并从列表中**移除**: `git stash pop`
*   应用**指定的**储藏但**保留**在列表中: `git stash apply stash@{n}`
*   查看指定储藏的内容: `git stash show stash@{n}`
*   查看指定储藏的详细差异: `git stash show -p stash@{n}`
*   删除指定的储藏: `git stash drop stash@{n}`
*   删除所有储藏: `git stash clear`
*   储藏**包括未跟踪**的文件: `git stash -u` 或 `git stash --include-untracked`
*   储藏**所有**文件 (包括忽略文件): `git stash -a` 或 `git stash --all`

## 二分查找 (`git bisect`): 快速定位引入 Bug 的提交

当发现一个 Bug，只知道它在某个旧版本是好的 (`good`)，在新版本是坏的 (`bad`) 时，`git bisect` 可以通过二分查找的方式，自动或半自动地帮你快速定位到**第一个**引入该 Bug 的提交。

**场景:** 项目部署后出现问题，你知道一周前的版本 `v1.0` 正常，当前 `HEAD` 有问题，期间有上百个提交。

**工作流程:**
1.  `git bisect start` # 开始二分查找
2.  `git bisect bad HEAD` # 标记当前版本有问题
3.  `git bisect good v1.0` # 标记一个已知无问题的版本
4.  Git 会自动检出历史记录中间的一个提交。
5.  **测试当前检出的版本**是否有问题。
6.  告知 Git 测试结果: `git bisect good` (如果这个版本没问题) 或 `git bisect bad` (如果这个版本有问题)。
7.  Git 会根据你的反馈，继续检出剩余范围的中间提交，重复步骤 5-6。
8.  最终，Git 会输出第一个被标记为 `bad` 的提交，这就是引入问题的提交。
9.  `git bisect reset` # 结束查找，回到原来的分支。

`git bisect` 可以极大地缩短调试回归 Bug 的时间。

## 工作树 (`git worktree`): 同时检出多个分支到不同目录

允许你在**同一个仓库**中，将**不同的分支**同时检出到**不同的文件系统目录**下，而无需克隆多次仓库或频繁 `stash` 和 `switch`。

**场景:** 你正在 `feature-A` 分支开发复杂功能，需要紧急修复 `hotfix` 分支的问题，但不想打断 `feature-A` 的环境（如编译产物、依赖）。

**操作:**
```bash
# 假设当前在 project/ 目录下
# 在 project/ 同级目录下创建 hotfix-build 目录，并检出 hotfix 分支
git worktree add ../hotfix-build hotfix

# 现在你可以 cd ../hotfix-build/ 并在那里独立工作，
# 同时 project/ 目录仍然是 feature-A 分支的环境。

# 查看当前的工作树
git worktree list

# 完成 hotfix 工作后，清理工作树 (确保没有未提交更改)
# 先切换出 hotfix-build 目录
cd ../project
git worktree remove ../hotfix-build
# 或者如果 hotfix 分支也删除了，可以用 prune
# git worktree prune
```

## Git 别名 (Aliases): 提升命令效率

Git 别名允许你创建自定义的简写命令，用于替代长度复杂的 Git 命令，从而显著提高工作效率。

**创建别名的两种方式:**

1. **使用 Git 配置命令**:
   ```bash
   git config --global alias.co checkout
   git config --global alias.br branch
   git config --global alias.ci commit
   git config --global alias.st status
   ```

2. **直接编辑 Git 配置文件** (`.gitconfig`):
   ```ini
   [alias]
       co = checkout
       br = branch
       ci = commit
       st = status
       unstage = reset HEAD --
       last = log -1 HEAD
       visual = !gitk
       lg = log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
   ```

**使用示例:**
* 简化的分支切换: `git co main` 代替 `git checkout main`
* 一目了然的状态查看: `git st` 代替 `git status`
* 撤销暂存: `git unstage file.txt` 代替 `git reset HEAD file.txt`
* 查看漂亮的日志图表: `git lg` 代替复杂的 log 格式化命令

**高级别名技巧:**
* 使用 `!` 可以运行外部命令或创建包含多个命令的别名:
  ```bash
  git config --global alias.visual '!gitk'
  git config --global alias.publish '!git push -u origin $(git branch --show-current)'
  git config --global alias.cleanup '!git branch --merged | grep -v "^*" | xargs git branch -d'
  ```

别名是个人工作流优化的重要部分，可以根据自己常用的命令和工作习惯创建适合自己的别名集合.

## Git 属性 (Attributes): 自定义文件处理规则

Git 属性允许你为特定文件或文件模式指定特殊处理规则，例如合并策略、差异比较方式、过滤器等。这些规则在 `.gitattributes` 文件中定义。

**常见使用场景:**

1. **行尾处理**: 确保跨平台项目中的行尾统一
   ```gitattributes
   # 自动将文本文件的行尾标准化为 LF
   * text=auto

   # 指定特定文件类型使用 LF
   *.js text eol=lf
   *.py text eol=lf

   # 指定二进制文件 (不进行任何转换)
   *.png binary
   *.jpg binary
   ```

2. **自定义差异显示**:
   ```gitattributes
   # 指定使用特定工具比较 Word 文档差异
   *.docx diff=word

   # 告诉 Git 将特定文件视为 JSON，使用 JSON 差异工具
   *.json diff=json
   ```

3. **合并策略指定**:
   ```gitattributes
   # 指定某些文件在合并冲突时使用 ours 策略 (保留当前分支的版本)
   database.xml merge=ours

   # 标记某些文件不应该合并 (如配置模板)
   config.template.json -merge
   ```

4. **标记导出归档时排除的文件**:
   ```gitattributes
   # 创建项目归档时不包含测试文件和文档
   tests/ export-ignore
   docs/ export-ignore
   .gitattributes export-ignore
   ```

**配置示例**: 设置 `word` 差异工具
```bash
git config diff.word.textconv catdoc
```

Git 属性是项目级别的元数据配置，能够确保团队所有成员以统一的方式处理特定文件，对于管理混合文本和二进制文件的项目特别有用.