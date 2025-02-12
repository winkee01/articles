## 简要版：使用 GPG 工具创建本地密钥链体系

### 密钥链层级结构图

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_allkeys_diagram.png)

#### 为什么要构建这么多 Key？

主要是基于网络安全中的最小权限和职责分离原则。

- Master Key 作为类似于 PKI 体系中的 Root CA 的功能，永不过期，只有在需要签发新的 subkey 和吊销 subkey 的时候才出来干活，平时放置在一个离线的安全存储中（比如硬件密钥盘 Yubikey）。
- 各个具有不同功能的 subkey 则作为真正的打工人，负责平时每日的干活，有效期短，过期后需要使用 Master key 签发新的。



#### 1. 创建 Master keys

Master Key 应该只用于 Certify，使用安全系数最高的 Ed25519 椭圆曲线算法，并且永远不过期。

```
gpg --expert --batch --passphrase "abc123" --quick-gen-key "Codenroad(Local Lab) alex924@codenroad.biz" ed25519 cert 0
```
注: 
- 这里使用 `--batch` 和 `--passphrase`两个选项来避免交互。效果是可以一键生成密钥。
- 如果没有添加 `--expert` 选项，将无法选择 `ed25519` 算法。
- passphrase 我们最好填写一个由密码软件所生成的随机且复杂的密码

效果：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_batch_masterkey.png)

查看：

```
gpg --list-secret-keys --keyid-format=short
```

命令主要会生成下面两个文件：

- 撤销证书：`openpgp-revocs.d/076CC6E463410ED3BE7F667D63BD63126141AA08.rev`
- 私钥：`private-keys-v1.d/58156E0F96DC0403390EFF9545A1CA8C2461F40E.key`

解释：

- 后面的这串 `076CC6E463410ED3BE7F667D63BD63126141AA08` 是 fingerprint。
- 撤销证书的作用是当私钥泄漏或丢失的时候，宣布该私钥作废。

我们可以通过下面的命令查看撤销证书的内容：

```
gpg --import-options show-only --dry-run --import ~/.gnupg/openpgp-revocs.d/076CC6E463410ED3BE7F667D63BD63126141AA08.rev
```

如果这个文件丢失，只要私钥还在，我们可以手动生成：

```
gpg --output new-revocation-certificate.asc --gen-revoke <masterkey-fingerprint>
```
后面的 `6141AA08` 是 fingerprint，我们用后 8 位即可。 

注意：撤销证书非常重要，千万不要泄漏它们，应该跟私钥一样，离线保存。


##### 关于 passphrase

不要手动输入因为可能是弱密码，建议使用 Apple 在 2024 年 6 月新发布的 **Passwords** 工具来创建和管理这个密码。

注：以前是使用 Keychain Access 来管理密钥和密码，但是现在用 Passwords 工具来管理密码更方便了。

另外，关于密码的管理，还可以参考我另一篇文章《如何管理你的密码》。

**更多参数解释：**

- `--expert` 是启用专家模式，让你对参数有更多的选择权。
- `--full-gen-key` 比 `--gen-key`提供更加丰富的选项。
- `--quick-gen-key` 可以避免交互。后面跟的值是 UserID，也就是 Email 信息。


#### 2. 编辑 Master key，为其添加 subkey

接下来，我们创建 3 个 subkey，分别赋予 **Sign Only**, **Encrypt Only** 和 **Authenticate Only** 功能，有效期均为 1 year。

```
gpg --expert --edit-key <keyid>
```

**Sign Only**
![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_addkey_sign_only.png)

**Encrypt Only**

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_addkey_encrypt_only.png)

**Authenticate Only**

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_addkey_authenticate_only.png)

**解释：** 上面添加 Authenticate Only subkey 时，选择的是 toggle capability 方法，默认情况下 Sign 和 Encrypt 是启用的，Authentication 是禁用的，用二进制表示就是 110；因此，我们要仅启用 Authentication 的话，就需要变成 001。

#### 3. 导出并删除 Master Key

添加完所有需要的 subkey 之后，Master Key 的工作就完成了（目的就是 **Certify** subkeys），这时候我们应该把 Master Key 保存到一个离线的安全地方，以避免因为设备被盗取、破坏或丢失等因素导致 Master Key 丢失。

日常的工作比如 Signing, Encryption, Authentication 等都交由 subkey 去完成。

**什么时候需要用到 Master Key？**

-   当你需要对别人（或自己）的公钥签名、或者需要吊销一个 subkey 时
-   当你需要新增一个 UID，或者需要标记一个 UID 为 primary 时
-   当你需要新增一个 subkey
-   当你需要吊销一个 UID 或 subkey
-   当你需要改变一个 UID 的设置 (e.g., with setpref) 
-   当你需要改变 primary key 或 它的任何 subkey 的过期日
-   当你需要吊销，或者生成一个吊销证书时

参考来源：https://wiki.debian.org/Subkeys


接下来我们需要执行（如果仅是实验目的，可以暂时忽略）
-   导出 Master Key 的私钥，保存到离线 U 盘中。
-   创建 Master Key 的吊销证书，防止 Master Key 的私钥丢失后无法吊销，保存到另一个 U 盘中。
-   在 Keyring 中删除 Master Key 的私钥。
-   验证

###### 导出私钥
```
gpg --export-secret-keys --armor --output my-private-key.asc 6141AA08
```

###### 创建 revocation certificate
略，前面已经介绍过。

###### 删除 private key

```
gpg --delete-secret-keys 6141AA08
```

注意，删除私钥的过程中会问你是否删除 subkey，记得勾选 **NO**


###### 验证 Master Key 私钥已经被删除

删除前
![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_key_before_deleted.png)

删除时
![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_key_before_deleted.png)

删除后
![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_key_after_deleted.png)

当私钥被删除后，会出现 `sec#` ，否则显示为 `sec`。

记得把 Master Key 保存到一个离线安全的地方。

#### 4. 导出 subkey 的私钥和公钥

我们可能在多台机器上工作，没必要在每台机器上都建立一个 GPG KEY 系统；因此，我们需要把 subkey 的**私钥**导出到我们各个工作时需要用到的机器上。此外，我们可能还需要把 subkey 的公钥也导出为文件，这样可以方便我们分发给需要的人；比如你需要别人给你的公钥签名以增加信任，或者别人需要用你的公钥加密邮件发给你，等等。

导出私钥

```
gpg --export-secret-subkeys -ao subkey_E_priv_45116F3D.gpg 45116F3D
```

导出公钥为文件
```
gpg --export -ao subkey_AD085F42_pub.gpg 45116F3D
```

导出公钥到屏幕
```
gpg --armor --export 45116F3D
```

导入私钥

```
gpg --import subkey_E_priv_45116F3D.gpg
```

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad 。

