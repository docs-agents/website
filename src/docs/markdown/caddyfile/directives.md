---
title: Caddyfile 指令
---

<style>
#directive-table table {
	margin: 0 auto;
	overflow: hidden;
}

#directive-table tr:hover {
	background: rgba(109, 226, 255, 0.11);
}

#directive-table tr td:first-child {
	position: relative;
}

#directive-table a:before {
	content: '';
	position: absolute;
	left: 0;
	top: 0;
	bottom: 0;
	display: block;
	width: 100vw;
}
</style>

# Caddyfile 指令

指令是出现在站点 [代码块](/docs/caddyfile/concepts#blocks) 内的职能关键字。有时，它们可以打开自己的代码块，其中可以包含 _子指令_，但除非另有说明，否则指令**不能**用于其他指令内。例如，您不能在 `file_server` 代码块内使用 `basic_auth`，因为 `file_server` 不知道如何进行身份验证。但是，您可以在 `handle` 和 `route` 等特殊指令代码块内使用某些指令，因为它们专门设计用于分组 HTTP 处理器指令。

- [语法](#syntax)
- [指令顺序](#directive-order)
- [排序算法](#sorting-algorithm)

以下是 Caddy 自带的指令，可用于 HTTP Caddyfile：

<div id="directive-table">

指令 | 描述
----------|------------
**[abort](/docs/caddyfile/directives/abort)** | 中止 HTTP 请求
**[acme_server](/docs/caddyfile/directives/acme_server)** | 嵌入的 ACME 服务器
**[basic_auth](/docs/caddyfile/directives/basic_auth)** | 强制执行 HTTP 基本身份验证
**[bind](/docs/caddyfile/directives/bind)** | 自定义服务器的套接字地址
**[encode](/docs/caddyfile/directives/encode)** | 编码（通常压缩）响应
**[error](/docs/caddyfile/directives/error)** | 触发错误
**[file_server](/docs/caddyfile/directives/file_server)** | 从磁盘提供文件
**[forward_auth](/docs/caddyfile/directives/forward_auth)** | 将身份验证委托给外部服务
**[fs](/docs/caddyfile/directives/fs)** | 设置用于文件 I/O 的文件系统
**[handle](/docs/caddyfile/directives/handle)** | 一组互斥的指令
**[handle_errors](/docs/caddyfile/directives/handle_errors)** | 定义用于处理错误的路由
**[handle_path](/docs/caddyfile/directives/handle_path)** | 类似于 handle，但剥离路径前缀
**[header](/docs/caddyfile/directives/header)** | 设置或删除响应头
**[import](/docs/caddyfile/directives/import)** | 包含片段或文件
**[intercept](/docs/caddyfile/directives/intercept)** | 拦截其他处理器写入的响应
**[invoke](/docs/caddyfile/directives/invoke)** | 调用命名路由
**[log](/docs/caddyfile/directives/log)** | 启用访问/请求日志记录
**[log_append](/docs/caddyfile/directives/log_append)** | 向访问日志添加字段
**[log_skip](/docs/caddyfile/directives/log_skip)** | 跳过匹配请求的访问日志记录
**[log_name](/docs/caddyfile/directives/log_name)** | 覆盖要写入的日志记录器名称
**[map](/docs/caddyfile/directives/map)** | 将输入值映射到一个或多个输出
**[method](/docs/caddyfile/directives/method)** | 在内部更改 HTTP 方法
**[metrics](/docs/caddyfile/directives/metrics)** | 配置 Prometheus 指标公开端点
**[php_fastcgi](/docs/caddyfile/directives/php_fastcgi)** | 通过 FastCGI 提供 PHP 站点
**[push](/docs/caddyfile/directives/push)** | 使用 HTTP/2 服务器推送向客户端推送内容
**[redir](/docs/caddyfile/directives/redir)** | 向客户端发出 HTTP 重定向
**[request_body](/docs/caddyfile/directives/request_body)** | 操作请求体
**[request_header](/docs/caddyfile/directives/request_header)** | 操作请求头
**[respond](/docs/caddyfile/directives/respond)** | 向客户端写入硬编码响应
**[reverse_proxy](/docs/caddyfile/directives/reverse_proxy)** | 强大且可扩展的反向代理
**[rewrite](/docs/caddyfile/directives/rewrite)** | 内部重写请求
**[root](/docs/caddyfile/directives/root)** | 设置站点根目录路径
**[route](/docs/caddyfile/directives/route)** | 一组作为单个单元按字面意义处理的指令
**[templates](/docs/caddyfile/directives/templates)** | 在响应上执行模板
**[tls](/docs/caddyfile/directives/tls)** | 自定义 TLS 设置
**[tracing](/docs/caddyfile/directives/tracing)** | 与 OpenTelemetry 追踪集成
**[try_files](/docs/caddyfile/directives/try_files)** | 依赖于文件存在性的重写
**[uri](/docs/caddyfile/directives/uri)** | 操作 URI
**[vars](/docs/caddyfile/directives/vars)** | 设置任意变量

</div>

## 语法

每个指令的语法如下所示：

```caddy-d
directive [<matcher>] <args...> {
	subdirective [<args...>]
}
```

`<尖括号>` 表示需要用实际值替换的标记。

`[方括号]` 表示可选参数。

省略号 `...` 表示延续，即一个或多个参数或行。

除非另有说明，否则子指令通常是可选的，即使它们没有出现在 `[方括号]` 中。


### 匹配器

大多数——但不是所有——指令接受 [匹配器标记](/docs/caddyfile/matchers#syntax)，这让您能够过滤请求。匹配器标记通常是可选的。如果您在指令的语法中看到此内容，则指令支持匹配器：

```caddy-d
[<matcher>]
```

由于所有匹配器标记的工作原理相同，因此匹配器标记的各种可能性不会在每页上描述，以减少重复。请参阅 [匹配器文档](/docs/caddyfile/matchers) 以获取语法的详细解释。


## 指令顺序

许多指令操作 HTTP 处理器链。这些指令的评估顺序很重要，因此 Caddy 中硬编码了默认顺序。

您可以使用 [`order` 全局选项](/docs/caddyfile/options#order) 或 [`route` 指令](/docs/caddyfile/directives/route) 来覆盖/自定义此顺序。

```caddy-d
tracing

map
vars
fs
root
log_append
log_skip
log_name

header
copy_response_headers # 仅在 reverse_proxy 的 handle_response 代码块中
request_body

redir

# 传入请求操作
method
rewrite
uri
try_files

# 中间件处理器；一些包装响应
basic_auth
forward_auth
request_header
encode
push
intercept
templates

# 特殊路由和调度指令
invoke
handle
handle_path
route

# 通常响应请求的处理器
abort
error
copy_response # 仅在 reverse_proxy 的 handle_response 代码块中
respond
metrics
reverse_proxy
php_fastcgi
file_server
acme_server
```



## 排序算法

为了方便使用，Caddyfile 适配器按以下规则对指令进行排序：

- 不同名的指令按其 [默认顺序](#directive-order) 中的位置进行排序。默认顺序可以使用 [`order` 全局选项](/docs/caddyfile/options) 覆盖。来自插件的指令 _没有_ 顺序，因此应使用 [`order`](/docs/caddyfile/options) 全局选项或 [`route`](/docs/caddyfile/directives/route) 指令来设置顺序。

- 同名指令按它们的 [匹配器](/docs/caddyfile/matchers#syntax) 进行排序。

  - 最高优先级是带有单个 [路径匹配器](/docs/caddyfile/matchers#path-matchers) 的指令。

    路径匹配器按具体性排序，从最具体到最不具体。
	
	通常，这是通过按路径匹配器的长度排序来执行。有一个例外，如果路径以 `*` 结尾，且两个匹配器的路径其他部分相同，则没有 `*` 的匹配器被认为是更具体的，并且排序更高。

    例如：
    - `/foobar` 比 `/foo` 更具体
    - `/foo` 比 `/foo*` 更具体
    - `/foo/*` 比 `/foo*` 更具体

  - 带有任何其他匹配器的指令接下来排序，按它们在 Caddyfile 中出现的顺序。

    这包括具有多个值的路径匹配器和 [命名匹配器](/docs/caddyfile/matchers#named-matchers)。

  - 没有匹配器的指令（即匹配所有请求）排在最后。

- [`vars`](/docs/caddyfile/directives/vars) 指令的匹配器排序顺序是相反的，因为它涉及设置值，这些值可能会相互覆盖，因此最具体的匹配器应该最后评估。

- [`route`](/docs/caddyfile/directives/route) 指令的内容忽略上述所有规则，并保留指令在其中的出现顺序。
