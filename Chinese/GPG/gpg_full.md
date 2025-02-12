## 使用 GPG Key 来构建签名、加密及认证体系

#### 前言

**GPG**, 或 **GnuPG** (GNU Privacy Guard) 是一个遵照 OpenPGP 协议的用于加密、数字签名以及认证的软件。它与 PGP (Pretty Good Privacy) 的区别是它是开源的，而 PGP 则是 Symantec 公司的专有软件。 

在数字世界，我们经常需要进行邮件加密、数字签名或者登陆认证等操作，GPG 就是这样一个既可以方便我们管理公私钥，又可以随时满足我们需求的密钥管理工具。


如果你使用 Linux 发行版，gpg 工具是自带的 (`/usr/bin/gpg`)，你可以执行 `gpg -h` 来查看帮助信息。本文介绍如何使用 GPG 来构建我们的签名、加密以及认证系统。


## 第一部分：GPG KEY 使用场景

为了有个直观的理解，我先列举几个使用场景：

**1. 使用 GPG 密钥来签名你的 git commits**

向 github 仓库提交代码时，如果你的 commit 经过已授权的 GPG Key 签名，那么会显示为绿色的 Verified 状态。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b7e88bf350246b2ac47b6579a0ff1cb~tplv-k3u1fbpfcp-zoom-1.image)

如果你使用了未授权的密钥进行签名，则会显示

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef8880b030ba4f0383f068ba8da778b0~tplv-k3u1fbpfcp-zoom-1.image)

或者你的密钥已经授权，但你进行 commit 时使用了非认证的 email（冒充别人或别人冒充你）

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/864d46ce00e84c97af242eb78e9dad29~tplv-k3u1fbpfcp-zoom-1.image)

对于要求多重签名却又缺少其中某个签名的：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70b6dd7d892f44109abe4fd002514db6~tplv-k3u1fbpfcp-zoom-1.image)

类似的，发布 release，或者添加 tag，通常都应该使用签名以便使得授权主体更加明确。

需要使用签名的场景非常多，可以说覆盖开发和管理的个个角落，有了 GPG 来帮助我们管理自己的密钥，面对各种场景就会得心应手。


**2. 使用**公钥**验证第三方软件的签名**

软件作者在软件发布时，Release 页面通常会同时提供程序文件和签名文件。通常，签名文件是软件作者使用私钥对软件的摘要(digest)进行签名而得来。我们可以在拿到作者的公钥后对签名进行验证。如果验证失败，那么说明下载的软件已经被篡改。这种情况通常发生在有人下载作者的软件之后，修改软件并注入木马，然后重新发布到假的镜像站点；如果你没有对签名进行验证，就会存在风险。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa722cd94335498191f699eda3a8d357~tplv-k3u1fbpfcp-zoom-1.image)

公钥或签名文件通常会以 `.sig` 或 `.asc` 为后缀。

```
gpg --import veracrypt_pgp_public_key.asc
gpg --verify veracrypt-1.25.9.sig veracrypt-1.25.9.deb
```

**3. 使用 gpg 公钥来加密你的邮件**

当我们苦苦寻找安全性更高，支持加密的邮箱时，有了 GPG Key，我们可以自己加密邮件了。下面演示使用 GPG KEY 对 GMAIL 邮件加密。


**1) 发送加密邮件**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b234c680267c4e15b5e592f48f072d93~tplv-k3u1fbpfcp-zoom-1.image)


**2) 收到加密邮件**

可借助插件自动使用私钥进行解密，解密后文本如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0c87d6096214f7e8ce8ef8b6e4450fa~tplv-k3u1fbpfcp-zoom-1.image)


**3) 未解密的邮件**

如果不自动解密，则会显示 ASC 格式的密文 (ASC 格式是 base64 略作修改而来，gpg 可通过 `--armor` 参数把二进制内容进行字符化）。

比如下图，我在手机上打开 Gmail 邮件 则会显示为密文，这是因为我手机上没有导入我的私钥，所以无法解密。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71db80d0b4dd4ba5825602ea89be05e4~tplv-k3u1fbpfcp-zoom-1.image)


