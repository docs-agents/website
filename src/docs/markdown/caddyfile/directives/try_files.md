---
title: try_files (Caddyfile 指令)
---

# try_files

将请求的 URI 路径重写为站点根目录下第一个存在的文件。如果没有文件匹配，则不执行重写。


## 语法

```caddy-d
try_files <files...> {
	policy first_exist|first_exist_fallback|smallest_size|largest_size|most_recently_modified
}
```

- **<files...>** 是要尝试的文件列表。URI 路径将被重写为第一个存在的文件。

  如需匹配目录，请在路径末尾添加斜杠 `/`。所有文件路径均相对于站点[根目录](root)，并且支持[glob 模式](https://pkg.go.dev/path/filepath#Match)展开。

  每个参数也可以包含查询字符串，此时如果匹配到该特定文件，查询字符串也将随之更改。

  如果 `try_policy` 为 `first_exist`（默认值），则列表中的最后一项可以是带有 `=` 前缀的数字（例如 `=404`），作为后备措施，将触发对应错误码的错误；该错误可通过 [`handle_errors`](handle_errors) 捕获并处理。

- **policy** 是从文件列表中选择文件的策略。

  默认值：`first_exist`



## 展开形式

`try_files` 指令本质上是以下内容的简写：

```caddy-d
@try_files file <files...>
rewrite @try_files {file_match.relative}
```

请注意，此指令不接受匹配器标记。如果您需要更复杂的匹配逻辑，请以上述展开形式为基础。

更多详情请参阅 [`file` 匹配器](/docs/caddyfile/matchers#file)。



## 示例

如果请求未匹配任何静态文件，则重写至您的 PHP 索引/路由入口点：

```caddy-d
try_files {path} /index.php
```

同样，但将原始路径添加到查询字符串（某些旧版 PHP 应用需要）：

```caddy-d
try_files {path} /index.php?{query}&p={path}
```

同样，但同时也匹配目录：

```caddy-d
try_files {path} {path}/ /index.php?{query}&p={path}
```

尝试重写为存在的文件或目录，否则抛出 404 错误（可通过 [`handle_errors`](handle_errors) 捕获并处理）：

```caddy-d
try_files {path} {path}/ =404
```

选择最新部署的静态文件版本（例如，当请求 `index.html` 时，提供 `index.be331df.html`）：

```caddy-d
try_files {file.base}.*.{file.ext} {
	policy most_recently_modified
}