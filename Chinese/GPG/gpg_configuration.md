## 介绍

上一篇文章介绍了 GPG 的核心概念，这篇文章着重介绍 GPG 的实战操作。
不过，在实战之前，我们需要先熟悉如何配置 GPG 工具，磨刀不误砍柴工。

### 1、安装

Linux 系统通常会自带，MacOS 我们可以使用 brew 来安装

    brew install gpg

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/a79d9749ae6f4a8ea77aa0f9c5ce7f8e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ29kZUFuZFJvYWQ=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNzYxMjgyMDA5NTA3OTAxIn0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1736358869&x-orig-sign=9ltycoIYlm2SnY2UsAu5aLUB4uY%3D)

### 2、HOME 目录

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/33418737a4384ee68d216e0548007c3e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ29kZUFuZFJvYWQ=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNzYxMjgyMDA5NTA3OTAxIn0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1736358869&x-orig-sign=ZIHpQlNX91bSRE8f7Q7gbd%2BD0Fo%3D)

`~/.gnupg` 目录结构：


![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/0e758c08ee954d1ba652bcd45b3a7530~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ29kZUFuZFJvYWQ=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNzYxMjgyMDA5NTA3OTAxIn0%3D&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1735840893&x-orig-sign=rp0QvyRuQyX%2B5O4kxTseEYZ1UEc%3D)


我们可以把默认目录强制指定为 `$HOME/.local/share/gnupg`，如下：

```
export GNUPGHOME="${XDG_DATA_HOME:-$HOME/.local/share}/gnupg"
```

也可以在执行 gpg 命令时用 `--homedir` 指定 home directory。

### 3、配置文件

(1) `~/.gnupg/gpg.conf` 这是标准的配置文件

(2) `~/.gnupg/common.conf` 可选的配置文件，如果 `~/.gnupg` 目录不存在，则会自动生成一个 `~/.gnupg/common.conf`，里面的内容是 use-keybox，意思是默认使用 keybox 格式。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/0b6ad02292b145e199c88f143d488414~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ29kZUFuZFJvYWQ=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNzYxMjgyMDA5NTA3OTAxIn0%3D&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1735841052&x-orig-sign=CeCzPI7lzbUNOYrr02mMJ%2FTc7vA%3D)

~/.gnupg/ 目录中的这两个配置文件有不同的用途：

1. gpg.conf:
- 专门用于 gpg 命令行工具的配置
- 影响加密、签名等核心操作的行为
- 用户最常修改的配置文件

常见的 gpg.conf 配置项包括：
```conf
# 默认密钥
default-key 0xFF00FF00

# 首选加密算法
personal-cipher-preferences AES256 AES192 AES CAST5

# 首选压缩算法
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed

# 首选摘要算法
personal-digest-preferences SHA512 SHA384 SHA256 SHA224

# 密钥服务器
keyserver hkps://keys.openpgp.org

# 显示照片 ID
photo-viewer "display %i"

# 总是显示密钥指纹
with-fingerprint

# 验证签名时显示公钥指纹
verify-options show-uid-validity
list-options show-uid-validity

# 自动获取密钥
auto-key-locate keyserver
```

2. common.conf:
- 包含所有 GnuPG 工具共享的基本配置
- 影响 gpg、gpgsm、gpg-agent 等所有组件
- 通常不需要修改

一般建议：
- 日常使用主要修改 gpg.conf
- 除非你明确知道要修改什么，否则保持 common.conf 默认
- 对安全性要求高的配置放在 gpg.conf 中
- 使用注释说明重要配置的用途


下面是各个文件的作用：

- `~/.gnupg/pubring.gpg` 这是旧格式的公钥环。**最新版本的 gpg 已经采用了新的 keybox 格式**。
新格式的文件是 `~/.gunpg/pubring.kbx`，或者 `~/.gnupg/public-keys.d/pubring.db`。

gpg 兼容旧格式，所以两种都可以使用。但是 gpg 默认会把所有密钥对保存在 `pubring.kbx` 中。

请注意，如果 `pubring.gpg` 和 `pubring.kbx` 两个文件同时存在，但如果后者中没有保存任何密钥的话，那么将使用旧文件 `pubring.gpg`。

注意：GnuPG 2.1之前的版本将始终使用文件 `pubring.gpg`，因为它们不知道新的 **keybox** 格式。如果你需要使用 GnuPG 1.4 解密数据，你应该保留这个文件。

如果我们直接执行 `gpg --list-keys` 命令，那就是把 `~/.gnupg/pubring.kbx` 中保存的 keys 信息列出来。

