---
title: '安全与签名'
---

# Git 安全：提交签名

通过 GPG 签名提交和标签，可以验证代码的真实来源，提高项目安全性，确保接收的代码确实来自可信的开发者。这在开源项目中尤为重要。

## 提交签名: 验证代码来源和完整性

**设置步骤:**

1. **生成 GPG 密钥**:
   ```bash
   gpg --full-generate-key
   # 选择密钥类型，推荐 RSA and RSA (default)
   # 输入密钥大小：4096
   # 设置有效期
   # 输入个人信息（与 Git 配置保持一致）
   ```

2. **获取并配置 GPG 密钥**:
   ```bash
   # 查看密钥列表
   gpg --list-secret-keys --keyid-format=long

   # 从输出中找到密钥 ID
   # sec   rsa4096/XXXXXXXXXXXXXXXX <-- 这是密钥 ID

   # 配置 Git 使用此密钥
   git config --global user.signingkey XXXXXXXXXXXXXXXX
   ```

3. **签名提交和标签**:
   ```bash
   # 启用自动提交签名（所有提交都会签名）
   git config --global commit.gpgsign true

   # 单个提交的签名
   git commit -S -m "添加安全功能"

   # 签名标签 (默认情况下 -s 标志就会签名)
   git tag -s v1.0.0 -m "版本 1.0.0 发布"
   ```

4. **验证签名**:
   ```bash
   # 验证已签名的提交
   git verify-commit HEAD

   # 验证已签名的标签
   git verify-tag v1.0.0
   ```

**配置 GitHub/GitLab 验证签名**:
1. 导出你的公钥: `gpg --armor --export XXXXXXXXXXXXXXXX`
2. 复制输出的公钥块（包括首尾行）
3. 将公钥添加到你的 GitHub/GitLab 账户设置中

**常见问题解决**:
* **签名失败**: 确保设置了正确的 GPG 密钥，在 Windows 上可能需要额外安装 GPG 工具。
* **验证签名失败**: 确保导入了签名者的公钥到你的 GPG 密钥环。

使用 GPG 签名可以增加一层安全保障，确认提交和标签的来源是可信的。