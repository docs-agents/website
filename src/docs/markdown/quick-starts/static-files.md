---
title: 静态文件快速入门
---

# 静态文件快速入门

本指南将向您展示如何快速搭建一个可用于生产环境的静态文件服务器。

**前提条件：**
- 基本的终端/命令行操作技能
- `caddy` 已添加到系统 PATH 中
- 一个包含您网站的文件夹

---

有两种简单方法可以快速启动文件服务器。

## 命令行

在终端中，切换到网站的根目录并运行：

<pre><code class="cmd bash">caddy file-server</code></pre>

如果遇到权限错误，通常意味着您的操作系统不允许绑定低端口——此时使用高位端口即可：

<pre><code class="cmd bash">caddy file-server --listen :2015</code></pre>

然后在浏览器中打开 [localhost](http://localhost)（或 [localhost:2015](http://localhost:2015)）查看您的网站！

如果没有索引文件但希望显示文件列表，请使用 `--browse` 选项：

<pre><code class="cmd bash">caddy file-server --browse</code></pre>

您也可以使用其他文件夹作为网站根目录：

<pre><code class="cmd bash">caddy file-server --root ~/mysite</code></pre>

## Caddyfile

在网站根目录下，创建一个名为 `Caddyfile` 的文件，内容如下：

```caddy
localhost

file_server
```

如果您没有绑定低端口的权限，请将 `localhost` 替换为 `localhost:2015`（或其他高位端口）。

然后，在同一目录下运行：

<pre><code class="cmd bash">caddy run</code></pre>

随后即可通过 [localhost](https://localhost)（或配置中指定的地址）访问您的网站！

[`file_server` 指令](/docs/caddyfile/directives/file_server) 提供了更多自定义选项。修改 Caddyfile 后，请务必[重载](/docs/command-line#caddy-reload) Caddy（或停止后重新启动）。

如果没有索引文件但希望显示文件列表，请使用 `browse` 参数：

```caddy
localhost

file_server browse
```

您也可以使用其他文件夹作为网站根目录：

```caddy
localhost

root /var/www/mysite
file_server