
#### 什么是 SNI？
SNI（Server Name Indication）是 TLS 协议的一个扩展，允许基于客户端请求的域名，在同一个 IP 地址和端口上提供多个 SSL/TLS 证书。使用 SNI 时，客户端会在 TLS 握手过程中包含它要访问的主机名，服务器根据该主机名提供相应的 SSL 证书。


#### SNI 使用场景简介
SNI 通常用在 Load Balancer 上，也就是一个 IP 提供多个域名的场景。

想象一下，一个 Load Balancer 内部管理着多个 Service, 每个 Service 有不同的 Domain Name，持有不同的 Certificate，当一个外部访问者连接到 Load Balancer 时，是先建立 TLS 连接再发送 HTTP 请求的，在建立 TLS 时，Load Balancer 只是与访问者的 IP 地址进行密钥交换，它并不知道应该发送哪个 Certificate 给访问者。而 SNI 作为 TLS 协议的扩展，可以附带想要访问的网站的 hostname 或 servername，简单来说就是 domain name。总之，是能标识 Server 主体的识别名字。这样，Load Balancer 就可以根据访问者提供的 SNI 来提供对应的 Certificate 了。

在没有 SNI 之前，一个 IP 地址只能提供一个 HTTPS 服务供客户访问。而有了 SNI 之后，一个 IP 地址就可以提供多个不同的 HTTPS 服务（使用不同的 Certificate）。


#### Nginx 中怎么识别 SNI 的？
Nginx 默认提供 SNI 功能。当我们在多个 Server Block 中都定义了 `listen 443 ssl;`，那么 Nginx 会自动为每个 domain 使用其相应的 Certificate。

比如，你有两个不同的域名 echo1.com 和 echo2.com，你可以这样配置：

```
server {
    listen 443 ssl;
    server_name echo1.com www.echo1.com;

    ssl_certificate /path/to/echo1.com/fullchain.pem;
    ssl_certificate_key /path/to/echo1.com/privkey.pem;

    ...
}

server {
    listen 443 ssl;
    server_name echo2.com www.echo2.com;

    ssl_certificate /path/to/echo2.com/fullchain.pem;
    ssl_certificate_key /path/to/echo2.com/privkey.pem;

    ...
}
```

公共的配置：

```
http {
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    # Other global settings...
}

```


#### 默认服务器（default server）
当用户提供的 SNI 不匹配任何一个已知的 Server Block 时，我们可以提供一个默认的服务器。

```
server {
    listen 443 ssl default_server;
    server_name default_server;

    ssl_certificate /path/to/default/fullchain.pem;
    ssl_certificate_key /path/to/default/privkey.pem;

    ...
}
```

#### 怎么验证 SNI

```
openssl s_client -connect your_server_ip:443 -servername echo1.com
```

我们可以在输出中查看使用到的 Certificate。

可见，服务器提供了正确的 echo1.com 的 Certificate。




### What is SNI
Server Name Indication (SNI) is an extension to the TLS protocol that allows multiple SSL/TLS certificates to be served on the same IP address and port, based on the domain name requested by the client. With SNI, the client includes the hostname it is trying to reach as part of the TLS handshake, and the server then presents the appropriate SSL certificate based on this hostname.


## Overview
If you want to accept HTTPS requests from your clients, the Internal or External HTTP(S) load balancer must have a certificate so it can prove its identity to your clients. The load balancer must also have a private key to complete the HTTPS handshake.

When the load balancer accepts an HTTPS request from a client, the traffic between the client and the load balancer is encrypted using TLS. However, the load balancer terminates the TLS encryption, and forwards the request without encryption to the application. When you configure an HTTP(S) load balancer through Ingress, you can configure the load balancer to present up to ten TLS certificates to the client.


Note: When using Internal HTTPS load balancing, it is not possible to use HTTP as well. It is necessary to disable HTTP in the Ingress manifest. This is not required for the external load balancer.
The load balancer uses Server Name Indication (SNI) to determine which certificate to present to the client, based on the domain name in the TLS handshake. If the client does not use SNI, or if the client uses a domain name that does not match the Common Name (CN) in one of the certificates, the load balancer uses the first certificate listed in the Ingress. The following diagram depicts the load balancer sending traffic to different backends, depending on the domain name used in the request:

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/sni_multiple_ssl_certs.png)


multiple SSL certificates with Ingress system diagram

You can provide an HTTPS load balancer with SSL certificates using one of three methods:

Google-managed SSL certificates. Refer to the managed certificates page for information on how to use them.
Google Cloud SSL Certificate that you are managing yourself. It uses a pre-shared certificate previously uploaded to your Google Cloud project.
Kubernetes Secrets. The Secret holds a certificate and key that you create yourself. To use a Secret, add its name in the tls field of your Ingress manifest.
You can use more than one method in the same Ingress. This allows for no-downtime migrations between methods.




查看这篇文章：
https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-multi-ssl