**4) 使用公钥认证来实现授权登陆 (Public Key Authentication)**

Publick Key Authentication （公钥认证）是服务器最常见的应用，比如 SSH 登陆，公钥认证是比密码登陆要安全得多的方式。具体的方式就是把需要登陆的用户的公钥提交到服务器上 `~/.ssh/authorized_keys`，服务器上的 sshd 程序使用公钥加密一个 shared key 发给用户，用户用私钥解密后得到 shared key，实现认证；并且双方在后续的通信中使用这把对称密钥进行加密通信。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/13eb49c0d70848ac9b78435624458f56~tplv-k3u1fbpfcp-zoom-1.image)


如果你在 `～/.ssh/` 目录中生成了太多的 key pairs，管理这些 keys 肯定会让你头疼，而如果你使用 gpg 来管理这些密钥的话，就会方便很多。gpg 生成的密钥格式与 `ssh-keygen` 所生成的密钥格式不相同，但是可以使用 `--export-ssh-keys` 命令选项来到处 ssh 格式的公钥，并借助 `gpg-agent` 加载私钥，从而使得 ssh 可以使用 gpg 的密钥建立加密连接。使用 gpg keys 来替代原来的 ssh keys 最大的好处就是管理更加方便和简单。而且，如果你使用了支持 gpg key 的硬件（比如 `Yubikey`），你的安全性也更加高了。


## 第二部分：GPG Keys 核心概念

下面介绍 GPG 中的核心概念以及 GPG 的公钥加密体系的原理

**1. Key** 密钥，每个 Key 都包含两部分：Private Key 和 Public Key。


**2. Fingerprint** 指纹，指纹是 Public Key 的散列值（默认使用 MD5 算法），gpg 中的 fingerprint 与 SSH 的计算方法略有不同，SSH 中 fingerprint 是直接对 base64 的结果再计算 MD5 得来，而 gpg 则对合并后的 `modulus` 和 `exponent` 等参数的二进制进行 MD5 计算而来。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91dc1e32d26642aa811b406008185080~tplv-k3u1fbpfcp-zoom-1.image)


**3. KeyID**，GPG 中使用 fingerprint 来作为 KeyID，用来标识一个 Key。总长为 32-bit。

- **Long ID** 用 fingerprint 的后 16 位字符 (HEX)来表示
- **Short ID** 则用 fringerprint 的后 8 位字符 (HEX) 来表示

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4af7a8931e1c48098111a5740124ad48~tplv-k3u1fbpfcp-watermark.image?)

如果不加 `--keyid-format`，则**默认显示的是 LONG ID**。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ec35393feaa4b7eb5b1e2404090352b~tplv-k3u1fbpfcp-zoom-1.image)

**4. UserID**，GPG 中也使用 UserID 来标识一个或多个 Key。由于 Key **必须绑定**至少一个 UserID（也就是 **Email 地址**），所以 UserID 也可以用来标识 Key。
区别是：**同一个 UserID（即 Email 地址） 可以绑定多个 Keys。**

**5. Keyring**，密钥环，是指存储密钥的数据库。默认情况下，所有的本地密钥都保存在 `~/.gnupg/pubring.kbx` 中

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b742cf1c95f64ff6b44a5fa27414633a~tplv-k3u1fbpfcp-zoom-1.image)

另外，我们可以把 gpg 命令的常用配置信息写入 `~/.gnupg/gpg.conf`，执行 `gpg` 命令的时候会自动读取该文件。

**6. Key Server**，专门用于存放 Public key 的服务器。gnugp 提供了一个免费的 key server `https://keys.openpgp.org/`，当我们执行 `gpg --send-keys [keyid]` 命令来发布一个 public key 的时候，它会自动发送到这个 key server 中去。另外，当我们执行 `gpg --search-keys [keyid]`或 `gpg --recv-keys [keyid]` 的时候，也都是会对默认的 key server 操作。

