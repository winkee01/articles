## 问题来源

Caddy 服务启动的时候出错，查看状态

```
systemctl status caddy
```

发现如下提示：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/caddy_service_systemctl.jpg)

可以看到，这个是在执行 ExecStart 的时候出错了，

也就是下面这条命令出错：

```
/usr/bin/caddy -conf /etc/caddy/caddy.conf -root /tmp -agree
```

由于 `systemctl status` 无法打印全部的出错信息，导致很难排查

我们单独执行这条命令，来查看整个执行过程

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/caddy_log_info.jpg)

可以看到，在获取证书的过程中，采用了 HTTP-1 的 challenge 的方式，但是却无法获得绑定 80 端口的权限，从而导致出错。

为何无法绑定 80 端口呢，这是因为：

Linux 默认不允许 non-root 用户绑定系统的低级端口（比如 80, 443 等）Linux doesn't allow processes to listen on low-level ports by default.

我们执行下面这句来进行授权

```
sudo setcap CAP_NET_BIND_SERVICE=+eip /usr/bin/caddy
```

再重启 `caddy.service` 就可以成功了

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/caddy_restart_success.jpg)

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 deliverit。
