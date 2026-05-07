Caddy 网站
=================

这是 Caddy 网站的源代码，[caddyserver.com](https://caddyserver.com)。


## 要求

- Caddy v2.7.6 或更新版本（已安装于 PATH 中，命令名为 `caddy`）
- 如需显示复古访问计数器（仅供娱乐），需安装 [caddy-hitcounter](https://github.com/mholt/caddy-hitcounter) 插件。然后在 Caddyfile 中取消相关行的注释。


## 快速开始

1. `git clone https://github.com/caddyserver/website.git`
2. `cd website`
3. `caddy run`

首次运行时，可能会提示您输入密码。这是为了让 Caddy 通过本地 HTTPS 提供站点服务。如果您无法绑定低端口，请修改 [Caddyfile 顶部的地址](https://github.com/caddyserver/website/blob/master/Caddyfile#L1)，例如改为 `localhost:2015`。

然后，您可以在浏览器中加载 [https://localhost](https://localhost)（或您配置的任何地址）。

### Docker

您可以通过 Docker 以无根模式运行，命令如下：
```
docker stop caddy-website || true && docker rm caddy-website || true
docker run --name caddy-website -it -p 8443:443 -v ./:/wd caddy sh -c "cd /wd && caddy run"
```

这样即可通过 https://localhost:8443 连接。