
# Title: 全面解析：OCSP Stapling 服务器搭建！

#### 简介

OCSP Stapling (Online Certificate Status Protocol)，也叫 OCSP 绑定。是一种优化 OCSP 协议的方法。它允许服务器在与客户端的 TLS 握手过程中，将证书的 OCSP 响应直接“绑定”到对客户端的响应中，以向客户端提供证书的实时状态，客户端无需再单独访问 CA 的 OCSP 响应服务器。


我们对比一下传统的 OCSP Responder 和 OCSP Stapling 的工作原理

1. 传统 OCSP：


![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ocsp_responder_working_diagram.jpg)


问题：

- 增加了连接延迟（需要额外查询 OCSP）
- 暴露用户隐私（OCSP 服务器知道用户访问了哪个网站）
- OCSP 服务器可能不可用

2. OCSP Stapling

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ocsp_stapling_working_diagram.jpg)

优点：

- 减少延迟（一次握手就完成）
- 保护隐私（客户端不需要联系 OCSP 服务器）
- 提高可靠性（即使 OCSP 服务器宕机，网站仍可访问）


#### 怎么实现 OCSP Stapling？

由于 OCSP Stapling 是需要网站服务器主动与 OCSP Server 交互的，我们在 Web 服务器的 Nginx 配置中需要重点配置如下信息：

```
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /path/to/ca-chain.crt;
resolver 8.8.8.8 8.8.4.4;  
```

注：
- `ssl_trusted_certificate` 字段是用来验证客户端身份的。Web server 需要验证 OCSP Server 返回的响应。它指定 OCSP 证书的签发机构的证书。
- `resolver` 字段指定了 DNS 解析器，用于解析 OCSP Server 的域名。


Nginx 配置文件


`/etc/nginx/conf.d/echocert.conf`

```nginx
server {
   listen 443 default_server ssl;
   listen [::]:443 ssl;

   server_name echocert.lab;

   access_log /var/log/nginx/echocert_access.log;
   error_log /var/log/nginx/echocert_error.log;

   ssl_certificate_key /etc/nginx/ssl/echocert.lab/rsa_echocert_encrypted.key;
   ssl_password_file /etc/nginx/ssl/echocert.lab/password.txt;
   ssl_certificate /etc/nginx/ssl/echocert.lab/fullchain.pem;

   ssl_stapling on;
   ssl_stapling_verify on;
   ssl_trusted_certificate /etc/nginx/ssl/ca-chain.pem;

   resolver 192.168.64.1 valid=300s;
   resolver_timeout 5;

   ssl_session_timeout 1d;
   ssl_session_cache shared:SSL:50m;
   ssl_session_tickets off;

   ssl_protocols TLSv1.2 TLSv1.3;

   # Only use ciphersuites that are considered modern and secure by Mozilla
   ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';

   # Always use server-side offered ciphersuites
   ssl_prefer_server_ciphers on;

   # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
   # add_header Strict-Transport-Security max-age=15768000;

   location / {
      root /var/www/html;
      try_files $uri /index.html;
   }
}
```

验证配置无误

```
sudo nginx -t
```

这一步，Nginx 会检查 Web 服务器的用户证书中的 AIA 扩展中的 OCSP 域名 `ocsp.servicelab.com` 是否可以访问。
所以，这里需要首先确保 OCSP Server 已经启动，并且 Web Server 所在的服务器中的 `/etc/hosts` 已经映射了域名 `ocsp.servicelab.com`，或者，在 DNS 服务器中（也即配置中的 `resolver`） 中可以正确解析该域名。

否则会出现如下提示：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ocsp_responder_host_not_found.jpg)


一切就绪后，重启 Web 服务器（Nginx）

```
sudo systemctl restart nginx
```

在 Firefox 浏览器中输入 `https://echocert.lab` 访问网站，会看到如下提示：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ocsp_stapling_certificate_revoked.jpg)


不过，由于不同的浏览器 OCSP Stapling 的策略不一样，所以在其他浏览器中访问时，可能依然能正常访问而不是提示证书已被吊销。


我们也可以这样来验证：

```
openssl s_client -connect echocert.lab:443 -status
```

或者

```
openssl s_client -connect 192.168.64.7:443 -status \
                 -servername echocert.lab -CAfile root_ca/CA/rsa_ca.crt \
                 -cert user_certs/echo/rsa_echo.crt \
                 -key user_certs/echo/private/rsa_echo_encrypted.key \
                 -pass pass:$(< user_certs/echo/private/password.txt)'
```

如果 OCSP Stapling 正常工作，会看到类似以下内容：

```
OCSP Response Status: successful (0x0)
Response verify OK
```

Edge, Chrome 和 Safari 可能都要求 Must Staple，所以





#### OCSP Stapling 优势总结

相比于前文介绍的 OCSP Responder 方法，OCSP Stapling 有如下几个优势：

1. **减少网络请求和延迟**：
   - 在没有 OCSP Stapling 的情况下，客户端在验证证书时需要直接连接到 CA 的 OCSP 响应服务器来获取证书状态，这会引入额外的网络延迟。
   - 使用 OCSP Stapling 时，服务器会提前获取并缓存 OCSP 响应，在客户端与服务器握手的过程中直接将该响应发送给客户端。这样，客户端无需额外请求 CA 的 OCSP 服务器，从而加速了连接。

2. **提高隐私性**：
   - 传统 OCSP 需要客户端直接与 CA 的 OCSP 响应服务器交互，这可能暴露用户正在访问的站点信息。
   - 通过 OCSP Stapling，客户端无需向 CA 服务器暴露其访问的站点信息，因为 OCSP 响应由服务器提供，从而保护了用户的隐私。

3. **减少 OCSP 服务器负载**：
   - 通过缓存 OCSP 响应并将其直接发送给客户端，OCSP Stapling 减少了 OCSP 响应服务器的请求压力，特别是当大量用户访问同一站点时，这种方式显著减轻了 CA 的基础设施负载。

4. **避免连接失败**：
   - 在某些情况下，如果 OCSP 服务器不可用或响应超时，客户端可能会出现连接失败的情况。使用 OCSP Stapling 后，服务器预先获取 OCSP 响应并定期更新，这样即便 OCSP 服务器暂时不可用，也不会影响客户端的连接请求。



全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad 。

