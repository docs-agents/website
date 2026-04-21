---
title: "自动 HTTPS"
---

# 自动 HTTPS

**Caddy 是首个默认自动使用 HTTPS 的 Web 服务器。**

自动 HTTPS 为您的所有站点自动配置 TLS 证书并进行续签。它还会为您将 HTTP 重定向到 HTTPS！Caddy 使用安全、现代化的默认设置——无需停机、无需额外配置、也无需额外工具。

<aside class="tip">
	Caddy 首创了自动 HTTPS 技术；我们从 2015 年该技术可行之初就开始使用。Caddy 的 HTTPS 自动化逻辑是世界上最成熟、最稳健的。
</aside>

这是一个 28 秒的视频，展示它是如何工作的：

<iframe width="100%" height="480" src="https://www.youtube-nocookie.com/embed/nk4EWHvvZtI?rel=0" frameborder="0" allowfullscreen=""></iframe>


**菜单：**

- [概述](#overview)
- [激活](#activation)
- [效果](#effects)
- [主机名要求](#hostname-requirements)
- [本地 HTTPS](#local-https)
- [测试](#testing)
- [ACME 挑战](#acme-challenges)
- [按需 TLS](#on-demand-tls)
- [错误](#errors)
- [存储](#storage)
- [通配符证书](#wildcard-certificates)
- [加密的 ClientHello (ECH)](#encrypted-clienthello-ech)



## 概述

**默认情况下，Caddy 通过 HTTPS 提供所有站点。**

- Caddy 使用自签名证书为 IP 地址和本地/内部主机名提供 HTTPS（如果允许，这些证书会自动被本地信任）。
	- 示例：`localhost`，`127.0.0.1`
- Caddy 使用公共 ACME CA（如 [Let's Encrypt <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org) 或 [ZeroSSL <img src="/old/resources/images/external-link.svg" class="external-link">](https://zerossl.com)）的证书通过 HTTPS 提供公共 DNS 名称。
	- 示例：`example.com`，`sub.example.com`，`*.example.com`

Caddy 会自动续签所有托管的证书，并自动将 HTTP（默认端口 `80`）重定向到 HTTPS（默认端口 `443`）。

**对于本地 HTTPS：**

- Caddy 可能会提示您输入密码，将其唯一的根证书安装到您的信任存储中。这每个根证书只发生一次；并且您可以随时删除它。
- 任何不信任 Caddy 根 CA 证书的客户端访问该站点时都会显示安全错误。

**对于公共域名：**

<aside class="tip">

这些是任何基本生产网站的通用要求，不仅适用于 Caddy。主要区别是**在运行 Caddy 之前**正确设置您的 DNS 记录，以便它可以配置证书。

</aside>


- 如果您的域名的 A/AAAA 记录指向您的服务器，
- 端口 `80` 和 `443` 在外部开放，
- Caddy 可以绑定到这些端口（_或_ 这些端口已转发到 Caddy），
- 您的 [数据目录](/docs/conventions#data-directory) 可写入且持久，
- 并且您的域名在配置中的某个相关位置出现，

那么站点将自动通过 HTTPS 提供服务。您无需对此做任何其他操作。它就能正常工作！

由于 HTTPS 利用共享的公共基础设施，作为服务器管理员，您应该了解此页面其余的信息，以便避免不必要的问题，在问题发生时进行故障排查，并正确配置高级部署。



## 激活

当 Caddy 知道它正在提供服务的域名（即主机名）或 IP 地址时，它会自动激活自动 HTTPS。根据您运行或配置 Caddy 的方式，有各种方法可以告诉 Caddy 您的域名/IP：

- [站点地址](/docs/caddyfile/concepts#addresses) 在 [Caddyfile](/docs/caddyfile) 中
- [主机匹配器](/docs/json/apps/http/servers/routes/match/host/) 在 [JSON 路由](/docs/modules/http#servers/routes) 的顶层
- 命令行标志，如 [`--domain`](/docs/command-line#caddy-file-server) 或 [`--from`](/docs/command-line#caddy-reverse-proxy)
- [automate](/docs/json/apps/tls/certificates/automate/) 证书加载器

以下任何情况都会阻止自动 HTTPS 激活（全部或部分）：

- 通过 [JSON](/docs/json/apps/http/servers/automatic_https/) 或 [Caddyfile](/docs/caddyfile/options#auto-https) 明确禁用它
- 配置中未提供任何主机名或 IP 地址
- 仅在 HTTP 端口上监听
- 在 Caddyfile 中 [站点地址](/docs/caddyfile/concepts#addresses) 前缀为 `http://`
- 手动加载证书（除非设置 [`ignore_loaded_certificates`](/docs/json/apps/http/servers/automatic_https/ignore_loaded_certificates/)）

**特殊情况：**

- 以 `.ts.net` 结尾的域名将不会由 Caddy 管理。相反，Caddy 将尝试在握手时从本地运行的 [Tailscale <img src="/old/resources/images/external-link.svg" class="external-link">](https://tailscale.com) 实例获取这些证书。这要求您在 [Tailscale 账户中启用 HTTPS <img src="/old/resources/images/external-link.svg" class="external-link">](https://tailscale.com/kb/1153/enabling-https/)，并且 Caddy 进程必须以 root 运行，或者您必须配置 `tailscaled` 以授予您的 Caddy 用户 [获取证书的权限](https://github.com/caddyserver/caddy/pull/4541#issuecomment-1021568348)。


## 效果

当自动 HTTPS 激活时，会发生以下情况：

- 为 [所有符合条件的域名](#hostname-requirements) 获取并续签证书
- HTTP 重定向到 HTTPS（使用 [HTTP 端口](/docs/modules/http#http_port) `80`）

自动 HTTPS 永远不会覆盖显式配置，它只是增强配置。

如果您已经有 [服务器](/docs/json/apps/http/servers/) 在 HTTP 端口上监听，HTTP->HTTPS 重定向路由将在您的主机匹配器路由之后、用户定义的全局捕获路由之前插入。

如有必要，您可以 [自定义或禁用自动 HTTPS](/docs/json/apps/http/servers/automatic_https/)；例如，您可以跳过某些域名或禁用重定向（对于 Caddyfile，使用 [全局选项](/docs/caddyfile/options) 执行此操作）。


## 主机名要求

所有满足以下条件的主机名（域名）都符合完全托管证书的条件：

- 非空
- 仅由字母数字、连字符、点和通配符 (`*`) 组成
- 不以点开头或结尾（[RFC 1034](https://tools.ietf.org/html/rfc1034#section-3.5)）

此外，符合公共信任证书条件的主机名需要满足：

- 不是本地主机（包括 `.localhost`、`.local`、`.internal` 和 `.home.arpa` 顶级域）
- 不是 IP 地址
- 仅有一个通配符 `*` 作为最左侧标签


## 本地 HTTPS

Caddy 自动为所有指定主机（域名、IP 或主机名）的站点使用 HTTPS，包括内部和本地主机。某些主机不是公有的（例如 `127.0.0.1`、`localhost`）或通常不符合公共信任证书的条件（例如 IP 地址——您可以从某些 CA 获取证书，但并非所有）。这些站点仍然通过 HTTPS 提供，除非被禁用。

要通过 HTTPS 提供非公共站点，Caddy 会生成自己的证书颁发机构 (CA) 并使用它来签署证书。信任链由根证书和中间证书组成。叶子证书由中间证书签署。它们存储在 [Caddy 的数据目录](/docs/conventions#data-directory) 的 `pki/authorities/local` 处。

Caddy 的本地 CA 由 [Smallstep 库 <img src="/old/resources/images/external-link.svg" class="external-link">](https://smallstep.com/certificates/) 提供动力。

本地 HTTPS 不使用 ACME，也不执行任何 DNS 验证。它仅在本机运行，并且仅在安装 CA 根证书的机器上被信任。

### CA 根

根证书的私钥是使用加密安全的伪随机源唯一生成的，并以有限权限持久存储。它仅在内存中加载以执行签名任务，之后离开作用域以便垃圾回收。

尽管可以配置 Caddy 直接使用根进行签名（以支持不合规的客户端），但默认禁用此选项，根密钥仅用于签名中间证书。

第一次使用根密钥时，Caddy 会尝试将其安装到系统的本地信任存储中。如果没有权限这样做，它将提示输入密码。此行为可以使用 [caddyfile 中的 `skip_install_trust`](/docs/caddyfile/options#skip-install-trust) 或 [json 配置中的 `"install_trust": false`](/docs/json/apps/pki/certificate_authorities/install_trust/) 禁用。如果因以非特权用户运行而失败，您可以运行 [`caddy trust`](/docs/command-line#caddy-trust) 以特权用户重试安装。

<aside class="tip">
	只要您的计算机未被入侵且您的唯一根密钥未泄露，信任 Caddy 的根证书在您自己的计算机上是安全的。
</aside>

安装 Caddy 的根 CA 后，您将在本地信任存储中看到它显示为 "Caddy Local Authority"（除非您配置了不同的名称）。如果您愿意，可以随时卸载它（[`caddy untrust`](/docs/command-line#caddy-untrust) 命令使这很容易）。

注意，自动将证书安装到本地信任存储仅出于便利目的，不能保证工作，特别是使用容器或作为非特权系统服务运行 Caddy 时。最终，如果您依赖于内部 PKI，确保 Caddy 的根 CA 正确添加到必要的信任存储中是系统管理员的责任（这超出了 Web 服务器的范围）。


### CA 中间证书

还会生成中间证书和密钥，用于签署叶子（单个站点）证书。

与根证书不同，中间证书的有效期较短，并且会根据需要自动续签。


## 测试

要测试或实验您的 Caddy 配置，请确保您 [更改 ACME 端点](/docs/modules/tls.issuance.acme#ca) 为临时或开发 URL，否则您很可能会命中速率限制，这可能会阻止您访问 HTTPS 长达一周，具体取决于您命中哪个速率限制。

Caddy 的默认 CA 之一是 [Let's Encrypt <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/)，它有一个 [临时端点 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/staging-environment/)，不受相同的 [速率限制 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/rate-limits/) 约束：

```
https://acme-staging-v02.api.letsencrypt.org/directory
```

## ACME 挑战

获取公共信任的 TLS 证书需要来自公共信任的第三方的验证。如今，此验证流程通过 [ACME 协议 <img src="/old/resources/images/external-link.svg" class="external-link">](https://tools.ietf.org/html/rfc8555) 自动化，可以通过以下三种方式之一（"挑战类型"）执行，如下所述。

前两种挑战类型默认启用。如果启用多个挑战，Caddy 会随机选择一个以避免意外依赖特定挑战。随着时间的推移，它会学习哪个挑战类型最成功，并会开始优先选择它，但在必要时会回退到其他可用的挑战类型。


### HTTP 挑战

HTTP 挑战对候选主机名的 A/AAAA 记录执行权威 DNS 查找，然后使用 HTTP 通过端口 `80` 请求临时加密资源。如果 CA 看到预期的资源，则颁发证书。

此挑战要求端口 `80` 可外部访问。如果 Caddy 无法在端口 80 上监听，来自端口 `80` 的数据包必须转发到 Caddy 的 [HTTP 端口](/docs/json/apps/http/http_port/)。

此挑战默认启用，无需显式配置。


### TLS-ALPN 挑战

TLS-ALPN 挑战对候选主机名的 A/AAAA 记录执行权威 DNS 查找，然后使用包含特殊 ServerName 和 ALPN 值的 TLS 握手通过端口 `443` 请求临时加密资源。如果 CA 看到预期的资源，则颁发证书。

此挑战要求端口 `443` 可外部访问。如果 Caddy 无法在端口 443 上监听，来自端口 `443` 的数据包必须转发到 Caddy 的 [HTTPS 端口](/docs/json/apps/http/https_port/)。

此挑战默认启用，无需显式配置。


### DNS 挑战

DNS 挑战对候选主机名的 `TXT` 记录执行权威 DNS 查找，并查找具有特定值的特殊 `TXT` 记录。如果 CA 看到预期的值，则颁发证书。

此挑战不要求任何开放端口，请求证书的服务器也不需要可外部访问。然而，DNS 挑战需要配置。Caddy 需要知道访问您的域名 DNS 提供者的凭据，以便设置（和清除）特殊 `TXT` 记录。如果启用 DNS 挑战，其他挑战默认会禁用。

由于 ACME CA 遵循 DNS 标准来查找 `TXT` 记录进行挑战验证，您可以使用 CNAME 记录将挑战应答委托给其他 DNS 区域。这可用于将 `_acme-challenge` 子域委托给 [另一个区域](/docs/caddyfile/directives/tls#dns_challenge_override_domain)。这在您的 DNS 提供者未提供 API 或 Caddy 的 DNS 插件未支持时特别有用。

DNS 提供者支持是社区努力。[在我们的维基上了解如何为您的提供者启用 DNS 挑战。](https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148)


## 按需 TLS

Caddy 首创了我们称为 **按需 TLS** 的新兴技术，它在首次 TLS 握手需要证书时动态获取新证书，而不是在配置加载时。关键是，这**不需要**在配置中提前硬编码域名。

许多企业依赖这一独特功能，以更低成本和没有运营难题来扩展其 TLS 部署，同时提供数万个站点。

按需 TLS 适用于：

- 您启动或重载服务器时不知道所有域名，
- 域名可能无法正确配置（DNS 记录尚未设置），
- 您不控制域名（例如它们是客户域名）。

启用按需 TLS 后，您无需在配置中指定域名即可为其获取证书。相反，当收到针对服务器名称（SNI）的 TLS 握手而 Caddy 尚未为其有证书时，握手会被挂起，同时 Caddy 获取证书以完成握手。延迟通常只有几秒，只有首次握手较慢。所有后续握手都很快，因为证书会被缓存和重用，续签在后台进行。后续握手可能触发证书维护以确保持续续签，但如果证书尚未过期，此维护在后台进行。

### 使用按需 TLS

**必须启用并限制按需 TLS 以防止滥用。**

在 [TLS 自动化策略](/docs/json/apps/tls/automation/policies/) 中启用按需 TLS（如果使用 JSON 配置），或使用 Caddyfile 时 [在站点块中使用 `tls` 指令](/docs/caddyfile/directives/tls)。

为防止滥用此功能，您必须配置限制。这通过 JSON 配置的 [`automation` 对象](/docs/json/apps/tls/automation/on_demand/) 或 Caddyfile 的 [`on_demand_tls` 全局选项](/docs/caddyfile/options#on-demand-tls) 完成。限制是"全局"的，不能按站点或域名配置。主要限制是"询问"端点，Caddy 将发送 HTTP 请求到此端点以询问是否有权为该域名获取和管理证书。这意味着您需要一些内部后端，例如查询数据库的客户表，查看是否已有客户使用该域名注册。

注意您的 CA 颁发证书的速度。如果超过几秒，将负面影响用户体验（仅首次客户端）。

由于其延迟性质和防止滥用所需的其他配置，我们仅建议在实际使用案例如上述情况时启用按需 TLS。

[查看我们的维基文章了解更多关于有效使用按需 TLS 的信息。](https://caddy.community/t/serving-tens-of-thousands-of-domains-over-https-with-caddy/11179)

## 错误

Caddy 尽力在证书管理出错时继续运行。

默认情况下，证书管理在后台执行。这意味着它不会阻塞启动或减慢您的站点。但这同时也意味着即使在所有证书不可用时服务器也在运行。在后台运行允许 Caddy 以指数退避方式长时间重试。

如果获取或续签证书出错，会发生以下情况：

1. Caddy 短暂暂停后重试一次，以防是偶发问题
2. Caddy 短暂暂停，然后切换到下一个启用的挑战类型
3. 所有启用的挑战类型尝试后，[尝试下一个配置的颁发者](#issuer-fallback)
	- Let's Encrypt
	- ZeroSSL
4. 所有颁发者尝试后，指数退避
	- 尝试间最多 1 天
	- 持续长达 30 天

重试 Let's Encrypt 时，Caddy 会切换到他们的 [临时环境 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/staging-environment/) 以避免速率限制问题。这不是完美策略，但通常很有帮助。

ACME 挑战至少需要几秒，内部速率限制有助于防止意外滥用。Caddy 使用内部速率限制，加上您或 CA 配置的速率限制，这样您就可以给 Caddy 一百万个域名的托盘，它会逐渐但尽可能快地为它们全部获取证书。Caddy 的内部速率限制目前是每 10 秒每个 ACME 账户 10 次尝试。

为避免资源泄漏，Caddy 在配置更改时中止进行中的任务（包括 ACME 事务）。虽然 Caddy 能够处理频繁的配置重载，但要注意如这类的运营考虑，并考虑批处理配置更改以减少重载并给予 Caddy 在后台完成获取证书的机会。

### 颁发者回退

Caddy 是首个（至今唯一）支持在无法成功获取证书时完全冗余、自动回退到其他 CA 的服务器。

默认情况下，Caddy 启用两个 ACME 兼容的 CA：[**Let's Encrypt** <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org) 和 [**ZeroSSL** <img src="/old/resources/images/external-link.svg" class="external-link">](https://zerossl.com)。如果 Caddy 无法从 Let's Encrypt 获取证书，它将尝试 ZeroSSL；如果两者都失败，它会回退并在稍后重试。在您的配置中，您可以自定义 Caddy 使用的颁发者来获取证书，无论是全局还是特定名称。


## 存储

Caddy 会将其 [配置的存储设施](/docs/json/storage/)（或未配置时的默认设施——参见链接了解详情）中存储公共证书、私钥和其他资源。

**使用默认配置您需要了解的主要事项是 `$HOME` 文件夹必须可写入且持久。** 为了帮助您故障排查，如果指定了 `--environ` 标志，Caddy 会在启动时打印其环境变量。

任何配置为使用相同存储的 Caddy 实例将自动共享这些资源，并作为集群协调证书管理。

尝试任何 ACME 事务之前，Caddy 将测试配置的存储以确保其可写入并有足够容量。这有助于减少不必要的锁争用。


## 通配符证书

当配置为提供符合资格通配符名称的站点时，Caddy 可以获取和管理通配符证书。如果仅最左侧域名标签为通配符，站点名称符合通配符资格。例如，`*.example.com` 符合资格，但以下不符合：`sub.*.example.com`、`foo*.example.com`、`*bar.example.com` 和 `*.*.example.com`。（这是 WebPKI 的限制。）

如果使用 Caddyfile，Caddy 对证书主体名称逐字处理站点名称。换句话说，定义为 `sub.example.com` 的站点将导致 Caddy 管理 `sub.example.com` 的证书，定义为 `*.example.com` 的站点将导致 Caddy 管理 `*.example.com` 的通配符证书。您可以在我们的 [常见 Caddyfile 模式](/docs/caddyfile/patterns#wildcard-certificates) 页面看到此示例。如果您需要不同的行为，[JSON 配置](/docs/json/) 为您提供更精确的证书主体和站点名称（"主机匹配器"）控制。

自 Caddy 2.10 起，自动化通配符证书时，Caddy 将在配置中使用通配符证书用于子域。除非显式配置（例如使用 `force_automate`），否则不会为子域获取证书。

通配符证书代表广泛的权限，仅当您有太多子域，为它们管理单独证书会拖累 PKI 或导致您命中 CA 强制的速率限制时，或在密钥泄露时暴露 DNS 区域过多的隐私权衡值得风险时才使用。注意通配符证书本身不隐私隐藏特定子域：除非启用加密的 ClientHello (ECH)，否则它们仍然在 TLS ClientHello 数据包中暴露。（见下文。）

**注意：** [Let's Encrypt 要求 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/challenge-types/) [DNS 挑战](#dns-challenge) 来获取通配符证书。


## 加密的 ClientHello (ECH)

正常情况下，TLS 握手涉及以明文发送 ClientHello，包括服务器名称指示 (SNI; 正在连接的域名)。这是因为它包含握手后加密连接所需的参数。当然，这向任何能够监听连接的人暴露了域名，即使他们不在您的直接物理范围内。它揭示了您正在连接哪个服务，当目标 IP 可能服务于多个不同站点时，也是某些政府审查互联网的方式。

使用加密的 ClientHello，客户端可以通过将真实 ClientHello 包装在"外部"ClientHello 中来自保域名，该"外部"ClientHello 建立解密"内部"ClientHello 的参数。然而，许多部分需要完美协同工作才能提供实际隐私保护。

首先，客户端需要知道用于加密 ClientHello 的参数或配置。此信息包括公钥和"外部"域名（"公共名称"）等。此配置必须通过某种可靠方式发布或分发。

您理论上可以写在一张纸上分发给每个人，但大多数主流浏览器支持在连接到站点时查找包含 ECH 参数的 HTTPS 类型 DNS 记录。因此，您需要：(1) 生成 ECH 配置（公钥/私钥对，以及其他参数），然后 (2) 创建包含 base64 编码 ECH 配置的 HTTPS 类型 DNS 记录。

或者……您可以让 Caddy 为您完成这一切。Caddy 是首个也是唯一能够自动生成、发布和提供 ECH 配置的 Web 服务器。

一旦发布 HTTPS 记录，客户端在连接到您的站点时将需要查找 HTTPS 记录以获取 ECH 参数。通常，DNS 查询是明文，这损害了 resulting ECH 握手的安全性，因此浏览器将需要使用安全 DNS 协议如 DNS-over-HTTPS (DoH) 或 DNS-over-TLS (DoT)。根据浏览器不同，可能需要手动启用。

一旦客户端安全下载 ECH 配置，它使用嵌入的公钥加密 ClientHello，然后连接到您的站点。Caddy 然后解密内部 ClientHello 并继续为您提供站点，域名从未以明文形式出现在线路上。

### 部署考虑

ECH 是微妙的技术。即使 Caddy 完全自动化 ECH，仍需考虑许多因素以实现最大隐私收益。您也应该了解各种权衡。

#### 发布

如果域名已有记录，Caddy 只会为该域名创建 HTTPS 记录。这防止破坏可能被通配符覆盖的子域的 DNS 查询。确保您的站点至少有一条 A/AAAA 记录指向您的服务器。如果仅使用通配符 DNS 记录，则通配符域名也必须出现在您的 Caddy 配置中。

Caddy 不会为有 CNAME 记录的域名发布 HTTPS 记录。

#### ECH GREASE

如果您打开 Wireshark 然后连接到任何站点（即使是不支持 ECH 的站点）在现代版本的主要浏览器如 Firefox 或 Chrome（即使 ECH 已禁用），您可能会注意到其握手包括 `encrypted_client_hello` 扩展：

![ECH GREASE](/resources/images/ech-grease.png)

这是为了使真实 ECH 握手与明文握手不可区分。如果 ECH 握手与普通握手不同，审查者可以轻松阻断 ECH 握手而最小伤亡/附带损害。但如果他们阻断任何看似有合理 ECH 扩展的握手，他们将实质上关闭大部分互联网。（目标是增加大规模审查的成本。）

这主要对故障排查连接很重要时要知道。

#### 密钥轮换

与证书密钥一样，长期使用相同密钥不是良好做法（可能完全不安全）。因此，ECH 密钥应定期轮换。与证书不同，ECH 配置不严格过期。但服务器仍应轮换它们。

密钥轮换很棘手，因为客户端需要知道更新后的密钥。如果服务器简单地用新密钥替换旧密钥，所有 ECH 握手将失败，除非客户端立即被告知新密钥。但仅仅发布更新后的密钥还不够。实际情况是，DNS 记录有 TTL，解析器缓存响应等。客户端查询更新的 HTTPS 记录并开始使用新 ECH 配置可能需要几分钟、几小时甚至几天。

因此，服务器应在一段时间内继续支持旧 ECH 配置。不这样做可能导致大规模暴露服务器名称明文。Caddy 会定期轮换密钥，并支持旋转密钥一段时间，直到最终删除。

然而，这可能不够。某些客户端由于各种原因仍无法获取更新后的密钥，每当发生这种情况，就有暴露服务器名称的风险。因此需要另一种方式将更新后的配置与连接**带内**提供给客户端。这就是**外部名称**（或**公共名称**）的作用。

#### 公共名称

"外部"ClientHello 是正常的 ClientHello，有两个细微差别只有源服务器知道：

1. SNI 扩展是假的
2. ECH 扩展是真的

该"SNI"扩展包含保护您真实域的公共名称。此名称可以是任何内容，但**您的服务器必须是公共名称的权威服务器**，因为 Caddy _将_ 为其获取证书。

如果客户端尝试 ECH 连接但服务器无法解密内部 ClientHello，它实际上可以使用外部 ClientHello 和外部名称的证书完成握手。此安全连接严格_仅_用于将客户端发送当前 ECH 配置；即仅用于完成初始 TLS 连接的临时 TLS 连接。不传输应用数据：仅 ECH 密钥。一旦客户端获取更新后的密钥，它可以按预期建立 TLS 连接。

通过这种方式，真实服务器名称保持保护，不同步的客户端仍可连接，这是安全的两个关键要素。

外部名称可以是您站点的一个域名、子域或任何其他指向您服务器的域名。我们建议选择一个通用名称。例如，Cloudflare 在 `cloudflare-ech.com` 后面为数百万站点提供服务。这对增加您的匿名集大小很重要。

公共名称不应为空；即必须配置公共名称才能使事物正常工作。Caddy 当前不强制（之后可能），但 ECH 规范要求公共名称至少 1 字节长。某些软件接受空名称，其他不接受。这可能导致令人困惑的行为，如浏览器使用 ECH 但服务器拒绝为无效；或浏览器不使用 ECH（因为无效）即使配置正确存在于 DNS 记录中。确保正确 ECH 配置和发布以确保隐私是站点所有者的责任。


#### 匿名集

为了最大化 ECH 的隐私收益，努力最大化您的_匿名集_大小。本质上，此集合由对观察者具有相同行为的面向客户端的服务器组成。想法是观察者无法轻易减少/推断客户端正在连接的可能站点或服务。

在实践中，我们建议所有站点仅使用一个公共名称。（每个 ECH 配置只有 1 个公共名称，这意味着任何时候只维护 1 个活动的 ECH 配置。）如果您在集群中运行 Caddy，Caddy 自动与其他实例共享和协调 ECH 配置，为您处理此问题。

推到极端，这意味着互联网上的每个站点都可以或应该在一个 IP 地址和一个公共名称后面。


#### 集中化

……这引出了我们下一个主题：集中化。对 ECH 的批评之一是该技术趋向于推动集中化。它通过至少两种方式做到这一点：(1) 通过客户端偏好 DoH/DoT 用于 DNS 查询，将所有 DNS 查询发送到少数几个提供者，以及 (2) 在大规模最大化匿名集大小。

使用 DoH 或 DoT 时，DNS 查询都通过 DoH/DoT 提供者。在客户端和提供者之间，DNS 数据加密，但在提供者和 DNS 服务器之间，未加密。全球 DoH/DoT 有效地将所有诱人的明文 DNS 流量汇聚到少数几个大管道中，便于观察……或故障。

类似地，如果我们确实在大规模最大化匿名集，所有站点都受单个公共名称保护，如 `cloudflare-ech.com`。这对隐私有益，但整个互联网都取决于 Cloudflare 和那个域名。现在，推进到那种程度并非必要或不切实际，但理论影响仍然有效。

我们建议每个组织或个人为他们的所有站点选择一个名称并使用，在大多数情况下这应提供足够的隐私。然而，请咨询专家根据您的个人威胁模型处理您的具体情况。


#### 子域隐私

使用 ECH，理论上可以保持子域从侧信道保密/私有，如果正确部署。

大多数站点不需要此，一般来说，子域是公开信息。我们建议不要在域名中放置敏感信息。话虽如此……

为避免向证书透明度 (CT) 日志泄露敏感子域，使用通配符证书代替。换句话说，在配置中放置 `*.example.com` 而不是 `sub.example.com`。（查看 [通配符证书](#wildcard-certificates) 获取重要信息。）

另一个泄露源是 DNSSEC，大多数权威 DNS 服务器默认使用。通过名为"区域遍历"的做法，通过查看 NSEC 记录可能进行子域枚举，这些记录用于提供认证的否认存在。为此，它们指向字母顺序中下一个可用的子域，形成所有记录的链接列表。确保您的域名至少使用 NSEC3 或理想情况下使用通配符 CNAME 记录以减轻此问题。

然后，在 Caddy 中启用 ECH。通配符证书结合 ECH 和通配符 CNAME 记录应正确隐藏子域，只要每个尝试连接的客户端使用 ECH 并有强大实现。（您仍然依赖客户端保护隐私。）


### 启用 ECH

由于运行的 ECH 需要发布配置到 DNS 记录，您需要为 DNS 提供者插入 [caddy-dns 模块](https://github.com/caddy-dns) 的 Caddy 构建。

然后，使用 Caddyfile，在全局选项中指定您的 DNS 提供者配置，以及您想使用的 ECH 公共名称：

```caddy
{
	dns <provider config...>
	ech example.com
}
```

记住：

- 必须插入 DNS 提供者模块，您必须为您的提供者/账户有正确配置。
- ECH 公共名称应指向您的服务器。Caddy 将为其获取证书。不必是您的站点域名之一。

如果使用 JSON，将这些属性添加到 `tls` 应用：

```json
"encrypted_client_hello": {
	"configs": [
		{
			"public_name": "example.com"
		}
	]
},
"dns": {
	"name": "<provider name>",
	// provider configuration
}
```

这些配置将启用 ECH 并为您的所有站点发布 ECH 配置。JSON 配置提供更多灵活性，如果您需要自定义行为或有高级设置。

### 验证 ECH

围绕 ECH 的工具仍然不多，所以在编写时，验证它是否工作的最好最通用方法是使用 Wireshark 并在 ServerName 字段中查找您的公共名称。

首先，启动您的服务器，看到日志提及您的域名类似"发布 ECH 配置列表"。（如果您遇到任何发布错误，确保您的 DNS 提供者模块支持 [libdns 1.0](https://github.com/libdns/libdns)，如果您遇到问题，在您的提供者仓库中提交问题。）Caddy 还应为公共名称获取证书。

接下来，确保您的浏览器启用了 ECH；这可能需要启用 DoH/DoT。清除浏览器（或系统）的 DNS 缓存也是个好主意，以确保它将拾取新发布的 HTTPS 记录。我们也建议关闭浏览器或至少打开一个新隐私标签页，以确保它不重用现有连接。

然后，打开 Wireshark 并开始监听适当的网络接口。当 Wireshark 收集数据包时，在浏览器中加载您的站点。然后您可以暂停 Wireshark。找到您的 TLS ClientHello，您应该在 ServerName 字段中看到_公共名称_，而不是实际连接的域名。

记住：即使不使用 ECH，您仍可能看到 `encrypted_client_hello` 扩展。关键指标是 SNI 值。如果 ECH 正常工作，您绝不应在 Wireshark 中以明文形式看到真实站点名称。

如果您遇到 ECH 部署问题，首先在我们的 [论坛](https://caddy.community) 询问。如果是错误，您可以在 GitHub 上 [提交问题](https://github.com/caddyserver/caddy/issues)。


### ECH 存储

ECH 配置存储在 [数据目录](/docs/conventions#data-directory) 的配置存储模块下（默认为文件系统）的 `ech/configs` 文件夹。

下一个文件夹是 ECH 配置 ID，随机生成，相对不重要。规范推荐随机性以帮助缓解指纹识别/跟踪。

元数据辅助文件帮助 Caddy 跟踪上次发布发生时间。这防止在每次配置重载时轰炸您的 DNS 提供者。如果您必须重置此状态，您可以安全删除元数据文件。但这也可能重置密钥轮换时间。您也可以进入文件并仅清除关于发布的信息。