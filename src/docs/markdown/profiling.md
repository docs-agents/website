---
title: 分析 Caddy
---

分析 Caddy
================

**程序 profile（性能分析）** 是程序在运行时资源使用情况的快照。Profile 对于识别问题区域、排查错误和崩溃以及优化代码非常有帮助。

Caddy 使用 Go 的工具来捕获 profile，该工具名为 [pprof](https://github.com/google/pprof)，它内置于 `go` 命令中。

Profile 报告 CPU 和内存的消耗者，显示 goroutine 的堆栈跟踪，并帮助追踪死锁或高争用的同步原语。

在报告 Caddy 的某些错误时，我们可能会要求提供 profile。本文可以提供帮助。它描述了如何通过 Caddy 获取 profile，以及通常如何使用和解释生成的 pprof profile。

开始之前需要了解两件事：

1. **Caddy 的 profile 对安全不敏感。** 它们包含无害的技术读数，而不是内存内容。它们不会授予系统访问权限。分享它们是安全的。
2. **Profile 是轻量级的，可以在生产环境中收集。** 实际上，对于许多用户来说，这是一个推荐的实践；参见本文后面的内容。

## 获取 profiles

Profile 可通过[管理 API](/docs/api) 在 `/debug/pprof/` 路径下获取。在运行 Caddy 的机器上，在浏览器中打开：

```
http://localhost:2019/debug/pprof/
```

<aside class="tip">
	默认情况下，管理 API 只能在本地访问。如果在远程、虚拟机或容器中运行，请参阅下一节了解如何访问此端点。
</aside>

你会看到一个简单的计数和链接表格，例如：

计数 | Profile
----- | --------------------
79    | allocs
0     | block
0     | cmdline
22    | goroutine
79    | heap
0     | mutex
0     | profile
29    | threadcreate
0     | trace
|     | full goroutine stack dump

这些计数是快速识别泄漏的便捷方法。如果你怀疑有泄漏，请反复刷新页面，你会看到其中一个或多个计数不断增长。如果堆计数增长，可能是内存泄漏；如果 goroutine 计数增长，可能是 goroutine 泄漏。

点击这些 profile，看看它们的样子。有些可能为空，这很多时候是正常的。最常用的有 <b>goroutine</b>（函数栈）、<b>heap</b>（内存）和 <b>profile</b>（CPU）。其他 profile 可用于调试互斥锁争用或死锁。

在底部，每个 profile 都有一个简单的描述：

- **allocs:** 对所有过去内存分配的采样
- **block:** 导致同步原语阻塞的堆栈跟踪
- **cmdline:** 当前程序的命令行调用
- **goroutine:** 所有当前 goroutine 的堆栈跟踪。使用 debug=2 作为查询参数，以未恢复的 panic 相同的格式导出。
- **heap:** 活动对象的内存分配采样。你可以指定 gc GET 参数，以便在获取堆样本之前运行 GC。
- **mutex:** 有争用的互斥锁持有者的堆栈跟踪
- **profile:** CPU profile。你可以在 seconds GET 参数中指定持续时间。获取 profile 文件后，使用 `go tool pprof` 命令进行调查。
- **threadcreate:** 导致创建新操作系统线程的堆栈跟踪
- **trace:** 当前程序执行的跟踪。你可以在 seconds GET 参数中指定持续时间。获取跟踪文件后，使用 `go tool trace` 命令进行调查。

<aside class="tip">

"goroutine" 和 "full goroutine stack dump" 之间的区别在于 `?debug=2` 参数：完全堆栈转储就像你在 panic 后看到的输出；它更详细，并且值得注意的是，不会合并相同的 goroutine。

</aside>


### 下载 profiles

点击上面 pprof 索引页面上的链接，你会得到文本格式的 profile。这对于调试很有用，也是我们 Caddy 团队偏爱的格式，因为我们可以扫描它以寻找明显的线索，而无需额外的工具。

但二进制格式实际上是默认格式。HTML 链接附加了 `?debug=` 查询字符串参数以将它们格式化为文本，但（CPU）"profile" 链接除外，它没有文本表示。

以下是你可以设置的查询字符串参数（来自 [Go 文档](https://pkg.go.dev/net/http/pprof#hdr-Parameters)）：

- **`debug=N`（所有 profile 除了 cpu）：** 响应格式：N = 0：二进制（默认），N > 0：纯文本
- **`gc=N`（heap profile）：** N > 0：在分析之前运行一次垃圾回收循环
- **`seconds=N`（allocs, block, goroutine, heap, mutex, threadcreate profiles）：** 返回增量 profile
- **`seconds=N`（cpu, trace profiles）：** 在给定的持续时间内进行分析

由于这些是 HTTP 端点，你也可以使用任何 HTTP 客户端（如 curl 或 wget）来下载 profile。

下载 profile 后，你可以将其上传到 GitHub 问题评论，或使用 [pprof.me](https://pprof.me/) 之类的网站。对于 CPU 配置文件，[flamegraph.com](https://flamegraph.com/) 是另一个选择。


## 远程访问

_如果你已经能够在本地访问管理 API，请跳过此部分。_

默认情况下，Caddy 的管理 API 只能通过本地环回套接字访问。但是，至少有 3 种方法可以远程访问 Caddy 的 `/debug/pprof` 端点：

### 通过你的站点反向代理

一个简单的选择是直接在你的站点上进行反向代理：

```caddy-d
reverse_proxy /debug/pprof/* localhost:2019 {
	header_up Host {upstream_hostport}
}
```

当然，这将使 profile 对能够连接到你站点的任何人可用。如果不需要，你可以使用你选择的 HTTP 认证模块添加一些身份验证。

（不要忘记 `/debug/pprof/*` 匹配器，否则你将代理整个管理 API！）


### SSH 隧道

另一种方法是使用 SSH 隧道。这是你的计算机和服务器之间使用 SSH 协议建立的加密连接。在你的计算机上运行如下命令：

<pre><code class="cmd bash">ssh -N username@example.com -L 8123:localhost:2019</code></pre>

这将 `localhost:8123`（在你的本地机器上）隧道到 `example.com` 上的 `localhost:2019`。确保根据需要替换 `username`、`example.com` 和端口。

<aside class="tip">

此命令将在前台运行。请记住，如果你尝试通过 <kbd>Ctrl</kbd>+<kbd>Z</kbd> 将进程置于后台，它将暂停隧道，使用隧道的连接将无法连接。

</aside>

然后在另一个终端中，你可以像这样运行 `curl`：

<pre><code class="cmd bash">curl -v http://localhost:8123/debug/pprof/ -H "Host: localhost:2019"</code></pre>

通过使用隧道的两端使用端口 `2019`，可以避免需要 `-H "Host: ..."`（但这要求你自己的电脑上尚未占用端口 `2019`，即本地没有运行 Caddy）。

在隧道处于活动状态时，你可以访问管理 API 的所有内容。在 `ssh` 命令上输入 <kbd>Ctrl</kbd>+<kbd>C</kbd> 关闭隧道。

#### 长期运行隧道

使用上述命令运行隧道需要你保持终端打开。如果你想在后台运行隧道，可以像这样启动它：

<pre><code class="cmd bash">ssh -f -N -M -S /tmp/caddy-tunnel.sock username@example.com -L 8123:localhost:2019</code></pre>

这将在后台启动，并在 `/tmp/caddy-tunnel.sock` 处创建一个控制套接字。然后，当你使用完隧道后，可以使用控制套接字关闭它：

<pre><code class="cmd bash">ssh -S /tmp/caddy-tunnel.sock -O exit e</code></pre>


### 远程管理 API

你还可以将管理 API 配置为接受来自授权客户端的远程连接。

（TODO：撰写关于此的文章。）



## Goroutine 配置分析

Goroutine 转储对于了解存在哪些 goroutine 以及它们的调用堆栈非常有用。换句话说，它让我们了解当前正在执行或阻塞/等待的代码。

如果你点击“goroutines”或访问 `/debug/pprof/goroutine?debug=1`，你会看到一个 goroutine 列表及其调用堆栈。例如：

```
goroutine profile: total 88
23 @ 0x43e50e 0x436d37 0x46bda5 0x4e1327 0x4e261a 0x4e2608 0x545a65 0x5590c5 0x6b2e9b 0x50ddb8 0x6b307e 0x6b0650 0x6b6918 0x6b6921 0x4b8570 0xb11a05 0xb119d4 0xb12145 0xb1d087 0x4719c1
#	0x46bda4	internal/poll.runtime_pollWait+0x84			runtime/netpoll.go:343
#	0x4e1326	internal/poll.(*pollDesc).wait+0x26			internal/poll/fd_poll_runtime.go:84
#	0x4e2619	internal/poll.(*pollDesc).waitRead+0x279		internal/poll/fd_poll_runtime.go:89
#	0x4e2607	internal/poll.(*FD).Read+0x267				internal/poll/fd_unix.go:164
#	0x545a64	net.(*netFD).Read+0x24					net/fd_posix.go:55
#	0x5590c4	net.(*conn).Read+0x44					net/net.go:179
#	0x6b2e9a	crypto/tls.(*atLeastReader).Read+0x3a			crypto/tls/conn.go:805
#	0x50ddb7	bytes.(*Buffer).ReadFrom+0x97				bytes/buffer.go:211
#	0x6b307d	crypto/tls.(*Conn).readFromUntil+0xdd			crypto/tls/conn.go:827
#	0x6b064f	crypto/tls.(*Conn).readRecordOrCCS+0x24f		crypto/tls/conn.go:625
#	0x6b6917	crypto/tls.(*Conn).readRecord+0x157			crypto/tls/conn.go:587
#	0x6b6920	crypto/tls.(*Conn).Read+0x160				crypto/tls/conn.go:1369
#	0x4b856f	io.ReadAtLeast+0x8f					io/io.go:335
#	0xb11a04	io.ReadFull+0x64					io/io.go:354
#	0xb119d3	golang.org/x/net/http2.readFrameHeader+0x33		golang.org/x/net@v0.14.0/http2/frame.go:237
#	0xb12144	golang.org/x/net/http2.(*Framer).ReadFrame+0x84		golang.org/x/net@v0.14.0/http2/frame.go:498
#	0xb1d086	golang.org/x/net/http2.(*serverConn).readFrames+0x86	golang.org/x/net@v0.14.0/http2/server.go:818

1 @ 0x43e50e 0x44e286 0xafeeb3 0xb0af86 0x5c29fc 0x5c3225 0xb0365b 0xb03650 0x15cb6af 0x43e09b 0x4719c1
#	0xafeeb2	github.com/caddyserver/caddy/v2/cmd.cmdRun+0xcd2					github.com/caddyserver/caddy/v2@v2.7.4/cmd/commandfuncs.go:277
#	0xb0af85	github.com/caddyserver/caddy/v2/cmd.init.1.func2.WrapCommandFuncForCobra.func1+0x25	github.com/caddyserver/caddy/v2@v2.7.4/cmd/cobra.go:126
#	0x5c29fb	github.com/spf13/cobra.(*Command).execute+0x87b						github.com/spf13/cobra@v1.7.0/command.go:940
#	0x5c3224	github.com/spf13/cobra.(*Command).ExecuteC+0x3a4					github.com/spf13/cobra@v1.7.0/command.go:1068
#	0xb0365a	github.com/spf13/cobra.(*Command).Execute+0x5a						github.com/spf13/cobra@v1.7.0/command.go:992
#	0xb0364f	github.com/caddyserver/caddy/v2/cmd.Main+0x4f						github.com/caddyserver/caddy/v2@v2.7.4/cmd/main.go:65
#	0x15cb6ae	main.main+0xe										caddy/main.go:11
#	0x43e09a	runtime.main+0x2ba									runtime/proc.go:267

1 @ 0x43e50e 0x44e9c5 0x8ec085 0x4719c1
#	0x8ec084	github.com/caddyserver/certmagic.(*Cache).maintainAssets+0x304	github.com/caddyserver/certmagic@v0.19.2/maintain.go:67

...
```

第一行 `goroutine profile: total 88` 告诉我们正在查看的内容以及共有多少个 goroutine。

接下来是 goroutine 列表。它们按调用堆栈分组，按出现频率降序排列。

一个 goroutine 行的语法是：`<count> @ <addresses...>`

行首是拥有该调用堆栈的 goroutine 数量。`@` 符号表示调用指令地址的开始，即函数指针，这些指针是 goroutine 的起源。每个指针都是一个函数调用，或调用帧。

你可能会注意到许多 goroutine 共享相同的第一个调用地址。这是程序的主入口点。有些 goroutine 不会从那里起源，因为程序有各种 `init()` 函数，Go 运行时也可能产生 goroutine。

接下来的行以 `#` 开头，实际上只是为了方便读者阅读的注释。它们包含 goroutine 的当前堆栈跟踪。顶部代表堆栈的顶部，即当前正在执行的代码行。底部代表堆栈的底部，即 goroutine 最初开始运行的代码。

堆栈跟踪的格式如下：

```
<address> <package/func>+<offset> <filename>:<line>
```

地址是函数指针，然后是 Go 包和函数名（如果是方法，则包含关联的类型名），以及函数内的指令偏移量。最后，也许是最有用的信息，文件和行号。

### 完整的 goroutine 堆栈转储

如果我们将查询字符串参数更改为 `?debug=2`，我们会得到完整的转储。这包括每个 goroutine 的详细堆栈跟踪，相同的 goroutine 不会被合并。在繁忙的服务器上，此输出可能非常大，但这是有趣的信息！

让我们看一个与上面第一个调用堆栈对应的转储（截断）：

```
goroutine 61961905 [IO wait, 1 minutes]:
internal/poll.runtime_pollWait(0x7f9a9a059eb0, 0x72)
	runtime/netpoll.go:343 +0x85
...
golang.org/x/net/http2.(*serverConn).readFrames(0xc001756f00)
	golang.org/x/net@v0.14.0/http2/server.go:818 +0x87
created by golang.org/x/net/http2.(*serverConn).serve in goroutine 61961902
	golang.org/x/net@v0.14.0/http2/server.go:930 +0x56a
```

尽管它很详细，但此转储唯一提供的最有用的信息是每个 goroutine 的第一行和最后一行。

第一行包含 goroutine 的编号 (61961905)、状态 ("IO wait") 和持续时间 ("1 minutes")：

- **goroutine 编号：** 是的，goroutine 有编号！但它们不会暴露给我们的代码。然而，这些编号在堆栈跟踪中特别有用，因为我们可以看到是哪个 goroutine 产生了这个（参见末尾："created by ... in goroutine 61961902"）。下面显示的工具帮助我们绘制视觉化图形。

- **状态：** 告诉我们 goroutine 当前正在做什么。以下是一些可能看到的状态：
	- `running`: 正在执行代码 - 很好！
	- `IO wait`: 等待网络。不消耗操作系统线程，因为它被挂起在非阻塞网络轮询器上。
	- `sleep`: 我们都需要的更多。
	- `select`: 阻塞在 select 上；等待某个 case 可用。
	- `select (no cases):` 专门阻塞在空 select `select {}` 上。Caddy 在其 main 中使用了一个，以保持运行，因为关闭是由其他 goroutine 发起的。
	- `chan receive`: 阻塞在通道接收上 (`<-ch`)。
	- `semacquire`: 等待获取信号量（低级同步原语）。
	- `syscall`: 正在执行系统调用。消耗一个操作系统线程。

- **持续时间：** goroutine 已经存在了多长时间。对于查找 goroutine 泄漏等错误非常有用。例如，如果我们期望所有网络连接在几分钟后关闭，那么当我们发现许多 netconn goroutine 存活数小时时，这意味着什么？

### 解读 goroutine 转储

在不查看代码的情况下，我们可以从上面的 goroutine 中学到什么？

它大约在一分钟前创建，正在等待网络套接字上的数据，并且它的 goroutine 编号非常大 (61961905)。

从第一个转储（debug=1）中，我们知道它的调用堆栈执行得相对频繁，并且大的 goroutine 编号加上短的持续时间表明存在数千万个这些相对短暂的 goroutine。它处于一个名为 `pollWait` 的函数中，它的调用历史包括从一个使用 TLS 的加密网络连接中读取 HTTP/2 帧。

因此，我们可以推断这个 goroutine 正在服务一个 HTTP/2 请求！它正在等待来自客户端的数据。此外，我们知道产生它的 goroutine 不是进程的第一个 goroutine，因为它也有一个高编号；在转储中找到该 goroutine 显示它是在一个现有请求期间为处理一个新的 HTTP/2 流而产生的。相比之下，其他具有高编号的 goroutine 可能由一个低编号的 goroutine（例如 32）产生，这表明这是一个全新的连接，刚刚从套接字的 `Accept()` 调用中出来。

每个程序都是不同的，但在调试 Caddy 时，这些模式往往是成立的。

## 内存分析

内存（或堆）profile 跟踪堆分配，这是系统内存的主要消耗者。分配也是性能问题的常见嫌疑人，因为分配内存需要系统调用，这可能会很慢。

堆 profile 看起来与 goroutine profile 几乎完全一样，除了顶行的开头。这是一个例子：

```
0: 0 [1: 4096] @ 0xb1fc05 0xb1fc4d 0x48d8d1 0xb1fce6 0xb184c7 0xb1bc8e 0xb41653 0xb4105c 0xb4151d 0xb23b14 0x4719c1
#	0xb1fc04	bufio.NewWriterSize+0x24					bufio/bufio.go:599
#	0xb1fc4c	golang.org/x/net/http2.glob..func8+0x6c				golang.org/x/net@v0.17.0/http2/http2.go:263
#	0x48d8d0	sync.(*Pool).Get+0xb0						sync/pool.go:151
#	0xb1fce5	golang.org/x/net/http2.(*bufferedWriter).Write+0x45		golang.org/x/net@v0.17.0/http2/http2.go:276
#	0xb184c6	golang.org/x/net/http2.(*Framer).endWrite+0xc6			golang.org/x/net@v0.17.0/http2/frame.go:371
#	0xb1bc8d	golang.org/x/net/http2.(*Framer).WriteHeaders+0x48d		golang.org/x/net@v0.17.0/http2/frame.go:1131
#	0xb41652	golang.org/x/net/http2.(*writeResHeaders).writeHeaderBlock+0xd2	golang.org/x/net@v0.17.0/http2/write.go:239
#	0xb4105b	golang.org/x/net/http2.splitHeaderBlock+0xbb			golang.org/x/net@v0.17.0/http2/write.go:169
#	0xb4151c	golang.org/x/net/http2.(*writeResHeaders).writeFrame+0x1dc	golang.org/x/net@v0.17.0/http2/write.go:234
#	0xb23b13	golang.org/x/net/http2.(*serverConn).writeFrameAsync+0x73	golang.org/x/net@v0.17.0/http2/server.go:851
```

第一行的格式如下：

```
<live objects> <live memory> [<allocations>: <allocation memory>] @ <addresses...>
```

在上面的例子中，有一个由 `bufio.NewWriterSize()` 进行的分配，但当前此调用堆栈没有活动对象。

有趣的是，我们可以从该调用堆栈推断出 http2 包使用了一个池化的 4 KB 来向客户端写入 HTTP/2 帧。如果热点路径已被优化以重用分配，你经常会在 Go 内存 profile 中看到池化对象。这减少了新的分配，堆 profile 可以帮助你了解池是否被正确使用！

## CPU 分析

CPU profile 帮助你了解 Go 程序在处理器上花费大部分调度时间的位置。

然而，这些没有纯文本形式，因此在下一节中，我们将使用 `go tool pprof` 命令来帮助我们读取它们。

要下载 CPU profile，请向 `/debug/pprof/profile?seconds=N` 发出请求，其中 N 是你希望收集 profile 的秒数。在 CPU profile 收集期间，程序性能可能会受到轻微影响。（其他 profile 几乎没有性能影响。）

完成后，它应该下载一个二进制文件，恰当地命名为 `profile`。然后我们需要检查它。

## `go tool pprof`

我们将以 CPU profile 为例使用 Go 内置的 profile 分析器，但你可以将其用于任何类型的 profile。

运行此命令（如果文件路径不同，则将 "profile" 替换为实际路径），它将打开一个交互式提示符：

<pre><code class="cmd bash">go tool pprof profile
File: caddy_master
Type: cpu
Time: Aug 29, 2022 at 8:47pm (MDT)
Duration: 30.02s, Total samples = 70.11s (233.55%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) </code></pre>

<aside class="tip">

你可以使用此命令检查任何类型的 profile，而不仅仅是 CPU profile。其他 profile 的原理相同，概念也可以沿用。

</aside>

这是你可以探索的内容。输入 `help` 会给出命令列表，`o` 会显示当前选项。如果你输入 `help <command>`，可以获取有关特定命令的信息。

有很多命令，但一些常见的有：

- `top`: 显示 CPU 使用最多的内容。你可以附加一个数字，例如 `top 20` 以查看更多内容，或者使用正则表达式来“聚焦”或忽略某些项。
- `web`: 在 Web 浏览器中打开调用图。这是可视化查看 CPU 使用情况的好方法。
- `svg`: 生成调用图的 SVG 图像。它与 `web` 相同，只是它不会打开 Web 浏览器，并且 SVG 会保存在本地。
- `tree`: 调用堆栈的表格视图。

让我们从 `top` 开始。我们看到的输出如下：

```
(pprof) top
Showing nodes accounting for 38.36s, 54.71% of 70.11s total
Dropped 785 nodes (cum <= 0.35s)
Showing top 10 nodes out of 196
      flat  flat%   sum%        cum   cum%
    10.97s 15.65% 15.65%     10.97s 15.65%  runtime/internal/syscall.Syscall6
     6.59s  9.40% 25.05%     36.65s 52.27%  runtime.gcDrain
     5.03s  7.17% 32.22%      5.34s  7.62%  runtime.(*lfstack).pop (inline)
     3.69s  5.26% 37.48%     11.02s 15.72%  runtime.scanobject
     2.42s  3.45% 40.94%      2.42s  3.45%  runtime.(*lfstack).push
     2.26s  3.22% 44.16%      2.30s  3.28%  runtime.pageIndexOf (inline)
     2.11s  3.01% 47.17%      2.56s  3.65%  runtime.findObject
     2.03s  2.90% 50.06%      2.03s  2.90%  runtime.markBits.isMarked (inline)
     1.69s  2.41% 52.47%      1.69s  2.41%  runtime.memclrNoHeapPointers
     1.57s  2.24% 54.71%      1.57s  2.24%  runtime.epollwait
```

CPU 的前 10 大消费者都在 Go 运行时中——特别是大量的垃圾回收（请记住系统调用用于释放和分配内存）。这暗示我们可以通过减少分配来提高性能，并且堆 profile 将是有价值的。

好的，但如果我们想查看我们自己代码的 CPU 利用率呢？我们可以像这样忽略包含 "runtime" 的模式：

```
(pprof) top -runtime  
Active filters:
   ignore=runtime
Showing nodes accounting for 0.92s, 1.31% of 70.11s total
Dropped 160 nodes (cum <= 0.35s)
Showing top 10 nodes out of 243
      flat  flat%   sum%        cum   cum%
     0.17s  0.24%  0.24%      0.28s   0.4%  sync.(*Pool).getSlow
     0.11s  0.16%   0.4%      0.11s  0.16%  github.com/prometheus/client_golang/prometheus.(*histogram).observe (inline)
     0.10s  0.14%  0.54%      0.23s  0.33%  github.com/prometheus/client_golang/prometheus.(*MetricVec).hashLabels
     0.10s  0.14%  0.68%      0.12s  0.17%  net/textproto.CanonicalMIMEHeaderKey
     0.10s  0.14%  0.83%      0.10s  0.14%  sync.(*poolChain).popTail
     0.08s  0.11%  0.94%      0.26s  0.37%  github.com/prometheus/client_golang/prometheus.(*histogram).Observe
     0.07s   0.1%  1.04%      0.07s   0.1%  internal/poll.(*fdMutex).rwlock
     0.07s   0.1%  1.14%      0.10s  0.14%  path/filepath.Clean
     0.06s 0.086%  1.23%      0.06s 0.086%  context.value
     0.06s 0.086%  1.31%      0.06s 0.086%  go.uber.org/zap/buffer.(*Buffer).AppendByte
```

好吧，很明显 Prometheus 指标是另一个主要消费者，但你会注意到，累积起来，它们的数量级远低于上面的 GC。这种显著差异表明我们应该专注于减少 GC。

<aside class="tip">

需要注意的是，CPU profile 是通过间歇性采样获得测量值的，并且采样频率永远不会超过采样率，默认情况下采样率为 10 毫秒。这就是为什么你不会看到任何小于 10 毫秒的累积持续时间（它们可能更小，但被向上取整了）。对于更具体的计时，你可以执行执行跟踪，它不使用采样。（TODO：添加关于跟踪的章节。）

</aside>

让我们使用 `q` 退出此 profile，并使用相同的命令分析堆 profile：

```
(pprof) top
Showing nodes accounting for 22259.07kB, 81.30% of 27380.04kB total
Showing top 10 nodes out of 102
      flat  flat%   sum%        cum   cum%
   12300kB 44.92% 44.92%    12300kB 44.92%  runtime.allocm
 2570.01kB  9.39% 54.31%  2570.01kB  9.39%  bufio.NewReaderSize
 2048.81kB  7.48% 61.79%  2048.81kB  7.48%  runtime.malg
 1542.01kB  5.63% 67.42%  1542.01kB  5.63%  bufio.NewWriterSize
 ...
 ```

找到了。几乎一半的内存分配严格用于我们使用 bufio 包的读写缓冲区。因此，我们可以推断，优化代码以减少缓冲将非常有益。（[Caddy 中的相关补丁](https://github.com/caddyserver/caddy/pull/4978) 正是这样做的。）

### 可视化

如果我们改为运行 `svg` 或 `web` 命令，我们将获得 profile 的可视化：

![CPU profile 可视化](/old/resources/images/profile.png)

这是 CPU profile，但其他 profile 类型也可用类似的图。

要了解如何阅读这些图，请阅读 [pprof 文档](https://github.com/google/pprof/blob/main/doc/README.md#interpreting-the-callgraph)。


### profile 差异分析

在进行代码更改后，你可以使用差异分析（“diff”）比较前后。这是堆的差异：

<pre><code class="cmd bash">go tool pprof -diff_base=before.prof after.prof
File: caddy
Type: inuse_space
Time: Aug 29, 2022 at 1:21am (MDT)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for -26.97MB, 49.32% of 54.68MB total
Dropped 10 nodes (cum <= 0.27MB)
Showing top 10 nodes out of 137
      flat  flat%   sum%        cum   cum%
  -27.04MB 49.45% 49.45%   -27.04MB 49.45%  bufio.NewWriterSize
      -2MB  3.66% 53.11%       -2MB  3.66%  runtime.allocm
    1.06MB  1.93% 51.18%     1.06MB  1.93%  github.com/yuin/goldmark/util.init
    1.03MB  1.89% 49.29%     1.03MB  1.89%  github.com/caddyserver/caddy/v2/modules/caddyhttp/reverseproxy.glob..func2
       1MB  1.84% 47.46%        1MB  1.84%  bufio.NewReaderSize
      -1MB  1.83% 49.29%       -1MB  1.83%  runtime.malg
       1MB  1.83% 47.46%        1MB  1.83%  github.com/caddyserver/caddy/v2/modules/caddyhttp/reverseproxy.cloneRequest
      -1MB  1.83% 49.29%       -1MB  1.83%  net/http.(*Server).newConn
   -0.55MB  1.00% 50.29%    -0.55MB  1.00%  html.populateMaps
    0.53MB  0.97% 49.32%     0.53MB  0.97%  github.com/alecthomas/chroma.TypeRemappingLexer</code></pre>

如你所见，我们减少了大约一半的内存分配！

差异也可以可视化：

![CPU profile 可视化](/old/resources/images/profile-diff.png)

这使得更改如何影响程序某些部分的性能变得非常明显。

## 进一步阅读

关于程序分析有很多需要掌握的内容，我们只是触及了表面。

要真正掌握“分析”，请考虑以下资源：

- [pprof 文档](https://github.com/google/pprof/blob/main/doc/README.md)
- [Caddy 中 profile 的实际使用示例](https://github.com/caddyserver/caddy/pull/4978)
- [Go 维基上的性能](https://github.com/golang/go/wiki/Performance)
- [`net/http/pprof` 包](https://pkg.go.dev/net/http/pprof)