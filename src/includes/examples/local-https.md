<div class="ex-local-https">

```caddy
localhost {
	respond "来自 HTTPS 的问候！"
}

192.168.1.10 {
	respond "同样也是 HTTPS！"
}

http://localhost {
	respond "普通 HTTP"
}
```

</div>

<script>
window.$_('.ex-local-https code').classList.add('light');
</script>