**7. Keygrip**，是一个 20 字节长度的，与 protocol 无关的的值，通过对 key 的 modulus 参数计算 md5 得来。比如，对于使用 RSA 算法的 key，具体实现是把 modules 作为 unsigned integer，并去除二进制高位的0，然后计算 sha1 得来。

Keygrip 相比 fingerprint 的特别之处是，它可以唯一标识一个 Key，因为它只跟你的密钥参数（如 modulus）相关，因此在 GPG 和 SSH 中，相同的 keygrip 都表示一个相同的 key。当我们执行 `gpg --gen-key` 命令时，会 keygrip 文件会自动生成在目录 `~/.gnupg/private-keys-v1.d/` 下，并以 `keygrip-id.key` 命名。

我们可以用 `gpg -k --with-keygrip` 命令来查看 Key 对应的 keygrip：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2dd47c2f502a4fc8a9ad0722ca495c8a~tplv-k3u1fbpfcp-zoom-1.image)

keygrip 一个用处是，如果我们只想删除 Master Key，当我们执行 `gpg --delete-secret-keys [master-keyid]`，他会继续追问我们是否删除 subkey，否则会删除失败。这是由于 KeyID/Fingerprint 会关联 Master Key 及所有的 subkey，所以如果你只想单独把 Master Key 删除，那么可以使用它的 keygrip：

`gpg-connect-agent "DELETE_KEY 5EACE229E5EA90792805777B9163400FE3D189D4" /bye`

**8. Master Key and Subkey**

当我们使用 `gpg --gen-key` 命令生成 GPG Keys 时，默认会创建一个 Master key 和一个 Subkey。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e6f751a6d7e047b2831facdc051f0646~tplv-k3u1fbpfcp-watermark.image?)

**Master Key 和 Subkey 是什么关系？**

本质上，Master key 和 Subkey 都是独立生成的，两者在生成时彼此之间并无依赖关系；但是在生成之后，GPG 用一个叫 **Binding Signature** 的东西把两种进行关联。简单来说就是 Master key 对 Subkey 进行签名，声明自己对 Subkey 的 Owner 关系；同时 Subkey 也对 Master Key 进行签名，声明自己对 Master Key 的 Member 关系。

RFC 4880 中对此有解释如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48a421957a814c62a40f3ae63aaded2d~tplv-k3u1fbpfcp-zoom-1.image)

我们可以使用 `--check-sigs` 选项来查看 Key 中的签名：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/915056faace347e2aa2fa1c97842a7da~tplv-k3u1fbpfcp-zoom-1.image)

使用 `--list-sigs` 来列出所有签名

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26af2b635bd146e6b6264fb3b414a126~tplv-k3u1fbpfcp-watermark.image?)

MasterKey 和 Subkey 互相对彼此签名，`--check-sig [keyid]` 显示的是 Master key 所拥有的签名数量，由于 Master key 会对自己签名，再加上 subkey 也会对他签名，所以最下面显示它有 2 个 goog signatures。如果我们只生成 Master Key，那么它会只有一个签名，就是自签名。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/453e9646187045da8ec8f96795d31fdb~tplv-k3u1fbpfcp-zoom-1.image)

**9. GPG Message Format**

如上所述，当我们执行 `gpg --list-keys` 或 `gpg --list-secret-keys` 命令查看 Key 信息时，显示结果包含了 fingerprint, date, expiration, usage, signature 等等许多信息，完整的 GPG Keys 格式是定义在 OpenPGP 协议中的，每个组成部分的描述都可以在 RFC 4880 中找到。 RFC 4880 把 Key 格式中每部分叫做一个 packet，每个 packet 分配一个 tag，这些规则都定义在 OpenPGP 协议中，RFC 4880 中的对应描述如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/918b8252d58d4cc8841cc4b410b9062e~tplv-k3u1fbpfcp-zoom-1.image)

