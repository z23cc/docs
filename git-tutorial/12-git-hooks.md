---
title: 'Git 钩子'
---

# Git 钩子：自动化你的工作流

Git 钩子（Hooks）是一种强大的自动化机制，允许你在 Git 工作流的特定事件点触发自定义脚本。通过钩子，你可以实现代码质量检查、自动化部署、工作流强制执行等多种功能，大幅提升团队协作效率和代码质量。

## 钩子的工作原理

Git 钩子是位于 `.git/hooks` 目录中的可执行脚本。当特定的 Git 事件发生时，Git 会自动查找并执行相应的钩子脚本。如果脚本返回非零值，相应的 Git 操作通常会被中止。

钩子脚本可以用任何可执行的脚本语言编写，如 Bash、Python、Ruby、Node.js 等。Git 默认提供了一系列 `.sample` 后缀的示例脚本，你可以通过移除 `.sample` 后缀来启用它们。

```bash
# 查看可用的钩子示例
ls -la .git/hooks
```

## 客户端钩子 vs 服务器端钩子

Git 钩子分为两大类：

### 客户端钩子

在本地仓库中执行，主要用于开发者的本地工作流。

#### 提交工作流钩子

1. **pre-commit**: 提交前执行，常用于代码风格检查、静态分析、单元测试等。
   ```bash
   #!/bin/bash
   # 运行代码风格检查
   eslint .

   # 如果 eslint 返回错误，阻止提交
   if [ $? -ne 0 ]; then
     echo "代码风格检查失败，请修复后再提交"
     exit 1
   fi
   ```

2. **prepare-commit-msg**: 在默认提交信息生成后、编辑器启动前执行，用于自定义提交信息模板。
   ```bash
   #!/bin/bash
   # 在提交信息中添加当前分支名
   BRANCH_NAME=$(git symbolic-ref --short HEAD)
   echo "[${BRANCH_NAME}] $(cat $1)" > $1
   ```

3. **commit-msg**: 提交信息编辑后执行，用于验证提交信息格式。
   ```bash
   #!/bin/bash
   # 检查提交信息是否符合约定式提交规范
   commit_msg=$(cat $1)
   pattern="^(feat|fix|docs|style|refactor|perf|test|chore)(\(.+\))?: .+"

   if ! [[ $commit_msg =~ $pattern ]]; then
     echo "提交信息不符合规范，请使用格式: type(scope): message"
     exit 1
   fi
   ```

4. **post-commit**: 提交完成后执行，通常用于通知或记录。
   ```bash
   #!/bin/bash
   # 提交后发送通知
   commit_hash=$(git rev-parse HEAD)
   commit_msg=$(git log -1 --pretty=%B)
   curl -X POST "https://api.example.com/notify" -d "hash=$commit_hash&message=$commit_msg"
   ```

#### 其他客户端钩子

1. **pre-rebase**: 变基前执行，可用于防止对已推送的提交进行变基。
2. **post-checkout**: 切换分支或克隆后执行，可用于调整工作目录环境。
3. **post-merge**: 合并完成后执行，可用于更新工作目录中的非版本控制文件。
4. **pre-push**: 推送前执行，可用于最终验证或阻止推送。
   ```bash
   #!/bin/bash
   # 推送前运行测试
   npm test

   if [ $? -ne 0 ]; then
     echo "测试失败，推送已取消"
     exit 1
   fi
   ```

### 服务器端钩子

在远程仓库中执行，用于服务器端的策略执行和自动化。

1. **pre-receive**: 服务器接收推送前执行，可用于验证所有推送的引用。
2. **update**: 类似 pre-receive，但针对每个被推送的分支单独执行。
3. **post-receive**: 推送完成后执行，常用于触发 CI/CD 流程、自动部署、通知等。
   ```bash
   #!/bin/bash
   # 推送到主分支后自动部署
   while read oldrev newrev refname; do
     branch=$(git rev-parse --symbolic --abbrev-ref $refname)
     if [ "$branch" = "main" ]; then
       echo "部署生产环境..."
       ssh user@production.server "cd /var/www/app && git pull"
     fi
   done
   ```

