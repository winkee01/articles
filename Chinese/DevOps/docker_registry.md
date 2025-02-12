# 全面搭建 Docker 私有镜像库，带认证服务器！


## 简介

网上有大量文章介绍如何搭建 Docker 私有镜像库，但多数都比较浅，属于一种临时方案，存在诸多的安全风险。
比如最常见的一种就是使用官方提供的 registry 容器来搭建：

```
$ docker run -d -p 5000:5000 \
    -v /opt/data/registry:/var/lib/registry registry
```

这样，就启动了一个 registry 服务器了，我们可以把镜像名标记为本地服务器名称之后就可以 push/pull 等操作了。
如下：

```
docker tag nginx:latest 127.0.0.1:5000/nginx:latest
docker push 127.0.0.1:5000/nginx:latest
docker image rm 127.0.0.1:5000/nginx:latest
```

最显然的问题就是使用本地作为镜像服务器，除非你为了好玩而不是真正的实际应用，在本机创建私有镜像库就是自己骗自己，电脑炸了什么都没了。

所以你会想到，我不用本机，我用局域网中其他主机来当服务器总可以吧，比如 NAS 机之类的。

虽然避免了单机关于故障问题，但这种使用 `127.0.0.1:5000` IP 地址访问的方式会显得很傻逼，也不方便。

现代人还有谁去输 IP 地址来访问服务的？

有人会想到说在 `/etc/hosts` 中添加一个别名不就搞定了？

```
127.0.0.1 docker.register
```

但是，你有没有想过，这种 HTTP 方式访问一个镜像服务器是不符合我们云原生的安全规范的，明文的方式传递认证信息当然会存在安全隐患。

所以，为了使我们所有云相关的操作有安全保障，我们应该使用 HTTPS 加密连接。

###### 一个最常见的方案是：

我们增加一个网关服务器（反向代理功能），把 registry 服务器当作是内网服务器，既然是内网，通常来说，用 HTTP 协议来访问问题不大。所有来自外部的访问全部使用 HTTPS 协议去请求网关服务器，这里的网关服务器我们可以使用 Nginx 来搭建，我们也常常把它称为反向代理服务器。

下面是 Nginx 的配置代码：

```nginx
upstream registryserver {
  server registry-server.io:5000;
}

map $upstream_http_docker_distribution_api_version $docker_distribution_api_version {
  '' 'registry/2.0';
}

server {
  listen      80;
  server_name home-reg.io reg.router.slab;

  access_log            /var/log/nginx/home-reg.access.log;
  error_log             /var/log/nginx/home-reg.error.log;

  location /{
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
      return 404;
    }
    proxy_set_header        Host $host;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto $scheme;

    proxy_pass              http://registryserver;
    proxy_read_timeout  90;

    add_header 'Docker-Distribution-Api-Version' $docker_distribution_api_version always;
  }
}

server {
  listen 443;
  server_name home-reg.io reg.router.slab;

  client_max_body_size 0; # Disabled to prevent 413's

  ssl_certificate           /etc/nginx/ssl/home-reg.crt;
  ssl_certificate_key       /etc/nginx/ssl/home-reg.key;
  ssl_trusted_certificate   /etc/nginx/ssl/ca.crt;

  # required to avoid HTTP 411: see Issue #1486 (https://github.com/moby/moby/issues/1486)
  chunked_transfer_encoding on;
  access_log            /var/log/nginx/home-reg.access.log;
  error_log             /var/log/nginx/home-reg.error.log;

  location / {
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
      return 404;
    }

    proxy_set_header        Host $host;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto $scheme;

    proxy_pass          http://registryserver;
    proxy_read_timeout  90;

    add_header 'Docker-Distribution-Api-Version' $docker_distribution_api_version always;
 }
}
```

到现在，我们解决了客户端访问网关服务器使用 HTTPS 连接的问题。但是，还有一个问题，如何向镜像仓库（docker registry）进行认证？

