## 申请 LetsEncrypt 泛域名证书（使用 Cloudflare DNS）

### 引言

本文将介绍如何使用 Docker Compose 申请和续签一个泛域名数字证书（例如 `*.rawhands.top`），以便 Nginx 服务器提供 HTTPS 服务。

与单域名证书使用 `HTTP-01` 挑战不同，泛域名证书需要使用 `DNS-01` 挑战方式，通过在 DNS 中添加和删除 TXT 记录来验证域名所有权。本教程假设你使用的是 Cloudflare 作为 DNS 服务提供商，并利用 Certbot 的 `dns-cloudflare` 插件与 Cloudflare API 交互。

需要的容器包括 Nginx 和 Certbot（带 `dns-cloudflare` 支持）。本地目录结构如下：

```
.
|-- certbot
|   |-- conf
|   |   |-- live/rawhands.top/
|   |   |       |-- cert.pem
|   |   |       |-- chain.pem
|   |   |       |-- fullchain.pem
|   |   |       `-- privkey.pem
|   |   |-- options-ssl-nginx.conf
|   |   `-- ssl-dhparams.pem
|-- cloudflare
|   `-- credentials.ini
|-- docker-compose.yml
|-- etc
|   `-- conf.d
|       `-- default.conf
|-- install.sh
|-- letsencrypt
|   `-- live
|       `-- rawhands.top
|           |-- fullchain.pem
|           `-- privkey.pem
|-- renew_cert.sh
`-- www
    `-- html
        `-- index.html
```

整个过程分为两个阶段：**Phase 1**（证书申请）和 **Phase 2**（配置 HTTPS 和续签）。

### 前置准备

在开始之前，需要准备以下内容：

1. **Cloudflare API Token**：

   - 登录 Cloudflare 仪表板，转到 "My Profile" > "API Tokens"。
   - 点击 "Create Token"，选择 "Edit zone DNS" 模板。
   - 设置权限：Zone > DNS > Edit。
   - 指定适用于你的域名（例如 `rawhands.top`）。
   - 生成 Token 并保存。

2. **环境变量文件**：
   在项目根目录创建 `.env` 文件，内容如下：

   ```
   EMAIL=your-email@rawhands.top
   DOMAIN=rawhands.top
   WILDCARD_DOMAIN=*.rawhands.top
   ```

3. **Cloudflare 凭证文件**：
   在 `cloudflare` 目录下创建 `credentials.ini` 文件，内容如下：
   ```
   dns_cloudflare_api_token = 你的Cloudflare_API_Token
   ```
   确保文件权限安全，例如运行 `chmod 600 cloudflare/credentials.ini`。

---

### Phase 1：申请泛域名证书

此阶段的目标是使用 Certbot 和 `dns-cloudflare` 插件申请泛域名证书。

#### 配置文件

`docker-compose.yml`（初始阶段直接使用此文件）：

```yaml
services:
  certbot:
    image: certbot/dns-cloudflare:latest
    container_name: certbot
    command: >-
      certonly --dns-cloudflare --dns-cloudflare-credentials /etc/cloudflare/credentials.ini
      --dns-cloudflare-propagation-seconds 60
      --email ${EMAIL} --agree-tos --no-eff-email
      -d ${DOMAIN} -d ${WILDCARD_DOMAIN}
    volumes:
      - ./certbot/conf:/etc/letsencrypt/:rw
      - ./certbot/log:/var/log/letsencrypt:rw
      - ./cloudflare:/etc/cloudflare:ro
```

#### 执行命令

```bash
# 启动 Certbot 容器申请证书
docker-compose up certbot

# 检查证书是否生成
ls -l certbot/conf/live/rawhands.top/
```

#### 解释

- **`dns-cloudflare`**：Certbot 使用此插件与 Cloudflare API 交互，自动添加和删除 TXT 记录。
- **`--dns-cloudflare-propagation-seconds 60`**：等待 DNS 记录传播的时间（Cloudflare 通常很快，但建议设置为 60 秒）。
- **证书存储**：证书会存储在 `certbot/conf/live/rawhands.top/` 目录下。
- **域名参数**：`-d ${DOMAIN} -d ${WILDCARD_DOMAIN}` 表示同时为 `rawhands.top` 和 `*.rawhands.top` 申请证书。

申请成功后，Certbot 会自动清理 TXT 记录，证书存储在本地。

---

### Phase 2：配置 HTTPS 和续签

此阶段的目标是配置 Nginx 使用泛域名证书提供 HTTPS 服务，并设置证书自动续签。

#### 下载 SSL 配置文件

```bash
# 下载 Nginx SSL 配置
curl -L --create-dirs -o certbot/conf/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf

# 生成 Diffie-Hellman 参数
openssl dhparam -out certbot/conf/ssl-dhparams.pem 2048
```

#### 更新 Nginx 配置文件

`etc/conf.d/default.conf`：

```nginx
server {
    listen 80;
    server_name rawhands.top *.rawhands.top;
    server_tokens off;
    root /var/www/html;
    index index.html;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name rawhands.top *.rawhands.top;

    ssl_certificate /etc/nginx/ssl/live/rawhands.top/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/rawhands.top/privkey.pem;
    include /etc/nginx/ssl/options-ssl-nginx.conf;
    ssl_dhparam /etc/nginx/ssl/ssl-dhparams.pem;

    root /var/www/html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### 更新 Docker Compose 文件

`docker-compose.yml`：

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    hostname: nginx
    restart: always
    volumes:
      - ./etc/conf.d:/etc/nginx/conf.d
      - ./www/html:/var/www/html
      - ./log:/var/log/nginx
      - ./certbot/conf:/etc/nginx/ssl:ro
    ports:
      - 80:80
      - 443:443

  certbot:
    image: certbot/dns-cloudflare:latest
    container_name: certbot
    depends_on:
      - nginx
    command: >-
      certonly --dns-cloudflare --dns-cloudflare-credentials /etc/cloudflare/credentials.ini
      --dns-cloudflare-propagation-seconds 60
      --email ${EMAIL} --agree-tos --no-eff-email
      -d ${DOMAIN} -d ${WILDCARD_DOMAIN}
    volumes:
      - ./certbot/conf:/etc/letsencrypt/:rw
      - ./certbot/log:/var/log/letsencrypt:rw
      - ./cloudflare:/etc/cloudflare:ro
```

#### 启动服务

```bash
docker-compose up -d nginx
```

此时，Nginx 将使用泛域名证书提供 HTTPS 服务，支持 `rawhands.top` 和 `*.rawhands.top`。

---

### 证书续签

#### 手动续签

```bash
# 运行 Certbot 续签
docker compose run --rm certbot

# 重启 Nginx 以应用新证书
docker compose restart nginx
```

#### 自动续签

创建一个续签脚本 `renew_cert.sh`：

```bash
#!/bin/bash
# 清理已退出的容器
EXITED_CONTAINERS=$(docker ps -a | grep Exited | awk '{ print $1 }')
if [ -z "$EXITED_CONTAINERS" ]; then
  echo "No exited containers to clean"
else
  docker rm $EXITED_CONTAINERS
fi

# 续签证书并重启 Nginx
docker compose -f /path/to/your/docker-compose.yml run --rm certbot
docker compose -f /path/to/your/docker-compose.yml restart nginx
```

添加 cron 任务：

```bash
sudo crontab -e
```

添加以下内容（每 60 天检查一次，凌晨 3 点执行）：

```
0 3 */60 * * /path/to/your/renew_cert.sh >> /var/log/renew_cert.log 2>&1
```

---

### 注意事项

1. **Cloudflare API Token 安全**：确保 `credentials.ini` 文件权限严格，避免泄露。
2. **DNS 传播时间**：如果遇到验证失败，可以适当增加 `--dns-cloudflare-propagation-seconds` 的值。
3. **测试证书**：可以用 `docker-compose run --rm certbot --dry-run` 进行模拟运行，检查配置是否正确。

---

### 结语

通过上述步骤，你可以使用 Docker Compose 和 Cloudflare DNS 申请并管理泛域名证书。Nginx 将为你的主域名和所有子域名提供 HTTPS 支持，且证书会自动续签。

如果喜欢这篇文章，欢迎关注我的微信公众号：codeandroad！

---
