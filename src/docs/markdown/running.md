---
title: 保持 Caddy 运行
---

# 保持 Caddy 运行

虽然可以直接通过[命令行界面](/docs/command-line)运行 Caddy，但使用服务管理器来保持其运行有许多优势，例如确保系统重启时自动启动，以及捕获 stdout/stderr 日志。


- [Linux 服务](#linux-服务)
  - [单元文件](#单元文件)
  - [手动安装](#手动安装)
  - [使用服务](#使用服务)
  - [本地 HTTPS（使用 systemd）](#本地-https使用-systemd)
  - [覆盖配置](#覆盖配置)
	- [环境变量](#环境变量)
	- [`run` 和 `reload` 覆盖](#run-和-reload-覆盖)
	- [崩溃时重启](#崩溃时重启)
  - [SELinux 注意事项](#selinux-注意事项)
- [Windows 服务](#windows-服务)
  - [sc.exe](#scexe)
  - [WinSW](#winsw)
- [Docker Compose](#docker-compose)
  - [设置](#设置)
  - [使用](#使用)
  - [本地 HTTPS（使用 Docker）](#本地-https使用-docker)


## Linux 服务

在带有 systemd 的 Linux 发行版上，推荐使用我们官方提供的 systemd 单元文件来运行 Caddy。


### 单元文件

我们提供了两个不同的 systemd 单元文件供您根据使用场景选择：

- [**`caddy.service`**](https://github.com/caddyserver/dist/blob/master/init/caddy.service) —— 如果您使用 [Caddyfile](/docs/caddyfile) 配置 Caddy。如果您倾向于使用其他配置适配器或 JSON 配置文件，可以[覆盖](#覆盖配置)`ExecStart` 和 `ExecReload` 命令。

- [**`caddy-api.service`**](https://github.com/caddyserver/dist/blob/master/init/caddy-api.service) —— 如果完全通过 [API](/docs/api) 配置 Caddy。该服务使用 [`--resume`](/docs/command-line#caddy-run) 选项，Caddy 将使用默认[持久化](/docs/json/admin/config/)的 `autosave.json` 启动。

这两个服务非常相似，但 `ExecStart` 和 `ExecReload` 命令有所不同，以适应不同的工作流程。

如果需要在两个服务之间切换，您应先禁用并停止前一个服务，再启用并启动另一个。例如，从 `caddy` 服务切换到 `caddy-api` 服务：
<pre><code class="cmd"><span class="bash">sudo systemctl disable --now caddy</span>
<span class="bash">sudo systemctl enable --now caddy-api</span></code></pre>


### 手动安装

某些[安装方法](/docs/install)会自动将 Caddy 设置为服务运行。如果您选择的方法未自动设置，可以按照以下说明手动配置：

**要求：**

- 已[下载](/download)或[从源码构建](/docs/build)的 `caddy` 二进制文件
- `systemctl --version` 232 或更新版本
- `sudo` 权限

将 caddy 二进制文件移动到您的 `$PATH` 中，例如：
<pre><code class="cmd bash">sudo mv caddy /usr/bin/</code></pre>

测试是否成功：
<pre><code class="cmd bash">caddy version</code></pre>

创建名为 `caddy` 的用户组：
<pre><code class="cmd bash">sudo groupadd --system caddy</code></pre>

创建名为 `caddy` 的用户，并为其分配一个可写入的家目录：
<pre><code class="cmd bash">sudo useradd --system \
    --gid caddy \
    --create-home \
    --home-dir /var/lib/caddy \
    --shell /usr/sbin/nologin \
    --comment "Caddy web server" \
    caddy</code></pre>

如果使用配置文件，请确保您刚创建的 `caddy` 用户有权限读取该文件。

接下来，根据您的使用场景[选择一个 systemd 单元文件](#单元文件)。

**仔细检查 `ExecStart` 和 `ExecReload` 指令。** 确保二进制文件的位置和命令行参数与您的安装一致！例如：如果使用配置文件，请根据实际路径修改 `--config` 参数（如果默认路径不同）。

服务文件通常保存于：`/etc/systemd/system/caddy.service`

保存服务文件后，您可以通过常规的 systemctl 操作首次启动服务：

<pre><code class="cmd"><span class="bash">sudo systemctl daemon-reload</span>
<span class="bash">sudo systemctl enable --now caddy</span></code></pre>

验证服务是否正在运行：
<pre><code class="cmd bash">systemctl status caddy</code></pre>

现在您就可以[使用该服务](#使用服务)了！



### 使用服务

如果使用 Caddyfile，您可以使用 `nano`、`vi` 或您喜欢的编辑器编辑配置：
<pre><code class="cmd bash">sudo nano /etc/caddy/Caddyfile</code></pre>

您可以将静态网站文件放置在 `/var/www/html` 或 `/srv` 中。确保 `caddy` 用户有权限读取这些文件。

验证服务是否正在运行：
<pre><code class="cmd bash">systemctl status caddy</code></pre>
`status` 命令还会显示当前运行的服务文件的位置。

使用我们官方提供的服务文件运行时，Caddy 的输出将被重定向到 `journalctl`。要查看完整日志并避免行被截断：
<pre><code class="cmd bash">journalctl -u caddy --no-pager | less +G</code></pre>

如果使用配置文件，您可以在进行任何更改后优雅地重新加载 Caddy：
<pre><code class="cmd bash">sudo systemctl reload caddy</code></pre>

您可以使用以下命令停止服务：
<pre><code class="cmd bash">sudo systemctl stop caddy</code></pre>

<aside class="advice">

不要通过停止服务来更改 Caddy 的配置。停止服务器会导致停机。请改用重新加载命令。

</aside>

Caddy 进程将以 `caddy` 用户运行，其 `$HOME` 设置为 `/var/lib/caddy`。这意味着：
- 默认的[数据存储位置](/docs/conventions#data-directory)（用于证书和其他状态信息）将位于 `/var/lib/caddy/.local/share/caddy`。
- 默认的[配置存储位置](/docs/conventions#configuration-directory)（用于自动保存的 JSON 配置，主要用于 `caddy-api` 服务）将位于 `/var/lib/caddy/.config/caddy`。


### 本地 HTTPS（使用 systemd）

当使用 Caddy 进行本地开发并启用 HTTPS 时，您可能会使用像 `localhost` 或 `app.localhost` 这样的[主机名](/docs/caddyfile/concepts#addresses)。这会启用[本地 HTTPS](/docs/automatic-https#local-https)，Caddy 会使用其本地 CA 签发证书。

由于 Caddy 作为服务运行时以 `caddy` 用户身份运行，它没有权限将其根 CA 证书安装到系统信任库中。要完成此操作，请运行 [`sudo caddy trust`](/docs/command-line#caddy-trust) 执行安装。

如果您希望其他设备在使用 [`internal` 签发者](/docs/caddyfile/directives/tls#internal)时连接到您的服务器，您还需要在这些设备上安装根 CA 证书。您可以在 `/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt` 找到根 CA 证书。许多现代浏览器使用自己的信任库（忽略系统的信任库），因此您可能还需要在那里手动安装证书。


### 覆盖配置

覆盖服务文件相关设置的最佳方法是使用以下命令：
<pre><code class="cmd bash">sudo systemctl edit caddy</code></pre>

这将在默认终端文本编辑器中打开一个空白文件，您可以在其中覆盖或添加单元定义的指令。这被称为“drop-in”文件。

#### 环境变量

如果您需要在配置中使用环境变量，可以这样定义：
```systemd
[Service]
Environment="CF_API_TOKEN=super-secret-cloudflare-tokenvalue"
```

同样，如果您更愿意使用单独的文件来维护环境变量（envfile），可以使用 [`EnvironmentFile`](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#EnvironmentFile=) 指令，如下所示：
```systemd
[Service]
EnvironmentFile=/etc/caddy/.env
```

然后您的 `/etc/caddy/.env` 文件可能如下所示（值周围不要使用 `"` 引号）：

```env
CF_API_TOKEN=super-secret-cloudflare-tokenvalue
```

#### `run` 和 `reload` 覆盖

如果您需要将配置文件从默认的 Caddyfile 更改为使用 JSON 文件（注意：`Exec*` 指令在设置新值之前[必须用空字符串重置](https://www.freedesktop.org/software/systemd/man/systemd.service.html#ExecStart=)）：
```systemd
[Service]
ExecStart=
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/caddy.json
ExecReload=
ExecReload=/usr/bin/caddy reload --config /etc/caddy/caddy.json
```

#### 崩溃时重启

如果您希望 caddy 在意外崩溃后 5 秒后自动重启：
```systemd
[Service]
# 自动重启 caddy，除非退出码为 1
RestartPreventExitStatus=1
Restart=on-failure
RestartSec=5s
```

然后，保存文件并退出文本编辑器，重启服务使其生效：
<pre><code class="cmd bash">sudo systemctl restart caddy</code></pre>



### SELinux 注意事项

在启用 SELinux 的系统上，您有两个选择：
1. 使用 [COPR 仓库](/docs/install#fedora-redhat-centos) 安装 Caddy。您的 systemd 文件和 caddy 二进制文件将已正确创建并标记（因此您可以忽略本部分）。如果您希望使用自定义构建的 Caddy，您需要按照下面描述的方式标记可执行文件。

2. [从此网站下载 Caddy](/download) 或使用 [`xcaddy`](https://github.com/caddyserver/xcaddy) 编译。无论哪种情况，您都需要自己标记文件。

Systemd 单元文件及其可执行文件只有在分别标记为 `systemd_unit_file_t` 和 `bin_t` 时才能运行。

`systemd_unit_file_t` 标签会自动应用于在 `/etc/systemd/...` 中创建的文件，因此请确保按照[手动安装](#手动安装)的说明在其中创建您的 `caddy.service` 文件。

要标记 `caddy` 二进制文件，您可以使用以下命令：
<pre><code class="cmd bash">semanage fcontext -a -t bin_t /usr/bin/caddy && restorecon -Rv /usr/bin/caddy
</code></pre>

## Windows 服务

在 Windows 上以服务方式运行 Caddy 有两种方法：[sc.exe](#scexe) 或 [WinSW](#winsw)。

### sc.exe

要创建服务，请运行：

<pre><code class="cmd bash">sc.exe create caddy start= auto binPath= "YOURPATH\caddy.exe run"</code></pre>

（将 `YOURPATH` 替换为 `caddy.exe` 的实际路径）

启动：

<pre><code class="cmd bash">sc.exe start caddy</code></pre>

停止：

<pre><code class="cmd bash">sc.exe stop caddy</code></pre>


### WinSW

按照以下说明在 Windows 上将 Caddy 安装为服务。

**要求：**

- 已[下载](/download)或[从源码构建](/docs/build)的 `caddy.exe` 二进制文件
- [WinSW](https://github.com/winsw/winsw/releases/latest) 服务包装器最新发布版中的任意 `.exe`（以下服务配置适用于 v2.x 版本）

将所有文件放入一个服务目录。在以下示例中，我们使用 `C:\caddy`。

将 `WinSW-x64.exe` 文件重命名为 `caddy-service.exe`。

在同一目录下添加 `caddy-service.xml`：

```xml
<service>
  <id>caddy</id>
  <!-- 服务的显示名称 -->
  <name>Caddy Web Server (powered by WinSW)</name>
  <!-- 服务描述 -->
  <description>Caddy Web Server (https://caddyserver.com/)</description>
  <executable>%BASE%\caddy.exe</executable>
  <arguments>run</arguments>
  <log mode="roll-by-time">
    <pattern>yyyy-MM-dd</pattern>
  </log>
</service>
```

现在您可以使用以下命令安装服务：
<pre><code class="cmd bash">caddy-service install</code></pre>

您可能想打开 Windows 服务控制台查看服务是否正常运行：
<pre><code class="cmd bash">services.msc</code></pre>

请注意，Windows 服务无法重新加载，因此您需要直接告诉 caddy 重新加载：
<pre><code class="cmd bash">caddy reload</code></pre>

可以通过常规的 Windows 服务命令进行重启，例如通过任务管理器的“服务”选项卡。

有关自定义服务包装器的信息，请参阅 [WinSW 文档](https://github.com/winsw/winsw/tree/master#usage)


## Docker Compose

通过 Docker 快速启动并运行的最简单方法是使用 Docker Compose。有关官方 Caddy Docker 镜像的更多详细信息，请参见 [Docker Hub](https://hub.docker.com/_/caddy) 上的文档。

<aside class="tip">

这里假设您使用的是 [Docker Compose V2](https://docs.docker.com/compose/reference/)，命令为 `docker compose`（空格），而不是 V1 的 `docker-compose`（连字符）。

</aside>

### 设置

首先，创建一个 `compose.yml` 文件（或将此服务添加到现有文件中）：

```yaml
services:
  caddy:
    image: caddy:<version>
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./conf:/etc/caddy
      - ./site:/srv
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

请确保将 `image: caddy:<version>` 中的 `<version>` 替换为最新的版本号，您可以在 [Docker Hub](https://hub.docker.com/_/caddy) 的“Tags”部分找到。

作用说明：

- 使用 `unless-stopped` 重启策略，确保您的机器重启时 Caddy 容器自动重启。
- 绑定端口 `80` 和 `443` 分别用于 HTTP 和 HTTPS，以及 `443/udp` 用于 HTTP/3。
- 绑定挂载 `conf` 目录，其中包含您的 Caddyfile 配置。
- 绑定挂载 `site` 目录，将静态文件从 `/srv` 提供服务。
- 为 `/data` 和 `/config` 使用命名卷，以[持久化重要信息](/docs/conventions#file-locations)。

然后，创建一个名为 `Caddyfile` 的文件作为 `conf` 目录中的唯一文件，并编写您的 [Caddyfile](/docs/caddyfile/concepts) 配置。

如果您有要提供的静态文件，可以将它们放在配置旁边的 `site/` 目录中，然后使用 `root /srv` 设置[根目录](/docs/caddyfile/directives/root)。如果没有，则可以移除 `/srv` 卷挂载。

<aside class="tip">

如果您使用 Caddy [反向代理](/docs/caddyfile/directives/reverse_proxy)到另一个容器，请记住在 Docker 网络中，`localhost` 指的是“此容器”，而不是“此机器”。因此，例如，不要使用 `reverse_proxy localhost:8080`，而应使用 `reverse_proxy other-container:8080`。

</aside>

如果您需要带有插件的自定义 Caddy 构建，请按照 [Docker 构建说明](/docs/build#docker)创建自定义 Docker 镜像。在 `compose.yml` 旁边创建 `Dockerfile`，然后将 `compose.yml` 中的 `image:` 行替换为 `build: .`。


### 使用

然后，您可以启动容器：
<pre><code class="cmd bash">docker compose up -d</code></pre>

在修改 Caddyfile 后重新加载 Caddy：
<pre><code class="cmd bash">docker compose exec -w /etc/caddy caddy caddy reload</code></pre>

从 v2.11.0 开始，您可以使用 `SIGUSR1` 重新加载，前提是 Caddy 是通过 `caddy run` 和配置文件启动的：
<pre><code class="cmd bash">docker compose kill -sUSR1 caddy</code></pre>

查看 Caddy 最近的 1000 条日志，并 `f` 跟随查看新的实时日志：
<pre><code class="cmd bash">docker compose logs caddy -n=1000 -f</code></pre>

### 本地 HTTPS（使用 Docker）

在 Docker 中进行本地开发并启用 HTTPS 时，您可能会使用像 `localhost` 或 `app.localhost` 这样的[主机名](/docs/caddyfile/concepts#addresses)。这会启用[本地 HTTPS](/docs/automatic-https#local-https)，Caddy 会使用其本地 CA 签发证书。这意味着容器外部的 HTTP 客户端不会信任 Caddy 提供的 TLS 证书。要解决此问题，您可以将 Caddy 的根 CA 证书安装到宿主机的信任库中：

<div x-data="{ os: $persist(defaultOS(['linux', 'mac', 'windows'], 'linux')) }" class="tabs">
<div class="tab-buttons">
	<button x-on:click="os = 'linux'" x-bind:class="{ active: os === 'linux' }">Linux</button>
	<button x-on:click="os = 'mac'" x-bind:class="{ active: os === 'mac' }">Mac</button>
	<button x-on:click="os = 'windows'" x-bind:class="{ active: os === 'windows' }">Windows</button>
</div>

<div x-show="os === 'linux'" class="tab bordered">

<pre><code class="cmd bash">docker compose cp \
    caddy:/data/caddy/pki/authorities/local/root.crt \
    /usr/local/share/ca-certificates/root.crt \
  && sudo update-ca-certificates</code></pre>

</div>

<div x-show="os === 'mac'" class="tab bordered">

<pre><code class="cmd bash">docker compose cp \
    caddy:/data/caddy/pki/authorities/local/root.crt \
    /tmp/root.crt \
  && sudo security add-trusted-cert -d -r trustRoot \
    -k /Library/Keychains/System.keychain /tmp/root.crt</code></pre>

</div>

<div x-show="os === 'windows'" class="tab bordered">

<pre><code class="cmd bash">docker compose cp \
    caddy:/data/caddy/pki/authorities/local/root.crt \
    %TEMP%/root.crt \
  && certutil -addstore -f "ROOT" %TEMP%/root.crt</code></pre>

</div>
</div>

许多现代浏览器使用自己的信任库（忽略系统的信任库），因此您可能还需要在那里手动安装证书，使用上面命令从容器中复制出的 `root.crt` 文件。

- 对于 Firefox，请进入首选项 > 隐私与安全 > 证书 > 查看证书 > 证书颁发机构 > 导入，然后选择 `root.crt` 文件。

- 对于 Chrome，请进入设置 > 隐私和安全 > 安全 > 管理证书 > 受信任的根证书颁发机构 > 导入，然后选择 `root.crt` 文件。