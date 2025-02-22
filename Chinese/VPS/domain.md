## Applying a LetsEncrypt certificate to a domain

#### Introduction

本文介绍如何使用 docker 来申请和续签一个数字证书，以便 Nginx 服务器能够提供
HTTPS 服务。

注意：本文的任务是申请**单域名的数字证书**，它使用的是 challenge 方式是 HTTP-01.
它有别与**泛域名的数字证书**，它使用的 challenge 方式是 DNS-01.

需要用到的容器有 Nginx 和 certbot。

本地的目录结构如下：

```
.
|-- certbot
|   |-- conf
|   |   |-- live/codenroad.biz/
|   |   |       |-- cert.pem -> ../../archive/codenroad.biz/cert1.pem
|   |   |       |-- chain.pem -> ../../archive/codenroad.biz/chain1.pem
|   |   |       |-- fullchain.pem -> ../../archive/codenroad.biz/fullchain1.pem
|   |   |       `-- privkey.pem -> ../../archive/codenroad.biz/privkey1.pem
|   |   |-- options-ssl-nginx.conf
|   |   |-- privkey.pem
|   |   `-- ssl-dhparams.pem
|   `-- www
|-- docker-compose-initiate.yml
|-- docker-compose.yml
|-- etc
|   |-- conf-initiate.d
|   |   `-- default.conf
|   `-- conf.d
|       `-- default.conf
|-- install.sh
|-- letsencrypt
|   `-- live
|       `-- codenroad.biz
|           |-- fullchain.pem
|           `-- privkey.pem
|-- renew_cert.sh
`-- www
    `-- html
        `-- index.html
```

我把整个过程分为两个阶段：Phase 1 和 Phase 2。

#### Phase 1

这个阶段的任务是新证书的申请。主要有下面三个动作：

- run `docker compose up` with the initiation configuration file
- obtain a certificate using Certbot and store it in a folder on the host system
- run `docker compose down` to finish the phase 1

#### Phase 2

这个阶段有两个任务，一是配置好 HTTPS 相关信息（已经有了数字证书），启动 Nginx 服务器。
二是配置好证书续签所需的参数。

下面我们来详细介绍这两个阶段：

首先，在所有操作之前，我们可以在当前目录保存一个 `.env`
文件记录环境信息，内容如下：

```
EMAIL=winkee01@gmail.com
DOMAIN=codenroad.biz
```

#### Phase 1

执行下列命令

```
docker-compose -f ./docker-compose-initiate.yaml up -d nginx    # 启动 nginx：只侦听 80 端口，conf 和 letsencrypt 目录映射
docker-compose -f ./docker-compose-initiate.yaml up certbot     # 启动 certbot：申请证书；letsencrypt 目录映射
docker-compose -f ./docker-compose-initiate.yaml down           # 关闭所有 container
```

解释：
启动 Nginx 的目的是为了能执行 HTTP-01 验证，配置文件如下：

`etc/conf-initiate.d/default.conf`

```
server {
    listen [::]:80;
    listen 80;
    server_name $DOMAIN;
    location ~/.well-known/acme-challenge {
        allow all;
        root /var/www/certbot;
    }
}
```

`./docker-compose-initiate.yml` 内容如下：

```
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    hostname: nginx
    restart: always
    environment:
      - DOMAIN
    ports:
      - 80:80
    volumes:
      - ./etc/conf-initiate.d:/etc/nginx/conf.d
      - ./certbot/conf/:/etc/nginx/ssl/live/codenroad.biz:ro
      - ./certbot/www:/var/www/certbot/:ro
  certbot:
    container_name: certbot
    image: certbot/certbot:latest
    depends_on:
      - nginx
    command: >-
             certonly --reinstall --webroot --webroot-path=/var/www/certbot
             --email ${EMAIL} --agree-tos --no-eff-email
             -d ${DOMAIN}
    volumes:
      - ./certbot/www/:/var/www/certbot/:rw
      - ./certbot/conf:/etc/letsencrypt/:rw
      - ./certbot/log:/var/log/letsencrypt
```