## 钩子共享与版本控制

Git 钩子目录 (`.git/hooks`) 不会随仓库克隆而复制，这使得钩子共享变得困难。解决方案包括：

### 1. 脚本安装机制

在项目中创建一个 `hooks` 目录，存放钩子脚本，并提供安装脚本：

```bash
#!/bin/bash
# install-hooks.sh
cp -a hooks/. .git/hooks/
chmod +x .git/hooks/*
```

团队成员克隆仓库后运行此脚本即可安装钩子。

### 2. 使用 Git 模板

Git 模板允许你定义新仓库的默认内容：

```bash
# 设置模板目录
git config --global init.templateDir ~/.git-templates
mkdir -p ~/.git-templates/hooks

# 添加钩子到模板
cp your-hook ~/.git-templates/hooks/
chmod +x ~/.git-templates/hooks/*
```

之后创建或克隆的仓库将自动包含这些钩子。

### 3. 使用 Husky 等工具

对于 JavaScript/Node.js 项目，[Husky](https://github.com/typicode/husky) 提供了更简单的钩子管理方式：

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  }
}
```

类似工具还有：
- Ruby: Overcommit
- Python: pre-commit
- PHP: CaptainHook

## 实用钩子示例

### 1. 防止敏感信息提交

```bash
#!/bin/bash
# pre-commit

# 检查是否有密码、API密钥等敏感信息
if git diff --cached | grep -E '(password|secret|api.?key|token).*["\047][^\s]{8,}["\047]'; then
  echo "警告: 可能包含敏感信息，请检查后再提交"
  exit 1
fi
```

### 2. 强制代码格式化

```bash
#!/bin/bash
# pre-commit

# 保存未提交的文件
git stash -q --keep-index

# 运行格式化工具
prettier --write "**/*.js"
eslint --fix "**/*.js"

# 添加格式化后的文件
git add -u

# 恢复未提交的更改
git stash pop -q
```

### 3. 自动版本号递增

```bash
#!/bin/bash
# post-commit

# 检查是否在主分支
branch=$(git rev-parse --abbrev-ref HEAD)
if [ "$branch" = "main" ]; then
  # 递增版本号
  npm version patch -m "Bump version to %s [skip ci]"
fi
```

### 4. 提交信息模板

```bash
#!/bin/bash
# prepare-commit-msg

# 获取当前任务编号（假设分支名格式为 feature/TASK-123）
branch=$(git symbolic-ref --short HEAD)
task=$(echo $branch | grep -oE '[A-Z]+-[0-9]+')

if [ -n "$task" ]; then
  # 在提交信息开头添加任务编号
  sed -i.bak -e "1s/^/[$task] /" $1
fi
```

## 最佳实践

1. **保持钩子简单**：复杂的钩子可能会减慢 Git 操作速度。
2. **提供跳过机制**：在必要时允许绕过钩子（如紧急修复）。
   ```bash
   # 在钩子中检查环境变量
   if [ "$SKIP_HOOKS" = "1" ]; then
     exit 0
   fi

   # 使用时
   SKIP_HOOKS=1 git commit -m "Emergency fix"
   ```
3. **良好的错误信息**：当钩子阻止操作时，提供清晰的错误信息和修复指导。
4. **测试钩子**：确保钩子在各种情况下都能正常工作。
5. **文档化**：记录钩子的功能和使用方法，特别是对团队成员来说。

---

Git 钩子是自动化和标准化开发流程的强大工具。通过精心设计的钩子，你可以提高代码质量，减少人为错误，并使团队协作更加顺畅。虽然设置钩子需要一些初始投入，但长期来看，它们将为你的项目带来显著的效率提升和质量改进。