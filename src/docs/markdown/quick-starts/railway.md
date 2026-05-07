---
title: Railway 快速入门
---

# Railway 快速入门

在 Railway 上部署 Caddy 是一种简单、无麻烦的方式，可以部署带有插件的自定义 Caddy 构建。

**前提条件：**
- 一个免费的 [Railway](https://railway.com) 账号

## 在 Railway 上部署 Caddy

前往我们的 [下载页面](/download)，选择你需要的任意插件，然后点击顶部的紫色“在 Railway 上部署”按钮。

<details>
	<summary>或者，手动配置模板</summary>

如果你希望自己配置 Railway 模板，以下是操作方法。

前往 Railway 上的模板：

<a href="https://railway.com/deploy/caddy?referralCode=YOPtw9&amp;utm_medium=integration&amp;utm_source=template&amp;utm_campaign=generic"><img src="https://railway.com/button.svg" alt="在 Railway 上部署"></a>

然后通过点击“配置”添加你需要的任意插件：

![部署界面](/resources/images/railway/deploy-screen.png)

接着将插件粘贴到 `CADDY_PLUGINS` 变量中，用空格分隔：

![添加插件](/resources/images/railway/deploy-config.png)

</details>

点击部署，部署完成后，你可以点击这里的链接进行测试：

![访问你的部署](/resources/images/railway/prod-link.png)

你应该会看到一个欢迎页面，显示你的新服务器正在运行！

接下来，你可以自定义部署以托管你自己的站点或作为代理连接到另一个 Railway 服务。

## 自定义部署

要托管你自己的网站或更改配置，只需将 [我们的模板](https://railway.com/deploy/caddy?referralCode=YOPtw9&amp;utm_medium=integration&amp;utm_source=template&amp;utm_campaign=generic) “弹出”到你自己的仓库中：

![弹出模板](/resources/images/railway/eject.png)

在你自己的仓库中，你可以：

- 将自己的站点放入 `www` 文件夹。
- 修改 Caddy 的配置，即 [Caddyfile](/docs/caddyfile)。

只需提交更改并推送，然后你就可以在 Railway 上重新部署。

如果你想更改 Caddy 构建中的插件，只需编辑 `CADDY_PLUGINS` 变量并重新部署：

![更改插件](/resources/images/railway/plugins-variable.png)

## 提示

Railway 会为你终止 TLS，因此你应当将 Caddy 配置视为被代理（事实也是如此）。因此，如果你在 Caddyfile 站点地址中使用主机名，则应在全局选项中设置 `auto_https off`。在我们的模板中，Caddy 不直接面向边缘。

## 变量

你可以在 Railway 项目中设置的环境变量，该模板可能会用到：

名称 | 描述 | 默认值 | 示例
---- | ----------- | ------- | ----------
`CADDY_PLUGINS` | 以空格分隔的 Caddy 插件列表 | ` ` | `github.com/caddy-dns/cloudflare github.com/mholt/caddy-ratelimit`