---
title: tls (Caddyfile 指令)
---

<script>
ready(function() {
	// 如果页面中找到匹配的锚点标签，我们将为所有子指令添加链接。
	addLinksToSubdirectives();
});
</script>

# tls

配置站点的 TLS。

**Caddy 默认的 TLS 设置是安全的。除非你有充分理由并理解其影响，否则不要更改这些设置。** 此指令最常见的用途是指定 ACME 账户邮箱地址、更改 ACME CA 端点或提供自己的证书。

兼容性说明：由于 TLS 作为安全协议具有敏感性，在新的次要版本或补丁版本中可能会故意调整 TLS 默认值。过时或有缺陷的 TLS 版本、密码套件、功能等可能会随时移除。如果你的部署对变更极为敏感，应明确指定必须保持不变的数值，并警惕升级。在几乎所有情况下，我们建议使用默认设置。


## 语法

```caddy-d
tls [internal|force_automate|<email>] | [<cert_file> <key_file>] {
	protocols <min> [<max>]
	ciphers   <cipher_suites...>
	curves    <groups...>
	alpn      <values...>
	load      <paths...>
	ca        <ca_dir_url>
	ca_root   <pem_file>
	key_type  ed25519|p256|p384|rsa2048|rsa4096
	dns       <provider_name> [<params...>]
	propagation_timeout <duration>
	propagation_delay   <duration>
	dns_ttl             <duration>
	dns_challenge_override_domain <domain>
	resolvers <dns_servers...>
	eab       <key_id> <mac_key>
	on_demand
	reuse_private_keys
	client_auth {
		mode                   [request|require|verify_if_given|require_and_verify]
		trust_pool             <module>
		verifier 			   <module>
	}
	issuer          <issuer_name>  [<params...>]
	get_certificate <manager_name> [<params...>]
	insecure_secrets_log <log_file>
	renewal_window_ratio <ratio>
	force_automate
}
```

