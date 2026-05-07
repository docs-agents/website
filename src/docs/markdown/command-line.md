---
title: "命令行"
---

# 命令行

Caddy 拥有标准的类 Unix 命令行界面。基本用法是：

```
caddy <command> [<args...>]
```

`<尖括号>` 表示需要由你的输入替换的参数。

`[方括号]` 表示可选参数。`(圆括号)` 表示必需参数。

省略号 `...` 表示可延续，即一个或多个参数。

`--标志` 可能有单字母快捷键，如 `-f`。

**快速开始：**`caddy`、`caddy help` 或 `man caddy`（如果已安装）

---

- **[caddy adapt](#caddy-adapt)**
  将配置文档适配为原生 JSON

- **[caddy build-info](#caddy-build-info)**
  打印构建信息

- **[caddy completion](#caddy-completion)**
  生成 shell 补全脚本

- **[caddy environ](#caddy-environ)**
  打印环境变量

- **[caddy file-server](#caddy-file-server)**
  一个简单但可用于生产的文件服务器

- **[caddy file-server export-template](#caddy-file-server-export-template)**
  文件服务器的辅助命令，用于导出默认文件浏览器模板

- **[caddy fmt](#caddy-fmt)**
  格式化 Caddyfile

- **[caddy hash-password](#caddy-hash-password)**
  对密码进行哈希并输出 base64

- **[caddy help](#caddy-help)**
  查看 caddy 命令的帮助信息

- **[caddy list-modules](#caddy-list-modules)**
  列出已安装的 Caddy 模块

- **[caddy manpage](#caddy-manpage)**
  生成手册页

- **[caddy reload](#caddy-reload)**
  更改正在运行的 Caddy 进程的配置

- **[caddy respond](#caddy-respond)**
  一个快速简洁、硬编码的 HTTP 服务器，用于开发和测试

- **[caddy reverse-proxy](#caddy-reverse-proxy)**
  一个简单但可用于生产的 HTTP(S) 反向代理

- **[caddy run](#caddy-run)**
  在前台启动 Caddy 进程

- **[caddy start](#caddy-start)**
  在后台启动 Caddy 进程

- **[caddy stop](#caddy-stop)**
  停止正在运行的 Caddy 进程

- **[caddy storage export](#caddy-storage)**
  将已配置存储的内容导出为 tar 包

- **[caddy storage import](#caddy-storage)**
  将之前导出的 tar 包导入到已配置的存储中

- **[caddy trust](#caddy-trust)**
  将证书安装到本地信任存储中

- **[caddy untrust](#caddy-untrust)**
  从本地信任存储中取消信任证书

- **[caddy upgrade](#caddy-upgrade)**
  将 Caddy 升级到最新版本

- **[caddy add-package](#caddy-add-package)**
  将 Caddy 升级到最新版本，并添加额外的插件

- **[caddy remove-package](#caddy-remove-package)**
  将 Caddy 升级到最新版本，并移除一些插件

- **[caddy validate](#caddy-validate)**
  测试配置文件是否有效

- **[caddy version](#caddy-version)**
  打印版本

- **[信号](#signals)**
  Caddy 如何处理信号

- **[退出码](#exit-codes)**
  Caddy 进程退出时发出的代码

## 子命令


### `caddy adapt`

<pre><code class="cmd bash">caddy adapt
	[-c, --config &lt;path&gt;]
	[-a, --adapter &lt;name&gt;]
	[-p, --pretty]
	[--validate]</code></pre>

将配置适配为 Caddy 的原生 JSON 配置结构，并将输出写入 stdout，同时将任何警告写入 stderr，然后退出。

`--config` 是配置文件的路径。如果省略，则假定当前目录中存在 `Caddyfile`；否则，此标志是必需的。如果你希望使用 stdin 而不是常规文件，请使用 `-` 作为路径。

`--adapter` 指定要使用的配置适配器；默认为 `caddyfile`。

`--pretty` 将格式化输出，添加缩进以提高可读性。

`--validate` 将加载并预配适配后的配置以检查其有效性（但不会实际启动运行配置）。

请注意，成功适配的配置仍可能验证失败。例如，使用以下 Caddyfile：

```caddy
localhost

tls cert_notexist.pem key_notexist.pem
```

尝试适配：

<pre><code class="cmd bash">caddy adapt --config Caddyfile</code></pre>

它会成功且无错误。然后尝试：

<pre><code class="cmd"><span class="bash">caddy adapt --config Caddyfile --validate</span>
adapt: validation: loading app modules: module name 'tls': provision tls: loading certificates: open cert_notexist.pem: no such file or directory
</code></pre>

尽管该 Caddyfile 可以成功适配为 JSON，但实际的证书和/或密钥文件不存在，因此验证失败，因为该错误发生在预配阶段。因此，验证是比适配更强的错误检查。

#### 示例

要将 Caddyfile 适配为易于手动阅读和调整的 JSON：

<pre><code class="cmd bash">caddy adapt --config /path/to/Caddyfile --pretty</code></pre>



### `caddy build-info`

<pre><code class="cmd bash">caddy build-info</code></pre>

打印 Go 提供的构建信息（主模块路径、包版本、模块替换）。




### `caddy completion`

<pre><code class="cmd bash">caddy completion [bash|zsh|fish|powershell]</code></pre>

生成 shell 补全脚本。这允许你在输入 `caddy` 命令时获得 tab 补全或自动补全（或类似功能，具体取决于你的 shell）。

要获取将此脚本安装到特定 shell 的说明，请运行 `caddy help completion` 或 `caddy completion -h`。



### `caddy environ`

<pre><code class="cmd bash">caddy environ</code></pre>

打印 caddy 所看到的环境变量，然后退出。在调试初始化系统或进程管理器单元（如 systemd）时很有用。




### `caddy file-server`

<pre><code class="cmd bash">caddy file-server
	[-r, --root &lt;path&gt;]
	[--listen &lt;addr&gt;]
	[-d, --domain &lt;example.com&gt;]
	[-b, --browse]
	[--reveal-symlinks]
	[-t, --templates]
	[--access-log]
	[-v, --debug]
	[-f, --file-limit &lt;number&gt;]
	[--no-compress]
	[-p, --precompressed]</code></pre>

启动一个简单但可用于生产的静态文件服务器。

`--root` 指定根文件路径。默认为当前工作目录。

`--listen` 接受一个监听地址。默认是 `:80`，除非使用了 `--domain`，此时默认使用 `:443`。

`--domain` 将仅通过该主机名提供文件服务，并且 Caddy 将尝试通过 HTTPS 提供它，因此如果是公共域名，请确保首先正确配置公共 DNS。默认端口将更改为 443。

`--browse` 将在请求不包含索引文件的目录时启用目录列表。

`--reveal-symlinks` 将在启用 `--browse` 时在目录列表中显示符号链接的目标。

`--templates` 将启用模板渲染。

`--access-log` 启用请求/访问日志。

`--debug` 启用详细日志记录。

`--file-limit` 设置目录列表中显示的最大文件数。默认值：`10000`。如果文件数超过此限制，将仅显示前 N 个文件，其中 N 是指定的限制。

`--no-compress` 禁用压缩。默认情况下，Zstandard 和 Gzip 压缩处于启用状态。

`--precompressed` 指定要搜索预压缩 sidecar 文件的编码格式。可以重复使用以指定多种格式。有关更多信息，请参阅 [file_server 指令](/docs/caddyfile/directives/file_server#precompressed)。

此命令禁用管理 API，从而更容易在本地开发机器上运行多个实例。


#### `caddy file-server export-template`

<pre><code class="cmd bash">caddy file-server export-template</code></pre>

将默认文件浏览模板导出到 stdout

### `caddy fmt`

<pre><code class="cmd bash">caddy fmt [&lt;path&gt;]
	[-w, --overwrite]
	[-d, --diff]</code></pre>

格式化或美化 Caddyfile，然后退出。结果默认打印到 stdout，除非使用了 `--overwrite`，并且如果存在任何差异，则退出码为 `1`。

`<path>` 指定 Caddyfile 的路径。如果是 `-`，则从 stdin 读取输入。如果省略，则假定当前目录中存在名为 Caddyfile 的文件。

`--overwrite` 将结果写入输入文件而不是打印到终端。如果输入不是常规文件，则此标志无效。

`--diff` 使输出与输入进行比较，并在差异处添加 `-` 和 `+` 前缀。请注意，未更改的行以两个空格为前缀以对齐，这不是有效的补丁格式；仅作为可视化工具。


### `caddy hash-password`

<pre><code class="cmd bash">caddy hash-password
	[-p, --plaintext &lt;password&gt;]
	[-a, --algorithm &lt;name&gt;]</code></pre>
	[--bcrypt-cost &lt;cost&gt;]</code></pre>

对明文密码进行哈希的便捷方法。生成的哈希以可直接在 Caddy 配置中使用的格式写入 stdout。

`--plaintext`
    要哈希的密码。如果省略，则从 stdin 读取。
    如果 Caddy 连接到控制 TTY，则输入将不会回显。

`--algorithm`
    选择哈希算法。有效选项为：
      * `argon2id`（推荐用于现代安全）
      * `bcrypt`（传统，较慢，可配置成本，默认成本为 `14`）

bcrypt 特定参数：

`--bcrypt-cost`
    设置 bcrypt 哈希难度。较高的值通过使哈希计算更慢且更消耗 CPU 来提高安全性。
    必须在有效范围 [bcrypt.MinCost, bcrypt.MaxCost] 内。
    如果省略或无效，则使用默认成本。

Argon2id 特定参数：

`--argon2id-time`
    执行的迭代次数。增加此值会使哈希变慢，并更能抵抗暴力攻击。

`--argon2id-memory`
    哈希期间使用的内存量。
    较大的值会增加对 GPU/ASIC 攻击的抵抗力。

`--argon2id-threads`
    要使用的 CPU 线程数。在多核系统上增加以获得更快的哈希速度。

`--argon2id-keylen`
    结果哈希的长度（以字节为单位）。更长的密钥可提高安全性，但会略微增加存储大小。


### `caddy help`

<pre><code class="cmd bash">caddy help [&lt;command&gt;]</code></pre>

打印 CLI 帮助文本，可选地针对特定子命令，然后退出。



### `caddy list-modules`

<pre><code class="cmd bash">caddy list-modules
	[--packages]
	[--versions]
	[-s, --skip-standard]
	[--json]</code></pre>

打印已安装的 Caddy 模块，可选地附带来自关联 Go 模块的包和/或版本信息，然后退出。

在某些脚本场景中，打印所有标准模块可能是多余的，因此你可以使用 `--skip-standard` 从输出中省略它们。

`--json` 以 JSON 格式输出模块信息，这对于程序化处理很有用。

注意：由于 [Go 中的一个 bug](https://github.com/golang/go/issues/29228)，版本信息仅在 Caddy 作为依赖项而非主模块构建时可用。使用 [xcaddy](/docs/build#xcaddy) 可以使这更容易。



### `caddy manpage`

<pre><code class="cmd bash">caddy manpage
	(-o, --directory &lt;path&gt;)</code></pre>

为 Caddy 命令生成手册/文档页面，并将其写入指定路径的目录。此命令的输出可由 `man` 命令读取。

`--directory`（必需）是写入手册页的目录路径。如果不存在，将创建该目录。

生成后，通常需要安装手册页。此过程因平台而异，但在典型的 Linux 系统上，如下所示：

<pre><code class="cmd"><b>$ caddy manpage --directory man
$ gzip -r man/
$ sudo cp man/* /usr/share/man/man8/
$ sudo mandb
</b></code></pre>

然后你可以运行 `man caddy`（或 `man caddy-*` 查看子命令）在终端中阅读文档。

手册页与我们网站上的文档是分开的。我们的网站有更全面的文档，并且经常更新。




### `caddy reload`

<pre><code class="cmd bash">caddy reload
	[-c, --config &lt;path&gt;]
	[-a, --adapter &lt;name&gt;]
	[--address &lt;interface&gt;]
	[-f, --force]</code></pre>

为正在运行的 Caddy 实例提供新配置。这具有将文档 POST 到 [/load 端点](/docs/api#post-load) 的效果，但此命令对于围绕配置文件的简单工作流程很方便。与 `stop`、`start` 和 `run` 命令相比，这个单一命令是更改/重新加载运行中配置的正确、语义化的方式。

由于此命令使用 API，管理端点不得禁用。

`--config` 是要应用的配置文件。如果是 `-`，则从 stdin 读取配置。如果未指定，它将尝试当前工作目录中名为 `Caddyfile` 的文件，如果存在，则使用 `caddyfile` 配置适配器进行适配；否则，如果不存在要加载的配置文件，则为错误。

`--adapter` 指定要使用的配置适配器（如果有）。如果 `--config` 文件名以 `Caddyfile` 开头或以 `.caddyfile` 结尾，则此标志不是必需的，因为会假定使用 `caddyfile` 适配器。否则，如果提供的配置文件不是 Caddy 的原生 JSON 格式，则此标志是必需的。

`--address` 在管理端点未监听默认地址且与提供的配置文件中的地址不同时需要。

`--force` 将强制重新加载，即使指定的配置与 Caddy 当前运行的配置相同。可用于强制 Caddy 重新预配其模块，这可能会产生副作用，例如：重新加载手动加载的 TLS 证书。




### `caddy respond`

<pre><code class="cmd bash">caddy respond
	[-s, --status &lt;code&gt;]
	[-H, --header "&lt;Field&gt;: &lt;value&gt;"]
	[-b, --body &lt;content&gt;]
	[-l, --listen &lt;addr&gt;]
	[-v, --debug]
	[--access-log]
	[&lt;status|body&gt;]</code></pre>


启动一个或多个简单的、硬编码的 HTTP 服务器，适用于开发、暂存以及某些生产用例。可用于验证或调试 HTTP 客户端、脚本甚至负载均衡器。

`--status` 是要返回的 HTTP 状态码。

`--header` 添加一个 HTTP 头；需要 `Field: value` 格式。此标志可以多次使用。

`--body` 指定响应体。或者，可以从 stdin 管道输入主体。

`--listen` 是监听地址，可以是 Caddy 识别的任何 [网络地址](/docs/conventions#network-addresses)，并且可以包括端口范围以启动多个服务器。

`--debug` 启用详细调试日志记录。

`--access-log` 启用访问/请求日志记录。

如果未指定任何选项，此命令会在随机可用端口上监听，并以空 200 响应回答 HTTP 请求。可以使用 `--listen` 标志自定义监听地址，并且始终将其打印到 stdout。如果监听地址包含端口范围，将启动多个服务器。

如果给出最后一个未命名参数，如果它是 3 位数字，则将其视为状态码（与 `--status` 标志相同）。否则，它将用作响应体（与 `--body` 标志相同）。`--status` 和 `--body` 标志将始终覆盖此参数。

主体可以通过三种方式给出：标志、命令的最后一个（未命名）参数，或通过管道从 stdin 输入（如果标志和参数均未设置）。主体支持有限的 [模板求值](https://pkg.go.dev/text/template)，具有以下变量：

变量 | 描述
------|----------
`.N`       | 服务器编号
`.Port`    | 监听端口
`.Address` | 监听地址


#### 示例

在随机端口上的空 200 响应：
<pre><code class="cmd bash">caddy respond</code></pre>

带主体的 HTTP 响应：
<pre><code class="cmd bash">caddy respond "Hello, world!"</code></pre>

多个服务器和模板：
<pre><code class="cmd"><b>$ caddy respond --listen :2000-2004 "{{printf "I'm server {{.N}} on port {{.Port}}"}}"</b>

Server address: [::]:2000
Server address: [::]:2001
Server address: [::]:2002
Server address: [::]:2003
Server address: [::]:2004

<b>$ curl 127.0.0.1:2002</b>
I'm server 2 on port 2002</code></pre>

管道输入维护页面：
<pre><code class="cmd bash">cat maintenance.html | caddy respond \
	--listen :80 \
	--status 503 \
	--header "Content-Type: text/html"</code></pre>




### `caddy reverse-proxy`

<pre><code class="cmd bash">caddy reverse-proxy
	[-f, --from &lt;addr&gt;]
	(-t, --to &lt;addr&gt;)
	[-H, --header-up "&lt;Field&gt;: &lt;value&gt;"]
	[-d, --header-down "&lt;Field&gt;: &lt;value&gt;"]
	[-c, --change-host-header]
	[-r, --disable-redirects]
	[-i, --internal-certs]
	[-v, --debug]
	[--access-log]
	[--insecure]</code></pre>

一个简单但可用于生产的反向代理。适用于快速部署、演示和开发。

只需将 HTTP(S) 流量从 `--from` 地址发送到 `--to` 地址。可以通过重复标志指定多个 `--to` 地址。至少需要一个 `--to` 地址。`--to` 地址可以有端口范围作为快捷方式，以扩展到多个上游。

除非在地址中另有指定，否则 `--from` 地址在给出主机名时将被假定为 HTTPS，而 `--to` 地址将被假定为 HTTP。

如果 `--from` 地址具有主机或 IP，Caddy 将尝试通过 HTTPS 提供代理服务（除非被 HTTP 方案或端口覆盖）。

如果提供 HTTPS：
  - `--disable-redirects` 可用于避免绑定到 HTTP 端口。

  - `--internal-certs` 可用于强制使用内部 CA 颁发证书，而不是尝试颁发公共证书。

对于代理：
  - `--header-up` 可用于设置发送到上游的请求头。
  
  - `--header-down` 可用于设置返回给客户端的响应头。
  
  - `--change-host-header` 将请求中的 Host 头设置为上游的地址，而不是默认为传入的 Host 头。

    这是 `--header-up "Host: {http.reverse_proxy.upstream.hostport}"` 的快捷方式。
  
  - `--insecure` 禁用与上游的 TLS 验证。警告：这将禁用安全性，因为不会验证上游的证书。
  
  - `--debug` 启用详细日志记录。

此命令禁用管理 API，以便更容易在本地开发机器上运行多个实例。



### `caddy run`

<pre><code class="cmd bash">caddy run
	[-c, --config &lt;path&gt;]
	[-a, --adapter &lt;name&gt;]
	[--pidfile &lt;file&gt;]
	[-e, --environ]
	[--envfile &lt;file&gt;]
	[-r, --resume]
	[-w, --watch]</code></pre>

运行 Caddy 并无限期阻塞；即“守护进程”模式。

`--config` 指定要立即加载和使用的初始配置文件。如果是 `-`，则从 stdin 读取配置。如果未指定配置，Caddy 将以空白配置运行，并使用 [管理 API 端点](/docs/api) 的默认设置，这些端点可用于提供新配置。作为一种特殊情况，如果当前工作目录有一个名为“Caddyfile”的文件，并且 `caddyfile` 配置适配器已插入（默认），那么即使没有任何命令行标志，也会加载并使用该文件来配置 Caddy。

`--adapter` 是在加载初始配置时使用的配置适配器的名称（如果有）。如果 `--config` 文件名以 `Caddyfile` 开头或以 `.caddyfile` 结尾，则此标志不是必需的，因为会假定使用 `caddyfile` 适配器。否则，如果提供的配置文件不是 Caddy 的原生 JSON 格式，则此标志是必需的。任何警告都将打印到日志中，但请注意，任何无错误的适配将立即使用，即使存在警告也是如此。如果你想先查看适配结果，请使用 [`caddy adapt`](#caddy-adapt) 子命令。

`--pidfile` 将 PID 写入指定文件。

`--environ` 在启动前打印环境变量。这与 `caddy environ` 命令相同，但打印后不会退出。

`--envfile` 从指定文件加载环境变量，格式为 `KEY=VALUE`。支持以 `#` 开头的注释；键可以带有 `export` 前缀；值可以用双引号括起来（内部的双引号可以转义）；支持多行值。

`--resume` 使用上次自动保存的配置，覆盖 `--config` 标志（如果存在）。使用此标志可确保通过机器重启或进程重启实现配置持久性。它在以 [API](/docs/api) 为中心的部署中最有用。

`--watch` 将监视配置文件并在其更改后自动重新加载。⚠️ 此功能仅用于本地开发环境！

<aside class="advice">

在生产环境中运行时，不要停止服务器来更改配置！这将导致停机。（这应该是显而易见的，但你会惊讶于我们收到多少关于此事的投诉。）请改用 [`caddy reload`](#caddy-reload) 命令，或向进程发送 `SIGUSR1` 信号，其效果与使用当前加载的配置执行 `caddy reload` 相同。

</aside>



### `caddy start`

<pre><code class="cmd bash">caddy start
	[-c, --config &lt;path&gt;]
	[-a, --adapter &lt;name&gt;]
	[--envfile &lt;file&gt;]
	[--pidfile &lt;file&gt;]
	[-w, --watch]</code></code></pre>

与 [`caddy run`](#caddy-run) 相同，但在后台运行。此命令仅在后台进程成功运行（或运行失败）后阻塞，然后返回。

注意：`--config` 标志 _不_ 支持 `-` 从 stdin 读取配置。

不建议在系统服务或 Windows 上使用此命令。在 Windows 上，子进程仍将附加到终端，因此关闭窗口将强制停止 Caddy，这一点并不明显。请考虑将 Caddy [作为服务运行](/docs/running)。

启动后，你可以使用 [`caddy stop`](#caddy-stop) 或 [`POST /stop`](/docs/api#post-stop) API 端点退出后台进程。



### `caddy stop`

<pre><code class="cmd bash">caddy stop
	[--address &lt;interface&gt;]
	[-c, --config &lt;path&gt; [-a, --adapter &&lt;name&gt;]]</code></pre>

<aside class="tip">

停止（和重启）服务器与配置更改是正交的。**不要在生产环境中使用 stop 命令来更改配置，除非你想要停机。** 请改用 [`caddy reload`](#caddy-reload) 命令。

</aside>


优雅地停止正在运行的 Caddy 进程（不是 stop 命令本身的进程）并使其退出。它使用管理 API 的 [`POST /stop`](/docs/api#post-stop) 端点执行优雅关闭。

此请求的地址可以使用 `--address` 标志自定义，或者如果正在运行的实例的管理 API 未使用默认监听地址，则可以从提供的 `--config` 中获取。

如果你想停止当前配置但不想退出进程，请使用带有空白配置的 [`caddy reload`](#caddy-reload)，或 [`DELETE /config/`](/docs/api#delete-configpath) 端点。


### `caddy storage`

<i>⚠️ 实验性</i>

允许导出和导入 Caddy 配置的数据存储的内容。

当需要从一个[存储模块](/docs/json/storage/)过渡到另一个时，通过从旧存储导出，更新配置，然后导入到新存储，这很有用。

以下命令可用于一次将存储复制到不同模块之间，使用新旧配置，将导出命令的输出管道输入到导入命令。

```
$ caddy storage export -c Caddyfile.old -o- |
  caddy storage import -c Caddyfile.new -i-
```

<aside class="advice">

请注意，当使用[文件系统存储](/docs/conventions#data-directory)时，你必须以 Caddy 通常运行的用户身份运行导出命令，否则可能会使用错误的存储位置。

例如，当 Caddy 作为 [systemd 服务](/docs/running#linux-service) 运行时，它将作为 `caddy` 用户运行，因此你应该以该用户身份运行导出或导入命令。这通常可以通过 `sudo -u caddy <command>` 完成。

</aside>


#### `caddy storage export`

<pre><code class="cmd bash">caddy storage export
	-c, --config &lt;path&gt;
	[-o, --output &lt;path&gt;]</code></pre>

`--config` 是要加载的配置文件。这是必需的，以便连接正确的存储模块。

`--output` 是要写入 tar 包的文件名。如果是 `-`，则输出写入 stdout。



#### `caddy storage import`

<pre><code class="cmd bash">caddy storage import
	-c, --config &lt;path&gt;
	-i, --input &lt;path&gt;</code></pre>

`--config` 是要加载的配置文件。这是必需的，以便连接正确的存储模块。

`--input` 是要读取的 tar 包的文件名。如果是 `-`，则从 stdin 读取输入。


### `caddy trust`

<pre><code class="cmd bash">caddy trust
	[--ca &lt;id&gt;]
	[--address &lt;interface&gt;]
	[-c, --config &lt;path&gt; [-a, --adapter &lt;name&gt;]]</code></pre>

将 Caddy 的 [PKI 应用](/docs/json/apps/pki/) 管理的 CA 的根证书安装到本地信任存储中。

Caddy 将在首次生成根证书时尝试自动将其安装到本地信任存储中，但如果 Caddy 没有写入信任存储的适当权限，则可能会失败。如果服务器进程以非特权用户身份运行（例如通过 systemd），则此命令对于在使用证书之前预安装证书是必需的。在 Unix 系统上，你可能需要使用 `sudo` 运行此命令。

默认情况下，此命令安装 Caddy 默认 CA（即“local”）的根证书。你可以使用 `--ca` 标志指定另一个 CA 的 ID。

此命令将尝试连接到 Caddy 的 [管理 API](/docs/api) 以获取根证书，使用 [`GET /pki/ca/<id>/certificates`](/docs/api#get-pkicaltidgtcertificates) 端点。如果正在运行的实例的管理 API 未使用默认监听地址，你可以显式指定 `--address`，或使用 `--config` 标志从配置中加载管理地址。

如果管理 API 可供其他机器访问，你也可以使用带有此命令的 `caddy` 二进制文件在网络中的其他机器上安装证书——但执行此操作时要小心，不要将管理 API 暴露给不受信任的客户端。


### `caddy untrust`

<pre><code class="cmd bash">caddy untrust
	[-p, --cert &lt;path&gt;]
	[--ca &lt;id&gt;]
	[--address &lt;interface&gt;]
	[-c, --config &lt;path&gt; [-a, --adapter &lt;name&gt;]]</code></pre>

从本地信任存储中取消信任根证书。

此命令卸载信任；它不一定完全从信任存储中删除根证书。因此，重复信任和取消信任新证书可能会填满信任数据库。

此命令不会从 Caddy 配置的存储中删除或修改证书文件。

此命令可以通过两种方式之一使用：
- 通过使用 `--cert` 标志指定要取消信任的根证书的直接路径。
- 通过从[管理 API](/docs/api) 使用 [`GET /pki/ca/<id>/certificates`](/docs/api#get-pkicaidcertificates) 端点获取根证书。如果未给出任何标志，这是默认行为。

如果使用管理 API，则 CA ID 默认为 "local"。你可以使用 `--ca` 标志指定另一个 CA 的 ID。如果正在运行的实例的管理 API 未使用默认监听地址，你可以显式指定 `--address`，或使用 `--config` 标志从配置中加载管理地址。


### `caddy upgrade`

<i>⚠️ 实验性</i>

<pre><code class="cmd bash">caddy upgrade
	[-k, --keep-backup]</code></pre>

用来自[我们的下载页面](/download)的最新版本替换当前的 Caddy 二进制文件，并安装相同的模块，包括在 Caddy 网站上注册的所有第三方插件。

升级不会中断正在运行的服务器；目前，该命令仅替换磁盘上的二进制文件。如果我们能找到一种好的方法，这可能在将来改变。

升级过程具有容错性；首先备份当前二进制文件（复制到当前二进制文件旁边），并在出现任何问题时自动恢复。如果你希望在升级过程完成后保留备份，可以使用 `--keep-backup` 选项。

如果你的用户没有写入可执行文件的权限，则此命令可能需要提升的权限。



### `caddy add-package`

<i>⚠️ 实验性</i>

<pre><code class="cmd bash">caddy add-package &lt;packages...&gt;
	[-k, --keep-backup]</code></pre>

与 `caddy upgrade` 类似，用最新版本替换当前的 Caddy 二进制文件，并安装相同的模块，_以及_ 在参数中列出的包包含在新的二进制文件中。从[我们的下载页面](/download)查找你可以安装的包列表。每个参数应该是完整的包名。

例如：

<pre><code class="cmd bash">caddy add-package github.com/caddy-dns/cloudflare</code></pre>



### `caddy remove-package`

<i>⚠️ 实验性</i>

<pre><code class="cmd bash">caddy remove-package &lt;packages...&gt;
	[-k, --keep-backup]</code></pre>

与 `caddy upgrade` 类似，用最新版本替换当前的 Caddy 二进制文件，并安装相同的模块，但 _不包含_ 参数中列出的包（如果它们存在于当前二进制文件中）。运行 `caddy list-modules --packages` 查看当前二进制文件中包含的非标准模块的包名列表。



### `caddy validate`

<pre><code class="cmd bash">caddy validate
	[-c, --config &lt;path&gt;]
	[-a, --adapter &<name>]
	[--envfile &<file>]</code></pre>

验证配置文件，然后退出。此命令反序列化配置，然后加载并预配其所有模块，就像要启动配置一样，但实际不会启动。这可以暴露在加载或预配阶段出现的配置错误，并且是比仅将配置序列化为 JSON 更强的错误检查。

`--config` 是要验证的配置文件。如果是 `-`，则从 stdin 读取配置。默认为当前目录中的 `Caddyfile`（如果存在）。

`--adapter` 是要使用的配置适配器的名称。如果 `--config` 文件名以 `Caddyfile` 开头或以 `.caddyfile` 结尾，则此标志不是必需的，因为会假定使用 `caddyfile` 适配器。否则，如果提供的配置文件不是 Caddy 的原生 JSON 格式，则此标志是必需的。

`--envfile` 从指定文件加载环境变量，格式为 `KEY=VALUE`。支持以 `#` 开头的注释；键可以带有 `export` 前缀；值可以用双引号括起来（内部的双引号可以转义）；支持多行值。



### `caddy version`
<pre><code class="cmd bash">caddy version</code></pre>

打印版本并退出。



## 信号

Caddy 捕获某些信号并忽略其他信号。信号可以触发特定的进程行为。

信号 | 行为
-------|----------
`SIGINT` | 优雅退出。再次发送信号以立即强制退出。
`SIGQUIT` | 立即退出 Caddy，但**仍然清理存储中的锁**，因为这很重要。
`SIGTERM` | 优雅退出。
`SIGUSR1` | 重新加载配置文件，但仅当使用 `caddy run`（不带 `--resume`）启动且未通过[API](/docs/api)（包括 [`caddy reload`](#caddy-reload)）对配置进行任何更改时。
`SIGUSR2` | 忽略。
`SIGHUP` | 忽略。

优雅退出意味着不再接受新连接，并在关闭套接字之前排空现有连接。可能会应用一个宽限期（并且是可配置的）。宽限期结束后，连接将被强制终止。在优雅关闭期间，会清理存储中的锁以及各个模块需要释放的其他资源。

当收到重新加载配置的信号（`SIGUSR1`）时，它就像强制重新加载配置一样（即使配置文本未更改也重新加载），这可能会重新加载依赖文件，如磁盘上的 TLS 证书。

基于信号的配置重新加载仅在 Caddy 使用配置文件通过 `caddy run` 启动时启用。如果 Caddy 使用 `--resume` 启动（因为这意味着 API 工作流），或者通过管理 API 收到任何配置更改，或者 `caddy reload` 使用与最初启动时 _不同_ 的文件名或配置适配器运行，则它们将被禁用（信号被忽略，并记录警告）。这是为了避免重新加载方法之间的冲突。



## 退出码

Caddy 在进程退出时返回一个代码：

代码 | 含义
-----|---------
`0` | 正常退出。
`1` | 启动失败。**不要自动重启进程；它很可能会再次出错，除非进行更改。**
`2` | 强制退出。Caddy 被强制退出，没有清理资源。
`3` | 退出失败。Caddy 在清理过程中出现一些错误。

在 bash 中，你可以使用 `echo $?` 获取最后一个命令的退出码。