解释：

- Nginx 侦听 80 端口，以便能够执行 HTTP-01 验证
- `certbot` 依赖于 Nginx，目的是申请证书。
- 注意目录映射，申请的证书存放目录为了以后能被 Nginx 访问到

证书申请成功之后，我们把所有容器停止。

#### Phase 2

到这个阶段的时候，已经申请好了数字证书，我们只需要做好正确的目录映射，把证书信息填写到
Nginx 的配置文件中，然后启动 Nginx 容器即可成功提供 HTTPS 服务。

首先，我们从准备一份 SSL 相关的配置文件（从网上下载）

```
curl -L --create-dirs -o etc/letsencrypt/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
openssl dhparam -out etc/letsencrypt/ssl-dhparams.pem 2048
```

更新 Nginx 的配置文件以便能够支持 HTTPS，如下：
`etc/conf.d/default.conf` (注意：有别于之前的 `etc/conf-initiate.d/default.conf`)

```
server {
    listen 80;

    server_name codenroad.biz www.codenroad.biz;
    server_tokens off;
    root /var/www/html;
    index index.html;

    access_log  /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 default_server ssl;
    server_name codenroad.biz;

    ssl_certificate /etc/nginx/ssl/live/codenroad.biz/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/codenroad.biz/privkey.pem;
    include /etc/nginx/ssl/options-ssl-nginx.conf;
    ssl_dhparam /etc/nginx/ssl/ssl-dhparams.pem;

    root /var/www/html;

    location /.well-known/acme-challenge/ {
        allow all;
        root /var/www/certbot;
    }

    location / {
	    try_files $uri $uri/ =404;
    }
}
```

docker 相关的配置文件

`docker-compose.yml`

```
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    hostname: nginx
    restart: always
    volumes:
      - ./etc/conf.d:/etc/nginx/conf.d       # change
      - ./www/html:/var/www/html             # add
      - ./log:/var/log/nginx                 # add
      - ./certbot/conf/:/etc/nginx/ssl/:ro
      - ./certbot/www:/var/www/certbot/:ro
    ports:
      - 80:80
      - 443:443                              # add

  certbot:
    container_name: certbot
    image: certbot/certbot:latest
    depends_on:
      - nginx
    command: >-
             certonly --reinstall --webroot --webroot-path=/var/www/certbot
             --email ${EMAIL} --agree-tos --no-eff-email
             -d ${DOMAIN}
    volumes:
      - ./certbot/www/:/var/www/certbot/:rw
      - ./certbot/conf:/etc/letsencrypt/:rw
```

有了上述的准备，接下来我们可以启动 Nginx 了。

```
docker-compose -f ./docker-compose.yaml -d up nginx
```

###### 证书更新

当我们需要更新证书的时候，我们可以执行：

```
docker compose -f ./docker-compose.yml run --rm certbot
docker compose -f ./docker-compose.yml restart nginx
```

想要实现自动更新，我们添加 cron job 任务即可。

先编写一个 shell 脚本 renew_cert.sh

```
#!/bin/bash
# cleanup exited docker containers
EXITED_CONTAINERS=$(docker ps -a | grep Exited | awk '{ print $1 }')
if [ -z "$EXITED_CONTAINERS" ]
then
  echo "No exited containers to clean"
else
  docker rm $EXITED_CONTAINERS
fi

# renew certbot certificate
docker compose -f /home/tipman/docker/certbot-nginx/docker-compose.yml run --rm certbot
docker compose -f /home/tipman/docker/certbot-nginx/docker-compose.yml restart nginx
```

添加 cron job 任务

```
sudo crontab -e
```

添加内容：

```
0 3  */60 * *  /home/tipman/docker/certbot-nginx/renew_cert.sh >> /var/log/renew_cert.log 2>&1
```

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号: codeandroad