如果我们想列出指定 `.kbx` 文件中的 keys 信息（比如 `/etc/apt/trusted.gpg.d/` 目录下的某 key 文件），我们可以用 `--no-default-keyring` 配合 `--keyring` 选项来实现，如下：

```
gpg --list-keys --no-default-keyring --keyring /etc/apt/trusted.gpg.d/ubuntu-keyring-2018-archive.gpg 
```
- `--no-default-keyring` 选项的意思不实用默认的 `~/.gnupg/pubring.kbx` 文件
- `--keyring` 选项是指定数据库。


### 版本变化

- `pubring.gpg` 是 GnuPG 2 版本之前所用来保存 公钥（Public Keys）的文件
- `pubring.kbx` 是 GnuPG 2.x 版本中默认用来存储 公钥（Public Keys） 的文件。
- `pubring.db` 是 GnuPG 2.4.0 及之后版本的新实现，它用来代替 pubring.kbx 文件。
    此改变主要是为了改进性能和支持更复杂的密钥管理。


### 迁移

下面是把旧格式（`~/.gnupg/pubring.gpg`) 中的密钥迁移到新格式 `~/.gnupg/pubring.kbx` 或 `~/.gnupg/public-keys.d/pubring.db` 中的具体步骤：

##### （1）使用 `gpg --list-keys` 会触发自动迁移

##### （2）也使用更明确的命令：

```bash
# 导出所有密钥
gpg --export-options backup --export > pubkeys.gpg
gpg --export-secret-keys --export-options backup > seckeys.gpg

# 创建新的 keybox 格式数据库
mv ~/.gnupg/pubring.gpg ~/.gnupg/pubring.gpg.old
mv ~/.gnupg/secring.gpg ~/.gnupg/secring.gpg.old

# 导入密钥到新格式
gpg --import pubkeys.gpg
gpg --import seckeys.gpg
```

完成后可以验证：
```bash
gpg --list-keys
```

如果一切正常，可以删除备份文件：
```bash
rm pubkeys.gpg seckeys.gpg
rm ~/.gnupg/*.gpg.old
```

注意：执行这些操作前最好先备份整个 `~/.gnupg` 目录。

如果想要将现有的 `pubring.gpg` 文件转换为 keybox 格式，应该首先备份 ownertrust 值，然后将 pubring.gpg 重命名为 publickeys.backup，以便任何 GnuPG 版本都无法识别。然后执行导入操作，最后恢复 ownertrust 值。具体步骤如下：

```
$ cd ~/.gnupg
$ gpg --export-ownertrust > otrust.lst
$ mv pubring.gpg publickeys.backup
$ gpg --import-options restore --import publickeys.backup
$ gpg --import-ownertrust otrust.lst
```

- `~/.gnupg/pubring.kbx.lock` pubring.kbx 的锁文件。
- `~/.gnupg/secring.gpg` GnuPG 2.1 之前版本使用的旧版秘密钥环。GnuPG 2.1及更高版本不使用它。如果你需要使用 GnuPG 1.4 解密存档数据，你可能需要保留它。
- `~/.gnupg/secring.gpg.lock` 旧版秘密钥环的锁文件。
- `~/.gnupg/.gpg-v21-migrated` 指示已迁移到 GnuPG 2.1 的文件。
- `~/.gnupg/random_seed` 用于保存内部随机池状态的文件。
- `~/.gnupg/openpgp-revocs.d/` 这是 gpg 存储预生成撤销证书的目录。文件名对应于相应密钥的 OpenPGP 指纹。建议备份这些证书，如果主私钥未存储在磁盘上，将它们移动到外部存储设备。任何能够访问这些文件的人都可以撤销相应的密钥。你可能需要打印出来。你应该备份此目录中的所有文件，并注意将此备份妥善保管。

#### 信任数据库 trustdb.gpg

- `~/.gnupg/trustdb.gpg` 
trustdb.gpg 是 GPG 的信任数据库文件，它存储了你对其他人的公钥的信任级别信息。具体来说：

1. 它记录了：
- 你对每个密钥的信任度评级
- 密钥的有效性判断
- 密钥所有者的身份验证状态

2. 信任级别包括：
- unknown (未知)
- none (不信任)
- marginal (部分信任)
- full (完全信任)
- ultimate (终极信任，通常只给自己的密钥)

3. 查看信任数据库：
```bash
gpg --list-keys --with-trust-model=trust-db
```

4. 编辑信任级别：
```bash
gpg --edit-key user@example.com
> trust
```

如果这个文件损坏，GPG 会自动重新创建，但你之前设置的信任关系会丢失，需要重新建立。
所以备份 `~/.gnupg` 目录时也要记得包含这个文件。