比如，我们想要查看 fingerprint 使用的散列算法，我们可以这样

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee14d8c11f8b460398630a8e3040ecd4~tplv-k3u1fbpfcp-zoom-1.image)

使用的是 digest algo 10，我们可以在 RFC 4880 9.4 Hash Algorithm 一章查到如下信息：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ccbff726efa405f87c39764e7b0b7b5~tplv-k3u1fbpfcp-zoom-1.image)

可以看到，ID 10 对应的就是 SHA512 算法。关于算法的选择，我们是可以通过参数设置的，比如常见的设置的方法是在 `~/.gnupu/gpg.conf` 中添加如下内容：

```
personal-digest-preference SHA512 SHA384 SHA256 SHA224
personal-cipher-preference AES256 AES192 AES CAST5 CAMELLIA192 BLOWFISH TWOFISH
 CAMELLIA128 3DES
personal-compress-preferences ZLIB BZIP2 ZIP
cert-digest-algo SHA512
digest-algo SHA256
```

如果你想要看更多或更加 human readable 的 detail，你可以安装 `pgpdump` 工具来查看。

**10. GPG Keys 的四种用途：**

-   Certify
-   Sign
-   Encryption
-   Authentication

在第一部分已经根据 GPG KEY 的用途介绍了多种使用场景，所以这部分内容并不难复杂，这里再稍微赘述一下以方便对文章后续内容的理解：

**Certify or Certification** 翻译成“证书授权，发证书”，本质就是 Sign 操作，也就是用私钥对目标进行签名（其实就是加密操作，之所以不能算是真正的加密是因为只要是公钥就可以解密，而公钥人人都可以获得，所以就失去了“加密”的内涵，人人都知道的东西肯定不叫秘密了）。但是签名的对象通常有两种，一种是对数据进行签名，而另一种则是对 Key 密钥进行签名。我们**把对数据进行签名叫做 Sign** **，而对密钥 (Public Key) 进行签名叫做 Certify**。对于前者 gpg 提供的命令选项是 `--sign (or --clearsign)`，对于后者，gpg 提供的命令选项是 `--sign-key`。


**信任链 Trust Chain**

当 A 用其私钥对 B 的公钥进行签名之后，B 就相当于找了一个人给他背书，这样，当 B 把公钥给第三个人 C 的时候，C 就更倾向于相信 B 的公钥确实是他的，当越多人对 B 的公钥进行签名，B 的公钥的受信任程度就越高。相比于 PKI 体系中的依靠权威证书机构所构建的信任体系，这是一种分布式的信任链。比特币的制造原理也是基于类似这样的分布式信任，当某个矿工打包一个区块并用自己的私钥对这个区块进行签名，发布到网络中之后，越来越多的矿工也对这个区块进行签名，这个区块就越加变得可信，最终达到难以摧毁的信任程度，也就是信任强化（**Trust Hardening**）。


**Encryption** 是只用公钥进行加密，在非对称密钥体系中，公钥加密的内容只能通过对应的私钥才能解密，但是由于加密速度非常慢，所以公钥加密通常用在传输体积较小的数据上，比如邮件就是最适合的应用场景。另外一个场合加密通信的握手阶段进行密钥交换。

另外，gpg 也支持对称密钥加密，比如使用下列命令就可以用对称密钥加密一个文件：

```
gpg --symmetric --cipher-algo AES256 hello.txt
```

会弹出对话框提示你输入 `passphrase`，输入之后就会生成加密文件，操作非常简单，这里不赘述。

