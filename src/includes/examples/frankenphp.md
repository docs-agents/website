<div class="ex-frankenphp">

```caddy
{
	# 启用 FrankenPHP
	frankenphp
}

example.com {
	# 从当前目录提供 PHP 应用
	php_server
}
```

</div>

<script>
window.$_('.ex-frankenphp code').classList.add('dark');
</script>