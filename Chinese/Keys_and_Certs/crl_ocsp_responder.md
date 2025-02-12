
#### 简介

本文继续介绍另一种验证证书有效性的方法：OCSP，Online Certificate Status Protocol

在线证书状态协议，是一个用于获取 X.509 数字证书撤销状态的作为证书吊销列表（CRL）替代品的网络协议，
它比传统的 CRL (Certificate Revocation List) 更高效。OCSP 能够实时提供证书的状态信息，允许客户端在不下载整个吊销列表的情况下快速验证证书的有效性。


#### OCSP 的工作原理
OCSP 的工作流程包括 客户端（请求者） 和 OCSP 响应者（通常是由 CA 提供的 OCSP 服务器） 之间的交互：

##### 1. 客户端请求：

- 当客户端（例如浏览器）需要验证服务器的证书状态时，它会构建一个 OCSP 请求，包含需要验证的证书的 序列号 以及该证书的 发行者（CA）信息。
- OCSP 请求是一个简单的 HTTP GET 或 POST 请求，内容为证书的序列号，遵循 OCSP 标准规范。


##### 2. OCSP 响应者验证：

- OCSP 响应者接收到请求后，会在 CA 的数据库中查找该证书的状态。
- 响应者会检查证书的状态，通常有三种可能的结果：
    - good：表示证书仍然有效，未被吊销。
    - revoked：表示证书已被吊销。
    - unknown：表示 OCSP 响应者无法找到该证书的信息，可能是因为该证书不是由此 CA 颁发的。

##### 3. 生成 OCSP 响应：

- OCSP 响应者会生成一个 OCSP 响应，包含证书的状态和有效性信息。
- OCSP 响应会使用 CA 或 OCSP 响应者的私钥进行签名，以确保响应的真实性。


##### 4. 客户端验证 OCSP 响应：

- 客户端收到 OCSP 响应后，会验证该响应的签名，确保它是由可信的 OCSP 响应者签名的。
- 如果响应为 good，则客户端认为证书有效并继续建立连接；如果为 revoked，则会终止连接。

#### OCSP 工作流程示意图

- 客户端发出请求：客户端发送 OCSP 请求，包含证书序列号和发行者信息。
- OCSP 响应者查找证书状态：OCSP 响应者查询证书的状态（good、revoked 或 unknown）。
- 生成并签名响应：OCSP 响应者生成响应并用私钥签名，保证响应的真实性。
- 客户端验证响应：客户端接收并验证响应，决定是否接受证书。


![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ocsp_service_flow.jpg)

OCSP 有两种方法实施：

- OCSP Responder
- OCSP Stapling

本文介绍第一种 OCSP Responder。



#### AIA 扩展（Authority Information Access)

想要实现 OCSP，用户证书必须包含这个 AIA 扩展，它主要包含两个重要信息：

- **CA Issuers** 这是一个指向 CA 证书的 URL 链接，这个 CA 证书是当前用户证书的签发者。
- **OCSP Responder** 这也是一个 URL 链接，它指向一个可以提供实时查询证书吊销状态的 OCSP 服务器。


###### 一个包含了 AIA 扩展的例子：

```
Authority Information Access:
    CA Issuers - URI:http://ca.servicelab.com/ca.crt
    OCSP - URI:http://ocsp.servicelab.com
```


为什么必须需要 **CA Issuer** 和 **OCSP Responder** 这两个字段？

首先，CA server 作为 OCSP Responder 要相应客户端的 OCSP request，当然要提供一个 API 查询接口。其次，客户端拿到 OCSP response 时，怎么确认就是 CA 发出的，未经篡改的呢？这就要求 OCSP Responder 提供一个证明自己是自己的证书，而这个证书是由 CA 所签发的，而这个 CA 证书有可能并没有内置到客户端的证书库中，所有，这里的 CA Issuers 就是提供这个 CA 证书的下载地址。客户端拿到这个 CA 证书后就可以用它来验证 OCSP Responder 的身份。

因此，在我们搭建一个 OCSP Responder 之前，我们需要先为它创建一个用户证书。



### 第一部分：创建 OCSP Responder 用户证书

