---
title: fs（Caddyfile 指令）
---

# fs

设置用于执行文件 I/O 的文件系统。

这可以让你连接到云中的远程文件系统、具有类文件接口的数据库，甚至读取嵌入在 Caddy 二进制文件中的文件。

首先，你必须使用 [`filesystem` 全局选项](/docs/caddyfile/options#filesystem) 声明一个文件系统名称，然后使用此指令指定要使用的文件系统。

此指令通常与 [`file_server` 指令](file_server) 配合使用以提供静态文件，或与 [`try_files` 指令](try_files) 配合使用以基于文件是否存在执行重写。通常还与 [`root` 指令](root) 一起使用以设置文件系统中的根路径。

## 语法

```caddy-d
fs [<匹配器>] <文件系统>
```

## 示例

使用名为 `foo` 的文件系统，该文件系统使用一个虚构的名为 `custom` 的模块（可能需要身份验证）：

```caddy
{
	filesystem foo custom {
		api_key abc123
	}
}

example.com {
	fs foo
	root /srv
	file_server
}
```

仅从 `foo` 文件系统提供图片，其余文件从默认文件系统提供：

```caddy
example.com {
	fs /images* foo
	root /srv
	file_server
}