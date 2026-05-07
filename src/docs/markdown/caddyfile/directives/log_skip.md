---
title: log_skip (Caddyfile 指令)
---

# log_skip

跳过匹配请求的访问日志记录。

此指令应与 [`log` 指令](log) 配合使用，以跳过记录那些您不需要关注的请求。

在 v2.8.0 之前，此指令名为 `skip_log`，但为了与其他指令保持一致而进行了重命名。


## 语法

```caddy-d
log_skip [<匹配器>]
```


## 示例

跳过子路径中静态文件的访问日志记录：

```caddy
example.com {
	root /srv

	log
	log_skip /static*

	file_server
}
```


跳过匹配特定模式的请求的访问日志记录；本例中为具有特定扩展名的文件：

```caddy-d
@skip path_regexp \.(js|css|png|jpe?g|gif|ico|woff|otf|ttf|eot|svg|txt|pdf|docx?|xlsx?)$
log_skip @skip
```


如果指令位于已包含匹配器的路由块内，则无需额外指定匹配器。例如，为特定子路径的文件服务器处理的示例如下：

```caddy-d
handle_path /static* {
	root /srv/static
	log_skip
	file_server
}