我们在目录 `ocsp_responder` 目录中操作：

OCSP Responder 证书的流程如下：

#### 1、创建私钥

#####（1）创建私钥

```
openssl rand -base64 32 > private/password.txt
chmod 600 private/password.txt

openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out private/rsa_ocsp.key
openssl pkcs8 -topk8 -v2 aes256 -iter 100000 -passout file:private/password.txt \
    -in private/rsa_ocsp.key -out private/rsa_ocsp_encrypted.key
shred -u private/rsa_ocsp.key

```

#####（2）检查私钥

```
openssl asn1parse -in private/rsa_ocsp_encrypted.key
```


#### 2、创建 CSR

#####（1）CSR 配置信息

`req_csr.conf`

```
[ req ]
default_bits       = 4096
default_keyfile    = rsa_encrypted.key
distinguished_name = req_distinguished_name
x509_extensions    = v3_ocsp
prompt             = no

[ req_distinguished_name ]
C  = US
ST = LA
O  = OCSP
CN = ocsp.servicelab.com

[ v3_ocsp ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature
extendedKeyUsage = OCSPSigning
subjectKeyIdentifier = hash
subjectAltName = @alt_names

[alt_names]
DNS.1 = ocsp.servicelab.com
```


#####（2）创建 CSR

```
openssl req -new \
    -passin file:private/password.txt \
    -config req_csr.cnf \
    -reqexts v3_ocsp \
    -key private/rsa_ocsp_encrypted.key \
    -out rsa_ocsp.csr
```


#####（3）检查 CSR

```
openssl req -text -noout -in rsa_ocsp.csr 
```

#### 3、签发证书

使用 `int_ca` 签发 ocsp 用户证书

#####（1）ca 配置信息

```
[ ca ]
default_ca              = CA_default

[ CA_default ]
dir                     = .
certs                   = $dir/certs
crl_dir                 = $dir/crl
database                = $dir/index.txt
new_certs_dir           = $dir/newcerts
private_key             = $dir/CA/private/rsa_encrypted.key
certificate             = $dir/CA/rsa_int_ca.crt
serial                  = $dir/serial
crlnumber               = $dir/crlnumber
default_md              = sha256
default_days            = 3650
default_crl_days        = 30
policy                  = policy_match

copy_extensions         = copy

[ policy_match ]
countryName             = match
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
```

###### 注：

使用 ca 签发 OCSP 证书，我们这里直接使用 `copy_extensions` 即可。

#####（2）签发 OCSP 用户证书

```
openssl ca -days 365 -notext -batch \
    -config ica.cnf \
    -cert CA/rsa_int_ca.crt \
    -keyfile CA/private/rsa_encrypted.key \
    -passin file:CA/private/password.txt \
    -in ../user_certs/ocsp/rsa_ocsp.csr \
    -out ../user_certs/ocsp/rsa_ocsp.crt
```

#####（3）检查 OCSP 用户证书

```
cd user_certs/ocsp
openssl x509 -text -noout -in rsa_ocsp.crt
openssl x509 -text -noout -in rsa_ocsp.crt | grep -A1 "X509v3 Extended Key Usage:"
```


### 第二部分：创建带 AIA 扩展的网站用户证书

像之前一样，我们以 `echocert.lab` 网站为例，现在，我们使用了 OCSP 方式来验证证书有效性，所以必须要为该网站证书添加 AIA 扩展。

我们可以复用之前的 CSR 文件，只需要在 [ ca ] 中再重新指定带 AIA 字段的扩展信息即可。

在 `int_ca` 目录中添加一个配置文件 `v3_ext_echocert.cnf`，内容如下：

```
[ v3_ext_echocert ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

crlDistributionPoints = @crl_info
authorityInfoAccess = @issuer_info

[alt_names]
IP = 127.0.0.1
DNS.1 = local.domain
DNS.2 = echocert.lab

[ crl_info ]
URI.0 = http://crl.servicelab.com/crl.pem

[ issuer_info ]
caIssuers;URI.0 = http://ca.servicelab.com/ca.crt
OCSP;URI.0 = http://ocsp.servicelab.com
```

##### 证书签名

