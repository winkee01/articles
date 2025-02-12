***本文正在参加[「金石计划」](https://juejin.cn/post/7207698564641996856/ "https://juejin.cn/post/7207698564641996856/")***


# Title: 全面解析！证书吊销列表服务器 CRL Server

#### 简介

前面的文章介绍了证书过期，吊销证书以及生成证书吊销列表（CRL）的知识和操作。 当一个证书已经失效后，访问者怎么知道它失效了呢？

显然，浏览器还需要跟 CA 进行某种形式的沟通才能确定当前证书是否已被吊销。主要有下面几种方式：

-   CRL
-   OCSP Responder
-   OCSP Stapling (Safari, Chrome, Edge)
-   CRLSets (Edge and Chrome)
-   OneCRL and CRLite (Firefox)

每种方式各有优缺点，这篇文章介绍如何通过 CRL 列表来验证证书是否失效。

首先，浏览器会在验证证书的时候，可以直接通过检查其过期时间来知道是否失效。 但并没法知道一个还在有效期内的证书是否被吊销。

这时候，它需要根据证书中提供的 CRL 地址去下载一份 CRL 列表，通过检测当前证书是否在 CRL 列表中，来确认当前证书已被吊销，如果确认，就会发出警告或者拒绝建立 HTTPS 连接。

这里有两个要求：

-   （1）CA 服务器必须提供一个 HTTP 下载链接以供浏览器下载 CRL 列表
-   （2）用户证书中必须提供一个 CRL 下载地址，这个是通过扩展字段来提供的。

我们知道，当 CA 吊销某个用户证书后，需要手动去更新 CRL 列表（执行 `openssl ca -gencrl -out crl/crl.pem`）

这个 `crl.pem` 只是在保存 CA 服务器本地的，浏览器没法访问。因此，CA 服务器需要把它以 HTTP 形式共享出去，这样，网站的访问者才能够下载它，从而能够校验证书是否被吊销。

根据原理，我们可以总结一个大致的流程：

1.  重新生成用户证书（如果之前的证书没有提供 CRL 下载地址的话） 生成用户证书时，必须包含扩展字段（CRL Distribution Points URL），CRL 下载地址就记录在这个扩展字段中
1.  CA 生成 CRL 列表（确保正确的 CRL extension）
1.  Nginx 提供 CRL 文件共享链接（HTTP 链接）

###### 下面介绍详细的操作步骤：

### 第一部分：生成网站用户证书（`echocert.lab`）

#### 1. 配置 [ ca ] 信息

假设我们之前已经生成过 CSR 了，可以直接复用，现在只需要重新配置 ca 以便能包含 CDP 扩展。

现在配置 `[ ca ]` 段落，确保包含扩展字段 **`crlDistributionPoints`**

只有这样，在使用 `openssl ca` 命令签发证书时，证书才会带有 CRL 下载地址。

我们使用中间证书签发用户证书，所以这里的配置文件名是`ica.cnf`

```
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = .
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
private_key       = $dir/private/ca.key
certificate       = $dir/certs/ca.crt
crl               = $dir/crl/ca.crl
crlnumber         = $dir/crlnumber
crl_extensions    = crl_ext
default_days      = 365
default_crl_days  = 30
default_md        = sha256
preserve          = no
policy            = policy_match

[ policy_match ]
countryName             = optional
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional


[ v3_ext_echocert ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
crlDistributionPoints = @crl_info
subjectAltName = @alt_names


[ crl_info ]
URI.0 = http://crl.servicelab.com/crl.pem


[alt_names]
IP = 127.0.0.1
DNS = echocert.lab

[ crl_ext ]
authorityKeyIdentifier = keyid:always
```

###### 注：

-   `[ crl_info ]` 中的 `URI.0` 字段请填写你自己的 CRL 下载链接的地址。

#### 2. 签发用户证书

重新签发带有 CDP 扩展字段的用户证书（如果之前的证书不包含该字段的话）

```
openssl ca -days 365 -notext -batch \
    -config ica.cnf \
    -extensions v3_ext_echocert \
    -cert CA/rsa_int_ca.crt \
    -keyfile CA/private/rsa_encrypted.key \
    -passin file:CA/private/password.txt \
    -in ../user_certs/echocert.lab/rsa_echocert.csr \
    -out ../user_certs/echocert.lab/echocert.crt
```

###### 检查证书

```
cd user_certs/echocert.lab
openssl x509 -text -noout -in echocert.crt
openssl x509 -text -noout -in echocert.crt | grep -A2 "CRL Distribution Points"
```

输出样式：

```
    X509v3 CRL Distribution Points:
        URI:
            http://crl.servicelab.com/crl.pem
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/echocert_cert_valid.jpg)


#### 3. 配置并启动 Nginx 服务

`/etc/nginx/conf.d/echocert.conf`

```
server {
    listen 80;
    listen [::]:80;

    server_name echocert.lab
    server_tokens off;

    location / {
        return 301 https://echocert.lab$request_uri;
    }
}

server {
    listen 443 default_server ssl;
    server_name echocert.lab;

    ssl_certificate_key /etc/nginx/ssl/echocert.lab/rsa_echocert_encrypted.key;
    ssl_password_file /etc/nginx/ssl/echocert.lab/password.txt;
    ssl_certificate /etc/nginx/ssl/echocert.lab/fullchain.pem;

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

重启 Nginx

```
sudo systemctl restart nginx
```

### 第二部分：CRL 及验证


#### 1. 生成 CRL

```
openssl ca -config ica.cnf -passin file:CA/private/password.txt -gencrl -out crl/crl.pem
```

查看 CRL

```
openssl crl -noout -text -in crl/crl.pem | grep -B1 "Revocation Date:"
```

#### 2. 验证证书是否在 CRL 中

注意，这里我们还需要把 `root_ca` 与 `int_ca` 合并起来，否则会提示找不到 `int_ca` 的 issuer。

```
cat ../root_ca/CA/rsa_ca.crt CA/rsa_int_ca.crt > ca-chain.crt
```

```
openssl verify -crl_check \
    -CAfile ca-chain.crt \
    -CRLfile crl/crl.pem \
    ../user_certs/echocert.lab/echocert.crt
```

输出结果：

```
../user_certs/echocert.lab/echocert.crt: OK
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/echocert_crl_verify_valid.jpg)

#### 3. 撤销证书

现在，我们把 `echocert.crt` 证书撤销，

```
openssl ca -config ica.cnf -passin file:CA/private/password.txt -revoke newcerts/08.pem
```

###### 别忘了更新 CRL

```
openssl ca -config ica.cnf -passin file:CA/private/password.txt -gencrl -out crl/crl.pem
```

更新 CRL 的操作是至关重要的，我们可以写个 bash 脚本把这个动作添加到定时任务中：

```
cat > /usr/local/bin/update-crl.sh << 'EOL'
#!/bin/bash
openssl ca -config ica.cnf -passin file:CA/private/password.txt -gencrl -out /var/www/crl/crl.pem
chmod 644 /var/www/crl/crl.pem
EOL

chmod +x /usr/local/bin/update-crl.sh

(crontab -l 2>/dev/null; echo "0 0 * * * /usr/local/bin/update-crl.sh") | crontab -
```

检查 `index.txt`

```
grep "Echo" index.txt
```

#### 4. 再次验证证书是否失效

```
openssl verify -crl_check \
    -CAfile ca-chain.crt \
    -CRLfile crl/crl.pem \
    ../user_certs/echocert.lab/echocert.crt
```

输出结果：

```
C=CN, ST=Unknown, O=Echo Certificate, CN=Echo Certificate
error 23 at 0 depth lookup: certificate revoked
error ../user_certs/echocert.lab/echocert.crt: verification failed
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/echocert_crl_verify_revoked.jpg)

###### 总结：

按照上述步骤，我们可以通过 CRL 来验证某用户证书是否被吊销。

接下来，我们用 Nginx 搭建一个 HTTP 服务，提供 CRL 下载链接。


### 第三部分：Nginx 搭建 CRL 下载链接

#### 1. Nginx 配置

由于这里只是搭建 HTTP 服务，我们不用为此服务创建用户证书。如果你想要使用 HTTPS，那么还需要为该服务创建证书。

`/etc/nginx/conf.d/crl.conf`

```
server {
    listen 80;
    server_name crl.servicelab.com;

    # Optional SSL configuration
    # listen 443 ssl;
    # ssl_certificate /etc/nginx/ssl/crl_server.crt;
    # ssl_certificate_key /etc/nginx/ssl/crl_server.key;

    access_log /var/log/nginx/crl_access.log;
    error_log /var/log/nginx/crl_error.log;

    location / {
        root /var/www/crl;
        autoindex on;
        
        # Configure proper MIME type for CRL files
        types {
            application/x-pkcs7-crl crl;
        }
        
        # Add headers for caching
        add_header Cache-Control "public, max-age=3600";
        
        # Enable compression
        gzip on;
        gzip_types application/x-pkcs7-crl;
    }
}
```

#### 2. 设置合适的权限

```
chmod 755 /var/www/crl
chmod 644 /var/www/crl/crl.pem
```

#### 3. 重启 Nginx

```
nginx -t
systemctl restart nginx
```

#### 4. 下载 CRL 进行验证

```
curl -v http://crl.servicelab.com/crl.pem -o test.crl
```

输出结果：

```
openssl crl -in test.crl -text -noout
```

既然 `crl.pem` 文件已经成功下载，我们可以像上面一样执行 `openssl verify -crl_check` 命令去手动验证。

通过 CRL 方式去验证证书是否失效有个缺点是很容易文件过大，这会极大的影响访问效率，而且，实时性也不高。因此，现代浏览器通常都不采用这种方法，只是把它作为备用方法。

后面的文章我们会继续介绍更现代的方法 **OCSP Stapling**。



全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 deliverit。