##### 补充知识：
我们在 Debian 系的 Linux 系统中使用 apt 时还会经常见到下面两个东西：
```
/etc/apt/trusted.gpg
/etc/apt/trusted.gpg.d
```
它们的作用也是类似的，是信任数据库。每当我们要在 `/etc/apt/sources.list` 中添加一个新的仓库源时，我们都需要把这个新源的公钥添加到 `/etc/apt/trusted.gpg` 中。

我们也可以这样做：

```
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
  http://packages.nginx.org/nginx-agent/ubuntu/ `lsb_release -cs` agent" \
  | sudo tee /etc/apt/sources.list.d/nginx-agent.list
```

### 为什么信任数据库（`trustdb.gpg`） 仍然使用 `.gpg` 格式而不是 `.kbx` 格式，

1. 历史原因：
- .kbx (keybox) 格式主要是为了替代密钥存储 (`pubring.gpg/secring.gpg`)
- trustdb.gpg 的功能和结构与密钥存储不同，没有迁移的迫切需求

2. 功能区别：
- keybox 格式针对密钥管理进行了优化
- **trustdb 只存储简单的信任关系数据，原有格式已经足够**

3. 兼容性考虑：
- 保持 trustdb.gpg 格式不变可以确保与旧版本 GnuPG 的兼容性
- 不同 GnuPG 版本间可以更容易地共享信任数据

实际上，到目前为止 GnuPG 的开发团队也没有计划将 trustdb 迁移到 .kbx 格式，因为现有格式运作良好，没有明显的改进需求。


### ~/.gnupg 目录下有 S.dirmngr 和 S.keyboxd 两个 socket 文件，它们是干啥用的？

**dirmngr** 和 **keyboxd** 是 GnuPG 的两个守护进程，而这些 `.S` 文件是它们的 Unix domain sockets：

1. S.dirmngr：
- dirmngr 是证书和密钥管理守护进程
- 负责与密钥服务器通信
- 处理 CRL（证书吊销列表）和 OCSP（在线证书状态协议）请求
- 通过这个 socket 文件与其他 GnuPG 组件通信

2. S.keyboxd：
- keyboxd 是密钥数据库访问守护进程
- 管理 pubring.kbx 文件的访问
- 提供密钥查询和管理服务
- 通过这个 socket 文件为其他组件提供密钥访问服务

这些 socket 文件：
- 用于进程间通信（IPC）
- 自动创建和管理
- 不需要手动修改
- 进程停止时会自动删除

如果这些 socket 文件出现问题，可以：
```bash
gpgconf --kill all    # 停止所有 GnuPG 守护进程
rm ~/.gnupg/S.*      # 删除所有 socket 文件
gpg --list-keys      # 重新启动守护进程
```


### 私钥数据库
前面介绍的 `~/.gnupg/pubring.kbx` 或 `~/.gnupg/public-keys.d/pubring.db` 保存的都是公钥，这里面没有私钥，因此即使文件泄露，也无法用于解密数据。
私钥是保存在 `~/.gnupg/private-keys-v1.d/` 目录中的，它的权限通常设置为 700，仅允许当前用户访问。
私钥通常会使用密码短语（passphrase）加密存储。如果攻击者获得了你的私钥文件，但不知道你的密码短语，他们仍然无法使用它。

如何查看私钥？
```
gpg --list-secret-keys
```

输出示例：

```
sec   rsa2048 2025-01-01 [SC]
      ABCDEF1234567890ABCDEF1234567890ABCDEF12
uid           [ultimate] Your Name <your-email@example.com>
ssb   rsa2048 2025-01-01 [E]
``

- sec：表示主私钥（Primary Secret Key）。
- ssb：表示子私钥（Sub Secret Key）。

在使用 GPG 创建密钥对时，公钥和私钥存储是分别存储的，公钥存储在 `pubring.kbx`（旧版本）或 `pubring.db`（新版本）中。
而私钥存储在 `~/.gnupg/private-keys-v1.d/` 目录中。具体文件名如：
```
~/.gnupg/private-keys-v1.d/ABCDEF1234567890.key
```

```
~/.gnupg/
├── private-keys-v1.d/
│   ├── ABCDEF1234567890.key    # 私钥文件
│   ├── ...（其他私钥）
├── public-keys.d/pubring.db    # 公钥环（取代 pubring.kbx）
├── trustdb.gpg                 # 信任数据库
├── gpg.conf                    # 配置文件
├── random_seed                 # 随机数种子
```



全文完！

如果你喜欢我的文章，请关注我的微信公众号 codeandroad。
