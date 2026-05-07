---
title: acme_server (Caddyfile指令)
---

# acme_server

一个嵌入式的 [ACME协议](https://tools.ietf.org/html/rfc8555) 服务器处理器。它允许 Caddy 实例为任何其他兼容 ACME 的软件（包括其他 Caddy 实例）颁发证书。

启用后，匹配路径 `/acme/*` 的请求将由 ACME 服务器处理。


## 客户端配置

使用 ACME 服务器默认配置时，ACME 客户端只需将 `https://localhost/acme/local/directory` 配置为其 ACME 端点即可。（`local` 是 Caddy 默认 CA 的 ID。）


## 语法

```caddy-d
acme_server [<匹配器>] {
	ca         <ID>
	lifetime   <持续时间>
	resolvers  <解析器列表...>
	challenges <挑战类型列表...>
	allow_wildcard_names
	allow {
		domains <域名列表...>
		ip_ranges <地址列表...>
	}
	deny {
		domains <域名列表...>
		ip_ranges <地址列表...>
	}
}
```

- **ca** 指定用于签署证书的证书颁发机构的 ID。默认值为 `local`，即 Caddy 的默认 CA，用于本地使用的自签名证书，这在开发环境中最为常见。对于更广泛的使用，建议指定不同的 CA 以避免混淆。如果给定 ID 的 CA 尚不存在，则会创建它。请参阅 [PKI 应用程序全局选项](/docs/caddyfile/options#pki-options) 来配置其他 CA。

- **lifetime**（默认值：`12h`）是一个 [持续时间](/docs/conventions#durations)，指定所颁发证书的有效期。此值必须小于用于签署证书的 [中间证书](/docs/caddyfile/options#intermediate-lifetime) 的有效期。除非绝对必要，否则不建议更改此值。

- **resolvers** 是用于在解决 ACME DNS 挑战时查找 TXT 记录的 DNS 解析器地址。接受 [网络地址](/docs/conventions#network-addresses)，默认使用 UDP 和端口 53（除非另行指定）。如果主机是 IP 地址，则会直接拨号以解析上游服务器。如果主机不是 IP 地址，则使用 Go 标准库的 [名称解析约定](https://golang.org/pkg/net/#hdr-Name_Resolution) 来解析这些地址。如果指定了多个解析器，则随机选择一个。

- **challenges** 设置启用的挑战类型。如果未设置或指令使用时未带值，则启用所有挑战类型。接受的值为：http-01、tls-alpn-01、dns-01。

- **allow_wildcard_names** 允许颁发包含通配符 SAN（主题备用名称）的证书。

- **allow**、**deny** 配置 `acme_server` 的操作策略。策略评估遵循 Step-CA 在此描述的 [标准](https://smallstep.com/docs/step-ca/policies/#policy-evaluation)。

	- **domains** 设置根据策略评估标准允许或拒绝的域名。

	- **ip_ranges** 设置根据策略评估标准允许或拒绝的 IP 范围。

## 示例

在域名 `acme.example.com` 上提供 ID 为 `home` 的 ACME 服务器，通过 [`pki` 全局选项](/docs/caddyfile/options#pki-options) 自定义 CA，并使用 `internal` 颁发者为其自身颁发证书：

```caddy
{
	pki {
		ca home {
			name "My Home CA"
		}
	}
}

acme.example.com {
	tls {
		issuer internal {
			ca home
		}
	}
	acme_server {
		ca home
	}
}
```

如果你有另一个 Caddy 服务器，它可以使用上述 ACME 服务器来颁发自己的证书：

```caddy
{
	acme_ca https://acme.example.com/acme/home/directory
	acme_ca_root /path/to/home_ca_root.crt
}

example.com {
	respond "Hello, world!"
}