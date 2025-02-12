
### ESNI, Encrypted SNI

**ESNI** 是一种安全特性，用于在 TLS 握手中加密 **服务器名称指示（SNI）** 字段，帮助保护客户端请求的域名不被网络中的第三方看到。在没有加密 SNI 的情况下，SNI 字段以明文形式传输，这意味着中间人可以看到客户端要连接的具体网站。

**ESNI** 现已基本被 **加密客户端问候（Encrypted Client Hello, ECH）** 所取代。ECH 是 TLS 1.3 引入的更全面的功能，它加密整个 ClientHello 消息，其中包含 SNI 和其他敏感信息。ECH 目前还在发展中，尚未完全被所有客户端和服务器采纳。

### ESNI/ECH 的工作原理

- **ESNI**：仅加密 TLS ClientHello 消息中的 SNI 字段，使得中间人难以看到客户端要连接的具体服务器。
- **ECH**：加密整个 ClientHello 消息，包括 SNI 和其他敏感信息，为隐私保护提供更强的防护。

### 当前对 ECH/ESNI 的支持

对 ECH/ESNI 的支持目前还比较有限，主要是实验性支持：

- **浏览器**：Firefox 和 Chrome 正在测试对 ECH 的支持。Firefox 提供了实验性 ECH 支持，可以在一些特定配置中启用。
- **服务器**：Nginx 和 Apache 目前不原生支持 ECH。但是，**Cloudflare** 和其他一些 CDN 服务提供商已经在其基础设施中实现了 ECH。

如果你希望使用 ECH（或 ESNI）来保护隐私，目前的实现方式通常是在 **支持 ECH 的 CDN 服务商**（例如 Cloudflare）上进行基于 DNS 的配置。

### 在 Cloudflare 上启用加密客户端问候（ECH）

Cloudflare 提供对 ECH 的支持（前身为 ESNI）。以下是在 Cloudflare 上启用 ECH 的步骤：

1. **在 Cloudflare 启用 ECH**：
   - 登录 Cloudflare 仪表盘。
   - 前往 **SSL/TLS > 边缘证书（Edge Certificates）**。
   - 向下滚动到 **TLS 客户端认证（TLS Client Auth）** 部分（或其他与 ECH 相关的部分），启用 **加密客户端问候（Encrypted Client Hello）**。

2. **为 ECH 配置 DNS**：
   - Cloudflare 会自动通过 DNS 为使用其 CDN 的域发布 ECH 密钥，无需手动配置，因为 Cloudflare 会管理密钥分发。
   - 支持 ECH 的客户端（如支持该功能的浏览器）会使用 ECH 密钥加密 ClientHello 消息。

### Firefox 的实验性 ECH 支持

在 Firefox 中测试 ECH 支持的步骤如下：

1. 打开 Firefox，并在地址栏输入 `about:config`。
2. 搜索 `network.dns.echconfig.enabled`，将其设置为 **true**。
3. 搜索 `network.dns.use_https_rr_as_altsvc`，将其设置为 **true**。

启用这些设置后，Firefox 将支持 ECH 和 HTTPS DNS（DoH），允许其在访问支持 ECH 的服务器（如 Cloudflare）时使用 ECH。

### 总结

- **ESNI** 已基本被 **ECH** 替代。
- **Cloudflare** 为其平台上的域提供 ECH 支持，并通过 DNS 自动管理密钥分发。
- **Nginx 和 Apache** 目前尚不原生支持 ECH/ESNI，但可以通过使用 Cloudflare 等 CDN 服务商来为域启用 ECH。
- **Firefox** 提供实验性 ECH 支持，可以在 `about:config` 中启用相关设置。

目前，若要实际使用 ESNI/ECH，最好的方式是选择像 Cloudflare 这样支持 ECH 的 CDN 服务商，直到更多客户端和服务器能够原生支持该功能。