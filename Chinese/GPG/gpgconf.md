
## gpgconf 介绍
gpgconf 是一个用于配置 GnuPG 的命令行工具。它提供了对 GnuPG 配置的灵活控制，让用户能够根据自己的需求定制化 GnuPG 的行为。
gpgconf 的主要作用
配置 GnuPG 参数： 可以设置各种 GnuPG 参数，如默认的加密算法、密钥服务器、日志级别等。
管理 GnuPG 代理： 可以控制 gpg-agent 的启动、停止和配置。gpg-agent 是 GnuPG 的一个辅助程序，用于管理密钥环和密码，提高 GnuPG 的性能。
自定义 GnuPG 行为： 通过配置，可以实现 GnuPG 在特定场景下的自动化操作，例如自动签名邮件、自动解密文件等。

#### 常见用法示例
查看 GnuPG 配置：
```
gpgconf --list-options
```

设置默认加密算法：
```
gpgconf --set default-cipher-algo AES256
```

启动 gpg-agent：
```
gpg-agent --daemon
```

重启 gpg-agent：
```
gpgconf --kill gpg-agent
```

#### 总结
gpgconf 是 GnuPG 用户必备的工具之一，通过它可以对 GnuPG 进行精细化的配置，满足各种加密需求。无论是初学者还是高级用户，都可以从 gpgconf 中获益。


#### 查看或修改配置组件或信息

```
gpgconf --list-components
gpgconf --list-dirs
```

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/cb88ec8e79d44a3c9752144536584bef~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ29kZUFuZFJvYWQ=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNzYxMjgyMDA5NTA3OTAxIn0%3D&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1735843345&x-orig-sign=y5b6t%2BLI0fNksXj%2B7HsMkdBvDpg%3D)


#### key server

更改默认的 key server

```
gpgconf --set-config gpg --keyserver hkp://keyserver.ubuntu.com
```

重置 keyserver

```
gpgconf --unset-config gpg --keyserver
```

#### gpg-agent
为 gpg-agent 组件创建新的配置文件
```
gpgconf --create-config gpg-agent
```

重载某个组件

先 kill 再 launch
```
gpgconf --kill gpg-agent
gpgconf --launch gpg-agent
```

导出配置
```
gpgconf --list-config > gpg-config-backup.txt
```

### dirmngr 组件
dirmngr 是 GnuPG 的一个组件，主要负责网络相关的操作。启动方法：

1. 直接启动：
```bash
dirmngr --daemon
```

2. 通过 gpgconf 启动：
```bash
gpgconf --launch dirmngr
```

dirmngr 的主要功能：
1. 密钥服务器通信
- 处理密钥的查找
- 处理密钥的获取
- 处理密钥的上传

2. 证书验证
- 获取 CRL（证书吊销列表）
- 处理 OCSP 请求
- 验证 X.509 证书链

3. 检查运行状态：
```bash
gpgconf --list-dirs dirmngr-socket  # 查看 socket 位置
gpgconf --status-of-socket dirmngr  # 检查状态
```

4. 查看日志：
```bash
dirmngr --daemon --debug-level basic --log-file ~/.gnupg/dirmngr.log
```

如果遇到网络问题，可以重启 dirmngr：
```bash
gpgconf --kill dirmngr
gpgconf --launch dirmngr
```

#### gpgsm

这个组件的作用是对 email 进行签名，遵循 S/MIME 规范。

    gpgsm --sign --output signed-email.eml email.eml

#### gpg-connect-agent

可以通过 **gpg-connect-agent** 给 **gpg-agent** 发送命令。比如

(1) 重载 gpg-agent。
修改了 gpg-agent 的配置信息，我们可以给它发送一个重载命令，以便配置生效。

    gpg-connect-agent reloadagent /bye

(2) 缓存某个 key 的 passphrase

    echo "PRESET_PASSPHRASE <key-id> <your-passphrase>" | gpg-connect-agent

#### gpgv

验证签名，验证文档未被篡改。

    gpgv signed-file.sig original-file

当我们从网站上下载了一个文件，我们不确定这个文件是不是被篡改过。我们就可以这样：

    gpgv package.tar.gz.sig package.tar.gz

#### gpg-agent

这个是个 daemon。最常见的配置项是 cache ttl，我们可以在启动时指定选项：

    gpg-agent --default-cache-ttl 600 --max-cache-ttl 7200

#### gpg.conf 配置文件

最后，我们来看下各个组件的配置文件案列：

### gpg.conf

```
# GPG Configuration File
keyserver hkp://keyserver.ubuntu.com
default-key ABCD1234
use-agent
compress-level 9
no-tty
```

### gpg-agent.conf

```
# GPG Agent Configuration File
default-cache-ttl 600
max-cache-ttl 7200
pinentry-program /usr/bin/pinentry-gtk-2
enable-ssh-support
```

### gpgsm.conf

```
# GPGSM Configuration File
keyserver hkp://keyserver.ubuntu.com
default-cert-digest-algo SHA256
use-agent
```