**解决方法是：** 搭建一个认证服务器（Authentication Server）！

#### Authentication Server

我们先简单了解一下 Docker 开源的这个 Registry 历史： 

Registry 最早的版本叫 V1， 它不提供登陆 （Authentication）功能，所以，用户只能匿名 pull/push。或者只能通过外部的程序来进行 authentication。

Registry V2 虽然提供 token-based Authentication，但是没有提供生成 token 的 server。比如 basic auth 中，可以用 htpasswd 来生成密码。

而 token 则没有工具可以生成，registry 可以生成，但是前提是你以 basic auth 方式登陆后才可以，否则没办法在没登录的情况下直接生成 token 。

因此，registry v2 常见的就是还是使用 username + passwd 的方式登陆。这种方式不够安全，所以我们接下来探索更安全的 token-based authentication 方法。

#### Token-based Authentication

使用 **Token-based Authentication** 第一个特点就是更加安全，因为 token 具有 pull/push 权限，但没有修改密码的权限，token 泄漏也没有致命危险（比泄漏 username/password 要好）。

Registry 认证的原理都是差不多的，比如 basic auth，registry 通过保存了密码的文件 registry.htpasswd 进行认证；而 token-based auth，registry 则通过 auth server 提供的 certificate 进行认证（auth server 用 private key 对 token 签名，证书则公开给 registry，registry 就可以用证书来验证了）。

这里，我们选择 **docker\_auth** 这个开源的认证服务器