```
openssl ca -days 365 -notext -batch \
    -config <(cat ica.cnf v3_ext_echocert.cnf)\
    -extensions v3_ext_echocert \
    -cert CA/rsa_int_ca.crt \
    -keyfile CA/private/rsa_encrypted.key \
    -passin file:CA/private/password.txt \
    -in ../user_certs/echocert.lab/rsa_echocert.csr \
    -out ../user_certs/echocert.lab/echocert.crt
```

##### 检查证书

```
cd user_certs/echocert.lab
openssl x509 -text -noout -in echocert.crt
openssl x509 -text -noout -in echocert.crt | grep -A2 "Authority Information Access"
```

##### 更新证书到 `echocert.lab` 服务

```
cat user_certs/echocert.lab/echocert.crt int_ca/CA/rsa_int_ca.crt > fullchain.pem
```

把 fullchain.pem 拷贝到 Web 服务器

```
scp fullchain.pem user_name@web_server:./
```

登录 Web 服务器

```
ssh user_name@web_server
mv fullchain.pem /etc/nginx/ssl/fullchain.pem
```

重启 Nginx 

```
sudo systemctl restart nginx
```

在浏览器中打开网站，查看证书：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/echocert_aia_extension_info.jpg)


现在 OCSP Responder 的证书和 `echocert.lab` 网站的证书都准备好了。我们需要为 OCSP Responder 搭建一个 HTTP 服务，以便能响应网站访问者（浏览器）的 OCSP request。


同样，我们使用 Nginx 来搭建这个 OCSP Server。

### 第三部分：搭建 OCSP Server

整体架构是先使用 OpenSSl 启动一个本地的 OCSP Server，然后 Nginx 服务器作为反向代理，提供外界对 OCSP 的访问。

#### 1、使用 OpenSSl 启动 OCSP Server

OpenSSl 可以启动一个本地的 OCSP Server，如下：

```
openssl ocsp -port 2560 \
             -index index.txt \
             -CA CA/rsa_int_ca.crt \
             -rkey ../user_certs/ocsp/private/rsa_ocsp_encrypted.key \
             -passin file:../user_certs/ocsp/private/password.txt \
             -rsigner ../user_certs/ocsp/rsa_ocsp.crt \
             -out ../user_certs/ocsp/ocsp_responder.log \
             -ndays 7
```

注：

可以看到，ocsp 是通过检查 `index.txt` 来确定证书是否被吊销的。


#### 2、Nginx


配置 OCSP Rensponder 服务

`/etc/nginx/conf.d/ocsp.conf`

```
server {
    listen 80;
    server_name ocsp.servicelab.com;

    access_log /var/log/nginx/ocsp_access.log;
    error_log /var/log/nginx/ocsp_error.log;

    location / {
        proxy_pass http://192.168.64.1:2560;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        add_header Cache-Control "public, max-age=3600";
    }
}

```

注：
这里为了方便，暂时没有添加 OCSP 的证书。我在后面会补充。


配置 Issuer 服务，用于提供 Issuer CA 的证书下载链接

`/etc/nginx/conf.d/issuer.conf`

```
server {
    listen 80;
    server_name ca.servicelab.com;

    access_log /var/log/nginx/issuer_access.log;
    error_log /var/log/nginx/issuer_error.log;

    location / {
        root /var/www/issuer;
        add_header Content-Type "application/x-pem-file";
        add_header Cache-Control "public, max-age=3600";
    }
}

```


重启 Nginx

```
sudo systemctl restart nginx
```



#### 3. 测试 OCSP Server 的响应

```
openssl ocsp -issuer ../../int_ca/CA/rsa_int_ca.crt \
    -CAfile ../../root_ca/CA/rsa_ca.crt \
    -cert echocert.crt \
    -url http://ocsp.servicelab.com \
```

输出结果：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/echocert_ocsp_verify_valid.jpg)

注：
- 由于 OCSP 证书是由 `int_ca` 签发的，而 `int_ca` 并没有添加到系统信任的证书库中去，因此不是 Trusted CA Certificate。所以，我们这里要用 `-CAfile` 选项来指定 `int_ca` 的签发者，它必须是一个受系统信任的 CA。
当然，如果我们已经把 `int_ca` 添加为受系统信任，那这里就可以省略 `-CAfile`。
- 这里通过 `-issuer` 选项手动指定了 issuer 证书；并没有使用 AIA 扩展中的 issuer URL。


