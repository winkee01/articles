没问题，既然你使用的是阿里云（aliyun.com）提供的 DNS 服务，我们可以调整之前的教程，使用阿里云 DNS API 来自动完成泛域名证书的申请和续签。Certbot 本身支持通过插件与多种 DNS 提供商交互，虽然目前没有官方的 `dns-aliyun` 插件，但我们可以通过以下方式实现：

1. 使用 Certbot 的手动模式结合脚本调用阿里云 DNS API。
2. 或者使用更适合自动化阿里云 DNS 的工具，例如 `acme.sh`，它原生支持阿里云 DNS。

考虑到你希望通过 Docker Compose 实现类似单域名证书的流程，我会提供一个基于 `acme.sh` 和 Docker Compose 的完整教程，结合阿里云 DNS API 来申请和续签泛域名证书。

---

## 申请 LetsEncrypt 泛域名证书（使用阿里云 DNS）

### 引言

本文将介绍如何使用 Docker Compose 和 `acme.sh` 工具，结合阿里云 DNS 服务，申请和续签泛域名证书（例如 `*.example.com`），以便 Nginx 提供 HTTPS 服务。泛域名证书需要通过 `DNS-01` 挑战方式验证，我们将利用阿里云 DNS API 自动管理 TXT 记录。

需要的工具包括 `acme.sh`（运行在 Docker 中）、Nginx 和阿里云 AccessKey。

本地目录结构如下：

```
.
|-- acme
|   |-- account.conf
|   `-- certificates
|       `-- example.com
|           |-- fullchain.cer
|           |-- example.com.cer
|           |-- example.com.key
|           `-- ca.cer
|-- docker-compose.yml
|-- etc
|   `-- conf.d
|       `-- default.conf
|-- update_cert.sh
`-- www
    `-- html
        `-- index.html
```

过程分为两个阶段：**Phase 1**（证书申请）和 **Phase 2**（配置 HTTPS 和续签）。

---

### 前置准备

1. **阿里云 AccessKey**：

   - 登录阿里云控制台，转到 "访问控制 (RAM)" > "用户"，创建一个 RAM 用户。
   - 为该用户分配权限：`AliyunDNSFullAccess`（DNS 完全管理权限）。
   - 生成 AccessKey ID 和 AccessKey Secret 并保存。

2. **环境变量文件**：
   创建 `.env` 文件，内容如下：
   ```
   EMAIL=your-email@example.com
   DOMAIN=example.com
   WILDCARD_DOMAIN=*.example.com
   ALIYUN_AK_ID=你的AccessKeyID
   ALIYUN_AK_SECRET=你的AccessKeySecret
   ```

---

### Phase 1：申请泛域名证书

此阶段使用 `acme.sh` 在 Docker 中通过阿里云 DNS API 申请泛域名证书。

#### 配置文件

`docker-compose.yml`（初始阶段）：

```yaml
services:
  acme:
    image: neilpang/acme.sh:latest
    container_name: acme
    command: >-
      --issue --dns dns_ali
      -d ${DOMAIN} -d ${WILDCARD_DOMAIN}
      --keylength 2048
      --server letsencrypt
            --config-home /acme
    environment:
      - Ali_Key=${ALIYUN_AK_ID}
      - Ali_Secret=${ALIYUN_AK_SECRET}
    volumes:
      - ./acme:/acme
```

#### 执行命令

```bash
# 启动 acme.sh 容器申请证书
docker compose up acme

# 检查证书是否生成
ls -l acme/
```

#### 解释

- **`neilpang/acme.sh`**：一个轻量级的 ACME 客户端，支持阿里云 DNS。
- **`--dns dns_ali`**：指定使用阿里云 DNS 插件，`acme.sh` 会通过环境变量 `ALIYUN_AK_ID` 和 `ALIYUN_AK_SECRET` 调用阿里云 API 添加和删除 TXT 记录。
- **证书存储**：证书会存储在 `acme/bitnote.fun` 目录下，包括：
  - `bitnote.fun.cer`（证书）
  - `fullchain.cer`（完整证书链）
  - `bitnote.fun.key`（私钥）

申请成功后，`acme.sh` 会自动清理阿里云 DNS 中的 TXT 记录。

---

### Phase 2：配置 HTTPS 和续签

此阶段配置 Nginx 使用泛域名证书，并设置自动续签。

#### 下载 SSL 配置文件

```bash
# 创建 Nginx SSL 配置目录
mkdir -p acme/conf

# 下载 Nginx SSL 配置
curl -L -o acme/conf/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf

# 生成 Diffie-Hellman 参数
openssl dhparam -out acme/conf/ssl-dhparams.pem 2048
```

#### 更新 Nginx 配置文件

`etc/conf.d/default.conf`：

```nginx
server {
    listen 80;
    server_name bitnote.fun *.bitnote.fun;
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
    server_name bitnote.fun *.bitnote.fun;

    ssl_certificate /home/winkee/apps/acme/bitnote.fun/fullchain.cer;
    ssl_certificate_key /home/winkee/apps/acme/bitnote.fun/bitnote.fun.key;
    include /home/winkee/apps/acme/options-ssl-nginx.conf;
    ssl_dhparam /home/winkee/apps/acme/ssl-dhparams.pem;

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
      - ./acme:/etc/nginx/ssl:ro
    ports:
      - 80:80
      - 443:443

  acme:
    image: neilpang/acme.sh:latest
    container_name: acme
    depends_on:
      - nginx
    command: >-
      --issue --dns dns_ali
      -d ${DOMAIN} -d ${WILDCARD_DOMAIN}
      --keylength 2048
      --server letsencrypt
      --config-home /acme
    environment:
      - Ali_Key=${ALIYUN_AK_ID}
      - Ali_Secret=${ALIYUN_AK_SECRET}
    volumes:
      - ./acme:/acme
```

#### 启动服务

```bash
docker compose up -d nginx
```

Nginx 现在将使用泛域名证书为 `bitnote.fun` 和 `*.bitnote.fun` 提供 HTTPS 服务。

---

### 证书续签

#### 手动续签

```bash
# 运行 acme.sh 续签
docker compose run --rm acme --renew -d ${DOMAIN} -d ${WILDCARD_DOMAIN} --dns dns_ali --server letsencrypt --config-home /acme --cert-home /acme/certificates

# 重启 Nginx
docker compose restart nginx
```

#### 自动续签

创建续签脚本 `renew_cert.sh`：

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
docker compose -f /path/to/your/docker-compose.yml run --rm acme --renew -d ${DOMAIN} -d ${WILDCARD_DOMAIN} --dns dns_ali --server letsencrypt --config-home /acme --cert-home /acme/certificates
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

1. **AccessKey 安全**：不要将 AccessKey 硬编码在文件中，建议使用环境变量或 Docker secrets。
2. **DNS 传播时间**：阿里云 DNS 更新通常很快，但如果验证失败，可尝试等待几分钟。
3. **证书路径**：`acme.sh` 的证书路径与 Certbot 不同，注意调整 Nginx 配置中的路径。

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号: codeandroad
