---
title: '分支与合并'
---

# Git 分支与合并

分支是 Git 最核心、最强大的功能之一，也是 Git 区别于许多旧版本控制系统的重要特性。它允许你从主开发线（如 `main` 分支）分离出来，在一个独立的环境中进行工作（例如开发新功能、修复 Bug），而不会影响到主线的稳定性。完成后，再将你的工作成果安全地合并回主线。

## 什么是分支？理解指针

想象一下你的项目提交历史是一条线性的节点序列。在 Git 中，一个**分支**本质上只是一个指向某个特定**提交 (commit)** 的轻量级**可移动指针**。

*   当你创建一个新分支时，Git 只是创建了一个新的指针，指向你当前所在的提交。
*   当你在这个新分支上进行提交时，只有这个新分支的指针会向前移动，其他分支指针保持不变。

Git 还有一个特殊的指针叫做 **`HEAD`**。`HEAD` 指向你**当前正在工作**的本地分支。当你切换分支时，实际上是移动 `HEAD` 指针，让它指向另一个分支指针，同时 Git 会更新你的**工作目录**，使其内容与 `HEAD` 指向的分支所对应的快照一致。

![Git 分支与指针](../images/git-tutorial/git-branches.png)

*分支本质上是指向特定提交的指针，而 HEAD 指针指向当前工作的分支*

这种基于指针的设计使得 Git 的分支操作极其**轻量级**和**快速**。创建、切换、合并分支几乎是瞬间完成的，鼓励开发者频繁使用分支。

默认情况下，Git 仓库的主分支名为 `main`（在较旧的 Git 版本或某些项目中可能仍然是 `master`）。

## 创建分支 (`git branch` / `git switch -c`)

1.  **创建新分支**:
    使用 `git branch <新分支名>` 命令可以创建一个新的分支指针，它会指向你当前所在的提交 (`HEAD` 指向的提交)。
    ```bash
    # 假设当前在 main 分支，创建一个名为 feature-login 的新分支
    git branch feature-login
    ```
    这**只创建**了分支指针，`HEAD` 仍然指向 `main`，你还在 `main` 分支上工作。

    ![Git 创建新分支](../images/git-tutorial/git-branch-create.png)

    *创建新分支时，只是添加了一个新指针，而 HEAD 仍然指向原来的分支*

2.  **创建并立即切换到新分支**:
    更常用的方式是使用 `git switch -c <新分支名>` (c 代表 create)：
    ```bash
    # 创建名为 feature-payment 的新分支，并立即切换过去
    git switch -c feature-payment
    ```
    这相当于执行了 `git branch feature-payment` 和 `git switch feature-payment` 两条命令。现在 `HEAD` 指向 `feature-payment` 分支。

    *(旧命令 `git checkout -b <新分支名>` 也能达到同样效果，但 `switch` 语义更清晰)*

## 切换分支 (`git switch`)

使用 `git switch <分支名>` 命令可以将 `HEAD` 指针移动到指定的分支，并更新工作目录。

```bash
# 从 feature-payment 切换回 main 分支
git switch main

# 再切换到之前创建的 feature-login 分支
git switch feature-login
```

![Git 切换分支](../images/git-tutorial/git-branch-switch.png)

*切换分支时，HEAD 指针移动到新的分支，工作目录也会相应更新*

**注意**: 切换分支前，请确保你的工作目录是干净的（没有未提交的修改），或者将修改提交或储藏 (`git stash`) 起来，否则 Git 可能会阻止你切换以防丢失修改。

## 查看与管理分支 (`git branch`)

*   **列出本地所有分支**:
    ```bash
    git branch
    ```
    当前 `HEAD` 指向的分支会用 `*` 标记。
*   **列出所有分支** (包括远程跟踪分支):
    ```bash
    git branch -a
    ```
*   **查看每个分支的最后一次提交**:
    ```bash
    git branch -v
    ```
*   **查看已合并到当前分支的分支**:
    ```bash
    git branch --merged
    ```
*   **查看尚未合并到当前分支的分支**:
    ```bash
    git branch --no-merged
    ```

## 合并分支 (`git merge`)

当你完成了某个特性分支（如 `feature-login`）的开发，并希望将其改动整合回主开发线（如 `main`）时，就需要进行**合并 (Merge)** 操作。

**合并步骤:**

1.  **切换到接收改动的分支** (目标分支，通常是 `main` 或 `develop`):
    ```bash
    git switch main
    ```
2.  **确保目标分支是最新状态** (如果是协作项目，先拉取远程更新):
    ```bash
    # git pull origin main  (如果与远程仓库协作)
    ```
3.  **执行合并命令**:
    ```bash
    git merge <要合并过来的分支名>
    ```
    例如，将 `feature-login` 合并到当前的 `main` 分支：
    ```bash
    git merge feature-login
    ```

**合并的两种主要情况:**

