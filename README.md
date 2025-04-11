# Mintlify 入门套件

点击 `Use this template` 复制 Mintlify 入门套件。该入门套件包含以下示例：

- 指南页面
- 导航
- 自定义设置
- API 参考页面
- 常用组件的使用

### 开发

安装 [Mintlify CLI](https://www.npmjs.com/package/mintlify) 在本地预览文档更改。使用以下命令安装：

```
npm i -g mintlify
```

在文档根目录（docs.json 所在目录）运行以下命令：

```
mintlify dev
```

### 发布更改

安装我们的 Github 应用程序，自动将您的存储库中的更改推送到部署中。更改将在推送到默认分支后自动部署到生产环境。在您的仪表板上找到安装链接。

#### 故障排除

- Mintlify dev 无法运行 - 运行 `mintlify install` 重新安装依赖项。
- 页面加载为 404 - 确保您在包含 `docs.json` 的文件夹中运行。
