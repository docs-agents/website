---
title: "自动 HTTPS"
---

# 自动 HTTPS

**Caddy 是第一个默认自动使用 HTTPS 的 Web 服务器。**

自动 HTTPS 为所有站点配置 TLS 证书并保持续期，同时自动将 HTTP 重定向到 HTTPS！Caddy 使用了安全且现代默认配置——无需停机、额外配置或单独工具。

<aside class="tip">
	Caddy 开创了自动 HTTPS 技术；自 2015 年该技术可行之日起，我们一直在实践。Caddy 的 HTTPS 自动化逻辑是世界上最成熟、最稳健的。
</aside>

以下是一个 28 秒的视频，展示其工作原理：

<iframe width="100%" height="480" src="https://www.youtube-nocookie.com/embed/nk4EWHvvZtI?rel=0" frameborder="0" allowfullscreen=""></iframe>

**目录：**

- [概述](#概述)
- [激活](#激活)
- [效果](#效果)
- [主机名要求](#主机名要求)
- [本地 HTTPS](#本地-https)
- [测试](#测试)
- [ACME 挑战](#acme-挑战)
- [按需 TLS](#按需-tls)
- [错误](#错误)
- [存储](#存储)
- [通配符证书](#通配符证书)
- [加密客户端问候 (ECH)](#加密客户端问候-ech)

## 概述

**默认情况下，Caddy 通过 HTTPS 提供所有站点服务。**

- Caddy 使用自签名证书为 IP 地址和本地/内部主机名提供 HTTPS 服务，这些证书会在本地自动信任（如果允许）。
  - 示例：`localhost`、`127.0.0.1`
- Caddy 使用来自公共 ACME CA（如 [Let's Encrypt <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org) 或 [ZeroSSL <img src="/old/resources/images/external-link.svg" class="external-link">](https://zerossl.com)）的证书为公共 DNS 名称提供 HTTPS 服务。
  - 示例：`example.com`、`sub.example.com`、`*.example.com`

Caddy 保持所有托管的证书续期，并自动将 HTTP（默认端口 `80`）重定向到 HTTPS（默认端口 `443`）。

**对于本地 HTTPS：**

- Caddy 可能会提示输入密码，以将其唯一根证书安装到您的信任存储中。每个根仅发生一次；您可以随时移除它。
- 任何未信任 Caddy 根 CA 证书的客户端访问站点时都会显示安全错误。

**对于公共域名：**

<aside class="tip">

这些是任何基本生产站点的常见要求，不仅仅是 Caddy。主要区别在于，**在运行 Caddy 之前**要正确设置 DNS 记录，以便它能配置证书。

</aside>


- 如果您的域名的 A/AAAA 记录指向您的服务器，
- 端口 `80` 和 `443` 对外部开放，
- Caddy 可以绑定到这些端口（_或者_这些端口被转发到 Caddy），
- 您的[数据目录](/docs/conventions#data-directory)是可写且持久的，
- 并且您的域名出现在配置中的相关位置，

那么站点将自动通过 HTTPS 提供服务。您无需再做任何其他事情。开箱即用！

由于 HTTPS 使用共享的公共基础设施，作为服务器管理员，您应该了解本页的其余信息，以便避免不必要的问题，在出现问题时进行故障排除，并正确配置高级部署。

## 激活

当 Caddy 知道其提供服务的域名（即主机名）或 IP 地址时，它会隐式激活自动 HTTPS。有多种方式将您的域名/IP 告知 Caddy，具体取决于您运行或配置 Caddy 的方式：

- [Caddyfile](/docs/caddyfile) 中的[站点地址](/docs/caddyfile/concepts#addresses)
- [JSON 路由](/docs/modules/http#servers/routes)中顶层的[主机匹配器](/docs/json/apps/http/servers/routes/match/host/)
- 命令行标志，如 [`--domain`](/docs/command-line#caddy-file-server) 或 [`--from`](/docs/command-line#caddy-reverse-proxy)
- [自动化](/docs/json/apps/tls/certificates/automate/)证书加载器

以下任一情况会阻止自动 HTTPS 的激活（全部或部分）：

- 显式禁用自动 HTTPS（[通过 JSON](/docs/json/apps/http/servers/automatic_https/) 或[通过 Caddyfile](/docs/caddyfile/options#auto-https)）
- 配置中未提供任何主机名或 IP 地址
- 仅监听 HTTP 端口
- 在 Caddyfile 中为[站点地址](/docs/caddyfile/concepts#addresses)添加 `http://` 前缀
- 手动加载证书（除非设置了 [`ignore_loaded_certificates`](/docs/json/apps/http/servers/automatic_https/ignore_loaded_certificates/)）

**特殊情况：**

- 以 `.ts.net` 结尾的域名不会被 Caddy 管理。相反，Caddy 会在握手时自动尝试从本地运行的 [Tailscale <img src="/old/resources/images/external-link.svg" class="external-link">](https://tailscale.com) 实例获取这些证书。这要求 [HTTPS 在您的 Tailscale 账户中已启用 <img src="/old/resources/images/external-link.svg" class="external-link">](https://tailscale.com/kb/1153/enabling-https/)，并且 Caddy 进程必须以 root 身份运行，或者您必须配置 `tailscaled` 以授予您的 Caddy 用户[获取证书的权限](https://github.com/caddyserver/caddy/pull/4541#issuecomment-1021568348)。

## 效果

激活自动 HTTPS 时，会发生以下情况：

- 为[所有符合条件的域名](#主机名要求)获取并续期证书
- HTTP 被重定向到 HTTPS（这使用了 [HTTP 端口](/docs/modules/http#http_port) `80`）

自动 HTTPS 永不会覆盖显式配置，只会增强配置。

如果您已经有一个监听 HTTP 端口的[服务器](/docs/json/apps/http/servers/)，HTTP->HTTPS 重定向路由将在您的路由之后插入（带有主机匹配器），但在用户定义的 catch-all 路由之前。

如果需要，您可以[自定义或禁用自动 HTTPS](/docs/json/apps/http/servers/automatic_https/)；例如，您可以跳过某些域名或禁用重定向（对于 Caddyfile，请使用[全局选项](/docs/caddyfile/options)）。

## 主机名要求

所有主机名（域名）符合完全托管证书的条件，只要它们：

- 非空
- 仅由字母数字、连字符、点和通配符（`*`）组成
- 不以点开头或结尾（[RFC 1034](https://tools.ietf.org/html/rfc1034#section-3.5)）

此外，主机名符合公开信任证书的条件，如果它们：

- 不是本地主机（包括 `.localhost`、`.local`、`.internal` 和 `.home.arpa` TLD）
- 不是 IP 地址
- 在最左侧标签中只有一个通配符 `*`

## 本地 HTTPS

Caddy 自动为所有指定了主机（域名、IP 或主机名）的站点使用 HTTPS，包括内部和本地主机。一些主机既不是公开的（例如 `127.0.0.1`、`localhost`），通常也不符合公开信任证书的条件（例如 IP 地址——您可以为它们获取证书，但仅限某些 CA）。除非禁用，这些站点仍然通过 HTTPS 提供服务。

为了通过 HTTPS 服务非公开站点，Caddy 会生成自己的证书颁发机构（CA）并使用它来签署证书。信任链由一个根证书和一个中间证书组成。叶子证书由中间证书签署。它们存储在 Caddy 的[数据目录](/docs/conventions#data-directory)下的 `pki/authorities/local` 中。

Caddy 的本地 CA 由 [Smallstep 库 <img src="/old/resources/images/external-link.svg" class="external-link">](https://smallstep.com/certificates/) 提供支持。

本地 HTTPS 不使用 ACME，也不执行任何 DNS 验证。它仅在本地机器上工作，并且仅当 CA 的根证书已安装时才受信任。

### CA 根

根私钥使用加密安全的伪随机源唯一生成，并以有限权限持久存储到存储中。它仅在执行签名任务时加载到内存中，之后离开作用域以被垃圾回收。

尽管可以将 Caddy 配置为直接使用根进行签名（以支持不符合标准的客户端），但默认禁用此功能，根密钥仅用于签署中间证书。

首次使用根密钥时，Caddy 会尝试将其安装到系统的本地信任存储中。如果没有权限，它会提示输入密码。此行为可以通过在 Caddyfile 中设置 [`skip_install_trust`](/docs/caddyfile/options#skip-install-trust) 或在 JSON 配置中设置 [`"install_trust": false`](/docs/json/apps/pki/certificate_authorities/install_trust/) 来禁用。如果因为以非特权用户身份运行而失败，您可以运行 [`caddy trust`](/docs/command-line#caddy-trust) 以特权用户身份重试安装。

<aside class="tip">
	只要您的计算机没有受到威胁，并且唯一的根密钥没有泄露，信任 Caddy 的根证书是安全的。
</aside>

在安装 Caddy 的根 CA 后，您将在本地信任存储中看到它，名称为“Caddy Local Authority”（除非您配置了不同的名称）。您可以随时卸载它（[`caddy untrust`](/docs/command-line#caddy-untrust) 命令可以轻松完成）。

请注意，自动将证书安装到本地信任存储仅为方便起见，并不保证有效，尤其是在使用容器或 Caddy 以非特权系统服务运行时。最终，如果您依赖内部 PKI，系统管理员有责任确保 Caddy 的根 CA 正确添加到必要的信任存储中（这超出了 Web 服务器的范围）。

### CA 中间证书

还会生成一个中间证书和密钥，用于签署叶子（单个站点）证书。

与根证书不同，中间证书的生命周期短得多，并会在需要时自动续期。

## 测试

要测试或试验您的 Caddy 配置，请务必将 [ACME 端点](/docs/modules/tls.issuance.acme#ca)更改为暂存或开发 URL，否则您很可能会遇到速率限制，这可能阻止您使用 HTTPS 长达一周，具体取决于您触发的速率限制。

Caddy 的默认 CA 之一是 [Let's Encrypt <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/)，它有一个[暂存端点 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/staging-environment/)，不受相同[速率限制 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/rate-limits/) 的影响：

```
https://acme-staging-v02.api.letsencrypt.org/directory
```

## ACME 挑战

获取公开信任的 TLS 证书需要经过公开信任的第三方权威机构的验证。如今，这一验证过程已通过 [ACME 协议 <img src="/old/resources/images/external-link.svg" class="external-link">](https://tools.ietf.org/html/rfc8555) 实现自动化，可以通过以下三种方式之一（“挑战类型”）执行。

默认启用前两种挑战类型。如果启用了多个挑战，Caddy 会随机选择一种，以避免意外依赖特定挑战。随着时间的推移，它会了解哪种挑战类型最成功，并开始优先选择它，但在必要时会回退到其他可用挑战类型。

### HTTP 挑战

HTTP 挑战对候选主机名的 A/AAAA 记录执行权威 DNS 查询，然后通过 HTTP 在端口 `80` 上请求临时加密资源。如果 CA 看到预期的资源，就会颁发证书。

此挑战要求端口 `80` 对外部可访问。如果 Caddy 无法监听端口 80，则必须将来自端口 `80` 的数据包转发到 Caddy 的 [HTTP 端口](/docs/json/apps/http/http_port/)。

此挑战默认启用，无需显式配置。

### TLS-ALPN 挑战

TLS-ALPN 挑战对候选主机名的 A/AAAA 记录执行权威 DNS 查询，然后通过 TLS 握手在端口 `443` 上请求临时加密资源，其中包含特殊的 ServerName 和 ALPN 值。如果 CA 看到预期的资源，就会颁发证书。

此挑战要求端口 `443` 对外部可访问。如果 Caddy 无法监听端口 443，则必须将来自端口 `443` 的数据包转发到 Caddy 的 [HTTPS 端口](/docs/json/apps/http/https_port/)。

此挑战默认启用，无需显式配置。

### DNS 挑战

DNS 挑战对候选主机名的 `TXT` 记录执行权威 DNS 查询，并查找具有特定值的特殊 `TXT` 记录。如果 CA 看到预期的值，就会颁发证书。

此挑战不需要任何开放端口，请求证书的服务器也不需要对外部可访问。但是，DNS 挑战需要配置。Caddy 需要知道访问域名 DNS 提供商的凭据，以便它可以设置（和清除）特殊的 `TXT` 记录。如果启用了 DNS 挑战，默认禁用其他挑战。

由于 ACME CA 在查找用于挑战验证的 `TXT` 记录时会遵循 DNS 标准，您可以使用 CNAME 记录将回答挑战委派给其他 DNS 区域。这可用于将 `_acme-challenge` 子域名委派给[另一个区域](/docs/caddyfile/directives/tls#dns_challenge_override_domain)。如果您的 DNS 提供商不提供 API，或者不受 Caddy 的 DNS 插件之一支持，这将特别有用。

DNS 提供商支持是社区共同努力的结果。[在我们的 wiki 上了解如何为您的提供商启用 DNS 挑战。](https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148)

## 按需 TLS

Caddy 开创了一项我们称为 **按需 TLS** 的新技术，该技术在第一次需要证书的 TLS 握手期间动态获取新证书，而不是在配置加载时。关键的是，这**不**需要事先在配置中硬编码域名。

许多企业依赖此独特功能，以较低成本扩展其 TLS 部署，并在服务数万个站点时避免运营麻烦。

按需 TLS 在以下情况下非常有用：

- 在启动或重新加载服务器时，您不知道所有域名；
- 域名可能尚未正确配置（DNS 记录尚未设置）；
- 您无法控制域名（例如，它们是客户域名）。

启用按需 TLS 后，您无需在配置中指定域名即可获取证书。相反，当收到一个服务器名称（SNI）的 TLS 握手，而 Caddy 尚未为该名称提供证书时，握手会被挂起，同时 Caddy 获取证书以完成握手。延迟通常只有几秒钟，并且仅第一次握手缓慢。所有未来的握手都很快，因为证书被缓存和重用，续期在后台进行。未来的握手可能会触发证书的维护以保持更新，但如果证书尚未过期，此维护会在后台进行。

### 使用按需 TLS

**按需 TLS 必须启用并加以限制，以防止滥用。**

如果使用 JSON 配置，在 [TLS 自动化策略](/docs/json/apps/tls/automation/policies/)中启用按需 TLS；如果使用 Caddyfile，则在[站点块中使用 `tls` 指令](/docs/caddyfile/directives/tls)启用。

为防止滥用此功能，您必须配置限制。这可以在 JSON 配置的 [`automation` 对象](/docs/json/apps/tls/automation/on_demand/)中完成，也可以在 Caddyfile 的 [`on_demand_tls` 全局选项](/docs/caddyfile/options#on-demand-tls)中完成。限制是“全局”的，不能按站点或按域名配置。主要的限制是一个“询问”端点，Caddy 将向该端点发送 HTTP 请求以询问是否有权限获取和管理握手域名的证书。这意味着您需要某种内部后端，例如，可以查询数据库的账户表，查看客户是否已使用该域名注册。

请注意 CA 颁发证书的速度。如果超过几秒钟，会对用户体验产生负面影响（仅对第一个客户端而言）。

由于其延迟性以及防止滥用所需的额外配置，我们建议仅在您的实际使用案例如上所述时才启用按需 TLS。

[在我们的 wiki 文章中查看更多关于有效使用按需 TLS 的信息。](https://caddy.community/t/serving-tens-of-thousands-of-domains-over-https-with-caddy/11179)

## 错误

Caddy 会尽力在证书管理出现错误时继续运行。

默认情况下，证书管理在后台执行。这意味着它不会阻塞启动或拖慢站点速度。然而，这也意味着服务器在所有证书可用之前就已经在运行。在后台运行允许 Caddy 在长时间内以指数退避进行重试。

如果获取或续期证书时出错，会发生以下情况：

1. Caddy 在短暂暂停后重试一次，以防是意外错误。
2. Caddy 短暂暂停，然后切换到下一个启用的挑战类型。
3. 在尝试了所有启用的挑战类型后，[它会尝试下一个配置的颁发者](#颁发者回退)
   - Let's Encrypt
   - ZeroSSL
4. 在尝试了所有颁发者后，它以指数退避暂停
   - 尝试之间最多间隔 1 天
   - 最长持续 30 天

在使用 Let's Encrypt 重试期间，Caddy 会切换到其[暂存环境 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/staging-environment/) 以避免速率限制问题。这不是完美的策略，但总体有帮助。

ACME 挑战至少需要几秒钟，内部速率限制有助于减少意外滥用。Caddy 在您或 CA 配置的基础上使用内部速率限制，以便您可以向 Caddy 提供一百万个域名，它会逐渐——但尽可能快地——为所有域名获取证书。Caddy 的当前内部速率限制是每个 ACME 账户每 10 秒 10 次尝试。

为避免资源泄漏，配置更改时 Caddy 会中止正在进行的任务（包括 ACME 事务）。虽然 Caddy 能够处理频繁的配置重新加载，但请注意此类操作方面的考虑，并考虑批量处理配置更改以减少重新加载次数，让 Caddy 有机会在后台实际完成证书获取。

### 颁发者回退

Caddy 是第一个（也是目前唯一）支持在其他 CA 无法成功获取证书时进行完全冗余、自动故障转移的服务器。

默认情况下，Caddy 启用两个兼容 ACME 的 CA：[**Let's Encrypt** <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org) 和 [**ZeroSSL** <img src="/old/resources/images/external-link.svg" class="external-link">](https://zerossl.com)。如果 Caddy 无法从 Let's Encrypt 获取证书，它会尝试 ZeroSSL；如果两者都失败，它会退避并在稍后重试。在您的配置中，您可以自定义 Caddy 用于获取证书的颁发者，无论是全局的还是针对特定名称。

## 存储

Caddy 会将公钥证书、私钥和其他资产存储在其[配置的存储设施](/docs/json/storage/)中（如果未配置，则使用默认存储——详细信息请参见链接）。

**使用默认配置时，您需要知道的主要事情是 `$HOME` 文件夹必须是可写且持久的。** 为帮助您进行故障排除，如果指定了 `--environ` 标志，Caddy 会在启动时打印其环境变量。

任何配置使用相同存储的 Caddy 实例将自动共享这些资源，并作为集群协调证书管理。

在任何 ACME 事务之前，Caddy 会测试配置的存储以确保其可写且容量充足。这有助于减少不必要的锁争用。

## 通配符证书

当 Caddy 配置为使用符合条件的通配符名称服务站点时，它可以获取和管理通配符证书。站点名称符合通配符条件仅当其最左侧的域名标签为通配符。例如，`*.example.com` 符合条件，但以下不符合：`sub.*.example.com`、`foo*.example.com`、`*bar.example.com` 和 `*.*.example.com`。（这是 WebPKI 的限制。）

如果使用 Caddyfile，Caddy 会严格按照证书主题名称的站点名称。换句话说，定义为 `sub.example.com` 的站点会导致 Caddy 管理 `sub.example.com` 的证书，而定为 `*.example.com` 的站点会导致 Caddy 管理 `*.example.com` 的通配符证书。您可以在我们的[通用 Caddyfile 模式](/docs/caddyfile/patterns#wildcard-certificates)页面中看到演示。如果您需要不同的行为，[JSON 配置](/docs/json/)可以让您更精确地控制证书主题和站点名称（“主机匹配器”）。

自 Caddy 2.10 起，在自动化通配符证书时，Caddy 将为配置中的单个子域名使用通配符证书。除非显式配置（例如使用 `force_automate`），否则不会为单个子域名获取证书。

通配符证书代表广泛的权限，应仅在您拥有大量子域名以至于管理单个证书会给 PKI 带来压力或导致 CA 强制速率限制时使用，或者隐私权衡值得冒密钥泄露时暴露大量 DNS 区域的风险。请注意，仅通配符证书本身并不能提供隐藏特定子域名的隐私：除非启用加密客户端问候 (ECH)，否则它们仍会在 TLS ClientHello 数据包中暴露。（见下文。）

**注意：** [Let's Encrypt 要求 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/challenge-types/) 使用 [DNS 挑战](#dns-challenge)来获取通配符证书。

## 加密客户端问候 (ECH)

通常，TLS 握手涉及发送 ClientHello，包括服务器名称指示（SNI；要连接的域名），以明文形式发送。这是因为它包含加密握手后连接所需的参数。这当然会将域名（ClientHello 中最敏感的部分）暴露给任何可以窃听连接的人，即使他们不在您的直接物理位置附近。它揭示了当目标 IP 可能服务许多不同站点时您正在连接的服务，这也是某些政府审查互联网的方式。

使用加密客户端问候，客户端可以通过将真正的 ClientHello 包装在“外部”ClientHello 中来保护域名，该外部 ClientHello 建立了解密“内部”ClientHello 的参数。然而，许多移动部件需要完美地协同工作才能实现这一点并提供实际的隐私益处。

首先，客户端需要知道用于加密 ClientHello 的参数或配置。此信息包括公钥和“外部”域名（“公共名称”）等。此配置必须以可靠的方式发布或分发。

理论上，您可以将其写在纸上分发给每个人，但大多数主流浏览器在连接站点时支持查找包含 ECH 参数的 HTTPS 类型 DNS 记录。因此，您需要：(1) 生成 ECH 配置（公钥/私钥对以及其他参数），然后 (2) 创建包含 base64 编码 ECH 配置的 HTTPS 类型 DNS 记录。

或者...您可以让 Caddy 为您完成所有这些工作。Caddy 是第一个也是唯一一个能够自动生成、发布和提供 ECH 配置的 Web 服务器。

一旦 HTTPS 记录发布，客户端在连接到您的站点时需要执行 DNS 查找以获取 HTTPS 记录。通常，DNS 查找是明文进行的，这会损害由此产生的 ECH 握手的安全性，因此浏览器需要使用安全的 DNS 协议，如 DNS-over-HTTPS (DoH) 或 DNS-over-TLS (DoT)。根据浏览器的不同，可能需要手动启用。

一旦客户端安全下载了 ECH 配置，它会使用嵌入的公钥加密 ClientHello，然后连接到您的站点。Caddy 随后解密内部 ClientHello，并继续为您的站点提供服务，而域名永远不会以明文形式出现在线路上。

### 部署注意事项

ECH 是一项细致的技术。即使 Caddy 完全自动化了 ECH，为了获得最大的隐私益处，还需要考虑许多事项。您也应该了解各种权衡。

#### 发布

Caddy 仅当域名已存在记录时才会为其创建 HTTPS 记录。这可以防止破坏可能由通配符覆盖的子域名的 DNS 查找。确保您的站点至少有一个指向您服务器的 A/AAAA 记录。如果您仅使用通配符用于 DNS 记录，那么通配符域名也需要出现在您的 Caddy 配置中。

Caddy 不会为具有 CNAME 记录的域名发布 HTTPS 记录。

#### ECH GREASE

如果您打开 Wireshark，然后在现代版本的主流浏览器（如 Firefox 或 Chrome）中连接到任何站点（即使是不支持 ECH 的站点，甚至禁用了 ECH），您可能会注意到其握手包括 `encrypted_client_hello` 扩展：

![ECH GREASE](/resources/images/ech-grease.png)

此目的使真正的 ECH 握手与明文握手无法区分。如果 ECH 握手看起来与正常握手不同，审查者可以阻止 ECH 握手，而附带损害最小。但如果他们阻止任何带有看似 ECH 扩展的握手，他们基本上会关闭大部分互联网。（目标是提高广泛审查的成本。）

这在排查连接问题时尤其重要。

#### 密钥轮换

与证书密钥一样，长时间使用相同的密钥不是好的做法（甚至可能不安全）。因此，ECH 密钥应定期轮换。与证书不同，ECH 配置没有严格的过期时间。但服务器仍然应该轮换它们。

然而，密钥轮换很棘手，因为客户端需要知道更新的密钥。如果服务器简单地用新密钥替换旧密钥，所有 ECH 握手都会失败，除非立即通知客户端新密钥。但仅仅发布更新的密钥是不够的。现实情况是，DNS 记录有 TTL，解析器会缓存响应等。客户端可能需要几分钟、几小时甚至几天才能查询到更新的 HTTPS 记录并开始使用新的 ECH 配置。

因此，服务器应该在一段时间内继续支持旧的 ECH 配置。不这样做可能会导致服务器名称在 _大规模_ 下以明文形式暴露。Caddy 会时不时地轮换密钥，并在一段时间内支持轮换的密钥，直到最终被移除。

然而，这可能还不够。一些客户端由于各种原因仍然无法获取更新的密钥，每当这种情况发生时，就有暴露服务器名称的风险。因此，需要另一种方式在连接中 _带内_ 向客户端提供更新的配置。这就是外部名称（或公共名称）的作用。

#### 公共名称

“外部”ClientHello 是一个普通的 ClientHello，有两个只有源服务器知道的细微差别：

1. SNI 扩展是假的
2. ECH 扩展是真的

那个“外部”SNI 扩展包含保护您真实域的公共名称。此名称可以是任何内容，但**您的服务器必须对该公共名称具有权威性**，因为 Caddy _会_ 为其获取证书。

如果客户端尝试进行 ECH 连接但服务器无法解密内部 ClientHello，它实际上可以使用 _外部_ ClientHello 完成握手，并使用外部名称的证书。此安全连接严格 _仅_ 用于向客户端发送当前的 ECH 配置；即，它是一个临时的 TLS 连接，仅用于完成初始 TLS 连接。不会传输任何应用程序数据：仅传输 ECH 密钥。一旦客户端获取了更新的密钥，它就可以按预期建立 TLS 连接。

通过这种方式，真实服务器名称保持受保护，不同步的客户端仍然能够连接，这两者都是安全的关键要素。

公共名称可以是您站点的域名之一、子域名，或指向您服务器的任何其他域名。我们建议只选择一个通用名称。例如，Cloudflare 在 `cloudflare-ech.com` 后面服务数百万个站点。这对于增加您的匿名集规模很重要。

公共名称不应为空；即，必须配置公共名称才能使事情正常工作。Caddy 目前不强制执行此要求（以后可能强制执行），但 ECH 规范要求公共名称至少为 1 字节长。一些软件会接受空名称，另一些则不会。这可能导致令人困惑的行为，例如浏览器使用 ECH 但服务器拒绝为无效；或者浏览器不使用 ECH（因为无效），即使配置正确存在于 DNS 记录中。站点所有者有责任确保正确的 ECH 配置和发布，以确保隐私。

#### 匿名集

为了最大化 ECH 的隐私益处，请努力最大化您的 _匿名集_ 的规模。本质上，此集由对外观察者行为相同的客户端服务器组成。其理念是观察者无法轻易缩小/推断客户端可能连接的站点或服务。

在实践中，我们建议为所有站点仅使用一个公共名称。（每个 ECH 配置只有一个公共名称，这意味着在任何给定时间只有一个活动的 ECH 配置。）如果您在集群中运行 Caddy，Caddy 会自动与其他实例共享和协调 ECH 配置，这为您处理了这一点。

极端情况下，这意味着互联网上的每个站点都可以（或应该）位于一个单一的 IP 地址和一个公共名称之后……

#### 集中化

……这引出了我们的下一个话题：集中化。对 ECH 的批评之一是其倾向于促进集中化。它至少通过两种方式做到这一点：(1) 客户端倾向于使用 DoH/DoT 进行 DNS 查找，这会将所有 DNS 查找发送给少数几个提供商，以及 (2) 大规模地最大化匿名集的规模。

当使用 DoH 或 DoT 时，所有 DNS 查找都通过 DoH/DoT 提供商。在客户端和提供商之间，DNS 数据是加密的，但在提供商和 DNS 服务器之间，它是不加密的。全局 DoH/DoT 有效地将所有有价值的明文 DNS 流量汇聚到少数几个容易被观察……或故障的大管道中。

类似地，如果我们真正大规模最大化匿名集，所有站点都将受到单一公共名称的保护，例如 `cloudflare-ech.com`。这对隐私有好处，但整个互联网将受制于 Cloudflare 和那一个域名。现在，扩展到那种程度既不是必要的也不实际，但理论上的含义仍然有效。

我们建议每个组织或个人为所有站点选择一个单一的名称并使用它，在大多数情况下这应该提供足够的隐私。但是，请根据您的具体情况咨询专家，考虑您的个人威胁模型。

#### 子域名隐私

借助 ECH，如果部署得当，现在理论上可以保持子域名的秘密/隐私免受侧信道攻击。

大多数站点不需要这个，因为一般来说，子域名是公开信息。我们建议不要将敏感信息放在域名中。话虽如此……

为了避免将敏感子域名泄露到证书透明度（CT）日志中，请改用通配符证书。换句话说，与其在配置中放入 `sub.example.com`，不如放入 `*.example.com`。（有关重要信息，请参见[通配符证书](#通配符证书)。）

另一个泄露源是 DNSSEC，大多数权威 DNS 服务器默认使用它。通过一种称为“区域遍历”的做法，通过查看 NSEC 记录可以实现子域名枚举，这些记录用于提供经过认证的不存在证明。为此，它们按字母顺序指向下一个可用的子域名，形成所有记录的链表。确保您的域名至少使用 NSEC3，最好使用通配符 CNAME 记录来缓解这种情况。

然后，在 Caddy 中启用 ECH。通配符证书结合 ECH 和通配符 CNAME 记录应该能够正确地隐藏子域名，前提是每个尝试连接到它的客户端都使用 ECH 并具有强大的实现。（您仍然受制于客户端以保护隐私。）

### 启用 ECH

由于功能正常的 ECH 需要将配置发布到 DNS 记录，因此您需要一个为您的 DNS 提供商插入了 [caddy-dns 模块](https://github.com/caddy-dns) 的 Caddy 构建。

然后，使用 Caddyfile，在全局选项中指定您的 DNS 提供商配置，以及您要使用的 ECH 公共名称：

```caddy
{
	dns <provider config...>
	ech example.com
}
```

请记住：

- DNS 提供商模块必须已插入，并且您必须为您的提供商/账户具备正确的配置。
- ECH 公共名称应指向您的服务器。Caddy 会为其获取证书。它不必是您站点的域名之一。

如果使用 JSON，请将这些属性添加到 `tls` 应用：

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

这些配置将启用 ECH 并为您的所有站点发布 ECH 配置。如果您需要自定义行为或拥有高级设置，JSON 配置提供了更大的灵活性。

### 验证 ECH

目前围绕 ECH 的工具仍然不多，因此在撰写本文时，验证其是否工作的最佳和最普遍的方法是使用 Wireshark 并在 ServerName 字段中查找您的公共名称。

首先，启动您的服务器，查看日志中是否提到如“published ECH configuration list”之类的关于您域名的信息。（如果在发布时出现任何错误，请确保您的 DNS 提供商模块支持 [libdns 1.0](https://github.com/libdns/libdns)，如果遇到问题，请在提供商的存储库中提交问题。）Caddy 还应该为公共名称获取证书。

接下来，确保您的浏览器已启用 ECH；这可能需要在浏览器中启用 DoH/DoT。清除浏览器（或系统）的 DNS 缓存也是一个好主意，以确保它会获取新发布的 HTTPS 记录。我们还建议关闭浏览器或至少打开一个新的隐私标签页，以确保它不会重用现有连接。

然后，打开 Wireshark 并开始监听相应的网络接口。当 Wireshark 正在收集数据包时，在浏览器中加载您的站点。之后可以暂停 Wireshark。找到您的 TLS ClientHello，您应该在 ServerName 字段中看到 _公共名称_，而不是您连接的 actual 域名。

请记住：即使未使用 ECH，您可能仍然会看到 `encrypted_client_hello` 扩展。关键指标是 SNI 值。如果 ECH 正常工作，您永远不应在 Wireshark 中以明文形式看到真实的站点名称。

如果遇到 ECH 部署问题，请先在我们的[论坛](https://caddy.community)上提问。如果是 bug，您可以在 GitHub 上[提交问题](https://github.com/caddyserver/caddy/issues)。

### 存储中的 ECH

ECH 配置存储在[数据目录](/docs/conventions#data-directory)中的配置存储模块（默认为文件系统）下的 `ech/configs` 文件夹中。

下一个文件夹是 ECH 配置 ID，它是随机生成的，相对不重要。规范建议使用随机数来帮助缓解指纹/跟踪。

元数据 sidecar 文件帮助 Caddy 跟踪上次发布时间。这可以防止在每次配置重新加载时频繁请求您的 DNS 提供商。如果您必须重置此状态，可以安全地删除元数据文件。但是，这也可能重置密钥轮换的时间。您也可以进入文件并仅清除有关发布的信息。