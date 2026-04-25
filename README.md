Caddy 网站
=========

这是 Caddy 网站的源代码，[caddyserver.com](https://caddyserver.com)。


## 要求

- Caddy v2.7.6 或更高版本（在您的 PATH 中安装为 `caddy`）
- 要显示复古点击计数器（纯属娱乐），需要 [caddy-hitcounter](https://github.com/mholt/caddy-hitcounter) 插件。然后在 Caddyfile 中取消注释相关行。


## 快速开始

1. `git clone https://github.com/caddyserver/website.git`
2. `cd website`
3. `caddy run`

首次运行时，系统可能会提示您输入密码。这是为了让 Caddy 能够通过本地 HTTPS 提供服务。如果您无法绑定到低端口，请更改 [Caddyfile 顶部的地址](https://github.com/caddyserver/website/blob/master/Caddyfile#L1)，例如 `localhost:2015`。

然后您可以在浏览器中加载 [https://localhost](https://localhost)（或您配置的任何地址）。

### Docker

您可以使用 Docker 以无根模式运行：
```
docker stop caddy-website || true && docker rm caddy-website || true
docker run --name caddy-website -it -p 8443:443 -v ./:/wd caddy sh -c "cd /wd && caddy run"
```

这将允许您连接到 https://localhost:8443
