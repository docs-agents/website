<div id="module-list-container">
	<h1>所有模块</h1>
	<p>
		本页面列出了所有已注册的 Caddy 模块。模块是扩展 Caddy <a href="/docs/json">JSON 配置结构</a>的插件。
	</p>
	<p>
		建议使用浏览器的“在页面中查找”功能进行快速查找。
	</p>
	<table id="module-list">
		<tr>
			<th></th>
			<th>模块 ID</th>
			<th>描述</th>
		</tr>
		<!--Populated by JS-->
	</table>
</div>

<div id="module-docs-container">
	<div class="pad"><h1 class="module-name"><!--Populated by JS--></h1></div>
	<div id="module-multiple-repos">
		存在多个名为 <b class="module-name"><!--Populated by JS--></b> 的模块。请根据其仓库选择一个。
	</div>
	<div id="module-template" class="module-repo-container">
		<div class="module-repo-selector"></div>
		<article>
			{{include "/includes/docs/renderbox.html"}}
			{{include "/includes/docs/details.html"}}
		</article>
	</div>
</div>

{{include "/includes/docs/hovercard.html"}}