- **internal** 表示使用 Caddy 内部的、本地信任的 CA 为该站点生成证书。要进一步配置 [`internal`](#internal) 签发者，请使用 [`issuer`](#issuer) 子指令。

- **force_automate** 强制 Caddy 自动化管理该站点的证书，即使其他受管理的证书已适用。

- **&lt;email&gt;** 是用于管理站点证书的 ACME 账户的邮箱地址。你可能更倾向于使用 [`email` 全局选项](/docs/caddyfile/options#email) 一次性为所有站点配置此选项。

<aside class="tip">

请记住，Let's Encrypt 可能会向你发送关于证书即将到期的电子邮件，但这可能具有误导性，因为 Caddy 可能在续期时选择了不同的签发者（例如 ZeroSSL）。请检查你的日志和/或证书本身（例如在浏览器中）以查看使用了哪个签发者，以及其有效期是否仍然有效；如果是，你可以安全地忽略 Let's Encrypt 的邮件。

</aside>

- **&lt;cert_file&gt;** 和 **&lt;key_file&gt;** 是证书和私钥 PEM 文件的路径。仅指定其中一个无效。

- **protocols** <span id="protocols"/> 指定最低和最高协议版本。除非你清楚自己在做什么，否则不要更改这些设置。通常无需配置，因为 Caddy 总是使用现代默认值。
  
  默认最低版本：`tls1.2`，默认最高版本：`tls1.3`

- **ciphers** <span id="ciphers"/> 按优先顺序降序指定密码套件列表。除非你清楚自己在做什么，否则不要更改这些设置。注意：密码套件在 TLS 1.3 中不可自定义；并且并非所有 TLS 1.2 密码套件都默认启用。支持的名称（按 Go 标准库的偏好顺序）为：
	- `TLS_AES_128_GCM_SHA256`
	- `TLS_CHACHA20_POLY1305_SHA256`
	- `TLS_AES_256_GCM_SHA384`
	- `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`
	- `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`
	- `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384`
	- `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
	- `TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256`
	- `TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256`
	- `TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA`
	- `TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA`
	- `TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA`
	- `TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA`
	- `TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA`

- **curves** <span id="curves"/> 指定要支持的 EC 组列表。建议不要更改默认值。支持的值有：
	- `x25519mlkem768`（后量子密码）
	- `x25519`
	- `secp256r1`
	- `secp384r1`
	- `secp521r1`

- **alpn** <span id="alpn"/> 是在 TLS 握手的 [ALPN 扩展 <img src="/old/resources/images/external-link.svg" class="external-link">](https://developer.mozilla.org/en-US/docs/Glossary/ALPN) 中通告的值列表。

- **load** <span id="load"/> 指定要从中加载包含证书+密钥捆绑的 PEM 文件的文件夹列表。

- **ca** <span id="ca"/> 更改 ACME CA 端点。这最常用于测试时设置 [Let's Encrypt 的临时端点 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/staging-environment/)，或设置内部 ACME 服务器。（要为整个 Caddyfile 更改此值，请改用 `acme_ca` [全局选项](/docs/caddyfile/options)。）

- **ca_root** <span id="ca_root"/> 指定一个 PEM 文件，其中包含 ACME CA 端点的受信任根证书（如果系统信任存储中没有）。

- **key_type** <span id="key_type"/> 是生成 CSR 时要使用的密钥类型。仅在你有特定要求时才设置。

- **dns** <span id="dns"/> 使用指定的提供者插件启用 [DNS 挑战](/docs/automatic-https#dns-challenge)。该插件必须来自 [`caddy-dns` <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddy-dns) 仓库之一。每个提供者插件可能在其名称后有自己特定的语法；详情请参考其文档。维护每个 DNS 提供者的支持是社区努力的结果。[在我们的维基中了解如何为你的提供者启用 DNS 挑战。](https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148)

- **propagation_timeout** <span id="propagation_timeout"/> 是一个[时长值](/docs/conventions#durations)，设置使用 DNS 挑战时等待 DNS TXT 记录出现的最长时间。设置为 `-1` 以禁用传播检查。默认 2 分钟。

- **propagation_delay** <span id="propagation_delay"/> 是一个[时长值](/docs/conventions#durations)，设置在使用 DNS 挑战时，开始 DNS TXT 记录传播检查前等待的时间。默认 `0`（不等待）。

- **dns_ttl** <span id="dns_ttl"/> 是一个[时长值](/docs/conventions#durations)，设置用于 DNS 挑战的 `TXT` 记录的 TTL。很少需要设置。

- **dns_challenge_override_domain** <span id="dns_challenge_override_domain"/> 覆盖用于 DNS 挑战的域名。这是为了将挑战委托给另一个域名。

  如果你的主域名的 DNS 提供者没有可用的 [DNS 插件 <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddy-dns)，你可能希望使用此选项。你可以改为在你的主域名中添加一个子域为 `_acme-challenge` 的 `CNAME` 记录，指向你确实拥有插件的辅助域名。此选项 _不需要_ 插件提供特殊支持。
  
  当 ACME 签发者尝试为你主域名解决 DNS 挑战时，它们将跟随 `CNAME` 记录到你的辅助域名以查找 `TXT` 记录。

  **注意：** 在此处使用 CNAME 记录中的完整规范名称 - `_acme-challenge` 子域不会自动附加。

- **resolvers** <span id="resolvers"/> 自定义执行 DNS 挑战时使用的 DNS 解析器；这些解析器优先于系统解析器或任何默认解析器。如果在此处设置，解析器将传播到所有配置的证书签发者。

  这通常是一个 IP 地址列表。例如，要使用 [Google Public DNS <img src="/old/resources/images/external-link.svg" class="external-link">](https://developers.google.com/speed/public-dns)：

  ```caddy-d
  resolvers 8.8.8.8 8.8.4.4
  ```

- **eab** <span id="eab"/> 使用你的 CA 提供的密钥 ID 和 MAC 密钥，为该站点配置 ACME 外部账户绑定（EAB）。

- **on_demand** <span id="on_demand"/> 为站点块地址中给定的主机名启用 [按需 TLS](/docs/automatic-https#on-demand-tls)。**安全警告：** 在生产环境中这样做是不安全的，除非你还配置了 [`on_demand_tls` 全局选项](/docs/caddyfile/options#on-demand-tls) 来减轻滥用风险。

- **reuse_private_keys** <span id="reuse_private_keys"/> 启用续期证书时重用私钥。默认情况下，每个新证书都会创建一个新密钥，以减轻固定密钥的风险并减少密钥泄露的影响。密钥固定违反行业最佳实践。除非你有特定理由使用它，否则不建议使用此选项；这在未来版本中可能会被移除。

- **client_auth** <span id="client_auth"/> 启用并配置 TLS 客户端认证：
  - **mode** <span id="mode"/> 是认证客户端的模式。允许的值有：

    | 模式 | 描述 |
    | --- | --- |
    | request | 向客户端请求证书，但即使没有也允许；不进行验证 |
    | require | 要求客户端提供证书，但不进行验证 |
    | verify_if_given | 向客户端请求证书；即使没有也允许，但有证书则进行验证 |
    | require_and_verify | 要求客户端提供经过验证的有效证书 |

    默认值：如果提供了 `trust_pool` 模块，则为 `require_and_verify`；否则为 `require`。
	
  - **trust_pool** <span id="trust_pool"/> 配置证书颁发机构（CA）的来源，这些 CA 提供用于验证客户端证书的证书。
	
	用于提供受信任证书池的证书颁发机构以及分段内的配置取决于所配置的信任池模块来源。Caddy 中可用的标准模块[如下所列](#trust-pool-providers)。完整模块列表（包括第三方）在 [`trust_pool` JSON 文档](/docs/json/apps/http/servers/tls_connection_policies/client_authentication/#trust_pool) 中列出。

    可以使用多个 `trusted_*` 指令来指定多个 CA 或叶证书。未列为叶证书之一或未由任何指定 CA 签名的客户端证书将根据 **mode** 被拒绝。

  - **verifier** <span id="verifier"/> 启用自定义客户端证书验证器模块。这些模块可以执行自定义客户端认证检查，例如确保证书未被吊销。

- **issuer** <span id="issuer"/> 配置自定义证书签发者，或从中获取证书的来源。

  使用哪个签发者以及此分段中跟随的选项取决于可用的[签发者模块](#issuers)。其他一些子指令，如 `ca` 和 `dns`，实际上是配置 `acme` 签发者的快捷方式（而此子指令是后来添加的），因此同时指定此指令和其他一些指令会引起混淆，因此被禁止。
  
  可以多次指定此子指令以配置多个冗余签发者；如果一个签发者无法签发证书，将尝试下一个。

- **get_certificate** <span id="get_certificate"/> 启用握手时从[管理器模块](#certificate-managers)获取证书。

- **insecure_secrets_log** <span id="insecure_secrets_log"/> 启用将 TLS 密钥记录到文件。这也称为 `SSLKEYLOGFILE`。使用 NSS 密钥日志格式，可由 Wireshark 或其他工具解析。⚠️ **安全警告：** 这是不安全的，因为它允许其他程序或工具解密 TLS 连接，从而完全破坏安全性。然而，此能力可用于调试和故障排除。

- **renewal_window_ratio** <span id="renewal_window_ratio"/> 是 0 到 1 之间的比率，确定证书剩余生命周期中，Caddy 应尝试续期的时间点。例如，如果证书有效期为 90 天，且此比率为 `0.3333`（默认值），则 Caddy 将在证书剩余 30 天或更少时持续尝试续期。也可以使用 [`renewal_window_ratio` 全局选项](/docs/caddyfile/options#renewal_window_ratio) 全局设置。

  你很少需要更改此值，但如果你的 CA 签发时间非常长，它可能有助于在证书生命周期后期续期。

  请记住，这只是一个建议，因为 ACME 签发者可能实现 [ARI 扩展](https://datatracker.ietf.org/doc/rfc9773/)。ARI 规定了 ACME 客户端（此处为 Caddy）应尝试续期的时间窗口，该窗口可能与此比率不一致。

- **force_automate** 与内联指定相同（见上文）。

### 信任池提供者

以下是可在 `trust_pool` 子指令中使用的标准信任池提供者：

#### inline

`inline` 模块直接在 Caddyfile 中解析以 base64 DER 编码格式列出的受信任根证书。`trust_der` 指令可以重复多次。

```caddy-d
trust_pool inline {
	trust_der      <base64_der>
}
```

- **trust_der** <span id="trust_der"/> 是一个 base64 DER 编码的 CA 证书，用于验证客户端证书。

#### file

`file` 模块从磁盘上的 PEM 文件读取受信任根证书。`pem_file` 指令可以在一行中接受多个文件路径，并可重复多次。

```caddy-d
... file [<pem_file>...] {
	pem_file <pem_file>...
}
```

- **pem_file** <span id="pem_file"/> 是一个 PEM CA 证书文件的路径，用于验证客户端证书。

#### pki_root

`pki_root` 模块从 [PKI 应用](/docs/caddyfile/options#pki-options) 定义的证书颁发机构获取 _根_ 证书并信任它们。`authority` 指令可以同时接受多个授权机构，并可重复多次。

```caddy-d
... pki_root [<ca_name>...] {
	authority <ca_name>...
}
```

- **authority** <span id="authority"/> 是在 PKI 应用中配置的证书颁发机构名称。

#### pki_intermediate

`pki_intermediate` 模块从 [PKI 应用](/docs/caddyfile/options#pki-options) 定义的证书颁发机构获取 _中间_ 证书并信任它们。`authority` 指令可以同时接受多个授权机构，并可重复多次。

```caddy-d
... pki_intermediate [<ca_name>...] {
	authority <ca_name>...
}
```

- **authority** <span id="authority"/> 是在 PKI 应用中配置的证书颁发机构名称。

#### storage

`storage` 模块从 Caddy [存储](/docs/caddyfile/options#storage) 中提取受信任的证书根。`authority` 指令可以同时接受多个授权机构，并可重复多次。

```caddy-d
... storage [<storage_keys>...] {
	storage <storage_module>
	keys    <storage_keys>...
}
```

- **storage** <span id="storage"/> 是一个可选的存储模块。如果未指定，将使用默认存储模块。如果指定，只能指定一次。

- **keys** <span id="keys"/> 是存储证书 PEM 文件的存储键列表。该指令接受同一行中的多个值，并可多次指定。

#### http

`http` 模块从 HTTP 端点获取受信任证书。`endpoints` 指令可以同时接受多个端点，并可重复多次。

```caddy-d
... http [<endpoints...>] {
	endpoints   <endpoints...>
	tls         <tls_config>
}
```

- **endpoints** <span id="endpoints"/> 是获取证书的 HTTP 端点列表。该指令接受同一行中的多个值，并可多次指定。

- **tls** <span id="tls"/> 是连接 HTTP 端点时使用的可选 TLS 配置。分段解析在[以下部分](#tls-1)中定义。

##### TLS

```caddy-d
... {
	ca                    <ca_module>
	insecure_skip_verify
	handshake_timeout     <duration>
	server_name           <name>
	renegotiation         <never|once|freely>
}
```

- **ca** <span id="ca"/> 是一个可选指令，用于定义信任池的提供者。配置遵循与 [`trust_pool`](#trust_pool) 相同的行为。如果指定，只能指定一次。

- **insecure_skip_verify** <span id="insecure_skip_verify"/> 关闭 TLS 握手验证，使连接不安全，容易受到中间人攻击。_不要在生产环境中使用。_ 验证针对系统信任的证书颁发机构或由 [`ca`](#ca) 指令决定。

- **handshake_timeout** <span id="handshake_timeout"/> 是等待 TLS 握手完成的最长[时长](/docs/conventions#durations)。默认值：无超时。

- **server_name** <span id="server_name"/> 设置验证 TLS 握手中收到的证书时使用的服务器名称。默认情况下，将使用上游地址的主机部分。

- **renegotiation** <span id="renegotiation"/> 设置 TLS 重协商级别。TLS 重协商是在首次握手之后执行后续握手的行为。级别可以是以下之一：
  - `never`（默认值）禁用重协商。
  - `once` 允许远程服务器每次连接请求一次重协商。
  - `freely` 允许远程服务器重复请求重协商。

### 验证器

客户端证书验证器模块在验证证书由受信任的证书颁发机构签发后执行（如果配置了 `trust_pool`）。目前，标准 Caddy 中附带的一个验证器是 `leaf`。

#### Leaf

`leaf` 验证器检查客户端证书是否属于定义的一组允许的证书。证书集使用[加载器](https://caddyserver.com/docs/modules/tls.client_auth.verifier.leaf#leaf_certs_loaders)模块加载。

##### 加载器

标准 Caddy 发行版捆绑了 4 个加载器，其中 3 个可在 Caddyfile 中使用。

###### File

`file` 加载器从指定的 PEM 文件加载证书集。

```caddy-d
... file <pem_files...>
```

###### Folder

`folder` 加载器递归遍历指定的目录，搜索要加载为接受客户端证书的 PEM 文件。

```caddy-d
... folder <folders...>
```

###### PEM

`pem` 加载器接受以 PEM 格式内联在 Caddyfile 中的证书。

```caddy-d
... pem <pem_strings...>
```

### 签发者

以下是 `tls` 指令附带的标准签发者：

#### acme

使用 ACME 协议获取证书。注意 `acme` 是默认签发者（使用 Let's Encrypt），因此明确配置它通常是不必要的。

```caddy-d
... acme [<directory_url>] {
	dir      <directory_url>
	test_dir <test_directory_url>
	email    <email>
	timeout  <duration>
	disable_http_challenge
	disable_tlsalpn_challenge
	alt_http_port    <port>
	alt_tlsalpn_port <port>
	eab <key_id> <mac_key>
	trusted_roots <pem_files...>
	dns [<provider_name> [<options>]]
	propagation_timeout <duration>
	propagation_delay   <duration>
	dns_ttl             <duration>
	dns_challenge_override_domain <domain>
	resolvers <dns_servers...>
	preferred_chains [smallest] {
		root_common_name <common_names...>
		any_common_name  <common_names...>
	}
	profile <name>
}
```

- **dir** <span id="dir"/> 是 ACME CA 目录的 URL。
  
  默认值：`https://acme-v02.api.letsencrypt.org/directory`

- **test_dir** <span id="test_dir"/> 是重试挑战时使用的可选备用目录；如果所有挑战失败，将在重试期间使用此端点；如果 CA 有一个临时端点，你希望避免对其生产端点施加速率限制，这将很有用。

  默认值：`https://acme-staging-v02.api.letsencrypt.org/directory`

- **email** <span id="email"/> 是 ACME 账户联系邮箱地址。

- **timeout** <span id="timeout"/> 是一个[时长值](/docs/conventions#durations)，设置 ACME 操作超时前等待的时间。

- **disable_http_challenge** <span id="disable_http_challenge"/> 将禁用 HTTP 挑战。

- **disable_tlsalpn_challenge** <span id="disable_tlsalpn_challenge"/> 将禁用 TLS-ALPN 挑战。

- **alt_http_port** <span id="alt_http_port"/> 是提供 HTTP 挑战的备用端口；必须发生在端口 80 上，因此需要将数据包转发到此备用端口。

- **alt_tlsalpn_port** <span id="alt_tlsalpn_port"/> 是提供 TLS-ALPN 挑战的备用端口；必须发生在端口 443 上，因此需要将数据包转发到此备用端口。

- **eab** <span id="eab"/> 指定外部账户绑定（EAB），某些 ACME CA 可能需要。

- **trusted_roots** <span id="trusted_roots"/> 是一个或多个根证书（作为 PEM 文件名），在连接到 ACME CA 服务器时信任它们。

- **dns** <span id="dns"/> 配置 DNS 挑战。必须在此处配置提供者，除非 [`dns` 全局选项](/docs/caddyfile/options#dns) 指定了全局适用的 DNS 提供者模块。

- **propagation_timeout** <span id="propagation_timeout"/> 是一个[时长值](/docs/conventions#durations)，设置使用 DNS 挑战时等待 DNS TXT 记录出现的最长时间。设置为 `-1` 以禁用传播检查。默认 2 分钟。

- **propagation_delay** <span id="propagation_delay"/> 是一个[时长值](/docs/conventions#durations)，设置在使用 DNS 挑战时，开始 DNS TXT 记录传播检查前等待的时间。默认 0（不等待）。

- **dns_ttl** <span id="dns_ttl"/> 是一个[时长值](/docs/conventions#durations)，设置用于 DNS 挑战的 `TXT` 记录的 TTL。很少需要设置。

- **dns_challenge_override_domain** <span id="dns_challenge_override_domain"/> 覆盖用于 DNS 挑战的域名。这是为了将挑战委托给另一个域名。

  如果你的主域名的 DNS 提供者没有可用的 [DNS 插件 <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddy-dns)，你可能希望使用此选项。你可以改为在你的主域名中添加一个子域为 `_acme-challenge` 的 `CNAME` 记录，指向你确实拥有插件的辅助域名。此选项 _不需要_ 插件提供特殊支持。
  
  当 ACME 签发者尝试为你主域名解决 DNS 挑战时，它们将跟随 `CNAME` 记录到你的辅助域名以查找 `TXT` 记录。

  **注意：** 在此处使用 CNAME 记录中的完整规范名称 - `_acme-challenge` 子域不会自动附加。

- **resolvers** <span id="resolvers"/> 自定义执行 DNS 挑战时使用的 DNS 解析器；这些解析器优先于系统解析器或任何默认解析器。如果在此处设置，解析器将传播到所有配置的证书签发者。

  这通常是一个 IP 地址列表。例如，要使用 [Google Public DNS <img src="/old/resources/images/external-link.svg" class="external-link">](https://developers.google.com/speed/public-dns)：

  ```caddy-d
  resolvers 8.8.8.8 8.8.4.4
  ```

- **preferred_chains** <span id="preferred_chains"/> 指定 Caddy 应优先使用的证书链；如果你的 CA 提供多条链，这很有用。使用以下选项之一：
	- **smallest** <span id="smallest"/> 将告诉 Caddy 优先选择字节数最少的链。

	- **root_common_name** <span id="root_common_name"/> 是一个或多个通用名称的列表；Caddy 将选择第一个根与至少一个指定通用名称匹配的链。

	- **any_common_name** <span id="any_common_name"/> 是一个或多个通用名称的列表；Caddy 将选择第一个签发者与至少一个指定通用名称匹配的链。

- **profile** 是在订购证书时要应用的 [ACME profile](https://datatracker.ietf.org/doc/draft-aaron-acme-profiles/) 名称。如果指定了，所有配置的（隐式或其他方式）CA 必须支持此 profile。请参考你的 CA 文档了解可用的 profiles；某些 CA 可能不支持 profiles。实验性：ACME profile 规范仍处于草案状态，因此此功能可能更改或移除。


#### zerossl

使用 [ZeroSSL 专有的证书签发 API](https://zerossl.com/documentation/api/) 获取证书。需要 API 密钥，根据你的计划可能还需要付费。请注意，此签发者与 [ZeroSSL 的 ACME 端点](https://zerossl.com/documentation/acme/) 不同。要使用 ZeroSSL 的 ACME 端点，请使用上面描述的 `acme` 签发者，并配置 ZeroSSL 的 ACME 目录端点。

```caddy-d
... zerossl <api_key> {
	validity_days <days>
	alt_http_port <port>
	dns <provider_name> ...
	propagation_delay <duration>
	propagation_timeout <duration>
	resolvers <list...>
	dns_ttl <duration>
}
```

- **validity_days** <span id="validity_days"/> 定义证书的有效期。只接受某些值；详情请参阅 [ZeroSSL 的文档](https://zerossl.com/documentation/api/create-certificate/)。
<!--   
  Default: `https://acme-v02.api.letsencrypt.org/directory`
 -->
- **alt_http_port** <span id="zerossl_alt_http_port"/> 是用于完成 ZeroSSL HTTP 验证的端口（如果不是端口 80）。
- **dns** <span id="zerossl_dns"/> 使用指定的 DNS 提供者和给定配置启用 CNAME 验证方法以自动配置记录。DNS 提供者插件必须从 [`caddy-dns` <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddy-dns) 仓库安装。每个提供者插件可能在其名称后有自己特定的语法；详情请参考其文档。维护每个 DNS 提供者的支持是社区努力的结果。
- **propagation_delay** <span id="zerossl_propagation_delay"/> 是在检查 CNAME 记录传播前等待的时间。
- **propagation_timeout** <span id="zerossl_propagation_timeout"/> 是等待 CNAME 记录传播的最长时间，超过则放弃。
- **resolvers** <span id="zerossl_resolvers"/> 定义在检查 CNAME 记录传播时使用的自定义 DNS 解析器。
- **dns_ttl** <span id="zerossl_dns_ttl"/> 配置作为验证过程一部分创建的 CNAME 记录的 TTL。



#### internal

从内部证书颁发机构获取证书。

```caddy-d
... internal {
	ca       <name>
	lifetime <duration>
	sign_with_root
}
```

- **ca** <span id="ca"/> 是要使用的内部 CA 名称。默认值：`local`。请参阅 [PKI 应用全局选项](/docs/caddyfile/options#pki-options) 来配置 `local` CA，或创建其他 CA。

  默认情况下，根 CA 证书的有效期为 `3600d`（10 年），中间证书的有效期为 `7d`（7 天）。

  Caddy 会尝试将根 CA 证书安装到系统信任存储中，但当 Caddy 以非特权用户身份运行或在 Docker 容器中运行时，此操作可能会失败。此时，需要手动安装根 CA 证书，可以使用 [`caddy trust`](/docs/command-line#caddy-trust) 命令，或[从容器中复制](/docs/running#usage)。

- **lifetime** <span id="lifetime"/> 是一个[时长值](/docs/conventions#durations)，设置内部颁发的叶证书的有效期。默认值：`12h`。除非绝对必要，否则不建议更改。它必须短于中间证书的有效期。

- **sign_with_root** <span id="sign_with_root"/> 强制使用根证书而不是中间证书作为签发者。不建议这样做，仅当设备/客户端未正确验证证书链时才应使用（非常罕见）。



### 证书管理器

证书管理器模块与签发者模块不同之处在于，使用管理器模块意味着外部工具或服务负责续期证书，而签发者模块意味着 Caddy 自己管理证书。（签发者模块将证书签名请求（CSR）作为输入，而证书管理器模块将 TLS ClientHello 作为输入。）

以下是 `tls` 指令附带的标准管理器模块：

#### tailscale

从本地运行的 [Tailscale <img src="/old/resources/images/external-link.svg" class="external-link">](https://tailscale.com) 实例获取证书。[必须在你 Tailscale 账户中启用 HTTPS](https://tailscale.com/kb/1153/enabling-https/)（或你的开源 [Headscale 服务器 <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/juanfont/headscale)）；并且 Caddy 进程必须以 root 身份运行，或者你必须配置 `tailscaled` 以授权你的 Caddy 用户[获取证书](https://github.com/caddyserver/caddy/pull/4541#issuecomment-1021568348)。

_**注意：这通常是不必要的！** Caddy 会自动为所有 `*.ts.net` 域名使用 Tailscale，无需额外配置。_

```caddy-d
get_certificate tailscale  # 通常不需要！
```


#### http

通过发出 HTTP(S) 请求获取证书。响应必须具有 `200` 状态码，并且正文必须包含 PEM 链，包括完整证书（带中间证书）以及私钥。

```caddy-d
get_certificate http <url>
```

- **url** <span id="url"/> 是请求的完整限定 URL。出于性能原因，强烈建议此端点为本地端点。URL 将附加以下查询字符串参数： 

  - `server_name`：SNI 值
  - `signature_schemes`：以逗号分隔的签名算法十六进制 ID 列表
  - `cipher_suites`：以逗号分隔的密码套件十六进制 ID 列表
  - `local_ip`：客户端发起请求的 IP 地址



## 示例

使用自定义证书和密钥。证书应具有与站点地址匹配的 [SAN](https://en.wikipedia.org/wiki/Subject_Alternative_Name)：

```caddy
example.com {
	tls cert.pem key.pem
}
```

对当前站点块中的所有主机使用[本地信任的](/docs/automatic-https#local-https)证书，而不是通过 ACME / Let's Encrypt 使用公共证书（在开发环境中很有用）：

```caddy
example.com {
	tls internal
}
```

使用本地信任的证书，但通过[按需](/docs/automatic-https#on-demand-tls)方式管理，而非后台管理。这允许你将任何域名指向你的 Caddy 实例，并让它自动为你配置证书。如果 Caddy 实例可公开访问，则不应使用此方法，因为攻击者可能利用它耗尽你的服务器资源：

```caddy
https:// {
	tls internal {
		on_demand
	}
}
```

为内部 CA 使用自定义选项（不能使用 `tls internal` 快捷方式）：

```caddy
example.com {
	tls {
		issuer internal {
			ca foo
		}
	}
}
```

为你的 ACME 账户指定邮箱地址（但如果所有站点只使用一个邮箱，我们建议使用 `email` [全局选项](/docs/caddyfile/options)代替）：

```caddy
example.com {
	tls your@email.com
}
```

为在 Cloudflare 上管理的域名启用 DNS 挑战，账户凭据存储在环境变量中。这会解锁通配符证书支持，而通配符证书需要 DNS 验证：

```caddy
*.example.com {
	tls {
		dns cloudflare {env.CLOUDFLARE_API_TOKEN}
	}
}
```

通过 HTTP 获取证书链，而不是让 Caddy 管理。注意 [`get_certificate`](#certificate-managers) 隐含启用了 [`on_demand`](#on_demand)，使用模块获取证书而不是触发 ACME 签发：

```caddy
https:// {
	tls {
		get_certificate http http://localhost:9007/certs
	}
}
```

启用 TLS 客户端认证，并要求客户端提供有效证书，该证书将通过 [`trust_pool`](#trust_pool) `file` 提供者提供的所有 CA 进行验证：

```caddy
example.com {
	tls {
		client_auth {
			trust_pool file ../caddy.ca.cer ../root.ca.cer
		}
	}
}