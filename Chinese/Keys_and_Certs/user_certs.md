# 全面解析！用户证书的使用，用 Nginx 搭建 HTTPS 服务器。


### 简介

前面的系列文章，我们已经创建了用户证书，这篇文章，我们继续用案例的方式来演示如何使用用户证书来搭建 HTTPS 服务。


### HTTPS 原理介绍

HTTPS 是一种用于在浏览器和服务器之间传输数据的安全协议，它通过加密确保数据在传输中不被第三方窃取或篡改。
数字证书是由权威机构颁发的（如 DigiCert、Let's Encrypt）的电子文件，它包含了网站的公钥和真实身份信息，用来证明是网站的拥有者。

当用户访问一个 HTTPS 网站时，以下步骤确保了通信的安全：
- 证书验证：用户打开浏览器输入网址时，浏览器会自动请求并接收网站的数字证书。浏览器会检查证书是否由可信的证书颁发机构签发，确保真实性。
- 建立加密连接：在验证了服务器的身份后，浏览器和服务器使用证书中的公钥来加密一段“会话密钥”，并将该密钥用于后续的数据加密。
- 数据加密传输：在整个会话过程中，浏览器和服务器使用这个会话密钥加密数据，确保数据只能被双方解读，保护隐私。

可以看到，数字证书不仅帮助验证网站的身份，还提供了“会话密钥”来加密双方的通信内容，避免被第三者窃听，能很好的防止中间人攻击等问题，极大的提高用户访问网站时的安全性。这也是我们建立整个证书链的一个重要目的。


重点：

数字证书验证身份过程中，权威机构（CA）的证书是被我们操作系统已经信任的，网站服务在被浏览时所提供的证书是尚未被信任的。我们使用已信任的 CA 证书去校验未信任的 User 证书，当然，前提是它是由已信任的 CA 所签发。

所以第一步，要确保我们之前所创建的 Root CA 的证书已经被导入到访问者的系统证书库中。

Linux

```
sudo cp rsa_ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

MacOS

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/user_certs_import_in_macos.png)



#### 实战案例

我在服务器中使用 Nginx 搭建一个简单的 echo 服务来验证用户证书的使用。假设服务器是 Ubuntu 系统，我们用 docker 容器来搭建 Nginx 服务

登录服务器

```
ssh <web-server>
```

创建一个目录保存 Nginx 相关的信息：

```
mkdir -pv docker/nginx/etc/conf.d
mkdir -pv docker/nginx/ssl/echo
```

创建 docker-compose 文件：

`docker-compose.yml`

```yml
services:
  echo:
    image: nginx:latest
    container_name: nginx
    restart: always
    volumes:
      - ./etc/conf.d:/etc/nginx/conf.d
      - ./ssl:/etc/nginx/ssl
      - ./www/html:/var/www/html
      - ./log:/var/log/nginx
    ports:
      - 80:80
      - 443:443
```

在 Nginx 配置文件目录中新建一个 echo 服务的配置文件 

`touch etc/conf.d/echo.conf`


```
server {
    listen 80;
    listen [::]:80;

    server_name echo1.example.com echo2.example.com
    server_tokens off;

    access_log   /var/log/nginx/access.log  main;

    location / {
        return 301 https://echo.example.com$request_uri;
    }
}

server {
    listen 443 default_server ssl http2;
    listen [::]:443 ssl http2;

    server_name *.example.com;

    ssl_certificate_key /etc/nginx/ssl/echo/rsa_echo.pem;
    ssl_password_file /etc/nginx/ssl/echo/password.txt
    ssl_certificate /etc/nginx/ssl/echo/fullchain.pem;

    location / {
    }
}
```

注意第二个 server 配置块中 `ssl_certificate` 和 `ssl_certificate_key` 两个字段的配置。

注：

我的三级证书系统是在工作主机（MacOS）上搭建的，这里为了方便演示，我把整个证书目录拷贝到了服务器主机（Ubuntu）上。
在实际场景中，我们通常还是在工作主机上（放在一个安全的位置）保存证书系统，方便管理。


假设当前在服务器的 `$HOME` 目录中，各目录结构如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/docker_nginx_certs_dir_structure.png)


- **`ssl_certificate_key`**


```
cp allcerts/user_certs/echo/rsa_echo.crt docker/nginx/ssl/echo/rsa_echo.pem
```



**ssl_certificate** 字段。


```
cat allcerts/user_certs/echo/rsa_echo.crt allcerts/int_ca/CA/rsa_int_ca.crt > fullchain.pem
mv fullchain.pem docker/nginx/ssl/echo/fullchain.pem
```


解释：

`**ssl_certificate**` 表示的是网站的数字证书，如果签发该证书的 CA 机构已被系统信任（不管该机构是 Root CA 还是 Intermediate CA），那么 ssl_certificate 填该证书名就可以了。如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/fullchain_rootca_usercert.jpg)

但是，这里的 `rsa_echo.crt` 是由 `int_ca` 签发的，而我们并没有把 `int_ca` 证书导入到系统证书库中，系统证书库只信任了 `root_ca`。所以，这里，我们需要把 `rsa_echo.crt` 和 `rsa_int_ca.crt` 两个证书合并，这样，Root CA 才能根据证书链去一一验证这两个证书的合法性。

以此类推，如果中间有多个 Intermediate CA 并且都没有被添加到系统证书库中，那我们应该把这些证书都合并为一个，这样才能得到验证。
如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/fullchain_rootca_intca_usercert.jpg)


这里，我把合并后的证书起名为 `fullchain.pem` 是参考了 Let's Encrypt 的命名方式；也有命名方式，比如 Hashicorp Value 中就命名为 `bundle.pem`。当然，你也可以根据自己的喜好起名，只要别与原名称造成混淆。


另外，由于我们映射了当前的 `./ssl` 的目录到 Nginx 容器中的的 `/etc/nginx/ssl`，所以我们直接把文件拷贝到 `./ssl/echo` 中即可。


至此，配置完成，启动 Nginx 容器：

```
docker compose up -d
```

服务启动没问题之后，我们就可以在浏览器中输入类似 `https://echo.example.com` 这样的网址打开网页了。


![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/echo_service_example_nginx.jpg)


查看网页所使用的证书：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/echo_service_https_certs.png)

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。