#### 4. 补充完善

前面我们用命令的方式启动了一个 OpenSSL 本地的 OCSP Server，为了方便启动和停止，我们可以创建一个系统服务。


`/etc/systemd/system/ocsp-responder.service`

```
[Unit]
Description=OpenSSL OCSP Responder
After=network.target

[Service]
Type=simple
User=nginx
Group=nginx
ExecStart=/usr/bin/openssl ocsp -index /etc/ssl/CA/index.txt \
    -port 2560 \
    -rsigner /etc/ssl/ICA/ocsp.crt \
    -rkey /etc/ssl/ICA/private/rsa_ocsp_encrypted.key \
    -passin file:/etc/ssl/ICA/private/password.txt \
    -CA /etc/ssl/CA/certs/rsa_ca.crt \
    -text \
    -out /var/log/ocsp/ocsp.log
Restart=always
WorkingDirectory=/etc/ssl/ocsp

[Install]
WantedBy=multi-user.target
```
注：
目录信息请根据本地实际情况填写。


#### 5. Nginx 服务反向代理 OCSP Server

`/etc/nginx/conf.d/ocsp.conf`

```
server {
    listen 80;
    listen 443 ssl;
    server_name ocsp.servicelab.com;

    ssl_certificate /etc/nginx/ssl/ocsp_server.crt;
    ssl_certificate_key /etc/nginx/ssl/ocsp_server.key;
    ssl_password_file /etc/nginx/ssl/password.txt;

    access_log /var/log/nginx/ocsp_access.log;
    error_log /var/log/nginx/ocsp_error.log;

    # OCSP specific settings
    location / {
        proxy_pass http://192.168.64.1:2560;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # OCSP specific headers
        proxy_set_header Content-Type application/ocsp-request;
        
        # Timeout settings
        proxy_connect_timeout 60;
        proxy_send_timeout 60;
        proxy_read_timeout 60;
        
        # Buffer settings
        proxy_buffering off;
        proxy_request_buffering off;
        
        # Cache settings
        proxy_cache off;
        
        # Security headers
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    }
}
```

重启 Nginx

```
sudo systemctl restart nginx
```


接下来，我们把证书吊销，再来验证 OCSP Server 的响应。


##### 吊销 `echocert.lab` 证书

```
openssl ca -config ica.cnf -passin file:CA/private/password.txt -revoke newcerts/0B.pem
openssl ca -config ica.cnf -passin file:CA/private/password.txt -gencrl -out crl/crl.pem
```


##### 测试响应：

```
openssl ocsp -issuer ../../int_ca/CA/rsa_int_ca.crt \
    -CAfile ../../root_ca/CA/rsa_ca.crt \
    -cert echocert.crt \
    -url http://ocsp.servicelab.com \
```

输出结果：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/echocert_ocsp_verify_revoked.jpg)


##### 我们可以把测试命令封装到脚本中，方便保存和后续调用：


`/usr/local/bin/check-ocsp.sh`

```bash
#!/bin/bash

# Check if OCSP responder is running
if ! pgrep -f "openssl ocsp" > /dev/null; then
    systemctl restart ocsp-responder
    echo "OCSP responder restarted at $(date)" >> /var/log/ocsp/monitor.log
fi

# Test OCSP response
openssl ocsp -issuer /etc/ssl/ICA/certs/ca.crt \
    -cert /etc/ssl/CA/certs/ocsp.crt \
    -url http://127.0.0.1:8888 \
    -resp_text > /dev/null 2>&1

if [ $? -ne 0 ]; then
    systemctl restart ocsp-responder
    echo "OCSP responder restarted due to failed test at $(date)" >> /var/log/ocsp/monitor.log
fi
```

添加可执行权限

```
chmod +x /usr/local/bin/check-ocsp.sh
```

#### 定时检查 ocsp

```
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/check-ocsp.sh") | crontab -
```


全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad 。