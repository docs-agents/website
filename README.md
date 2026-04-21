Caddy 网站
==========

这是 Caddy 网站 [caddyserver.com](https://caddyserver.com) 的源码。


## 要求

- Caddy v2.7.6 或更新版本（已安装到你的 PATH 中，命令为 `caddy`）
- 要显示复古访问计数器（仅供娱乐），需要 [caddy-hitcounter](https://github.com/mholt/caddy-hitcounter) 插件。然后取消注释 Caddyfile 中的相关行。


## 快速开始

1. `git clone https://github.com/caddyserver/website.git`
2. `cd website`
3. `caddy run`

第一次使用时，系统可能会提示你输入密码。这是为了让 Caddy 通过本地 HTTPS 提供服务。如果你无法绑定到低端口，请修改 [Caddyfile 顶部的地址](https://github.com/caddyserver/website/blob/master/Caddyfile#L1)，例如 `localhost:2015`。

然后你可以在浏览器中访问 [https://localhost](https://localhost)（或你配置的任何地址）。

### Docker

你可以使用 Docker 以无 root 用户方式运行：
```
docker stop caddy-website || true && docker rm caddy-website || true
docker run --name caddy-website -it -p 8443:443 -v ./:/wd caddy sh -c "cd /wd && caddy run"
```

这样你就可以通过 https://localhost:8443 进行连接。