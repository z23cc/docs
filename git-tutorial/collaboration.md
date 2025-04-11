# Git 团队协作流程

当多个人在同一个项目上工作时，需要一套规范的流程来确保代码的有效管理和集成。Git 提供了灵活的基础，可以支持多种协作模式。以下介绍几种常见的流程。

## 集中式工作流 (Centralized Workflow)

这种工作流类似于传统的版本控制系统（如 SVN）。团队成员都从一个中央仓库（通常是 `origin`）克隆代码，并在本地进行修改。

1.  **克隆**: `git clone <中央仓库URL>`
2.  **修改**: 在本地工作目录进行开发。
3.  **提交**: `git add .` 然后 `git commit -m "提交信息"`
4.  **拉取更新**: 在推送前，先拉取中央仓库的最新代码并合并，确保本地代码是基于最新版本的：`git pull origin main` （假设主分支是 `main`）。解决可能出现的冲突。
5.  **推送**: 将本地提交推送到中央仓库：`git push origin main`

**优点**: 简单易懂，适合小团队或从 SVN 过渡来的团队。
**缺点**: 所有人都直接向主分支推送，可能导致主分支不稳定或频繁冲突。

## 功能分支工作流 (Feature Branch Workflow)

这是非常流行的一种工作流。核心思想是，任何新功能的开发或 Bug 修复都在专门的分支上进行，而不是直接在主分支（如 `main`）上操作。

1.  **创建分支**: 基于最新的 `main` 分支创建一个新的功能分支：
    ```bash
    git switch main
    git pull origin main  # 确保 main 是最新的
    git switch -c feature/新功能名称
    ```
2.  **开发**: 在功能分支上进行开发、提交。
    ```bash
    # ... 修改代码 ...
    git add .
    git commit -m "开发新功能..."
    ```
3.  **推送分支**: 将功能分支推送到远程仓库，方便备份和协作（如果需要他人Review）：
    ```bash
    git push -u origin feature/新功能名称
    ```
4.  **创建合并请求 (Pull Request / Merge Request)**: 当功能开发完成，通过 GitHub/GitLab 等平台创建一个合并请求，请求将 `feature/新功能名称` 分支合并回 `main` 分支。
5.  **代码审查与讨论**: 团队成员审查代码，提出修改意见。开发者根据反馈继续在功能分支上修改并推送。
6.  **合并**: 代码审查通过后，由项目维护者或开发者通过平台界面将功能分支合并到 `main` 分支。
7.  **删除分支**: 合并完成后，可以删除远程和本地的功能分支。
    ```bash
    # 在平台上删除远程分支
    git switch main
    git pull origin main # 更新本地 main
    git branch -d feature/新功能名称 # 删除本地分支
    ```

**优点**:
*   `main` 分支始终保持稳定和可发布状态。
*   代码审查流程清晰。
*   并行开发更容易，减少冲突。

## Gitflow 工作流 (Gitflow Workflow)

Gitflow 是一种更复杂但结构化更强的分支模型，适用于需要维护多个发布版本的大型项目。它定义了更严格的分支角色：

*   **`main` (或 `master`)**: 永远代表生产就绪状态，只包含发布版本的代码。只接受来自 `release` 或 `hotfix` 分支的合并。
*   **`develop`**: 代表下一个发布版本的最新开发状态。所有功能分支都从 `develop` 创建，并合并回 `develop`。
*   **`feature/*`**: 用于开发新功能，从 `develop` 分支创建，完成后合并回 `develop`。
*   **`release/*`**: 用于准备新的生产发布。从 `develop` 分支创建。在这个分支上只进行 Bug 修复、文档生成等发布相关的任务。完成后，**同时合并到 `main` 和 `develop`**，并在 `main` 分支上打上版本标签。
*   **`hotfix/*`**: 用于紧急修复生产环境 `main` 分支上的 Bug。从 `main` 分支创建，完成后**同时合并到 `main` 和 `develop`**，并在 `main` 分支上打上修复后的版本标签。

**优点**: 分支职责清晰，适合管理复杂的发布周期。
**缺点**: 流程相对复杂，对于简单项目可能过于繁琐。

## Forking 工作流 (Forking Workflow)

这种工作流在开源项目中非常常见。贡献者不是直接向原始仓库推送代码，而是先将原始仓库“Fork”（复刻）到自己的账户下，然后在自己的 Fork 仓库中进行修改。

1.  **Fork**: 在 GitHub/GitLab 等平台上，点击原始仓库的 "Fork" 按钮，创建一份属于自己的副本仓库。
2.  **克隆**: 将**自己的** Fork 仓库克隆到本地：
    ```bash
    git clone https://github.com/你的用户名/forked-repo.git
    ```
3.  **添加上游仓库**: 为了方便同步原始仓库的更新，添加原始仓库作为远程仓库，通常命名为 `upstream`：
    ```bash
    git remote add upstream https://github.com/原始作者/original-repo.git
    ```
4.  **创建分支**: 在本地基于 `main`（或其他基础分支）创建功能分支：
    ```bash
    git switch main
    git pull upstream main # 从原始仓库拉取最新代码
    git switch -c feature/我的贡献
    ```
5.  **开发与提交**: 在功能分支上进行开发和提交。
6.  **推送分支**: 将功能分支推送到**自己的** Fork 仓库：
    ```bash
    git push -u origin feature/我的贡献
    ```
7.  **创建合并请求 (Pull Request)**: 在 GitHub/GitLab 上，从你的 Fork 仓库的 `feature/我的贡献` 分支向原始仓库的 `main` 分支发起 Pull Request。
8.  **讨论与合并**: 原始仓库的维护者审查代码，讨论修改，最终决定是否合并你的贡献。

**优点**:
*   对原始仓库的写入权限控制非常严格，适合大规模、开放的协作。
*   贡献者可以在自己的 Fork 中自由实验。

选择哪种工作流取决于项目的规模、团队的习惯以及发布需求。对于大多数现代项目，**功能分支工作流**是一个很好的起点。