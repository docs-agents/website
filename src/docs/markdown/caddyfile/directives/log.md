---
title: log (Caddyfile 指令)
---

<script>
ready(function() {
	// 修正代码块中的 >
	$$_('pre.chroma .k').forEach(item => {
		if (item.innerText.includes('>')) {
			// 如果以 > 结尾则跳过
			if (item.textContent.trim().endsWith('>')) return;
			// 将 > 替换为 <span class="p">&gt;</span>
			item.innerHTML = item.innerHTML.replace(/&gt;/g, '<span class="p">&gt;</span>');
		}
	});

	// 如果页面上存在匹配的锚点标签，则为所有子指令添加链接
	addLinksToSubdirectives();
});
</script>

# log

启用并配置 HTTP 请求日志记录（也称为访问日志）。

<aside class="tip">

要配置 Caddy 的运行日志，请参阅 [`log` 全局选项](/docs/caddyfile/options#log)。

</aside>

`log` 指令适用于其所在的站点块的主机名，除非使用 `hostnames` 子指令覆盖。

配置后，默认情况下，对该站点的所有请求都会被记录。要按条件跳过某些请求的记录，请使用 [`log_skip` 指令](log_skip)。

要向日志条目添加自定义字段，请使用 [`log_append` 指令](log_append)。

- [语法](#syntax)
- [输出模块](#output-modules)
  - [stderr](#stderr)
  - [stdout](#stdout)
  - [discard](#discard)
  - [file](#file)
  - [net](#net)
- [格式模块](#format-modules)
  - [console](#console)
  - [json](#json)
  - [filter](#filter)
    - [delete](#delete)
    - [rename](#rename)
    - [replace](#replace)
    - [ip_mask](#ip-mask)
    - [query](#query)
    - [cookie](#cookie)
    - [regexp](#regexp)
    - [hash](#hash)
  - [append](#append)
- [示例](#examples)

默认情况下，包含潜在敏感信息的请求头（`Cookie`、`Set-Cookie`、`Authorization` 和 `Proxy-Authorization`）在访问日志中会被记录为 `REDACTED`。此行为可以通过全局服务器选项 [`log_credentials`](/docs/caddyfile/options#log-credentials) 禁用。

## 语法

```caddy-d
log [<logger_name>] {
	hostnames <hostnames...>
	no_hostname
	output <writer_module> ...
	format <encoder_module> ...
	level  <level>
	sampling {
		interval   <duration>
		first      <number>
		thereafter <number>
	}
}
```

- **logger_name** <span id="logger_name"/> 是该站点日志器名称的可选覆盖。

  默认情况下，日志器名称会自动生成，例如 `log0`、`log1` 等，具体取决于站点在 Caddyfile 中的顺序。只有当你想可靠地从全局选项中定义的其他日志器引用该日志器的输出时，此参数才有用。参见下面的[多输出示例](#multiple-outputs)。

- **hostnames** <span id="hostnames"/> 是该日志器适用的主机名的可选覆盖。

  默认情况下，日志器适用于它所在的站点块的主机名，即站点地址。这在你想为[通配符站点块](/docs/caddyfile/patterns#wildcard-certificates)中的每个子域名定义不同的日志器时很有用。参见下面的[通配符日志示例](#wildcard-logs)。

- **no_hostname** <span id="no_hostname"/> 阻止日志器与站点块的任何主机名关联。默认情况下，日志器与 `log` 指令所在的[站点地址](/docs/caddyfile/concepts#addresses)关联。

  当你希望根据某些条件（如请求路径或方法）将请求记录到不同文件时，结合 [`log_name` 指令](/docs/caddyfile/directives/log_name) 使用此选项非常有用。

- **output** <span id="output"/> 配置日志的写入位置。参见下面的 [`output` 模块](#output-modules)。

  默认值：`stderr`。

- **format** <span id="format"/> 描述如何编码或格式化日志。参见下面的 [`format` 模块](#format-modules)。

  默认值：如果检测到 `stderr` 是终端，则为 `console`，否则为 `json`。

- **level** <span id="level"/> 是记录日志的最小条目级别。默认值：`INFO`。

  请注意，访问日志目前仅发出 `INFO` 和 `ERROR` 级别的日志。

- **sampling** <span id="sampling"/> 配置日志采样以减少日志量。如果指定了采样，则启用采样，并使用下面的默认值。省略此选项则禁用采样。

  - **interval** 是进行采样的[时间窗口](/docs/conventions#durations)。默认值：`1s`（禁用）。

  - **first** 是在每个时间窗口内，对给定级别和消息保留的日志数量。默认值：`100`。

  - **thereafter** 是在每个时间窗口内，跳过已经保留的日志之后的日志数量。默认值：`100`。

  例如，使用 `interval 1s`、`first 5` 和 `thereafter 10`，在每 10 秒的窗口中，前 5 条日志条目会被保留，然后在该秒内，每 10 条具有相同级别和消息的日志条目才会通过一次。

### 输出模块

**output** 子指令允许你定制日志的写入位置。

#### stderr

标准错误（控制台，默认）。

```caddy-d
output stderr
```

#### stdout

标准输出（控制台）。

```caddy-d
output stdout
```

#### discard

无输出。

```caddy-d
output discard
```

#### file

文件。默认情况下，日志文件会根据大小进行轮转以防止磁盘空间耗尽。

日志轮转由 [timberjack <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/DeRuina/timberjack) 提供。

<aside class="tip">

**关于重新加载日志文件选项的说明：** 要对给定的输出文件应用配置更改，需要重启服务器。更改不会在服务器重载时生效，除非你添加一个新的日志文件名。

</aside>

```caddy-d
output file <filename> {
	mode          <mode>
	roll_disabled
	roll_size     <size>
	roll_interval <duration>
	roll_minutes  <minutes...>
	roll_at	      <times...>
	roll_uncompressed
	roll_local_time
	roll_keep     <num>
	roll_keep_for <days>
	backup_time_format <format>
}
```

- **&lt;filename&gt;** 是日志文件的路径。

  轮转时，文件会按照模板 `<name>-<timestamp>-<reason>.log` 重命名。时间戳根据 [`backup_time_format`](#backup_time_format) 选项格式化。`reason` 为 `size` 或 `time`，具体取决于触发轮转的条件。如果文件被压缩，文件名会附加 `.gz`。

  例如，如果文件名是 `access.log`，则轮转后的文件名可能是 `access-2026-01-30T22-15-42.123-size.log`（因大小触发轮转）或 `access-2025-01-30T00-00-00.000-time.log`（因时间触发轮转）。

- **mode** <span id="mode"/> 是日志文件的 Unix 文件模式/权限。模式由 1 到 4 位八进制数字组成（与 Unix [chmod <img src="/old/resources/images/external-link.svg" class="external-link">](https://en.wikipedia.org/wiki/Chmod) 命令接受的数字格式相同，但全零模式将被解释为默认模式 `600`）。

  例如：`0600` 表示模式为 `rw-,---,---`（日志文件所有者具有读写权限，其他人无权限）；`0640` 表示模式为 `rw-,r--,---`（文件所有者读写，组内成员只读）；`644` 表示模式为 `rw-,r--,r--`（文件所有者读写，组内成员和其他用户只读）。

- **roll_disabled** <span id="roll_disabled"/> 禁用日志轮转。这可能导致磁盘空间耗尽，因此仅当日志文件通过其他方式维护时才使用此选项。

- **roll_size** <span id="roll_size"/> 是触发日志文件轮转的大小。当前实现支持兆字节分辨率；小数值向上取整到下一个整兆字节。例如，`1.1MiB` 会向上取整为 `2MiB`。

  此选项始终启用。如果写入日志导致文件超过指定大小，日志会立即轮转。备份文件名中会包含 `size` 作为原因。

  默认值：`100MiB`

- **roll_interval** <span id="roll_interval"/> 是日志轮转的最大时间间隔。该值是一个[持续时间字符串](/docs/conventions#durations)，指定后，在自上次轮转以来经过该持续时间后，下次写入日志时会触发轮转。备份文件名中会包含 `time` 作为原因。

  请注意，如果设置为 `24h`，则不一定在午夜轮转，而是在自上次轮转后的 24 小时标记处轮转。如果因大小触发轮转，则下次轮转的时间会相对于上次轮转偏移。你可以使用 `roll_at` 或 `roll_minutes` 选项在特定时间轮转。

  默认值：禁用

- **roll_minutes** <span id="roll_minutes"/> 是一个分钟值列表（0-59），在这些分钟时刻轮转日志文件。例如，`10 40` 会在每小时的 `xx:10` 和 `xx:40` 每 30 分钟轮转一次日志文件。轮转与时钟分钟对齐（第 0 秒）。

  启用此选项会启动一个 goroutine 计时器，在指定的分钟值触发日志轮转（即引入少量后台处理）。此选项与 `roll_interval` 和 `roll_size` 协同工作。备份文件名中会包含 `time` 作为原因。

  默认值：禁用

- **roll_at** <span id="roll_at"/> 是一个时间值列表（24 小时制），在这些时间点轮转日志文件。例如，`00:00 12:00` 会在每天午夜和中午轮转两次日志文件。轮转与时钟分钟对齐（第 0 秒）。

  启用此选项会启动一个 goroutine 计时器，在指定的时间触发日志轮转（即引入少量后台处理）。此选项与 `roll_interval` 和 `roll_size` 协同工作。备份文件名中会包含 `time` 作为原因。

  默认值：禁用

- **roll_uncompressed** <span id="roll_uncompressed"/> 关闭 gzip 日志压缩。

  默认值：启用 `gzip` 压缩。

- **roll_local_time** <span id="roll_local_time"/> 设置轮转使用本地时间戳作为文件名。 
  默认值：使用 UTC 时间。

- **roll_keep** <span id="roll_keep"/> 是在删除最旧的日志文件之前保留的日志文件数量。在创建新日志文件时触发。

  默认值：`10`

- **roll_keep_for** <span id="roll_keep_for"/> 是保留轮转文件的时长，表示为[持续时间字符串](/docs/conventions#durations)。在创建新日志文件时触发。当前实现支持天分辨率；小数值向上取整到下一个整天。例如，`36h`（1.5 天）会向上取整为 `48h`（2 天）。
  
  默认值：`2160h`（90 天）

- **backup_time_format** <span id="backup_time_format"/> 是备份文件名中使用的时间格式。必须是有效的时间布局字符串；有关完整详情，请参见 [Go 文档](https://pkg.go.dev/time#pkg-constants)。

  默认值：`2006-01-02T15-04-05`

#### net

网络套接字。如果套接字断开，将在尝试重新连接时将日志转储到 stderr。

```caddy-d
output net <address> {
	dial_timeout <duration>
	soft_start
}
```

- **&lt;address&gt;** 是写入日志的[地址](/docs/conventions#network-addresses)。

- **dial_timeout** <span id="dial_timeout"/> 是等待与日志套接字成功连接的最长时间。如果套接字断开，日志发送可能会被阻塞长达此时间。

- **soft_start** <span id="soft_start"/> 将忽略连接套接字时的错误，允许即使在远程日志服务宕机时也能加载配置。日志将改为发送到 stderr。

### 格式模块

**format** 子指令允许你定制日志的编码方式（格式化）。它出现在 `log` 块内部。

<aside class="tip">

**关于通用日志格式 (CLF) 的说明：** CLF 与现代结构化日志不兼容。要将访问日志转换为已弃用的通用日志格式，请使用 [`transform-encoder` 插件 <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddyserver/transform-encoder)。

</aside>

除了每个编码器的特定语法外，这些通用属性可以在大多数编码器上设置：

```caddy-d
format <encoder_module> {
	message_key     <key>
	level_key       <key>
	time_key        <key>
	name_key        <key>
	caller_key      <key>
	stacktrace_key  <key>
	line_ending     <char>
	time_format     <format>
	time_local
	duration_format <format>
	level_format    <format>
}
```

- **message_key** <span id="message_key"/> 日志条目中消息字段的键。默认值：`msg`

- **level_key** <span id="level_key"/> 日志条目中级别字段的键。默认值：`level`

- **time_key** <span id="time_key"/> 日志条目中时间字段的键。默认值：`ts`
- **name_key** <span id="name_key"/> 日志条目中名称字段的键。默认值：`name`

- **caller_key** <span id="caller_key"/> 日志条目中调用者字段的键。

- **stacktrace_key** <span id="stacktrace_key"/> 日志条目中堆栈跟踪字段的键。

- **line_ending** <span id="line_ending"/> 使用的行结束符。

- **time_format** <span id="time_format"/> 时间戳的格式。
  默认值：如果格式默认为 `console`，则为 `wall_milli`；否则为 `unix_seconds_float`。
  
  可以是以下之一：
  - `unix_seconds_float` 自 Unix 纪元以来的浮点秒数。
  - `unix_milli_float` 自 Unix 纪元以来的浮点毫秒数。
  - `unix_nano` 自 Unix 纪元以来的整型纳秒数。
  - `iso8601` 示例：`2006-01-02T15:04:05.000Z0700`
  - `rfc3339` 示例：`2006-01-02T15:04:05Z07:00`
  - `rfc3339_nano` 示例：`2006-01-02T15:04:05.999999999Z07:00`
  - `wall` 示例：`2006/01/02 15:04:05`
  - `wall_milli` 示例：`2006/01/02 15:04:05.000`
  - `wall_nano` 示例：`2006/01/02 15:04:05.000000000`
  - `common_log` 示例：`02/Jan/2006:15:04:05 -0700`
  - 或者，任何兼容的时间布局字符串；有关完整详情，请参见 [Go 文档](https://pkg.go.dev/time#pkg-constants)。
  
  请注意，格式字符串中的部分是布局的特殊常量；因此 `2006` 表示年份，`01` 表示月份，`Jan` 表示月份的字符串形式，`02` 表示日期。不要在格式字符串中使用实际的当前日期数字。

- **time_local** <span id="time_local"/> 使用本地系统时间记录日志，而不是默认的 UTC 时间。

- **duration_format** <span id="duration_format"/> 持续时间的格式。

  默认值：`seconds`。
  
  可以是以下之一：
  - `s`, `second` 或 `seconds` 经过的浮点秒数。
  - `ms`, `milli` 或 `millis` 经过的浮点毫秒数。
  - `ns`, `nano` 或 `nanos` 经过的整型纳秒数。
  - `string` 使用 Go 内置的字符串格式，例如 `1m32.05s` 或 `6.31ms`。

- **level_format** <span id="level_format"/> 级别的格式。

  默认值：如果格式默认为 `console`，则为 `color`；否则为 `lower`。
  
  可以是以下之一：
  - `lower` 小写。
  - `upper` 大写。
  - `color` 大写，带 ANSI 颜色。

#### console

控制台编码器以人类可读性格式化日志条目，同时保留一些结构。

```caddy-d
format console
```

#### json

将每个日志条目格式化为 JSON 对象。

```caddy-d
format json
```

#### filter

允许逐字段过滤。

```caddy-d
format filter {
	fields {
		<field> <filter> ...
	}
	<field> <filter> ...
	wrap <encode_module> ...
}
```

嵌套字段可以通过使用 `>` 表示一层嵌套来引用。换句话说，对于像 `{"a":{"b":0}}` 这样的对象，内部字段可以引用为 `a>b`。

以下字段是日志的基础字段，由于底层日志库将其作为特殊情况添加，因此不能过滤：`ts`、`level`、`logger` 和 `msg`。

`wrap` 是可选的；如果省略，会根据当前输出模块是 [`stderr`](#stderr) 还是 [`stdout`](#stdout) 以及是否为交互式终端自动选择默认编码器：如果是终端则选择 [`console`](#console)，否则选择 [`json`](#json)。

作为一种快捷方式，可以省略 `fields` 块，直接在 `filter` 块内指定过滤器。

以下是可用的过滤器：

##### delete

标记一个字段在编码时被跳过。

```caddy-d
<field> delete
```

##### rename

重命名日志字段的键。

```caddy-d
<field> rename <key>
```

##### replace

标记一个字段在编码时被提供的字符串替换。

```caddy-d
<field> replace <replacement>
```

##### ip_mask

使用 CIDR 掩码屏蔽字段中的 IP 地址，即保留 IP 的位数（从左侧开始）。如果字段是字符串数组（例如 HTTP 头），则数组中的每个值都会被屏蔽。该值可以是一个逗号分隔的 IP 地址字符串。

IPv4 和 IPv6 地址有单独的配置，因为它们的总位数不同。

最常见的需要过滤的字段是：
- `request>remote_ip`（直接连接的客户端）
- `request>client_ip`（配置了 [`trusted_proxies`](/docs/caddyfile/options#trusted-proxies) 时解析的“真实客户端”）
- `request>headers>X-Forwarded-For`（如果位于反向代理之后）

```caddy-d
<field> ip_mask [<ipv4> [<ipv6>]] {
	ipv4 <cidr>
	ipv6 <cidr>
}
```

##### query

标记一个字段，对其执行一个或多个操作，以操控 URL 字段的查询部分。最常见的字段是 `request>uri`。

```caddy-d
<field> query {
	delete  <key>
	replace <key> <replacement>
	hash    <key>
}
```

可用的操作有：

- **delete** 从查询中删除给定的键。

- **replace** 将给定查询键的值替换为 **replacement**。用于插入占位符；你可以看到查询键存在于 URL 中，但值被隐藏。

- **hash** 将给定查询键的值替换为其 SHA-256 哈希的前 4 个字节（小写十六进制）。用于在敏感时隐藏值，同时能够注意到每个请求是否有不同的值。

##### cookie

标记一个字段，对其执行一个或多个操作，以操控 `Cookie` HTTP 头的值。最常见的字段是 `request>headers>Cookie`。

```caddy-d
<field> cookie {
	delete  <name>
	replace <name> <replacement>
	hash    <name>
}
```

可用的操作有：

- **delete** 按名称从请求头中删除给定的 cookie。

- **replace** 将给定 cookie 的值替换为 **replacement**。用于插入占位符；你可以看到 cookie 存在于请求头中，但值被隐藏。

- **hash** 将给定 cookie 的值替换为其 SHA-256 哈希的前 4 个字节（小写十六进制）。用于在敏感时隐藏值，同时能够注意到每个请求是否有不同的值。

如果为同一个 cookie 名称定义了多个操作，则仅应用第一个操作。

##### regexp

标记一个字段，在编码时应用正则表达式替换。如果字段是字符串数组（例如 HTTP 头），则对数组中的每个值应用替换。

```caddy-d
<field> regexp <pattern> <replacement>
```

使用的正则表达式语言是 Go 中的 RE2。请参见 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax) 和 [Go regexp 语法概述](https://pkg.go.dev/regexp/syntax)。

在替换字符串中，捕获组可以用 `${group}` 引用，其中 `group` 是捕获组的名称或编号。捕获组 `0` 是整个正则表达式匹配，`1` 是第一个捕获组，`2` 是第二个捕获组，以此类推。

##### hash

标记一个字段，在编码时替换为其值的 SHA-256 哈希的前 4 个字节（8 个十六进制字符）。如果字段是字符串数组（例如 HTTP 头），则对数组中的每个值进行哈希。

用于在敏感时隐藏值，同时能够注意到每个请求是否有不同的值。

```caddy-d
<field> hash
```

#### append

向所有日志条目追加字段。

```caddy-d
format append {
	fields {
		<field> <value>
	}
	<field> <value>
	wrap <encode_module> ...
}
```

它最适用于添加有关生成日志条目的 Caddy 实例的信息，可能通过环境变量实现。字段值可以是全局占位符（例如 `{env.*}`），但**不能**是逐请求占位符，因为日志是在 HTTP 请求上下文之外写入的。

`wrap` 是可选的；如果省略，会根据当前输出模块是 [`stderr`](#stderr) 还是 [`stdout`](#stdout) 以及是否为交互式终端自动选择默认编码器：如果是终端则选择 [`console`](#console)，否则选择 [`json`](#json)。

可以省略 `fields` 块，直接在 `append` 块内指定字段。

## 示例

为默认日志器启用访问日志记录。

换句话说，默认情况下这会将日志记录到 `stderr`，但可以通过使用 [`log` 全局选项](/docs/caddyfile/options#log) 重新配置 `default` 日志器来更改：

```caddy
example.com {
	log
}
```

将日志写入文件（默认启用日志轮转）：

```caddy
example.com {
	log {
		output file /var/log/access.log
	}
}
```

自定义日志轮转，每天午夜或日志文件达到 1 GB 时轮转（以先到者为准），并保留 5 个轮转文件或 30 天的日志：

```caddy
example.com {
	log {
		output file /var/log/access.log {
			roll_at 00:00
			roll_size 1gb
			roll_keep 5
			roll_keep_for 720h
		}
	}
}
```

从日志中删除 `User-Agent` 请求头：

```caddy
example.com {
	log {
		format filter {
			request>headers>User-Agent delete
		}
	}
}
```

脱敏多个敏感 cookie。（注意，默认情况下某些敏感请求头会记录为空值；参见 [`log_credentials` 全局选项](/docs/caddyfile/options#log-credentials) 以启用记录 `Cookie` 头值）：

```caddy
example.com {
	log {
		format filter {
			request>headers>Cookie cookie {
				replace session REDACTED
				delete secret
			}
		}
	}
}
```

屏蔽请求中的远程地址，保留 IPv4 地址的前 16 位（即 255.255.0.0），以及 IPv6 地址的前 32 位。

请注意，自 Caddy v2.7 起，`remote_ip` 和 `client_ip` 都会被记录，其中 `client_ip` 是配置了 [`trusted_proxies`](/docs/caddyfile/options#trusted-proxies) 时的“真实 IP”：

```caddy
example.com {
	log {
		format filter {
			request>remote_ip ip_mask 16 32
			request>client_ip ip_mask 16 32
		}
	}
}
```

从环境变量追加服务器 ID 到所有日志条目，并与 `filter` 链接以删除一个请求头：

```caddy
example.com {
	log {
		format append {
			server_id {env.SERVER_ID}
			wrap filter {
				request>headers>Cookie delete
			}
		}
	}
}
```

<span id="wildcard-logs" /> 通过为每个日志器覆盖 `hostnames`，为[通配符站点块](/docs/caddyfile/patterns#wildcard-certificates)中的每个子域名写入单独的日志文件。此处使用[代码片段](/docs/caddyfile/concepts#snippets)避免重复：

```caddy
(subdomain-log) {
	log {
		hostnames {args[0]}
		output file /var/log/{args[0]}.log
	}
}

*.example.com {
	import subdomain-log foo.example.com
	@foo host foo.example.com
	handle @foo {
		respond "foo"
	}

	import subdomain-log bar.example.com
	@bar host bar.example.com
	handle @bar {
		respond "bar"
	}
}
```

<span id="multiple-outputs" /> 将特定子域名的访问日志写入两个不同的文件，并使用不同的格式（一个使用 [`transform-encoder` 插件 <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddyserver/transform-encoder)，另一个使用 [`json`](#json)）。

这是通过在站点块中将日志器名称覆盖为 `foo`，然后在全局选项中的两个日志器中通过 `include http.log.access.foo` 包含该日志器产生的访问日志来实现的：

```caddy
{
	log access-formatted {
		include http.log.access.foo
		output file /var/log/access-foo.log
		format transform "{common_log}"
	}

	log access-json {
		include http.log.access.foo
		output file /var/log/access-foo.json
		format json
	}
}

foo.example.com {
	log foo
}
```

<span id="sampling-example" /> 通过采样减少日志量，例如每秒保留前 5 个请求，然后每 10 个请求中记录 1 个：

```caddy
example.com {
	log {
		sampling {
			interval   1s
			first      5
			thereafter 10
		}
	}
}