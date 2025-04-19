---
title: '标签与发布'
---

# Git 标签与发布管理

在项目开发过程中，标记重要的里程碑，尤其是软件发布版本，是非常关键的。Git 提供了标签 (Tag) 功能来实现这一目标。

## 标签 (Tagging): 标记重要的里程碑

标签 (Tag) 是指向特定提交的**固定引用**，通常用于标记项目历史中的重要节点，最常见的用途是标记**软件发布版本**（如 `v1.0`, `v2.1.3-beta`）。

**类型:**

*   **轻量标签 (Lightweight Tag):** 只是一个指向特定提交的指针（像一个不会移动的分支名）。创建简单：
    ```bash
    # 为当前 HEAD 创建轻量标签
    git tag v1.0-lw
    # 为特定提交创建轻量标签
    git tag v0.9-lw <commit-hash>
    ```
*   **附注标签 (Annotated Tag):** 是存储在 Git 数据库中的完整对象。它包含标签创建者、日期、注释信息，并且可以进行 GPG 签名以验证。**强烈推荐用于正式发布。**
    ```bash
    # 为当前 HEAD 创建附注标签
    git tag -a v1.0 -m "正式发布版本 1.0"
    # 为特定提交创建附注标签并签名
    git tag -s v1.0 -m "正式发布版本 1.0" <commit-hash>
    ```

**常用操作:**

*   查看所有标签: `git tag` 或 `git tag -l "v1.*"` (支持通配符)
*   查看特定标签信息 (包括附注信息): `git show <tagname>`
*   **推送标签到远程**: 默认 `git push` 不会推送标签。
    *   推送单个标签: `git push origin <tagname>`
    *   推送所有本地标签: `git push origin --tags`
*   删除本地标签: `git tag -d <tagname>`
*   删除远程标签 (需要先删除本地标签): `git push origin --delete <tagname>`
*   检出标签 (进入 "detached HEAD" 状态): `git checkout <tagname>`

标签是管理软件发布版本和标记重要历史节点的有效方式。附注标签提供了更丰富的元数据，是正式发布的推荐选择。