docker\_auth 实现了 docker spec 中的 [token 协议](https://github.com/docker/distribution/blob/master/docs/spec/auth/token.md)

docker\_auth 支持的登陆方法有：

Supported authentication methods（登陆方式）:

- Static list of users
- Google Sign-In (incl. Google for Work / GApps for domain) (documented [here](https://github.com/cesanta/docker_auth/blob/master/examples/reference.yml))
- Github Sign-In
- Gitlab Sign-In
- LDAP bind ([demo](https://github.com/kwk/docker-registry-setup))
- MongoDB user collection
- MySQL/MariaDB, PostgreSQL, SQLite database table
- External program

Supported authorization methods（授权方式）:
- Static ACL
- MongoDB-backed ACL
- MySQL/MariaDB, PostgreSQL, SQLite backed ACL
- External program

项目地址：<https://github.com/cesanta/docker_auth>

<br>

接下来，实战操作：

#### 第一步：创建密钥和证书

**docker\_auth** 需要使用 HTTPS 连接以确保通讯安全，因此我们需要使用 CA 为其签发一个 `server.crt`。

docker\_auth 还需要对 token 进行签名，我们还需要用 CA 再签发一个专门用于对 token 签名的 `token-sign.crt`。同时，这个 `token-sign.crt` 也是需要放入 registry 的，因为它需要对 token 进行验证。

为了简便，我们可以把 `server.crt` 同时作为 `token-sign.crt` 用。

```
openssl req -new -nodes \
    -keyout auth-server.key \
    -out auth-server.csr \
    -subj "/CN=auth-server"

openssl ca -days 391 \
  -in auth-server.csr \
  -out auth-server.crt \
  -cert ca.crt \
  -keyfile ca.key \
  -create_serial \
  -extensions x509_ext \
  -extfile <(cat /etc/ssl/openssl.cnf - <<END
[ x509_ext ]
basicConstraints = CA:false
subjectKeyIdentifier = hash
keyUsage = digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS.1:auth-server.io,IP:10.10.0.3,IP:127.0.0.1
END
)
```

第二步：编写 `docker_compose.yml`

```yml
version: "3.9"
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
      - "443:443"
    restart: always
    environment:
      - REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/var/lib/registry
      - REGISTRY_HTTP_ADDR="0.0.0.0:443"
      - REGISTRY_AUTH=token # 使用 token auth 的方式
      - REGISTRY_AUTH_TOKEN_REALM=https://auth-server.io:5001/auth
      - REGISTRY_AUTH_TOKEN_SERVICE="Docker registry"
      - REGISTRY_AUTH_TOKEN_ISSUER="Auth Service"
      - REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE=/certs/auth-server.crt
      - REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry-server.crt
      - REGISTRY_HTTP_TLS_KEY=/certs/registry-server.key
      #- REGISTRY_HTTP_TLS_CLIENTCAS=' - /certs/domain.ca-bundle' # 如果需要 client HTTPS，请加上这行
      - TZ=Asia/Shanghai
    volumes:
      - /opt/docker/registry/certs:/certs
      - /opt/docker/registry/auth:/auth
      - registry-vol:/var/lib/registry
        #- $PWD/reg_config.yml:/etc/docker/registry/config.yml

  dockerauth:
    image: cesanta/docker_auth
    ports:
      - "5001:5001"
    volumes:
      - ./config.yml:/config/auth_config.yml:ro
      - ./ssl:/ssl
      - ./logs:/logs
    #command: -alsologtostderr=true -log_dir=/logs /config/extAuth.yml
    environment:
      - TZ=Asia/Shanghai
    restart: always

volumes:
  registry-vol:
    driver: local
    driver_opts:
      o: bind
      type: none
      device: /opt/docker/registry/data
```

启动后

#### 第三步：登录 Registry

执行下列命令登陆：

```
docker login https://registry-server.io/v2
```

输入正确的用户名（配置在 docker\_auth 的 `config.yml` 中）后，会出现如下提示：

>
>    Error response from daemon: login attempt to https://registry-server.io/v2/ failed with status: 401 Unauthorized

最后发现是 Issuer 写错了，registry server 和 auth server 配置文件中的 Issuer 必须相同。

**登陆成功：**

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/docker_login_message.jpg)

输入 curl 命令访问 auth server 看看：

```
curl --cacert /opt/docker/registry/certs/ca.crt --user admin:badmin https://auth-server.io:5001/auth | jq
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IkRRTTQ6TjJXVTozRkVKOjNLNUo6TDNVUzpIT0pQOkdZRUk6VVZRQTpaSTY1OkREQ1Q6N002Tjo3WVRZIn0.eyJpc3MiOiJBY21lIGF1dGggc2VydmVyIiwic3ViIjoiYWRtaW4iLCJhdWQiOiIiLCJleHAiOjE2NDAwMTA5NDgsIm5iZiI6MTY0MDAxMDAzOCwiaWF0IjoxNjQwMDEwMDQ4LCJqdGkiOiIzNzIzMzA0MjA1MTAzNDAwMDEwIiwiYWNjZXNzIjpbXX0.Vs19bXQIcwoF7qgxyEpMtmo9wHZtCRNIptz_rnWEzf8nIFdhPCDiC8AO94jNhpGxfpuz9XnksodMu7o1HsQyhhtR4_9m00MJInrk7fq-GBZV90C6S_W5tamHvwh1PM5Qzsg4LAvqKJb9xnXbOk3e-LYDsQglPkRsb_3lkTCbTO9rkb8itBuTNbY45XDbvdPpKa-mSLYt0AaXAKSHxoXOOq00JZU2VaECh2G4g5bvY7SaP30L5HB4MqNQU3m5Nnmtqo7V46Zwm_mU-2rzRdc1UtXEL7M0MwqhvNXE4M7dmNEZLdp8nErJI8fpsEzr9QZWvWFEsEGB8YkiW8uUb0SHDw",
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IkRRTTQ6TjJXVTozRkVKOjNLNUo6TDNVUzpIT0pQOkdZRUk6VVZRQTpaSTY1OkREQ1Q6N002Tjo3WVRZIn0.eyJpc3MiOiJBY21lIGF1dGggc2VydmVyIiwic3ViIjoiYWRtaW4iLCJhdWQiOiIiLCJleHAiOjE2NDAwMTA5NDgsIm5iZiI6MTY0MDAxMDAzOCwiaWF0IjoxNjQwMDEwMDQ4LCJqdGkiOiIzNzIzMzA0MjA1MTAzNDAwMDEwIiwiYWNjZXNzIjpbXX0.Vs19bXQIcwoF7qgxyEpMtmo9wHZtCRNIptz_rnWEzf8nIFdhPCDiC8AO94jNhpGxfpuz9XnksodMu7o1HsQyhhtR4_9m00MJInrk7fq-GBZV90C6S_W5tamHvwh1PM5Qzsg4LAvqKJb9xnXbOk3e-LYDsQglPkRsb_3lkTCbTO9rkb8itBuTNbY45XDbvdPpKa-mSLYt0AaXAKSHxoXOOq00JZU2VaECh2G4g5bvY7SaP30L5HB4MqNQU3m5Nnmtqo7V46Zwm_mU-2rzRdc1UtXEL7M0MwqhvNXE4M7dmNEZLdp8nErJI8fpsEzr9QZWvWFEsEGB8YkiW8uUb0SHDw"
}
```

可见，能得到 token。

#### 总结：

前面我们测试过，Registry 使用 basic auth 的话，也会给用户颁发 token 来认证。

那为什么还要用一个额外的第三方 auth server（docker\_auth）呢，**使用 docker\_auth 到底有什么优势？**

1.  使用 docker\_auth 后，登陆 regsitry server 时使用的不再是 registry 中的 user 了，而是 docker\_auth 中的 user。这样的好处有多个：

*   不会泄漏 registry 中的 user:password。
*   可以在 docker\_auth 创建具有不同权限的 user，有的 user 只能 pull，有的能 pull 也能 push。
*   有 acl 支持不同权限操作

2.  现在访问 registry server 的实际是由 auth server 签发的 token，有过期时间，而且 token 泄漏不会造成致命风险，因此是一种更加安全的访问方式。

3.  docker\_auth 支持许多第三方的 token 机构，比如 github, gitlab, google, 等等。

##### 唯一遗憾的是：

使用 docker\_auth 中的用户登陆 docker，对 docker 来说依然是匿名的，因此 **pull limit 会消耗 anonymose 的额度**。

我们可以用用下面的办法来验证，

```
TOKEN=(curl "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)
curl --head -H "Authorization: Bearer $TOKEN" https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/bearer_token.jpg)

可以看到，匿名用户的额度确实减少了！

下面，我们测试一下用户权限：

使用 test 用户登陆，它没有 push 权限，我们来验证一下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/docker_test_login.jpg)

再使用 admin 登陆试试：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/docker_admin_login.jpg)

#### 思考：

在上面 docer\_auth 的配置中，我们把 user 和 password 都明文（base64)保存到了 `config.yaml`，这样显然是不安全的。有没有办法通过外部程序来管理 user:password 呢？（其实，这也回到了我们怎么保存 docker credential 的问题）

答案是：有的！

docker\_auth 提供了下面几种方式：

- OAuth
  - Google Sign-in
  -   Github
  *   Gitlab
  *   LDAP bind
- Database
  - MongoDB user collection
  - MySQL/MariaDB, PostgreSQL, SQLite
* External program

配置样例：

```yml
# config.yaml for Registry
#  auth:
#    token:
#      realm: "https://127.0.0.1:5001/auth"
#      service: "Docker registry"
#      issuer: "Acme auth server"
#      rootcertbundle: "/path/to/server.pem"

server:
  addr: ":5001"
  certificate: "/ssl/auth-server.crt"
  key: "/ssl/auth-server.key"

token:
  issuer: "Auth Service"  # Must match issuer in the Registry config.
  expiration: 900

users:
  # Password is specified as a BCrypt hash. Use `htpasswd -nB USERNAME` to generate.
  "admin":
    password: "$2y$05$LO.vzwpWC5LZGqThvEfznu8qhb5SGqvBSWY1J3yZ4AxtMRZ3kN5jC"  # badmin
  "test":
    password: "$2y$05$WuwBasGDAgr.QCbGIjKJaep4dhxeai9gNZdmBnQXqpKly57oNutya"  # 123
  "": {}  # Allow anonymous (no "docker login") access.

acl:
  - match: {account: "admin"}
    actions: ["*"]
    comment: "Admin has full access to everything."
  - match: {account: "test"}
    actions: ["pull"]
    comment: "User \"test\" can pull stuff."
  # All logged in users can pull all images.
  - match: {account: "/.+/"}
    actions: ["pull"]
  # Anonymous users can pull "hello-world".
  - match: {account: "", name: "hello-world"}
    actions: ["pull"]
  # Access is denied by default.
```

我们可以把 auth 中的信息写在 `docker-compose.yml` 的环境变量中

##### 需要注意的是：

Authentication 和 HTTPS Connection 是不同的概念，使用 HTTPS 和 mutial TLS 只是保证通信是加密的；但 Registry，还需要认证用户身份，也就是 Authentication。当然，Authentication 也可以用 HTTP 连接，但是非常不安全。

最常见的认证方式是 **username:password** 的方式，但是 Registry V2 可以采用 **Bearer Token**。

Registry V2 提供的 auth 方式有（在 `config.yml` 中配置）：

```yml
auth:
  silly:
    realm: silly-realm
    service: silly-service
  token:
    autoredirect: true
    realm: token-realm
    service: registry-server.io
    issuer: registry-token-issuer  # my.portus
    rootcertbundle: /root/certs/bundle.crt
  htpasswd:
    realm: basic-realm
    path: /path/to/htpasswd
```

docker\_auth 提供了我们生成 bearer token 并使用 bearer token 进行认证的功能。

```yml
version: "3.9"
services:
 registry:
  image: registry:2
  ports:
    - "5000:5000"
    - "443:443"
  restart: always
  environment:
    - REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/data
    - REGISTRY_AUTH=token
    - REGISTRY_AUTH_TOKEN_REALM=https://example.docker.com:5001/auth
    - REGISTRY_AUTH_TOKEN_SERVICE="Docker registry"
    - REGISTRY_AUTH_TOKEN_ISSUER="Auth Service"
    - REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE=/ssl/domain.crt
    - REGISTRY_HTTP_TLS_CERTIFICATE=/ssl/domain.crt
    - REGISTRY_HTTP_TLS_KEY=/ssl/domain.key
    - REGISTRY_HTTP_TLS_CLIENTCAS=' - /certs/domain.ca-bundle'
  volumes:
    - ./ssl:/ssl
    - ./data:/data
 dockerauth:
   image: cesanta/docker_auth
   ports:
     - "5001:5001"
   volumes:
     - ./:/config:ro
     - ./ssl:/ssl
     - ./extensions:/extensions
   #command: -alsologtostderr=true -log_dir=/logs /config/extAuth.yml
   restart: always
```

对比 htpasswd 的 auth 方式：

```yml
version: '3.9'
services:
  registry:
    image: registry:2
    ports:
    - "5000:5000"
    - "443:443"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.password
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /var/lib/registry
      REGISTRY_HTTP_ADDR: 0.0.0.0:443
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/home-registry.crt
      REGISTRY_HTTP_TLS_KEY: /certs/home-registry.key
      REGISTRY_HTTP_TLS_CLIENTCAS: ' - /certs/domain.ca-bundle'
    volumes:
      - /opt/docker/registry/certs:/certs
      - /opt/docker/registry/auth:/auth
      - registry-vol:/var/lib/registry
      - $PWD/config.yml:/etc/docker/registry/config.yml
volume:
    registry-vol:
      driver: local
      driver_opts:
        o: bind
        type: none
        device: /opt/docker/registry/data
```

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。