**Authentication** 是认证，我们常见的认证方式就是密码登陆，但是由于密码存在暴力破解的问题，所以并不是一种非常安全的方式，如果你购买过公网服务器就知道，只要一上线，/var/log/auth.log 中就有无数的密码尝试登陆记录。所以，通常更加安全的方式就是使用公钥认证，它的加密强度更高，目前无法破解。公钥认证的原理跟使用公钥加密一样，服务器持有用户的公钥，只要用户持有对应的私钥（可以解密服务器下发的加密后的数据），就可以验证用户身份。最常见的比如 SSH 登陆，另外 Kubernetes 也是用客户端证书来校验用户身份，把证书中所授权的 Subject 映射为集群中的 Authorized User，Organization 则为组 Group。证书与公钥的主要区别在于前者绑定了用户身份及域名信息(Subject, SAN, etc.)，在 GPG 的消息格式中，Public Key 也同样绑定了用户身份（User ID）。

上面介绍了 Key 的四种功能，只有 Master Key 有 Certify 的功能。也就是说，**只有 Master Key 可以对其它 Key 进行签名，subkey 是无法这么做的。**

## 第三部分：构建 GPG Keys

下面介绍如何使用 gpg 工具来构建 GPG keys。

gpg 是一个 Linux 下自带的一个命令行工具（多数发行版都自带这个工具），如果你是使用 macOS 或 Windows，那么你可以分别下载 GPG Suite 和 Gpg4win)。官网就提供了多种平台的下载链接：https://gnupg.org/download/ 。

根据前面介绍的 KEY 用途和功能的不同，我们需要构建如下的 Keys

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0eb502d27c9f4001b60e9b06658c7d45~tplv-k3u1fbpfcp-watermark.image?)

**为什么要构建这么多 Key？**

主要是基于网络安全中的最小权限和职责分离原则。

Master Key 作为类似于 PKI 体系中的 Root CA 的功能，永不过期，只有在需要签发新的 subkey 和 吊销 subkey 的时候才出来干活，平时放置在一个离线的安全存储中（比如硬件密钥盘 Yubikey）。而各个具有不同功能的 subkey 则作为真正的打工人，负责平时每日的干活，有效期短，过期后需要使用 Master key 签发新的。

**1. 创建 Master keys**

Master Key 应该只用于 Certify，使用安全系数最高的 Ed25519 椭圆曲线算法，并且永远不过期。

![aaaaaaa.jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb44292596fa4e0fb4190244b35ee002~tplv-k3u1fbpfcp-watermark.image?)

命令末尾会提示你是否输入一个 passphrase 密码来保护密钥的访问，建议输入一个保护密码。如果嫌密码不好记忆，可以暂时不设密码，因为后续还可以手动再添加。另外，关于密码的管理，可以参考我另一篇文章《如何管理你的密码》。


**参数解释：**

- `--expert` 是启用专家模式，让你对参数有更多的选择权。
- `--full-gen-key` 比 `--gen-key`提供更加丰富的选项。

上面两个选项都是交互性的，我们也可以使用 `--quick-gen-key` 选项来一键生成，命令格式是：
>--quick-generate-key user-id [algo [usage [expire]]]

示例:

```
gpg --expert --quick-gen-key "Elon Musk(Test for lab) emusk@spacex.com" ed25519 cert 0
```

上述命令依然会弹出交互对话框询问你是否需要设置 passphrase，我们可以用选项 `--batch --passphrase ""` 来彻底一键化

```
gpg --expert --batch --passphrase "" --quick-gen-key "Elon Musk(Test for lab) emusk@spacex.com" ed25519 cert 0
```
注: 如果没有添加 `--expert` 选项，将无法选择 `ed25519` 算法。


**2. 编辑 Mster key，为其添加 subkey**

创建 3 个 subkey，分别赋予 Sign Only, Encrypt Only 和 Authenticate Only 功能，有效期均为 1 year。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5895a31af4f482fb99d0e0901da7e62~tplv-k3u1fbpfcp-watermark.image?)