*   **快进合并 (Fast-forward)**: 如果目标分支 (`main`) 在你创建特性分支 (`feature-login`) 后**没有任何新的提交**，那么 Git 会执行一次“快进”合并。它只是简单地将目标分支 (`main`) 的指针直接移动到特性分支 (`feature-login`) 所指向的提交。没有新的合并提交产生，历史记录保持线性。

    ![Git 快进合并](../images/git-tutorial/git-merge-ff.png)

    *快进合并：当目标分支没有新提交时，只需要移动指针到特性分支的最新提交*

*   **三方合并 (Three-way Merge)**: 如果目标分支 (`main`) 在你创建特性分支后**也有了新的提交**，那么 Git 无法进行快进合并。它会执行一次“三方合并”。Git 会找到两个分支的**共同祖先提交**，以及两个分支各自的最新提交，将这三方的更改进行合并，并**创建一个新的合并提交 (Merge Commit)**。这个合并提交有两个父提交（分别是 `main` 和 `feature-login` 的最新提交）。

    ![Git 三方合并](../images/git-tutorial/git-merge-3way.png)

    *三方合并：当两个分支都有新提交时，需要创建一个新的合并提交，它有两个父提交*

## 解决合并冲突 (Merge Conflicts)

如果在两个待合并的分支上，修改了**同一个文件的同一区域**，Git 无法自动判断应该保留哪个版本，这时就会发生**合并冲突**。Git 会暂停合并过程，让你手动解决冲突。

**解决冲突步骤:**

1.  执行 `git merge` 后，Git 会在输出中提示哪些文件存在冲突，并且 `git status` 也会显示这些文件处于 "Unmerged paths" 状态。
2.  打开包含冲突的文件。Git 会在冲突区域用特殊的标记符标示出来：
    ```diff
    <<<<<<< HEAD
    这是当前分支 (例如 main) 的内容。
    =======
    这是来自被合并分支 (例如 feature-login) 的内容。
    >>>>>>> feature-login
    ```
    `<<<<<<< HEAD` 到 `=======` 之间是当前分支 (`HEAD` 指向的分支) 的内容。
    `=======` 到 `>>>>>>> <分支名>` 之间是被合并分支的内容。
3.  **手动编辑文件**：你需要仔细阅读冲突部分的代码，决定最终想要保留的内容。这可能意味着：
    *   保留 `HEAD` 的版本。
    *   保留被合并分支的版本。
    *   结合两者的内容。
    *   完全重写这部分代码。
    **完成后，务必删除所有冲突标记符 (`<<<<<<<`, `=======`, `>>>>>>>`)**。
4.  **标记为已解决**: 在解决了文件中的所有冲突并保存后，使用 `git add` 将该文件标记为已解决冲突状态：
    ```bash
    git add <包含冲突的文件名>
    ```
    如果你有多个冲突文件，需要对每个文件都执行此操作。
5.  **提交合并结果**: 当所有冲突都解决并通过 `git add` 标记后，执行 `git commit` 来完成合并。Git 通常会自动生成一个合并提交信息（如 "Merge branch 'feature-login' into main"），你可以直接使用它，或者用 `-m` 提供自定义信息。
    ```bash
    git commit
    # 或者 git commit -m "合并 feature-login，手动解决登录逻辑冲突"
    ```

**使用合并工具 (`git mergetool`)**:
对于复杂的冲突，可以使用图形化的合并工具。配置好工具后，运行 `git mergetool`，它会为每个冲突文件启动工具，帮助你更直观地解决冲突。

## 删除分支 (`git branch -d`)

当一个特性分支的工作已经完成并且成功合并到目标分支后，通常这个特性分支就不再需要了，可以将其删除以保持仓库整洁。

```bash
# 删除已合并的 feature-login 分支
git branch -d feature-login
```

*   `-d` (小写 d, `--delete`) 选项只能删除**已经被完全合并**到当前 `HEAD` 所在分支的分支。这是一种安全措施，防止意外丢失未合并的工作。
*   如果你确定要删除一个**尚未合并**的分支（例如，你放弃了这个功能的开发），需要使用 `-D` (大写 D, `--delete --force`) 选项强制删除：
    ```bash
    # 强制删除未合并的 feature-abandoned 分支
    git branch -D feature-abandoned
    ```
    **强制删除前请务必确认不会丢失重要工作！**

## 常见分支策略：特性分支工作流

一种非常流行且推荐的工作流程是**特性分支 (Feature Branch)** 工作流：

1.  从主开发分支（如 `main` 或 `develop`）创建一个新的特性分支来开发一个新功能或修复一个 Bug。分支名应具有描述性（如 `feature/user-authentication`, `bugfix/issue-123`）。
2.  在该特性分支上进行开发、提交。
3.  开发完成后，将特性分支合并回主开发分支。
4.  删除已合并的特性分支。

这种方式可以保持主分支的整洁和稳定，让多人协作和并行开发更加顺畅。

---

分支是 Git 的精髓所在。理解并熟练运用分支和合并是进行高效、安全的版本控制和团队协作的关键。