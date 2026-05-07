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

指令是出现在站点[块](/docs/caddyfile/concepts#blocks)内的功能关键字。有时，它们可能会开启自己的块，其中可以包含_子指令_，但除非另有说明，指令**不能**在其他指令内部使用。例如，你不能在 `file_server` 块内使用 `basic_auth`，因为 `file_server` 不知道如何进行身份验证。然而，你_可以_在某些特殊指令块（如 `handle` 和 `route`）内使用一些指令，因为它们专门设计用于对HTTP处理程序指令进行分组。

- [语法](#语法)
- [指令顺序](#指令顺序)
- [排序算法](#排序算法)

以下指令是 Caddy 标准自带的，可以在 HTTP Caddyfile 中使用：

<div id="directive-table">

指令 | 描述
----------|------------
**[abort](/docs/caddyfile/directives/abort)** | 中止 HTTP 请求
**[acme_server](/docs/caddyfile/directives/acme_server)** | 嵌入式 ACME 服务器
**[basic_auth](/docs/caddyfile/directives/basic_auth)** | 强制 HTTP 基本认证
**[bind](/docs/caddyfile/directives/bind)** | 自定义服务器的套接字地址
**[encode](/docs/caddyfile/directives/encode)** | 对响应进行编码（通常为压缩）
**[error](/docs/caddyfile/directives/error)** | 触发错误
**[file_server](/docs/caddyfile/directives/file_server)** | 从磁盘提供文件服务
**[forward_auth](/docs/caddyfile/directives/forward_auth)** | 将身份验证委托给外部服务
**[fs](/docs/caddyfile/directives/fs)** | 设置用于文件 I/O 的文件系统
**[handle](/docs/caddyfile/directives/handle)** | 互斥的指令组
**[handle_errors](/docs/caddyfile/directives/handle_errors)** | 定义用于处理错误的路由
**[handle_path](/docs/caddyfile/directives/handle_path)** | 类似 handle，但会去除路径前缀
**[header](/docs/caddyfile/directives/header)** | 设置或移除响应头
**[import](/docs/caddyfile/directives/import)** | 包含代码片段或文件
**[intercept](/docs/caddyfile/directives/intercept)** | 拦截其他处理程序写入的响应
**[invoke](/docs/caddyfile/directives/invoke)** | 调用命名路由
**[log](/docs/caddyfile/directives/log)** | 启用访问/请求日志记录
**[log_append](/docs/caddyfile/directives/log_append)** | 向访问日志追加字段
**[log_skip](/docs/caddyfile/directives/log_skip)** | 跳过匹配请求的访问日志记录
**[log_name](/docs/caddyfile/directives/log_name)** | 覆盖写入的日志名称
**[map](/docs/caddyfile/directives/map)** | 将输入值映射到一个或多个输出
**[method](/docs/caddyfile/directives/method)** | 内部更改 HTTP 方法
**[metrics](/docs/caddyfile/directives/metrics)** | 配置 Prometheus 指标暴露端点
**[php_fastcgi](/docs/caddyfile/directives/php_fastcgi)** | 通过 FastCGI 提供 PHP 站点服务
**[push](/docs/caddyfile/directives/push)** | 使用 HTTP/2 服务器推送向客户端推送内容
**[redir](/docs/caddyfile/directives/redir)** | 向客户端发出 HTTP 重定向
**[request_body](/docs/caddyfile/directives/request_body)** | 操作请求体
**[request_header](/docs/caddyfile/directives/request_header)** | 操作请求头
**[respond](/docs/caddyfile/directives/respond)** | 向客户端写入硬编码响应
**[reverse_proxy](/docs/caddyfile/directives/reverse_proxy)** | 强大且可扩展的反向代理
**[rewrite](/docs/caddyfile/directives/rewrite)** | 内部重写请求
**[root](/docs/caddyfile/directives/root)** | 设置站点根路径
**[route](/docs/caddyfile/directives/route)** | 一组被当作单一单元处理的指令
**[templates](/docs/caddyfile/directives/templates)** | 对响应执行模板
**[tls](/docs/caddyfile/directives/tls)** | 自定义 TLS 设置
**[tracing](/docs/caddyfile/directives/tracing)** | 与 OpenTelemetry 追踪集成
**[try_files](/docs/caddyfile/directives/try_files)** | 依赖文件存在性的重写
**[uri](/docs/caddyfile/directives/uri)** | 操作 URI
**[vars](/docs/caddyfile/directives/vars)** | 设置任意变量

</div>

## 语法

每条指令的语法大致如下：

```caddy-d
directive [<matcher>] <args...> {
	subdirective [<args...>]
}
```

`<尖括号>` 表示需要替换为实际值的标记。

`[方括号]` 表示可选参数。

省略号 `...` 表示可延续，即一个或多个参数或行。

除非文档另有说明，子指令通常是可选的，即使它们没有出现在 `[方括号]` 中。

### 匹配器

大多数（但不是全部）指令接受[匹配器标记](/docs/caddyfile/matchers#语法)，用于过滤请求。匹配器标记通常是可选的。如果在指令的语法中看到以下内容，则该指令支持匹配器：

```caddy-d
[<matcher>]
```

由于匹配器标记的工作方式相同，为了减少重复，不会在每个页面上描述匹配器标记的各种可能性。相反，请参考[匹配器文档](/docs/caddyfile/matchers)以了解语法的详细解释。

## 指令顺序

许多指令会操作 HTTP 处理程序链。这些指令的评估顺序很重要，因此 Caddy 中硬编码了默认顺序。

你可以通过使用 [`order` 全局选项](/docs/caddyfile/options#order) 或 [`route` 指令](/docs/caddyfile/directives/route) 来覆盖/自定义此顺序。

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
copy_response_headers # 仅在 reverse_proxy 的 handle_response 块中
request_body

redir

# 入站请求操作
method
rewrite
uri
try_files

# 中间件处理程序；部分会包装响应
basic_auth
forward_auth
request_header
encode
push
intercept
templates

# 特殊路由与分发指令
invoke
handle
handle_path
route

# 通常对请求进行响应的处理程序
abort
error
copy_response # 仅在 reverse_proxy 的 handle_response 块中
respond
metrics
reverse_proxy
php_fastcgi
file_server
acme_server
```

## 排序算法

为便于使用，Caddyfile 适配器根据以下规则对指令进行排序：

- 不同名称的指令按其[默认顺序](#指令顺序)中的位置排序。默认顺序可以通过 [`order` 全局选项](/docs/caddyfile/options) 覆盖。来自插件的指令_没有_顺序，因此应使用 [`order`](/docs/caddyfile/options) 全局选项或 [`route`](/docs/caddyfile/directives/route) 指令来设置顺序。

- 相同名称的指令根据其[匹配器](/docs/caddyfile/matchers#语法)排序。

  - 最高优先级是带有单个[路径匹配器](/docs/caddyfile/matchers#路径匹配器)的指令。

    路径匹配器按特异性排序，从最具体到最不具体。

    通常，这是通过按路径匹配器的长度排序来完成的。有一个例外：如果路径以 `*` 结尾，并且两个匹配器的路径其余部分相同，则不带 `*` 的匹配器被认为更具体，排序更靠前。

    例如：
    - `/foobar` 比 `/foo` 更具体
    - `/foo` 比 `/foo*` 更具体
    - `/foo/*` 比 `/foo*` 更具体

  - 带有任何其他匹配器的指令排在其次，按它们在 Caddyfile 中出现的顺序排序。

    这包括具有多个值的路径匹配器和[命名匹配器](/docs/caddyfile/matchers#命名匹配器)。

  - 没有匹配器（即匹配所有请求）的指令排在最后。

- [`vars`](/docs/caddyfile/directives/vars) 指令的匹配器排序顺序相反，因为它涉及设置可以相互覆盖的值，所以最具体的匹配器应在最后评估。

- [`route`](/docs/caddyfile/directives/route) 指令的内容忽略上述所有规则，并保留指令在内部出现的顺序。