需要注意的是：上面添加 Authenticate Only subkey 时，选择的是 toggle capability 方法，默认情况下 Sign, Encrypt 是启用的，Authentication 是禁用的，用二进制表示就是 110；因此，我们要仅启用 Authentication 的话，就需要变成 001。

**3. 导出并删除 Master Key**

添加完所有需要的 subkey 之后，Master Key 的主要工作就完成了，平时的 Signing, Encryption, Authentication 等功能都由 subkey 去完成。这时候我们应该把 Master Key 保存到一个离线的安全地方，以避免因为设备被盗取、破坏或丢失等因素导致 Master Key 丢失。

**什么时候需要用到 Master Key？**

-   when you sign someone else's key or revoke an existing signature,
-   when you add a new UID or mark an existing UID as primary,
-   when you create a new subkey,
-   when you revoke an existing UID or subkey,
-   when you change the preferences (e.g., with setpref) on a UID,
-   when you change the expiration date on your primary key or any of its subkey, or
-   when you revoke or generate a revocation certificate for the complete key.

参考来源：https://wiki.debian.org/Subkeys


接下来我们需要执行
-   导出 Master Key 的私钥，保存到离线 U 盘中。
-   创建 Master Key 的吊销证书，防止私钥丢失后无法吊销，保存到另一个 U 盘中。
-   在 Keyring 中删除 Master Key 的私钥。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/83437714ecff453eb7b9f756270b437d~tplv-k3u1fbpfcp-zoom-1.image)

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6eb876b545d84d61af8080d9b8053e43~tplv-k3u1fbpfcp-zoom-1.image)

验证 Master Key 私钥已经被删除

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f60c64a8d704d0fbe3ea3013b4b6489~tplv-k3u1fbpfcp-zoom-1.image)

当私钥被删除后，会出现 `sec#` ，否则显示为 `sec`。

4. **导出 subkey 的私钥和公钥**

我们可能在多台机器上工作，没必要在每台机器上都建立一个 GPG KEY 系统；所以我们把 subkey 的私钥导出到我们各个工作时需要用到的机器上。导出 subkey 的公钥为文件主要是方便我们分发给需要的人，比如你需要别人给你的公钥签名以增加信任，或者别人需要用你的公钥加密邮件发给你，等等。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc249f498918409aa833cfcdf68c33eb~tplv-k3u1fbpfcp-zoom-1.image)

**5. 操作实验**

**(1) Encrypting**

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99fdc24c10cb49cbb8c40c42a3d473e0~tplv-k3u1fbpfcp-zoom-1.image)

`--recipient` 指的就是接收方，你用接收方的公钥加密文件并发给对方，对方持有私钥，所以才能解密。

**(2) Decrypting**

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d5aedb5df3040c4a6b89b261c3cedb1~tplv-k3u1fbpfcp-zoom-1.image)

**(3) Signing keys**

当需要给别人的 public key 签名时，我们需要先导入 Master Key 的私钥（前面提到我们已经删除 Master Key）。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d1c2cf776984aac9119d1aafc953592~tplv-k3u1fbpfcp-zoom-1.image)

选项 `--default-key` 的意思是指定 signer，也就是我们用来给别人签名的 Master Key。如果没有指定，那么 gpg 会使用 keyring 中默认的 signer，通常是 keyring 中的第一个 Key。我们可以在 `~/.gnupg/gpg.conf` 中修改：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1478ae9038754a79b3e57e3dcdafcc87~tplv-k3u1fbpfcp-zoom-1.image)

**(4) Signing data**

gpg 提供 `--sign` 和 `--clearsign` 对数据进行签名，两个选项都会把签名与原数据文件打包成一个整体，--sign 还会对数据进行压缩，后者不压缩。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95dcc997207a4042a2017af0b63950c5~tplv-k3u1fbpfcp-zoom-1.image)

上述两个选项把签名和数据写在一起，所以对使用方来说很不方便，所以实际用途不大，更常见的还是把签名文件和数据文件分开来。gpg 提供 `--detach-sign` 选项来实现：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cab79364f9f64c5d8ea64fcf8a69cebc~tplv-k3u1fbpfcp-zoom-1.image)

**(5) Revoking**

Revoke 分两种：Revoke Master Key 和 Revoke Subkey

**注意：只有 Master Key 才能 Revoke 自己和 subkey，所以操作之前记得先把 Master Key 私钥导入。**

**1) Revoke Master Key**

**主要分三步：**
- 生成 Revoke Key 
实际上这一步可以省略，还记得在执行 `gpg --gen-key` 的时候，会默认生成一个 revoke key 在 `~/.gnupg/openpgp-revocs.d/` 目录吗？
- 把 Revoke key 导入 Keyring
- 把 Revoke key 分发出去（上传到 key server 或传给别人）

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfb3148830fe415baf5ede581bbbb245~tplv-k3u1fbpfcp-zoom-1.image)

**2. Revoke subkey**

通过编辑 Master Key 实现：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e4e024246794c2ab2194cfdcecf0141~tplv-k3u1fbpfcp-zoom-1.image)

完成操作之后，不要忘记使用 `--send-keys [keyid]` 命令选项把 key 更新到 Key Server，以便别人的值你的 subkey 已经被吊销。

## 第四部分：使用 GPG Keys

看到这里，应该已经熟悉了 GPG 工具的大部分操作方法，个人的 GPG KEY 也已经建立得相对完善了，那就回到开头第一部分介绍的应用场景，简单介绍一下如何实现。

有了上面的理论基础，下面的步骤和操作都将变得简单：

**1. 使用 GPG Key 对 git commit 进行签名**

步骤如下：
-   导出具有 Sign 功能的 subkey 的公钥
-   把公钥导入到 Github 设置页面
-   设置本地的 git client，以便它使用 keyring 中的私钥对 commit 签名。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/897bbf5079d8496d8140586cd84f8a29~tplv-k3u1fbpfcp-zoom-1.image)

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9bc626de6bfe402a875988df6a366329~tplv-k3u1fbpfcp-zoom-1.image)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f67c23586434135812cae3ec6bc027d~tplv-k3u1fbpfcp-watermark.image?)

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c15b7fab0c374fd6bbb6750b2dcaa16c~tplv-k3u1fbpfcp-zoom-1.image)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19fc12b7f3254b4c8232d22c695d1c30~tplv-k3u1fbpfcp-watermark.image?)

**2. 使用 GPG Key 加密邮件**

如果使用 Gmail，我们可以安装一个浏览器插件 Flowcrypt，它支持 Chrome, Firefox, Brave 三个浏览器，同时还提供 Androd 和 iOS App。这个插件可以自动使用我们 GPG Key 对 Gmail 邮件（不支持其它邮箱）加密。

操作方法非常简单，安装插件时会提示你创建或者导入 GPG Key，由于我们已经建立了自己的 GPG Keys，所以我们直接导入有 Encryption 功能的 subkey 的私钥。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4d79f42e2a0452ba144b2e3479dc5c9~tplv-k3u1fbpfcp-watermark.image?)

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5f17f89fd404f4cade1bee3deb3c5f5~tplv-k3u1fbpfcp-zoom-1.image)

完成设置后，Flowcrypt 会把我们的公钥上传到它的 Key Server 中，这样其它也安装了 Flowcrypt 插件的人也可以获得我们的公钥，也就能给我们发加密邮件了。

打开邮箱后，我们会收到 Flowcrypt 用我们的公钥加密的一封邮件，我们打开后立即是解密状态，这是因为 Flowcrypt 用我们的私钥自动解密了。

需要记住的是，我们目前还不能给别人发加密邮件，因为我们没有别人的公钥。

所以，如果我们想要给别人发加密邮件，就需要有他/她的一个公钥。目前 Flowcrypt 虽然有自己的 Key Server，但是并不支持手动上传公钥，因此要求对方也安装 Flowcrypt，这样他/她的公钥也在 Flowcrypt 的 Key Server 上，这边就能自动检索到。

下面演示一下如何给别人发加密邮件：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74961fdcd79f4ad88a83339faf26a6cb~tplv-k3u1fbpfcp-zoom-1.image)

地址栏里的 recipient 就是你的发送对象，如果对方也安装了 Flowcrypt 并载入过私钥，那么这里就会显示绿色，说明你可以发加密邮件给对方。

另外，如果你的邮件客户端需要支持 GPG Key，否则收到加密邮件后无法自动解密。上面的 Chrome 浏览器是就是 Gmail 的客户端，我们安装了 Flowcrypt 所以它可以帮我们自动解密，如果我们换成用其它客户端（比如 Outlook 等）打开邮件，则会显示密文，如下：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/757690f379ca4cab89650eee40ecea6e~tplv-k3u1fbpfcp-watermark.image?)

在 Thunderbird 中导入对应的 GPG 私钥，


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b7113936c4f42cca929f0467bf2ac76~tplv-k3u1fbpfcp-watermark.image?)

又可以自动解密了。


**3. 使用 GPG Key 做 SSH 认证**

做法很简单，步骤如下：

-1) 导出 SSH 格式的公钥，并上传到服务器

```
gpg --export-ssh-keys 64810DE8 > ~/.ssh/gpg_subkey.pubssh-copy-id -i ~/.ssh/gpg_subkey.pub server:./ssh servercat gpg_subkey.pub >> ~/.ssh/authorized_keys
```

- 2) 关掉 ssh-agent，启动 gpg-agent

```
echo enable-ssh-support >> $HOME/.gnupg/gpg-agent.conf
cat >> ~/.bashrc << EOFunset SSH_AGENT_PIDif [ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ]; then  export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"fiexport GPG_TTY=$(tty)EOF
gpg-connect-agent updatestartuptty /bye >/dev/null
```

- 3) 测试登陆 SSH

```
ssh -T git@github.com
```


#### Reference:

https://datatracker.ietf.org/doc/html/rfc4880

https://infra.apache.org/openpgp.html

https://blog.djoproject.net/2020/05/03/main-differences-between-a-gnupg-fingerprint-a-ssh-fingerprint-and-a-keygrip/

https://wiki.archlinux.org/index.php/GnuPG#SSH_agent

https://www.linode.com/docs/security/authentication/gpg-key-for-ssh-authentication/

https://wiki.debian.org/Subkeys

https://superuser.com/questions/1113308/what-is-the-relationship-between-an-openpgp-key-and-its-subkey?newreg=0aa6047192a443b897b6489403efea0e

https://unix.stackexchange.com/questions/339077/set-default-key-in-gpg-for-signing

https://danielpecos.com/2019/03/30/how-to-rotate-your-openpgp-gnupg-keys/

https://oguya.ch/posts/2016-04-01-gpg-subkeys/

https://www.reddit.com/r/pgp/comments/7wf7e1/confusion_re_master_keys_vs_subkeys/du1f4dg/

https://security.stackexchange.com/questions/104059/help-me-understand-the-relationship-between-gpg-public-keys-sub-keys-and-expira

https://davesteele.github.io/gpg/2014/09/20/anatomy-of-a-gpg-key/

https://www.digitalneanderthal.com/post/gpg/

https://dev.to/benjaminblack/signing-git-commits-with-modern-encryption-1koh

https://fedoraproject.org/wiki/Creating_GPG_Keys

https://github.com/gpg/gnupg

https://github.com/gpg/libgcrypt

https://security.stackexchange.com/questions/181453/why-does-github-need-to-offer-a-gpg-signature-feature?rq=1

https://flowcrypt.com/pub

https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits


如果你觉得我的文章对你有帮助，欢迎留言或者关注我的专栏。


微信公众号：“知辉”


搜索“codeandroad”


