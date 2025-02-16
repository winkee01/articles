
#### gpg 命令操作实验

这篇文章通过一些练习来掌握 gpg 不同功能的用法

##### (1) Encrypting 非对称加密
使用 `--encrypt` 是用公钥进行加密（非对称加密），需要指定所使用公钥的 UID，这里使用 `--recipient` 选项来指定，你用接收方的公钥加密文件并发给对方，对方持有私钥，所以才能解密。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_encrypting.png)

##### (2) Decrypting 非对称解密
使用 `--decrypt` 是用私钥进行解密，同样需要用 `--recipient` 指定所使用的密钥的 UID。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_decrypting.png)



##### (3) 对称加解密

```bash
gpg --symmetric --cipher-algo AES256 ~/.ssh/id_rsa
gpg --decrypt id_rsa.gpg > ~/.ssh/id_rsa
```

##### (4) Signing keys 对密钥签名

当需要给别人的 public key 签名时，我们需要先导入 Master Key 的私钥（前面提到我们已经删除 Master Key）。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_signing.png)

选项 `--default-key` 的意思是指定 signer，也就是我们用来给别人签名的 Master Key。如果没有指定，那么 gpg 会使用 keyring 中默认的 signer，通常是 keyring 中的第一个 Key。我们可以在 `~/.gnupg/gpg.conf` 中修改：

```bash
~$ head -1 ~/.gnupg/gpg.conf 
default-key emusk@spacex.com
```

##### (5) Signing data 对数据签名

gpg 提供 `--sign` 和 `--clearsign` 对数据进行签名，两个选项都会把签名与原数据文件打包成一个整体，`--sign` 还会对数据进行压缩，后者不压缩。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_signing_data.png)

上述两个选项把签名和数据写在一起，所以对使用方来说很不方便，所以实际用途不大，更常见的还是把签名文件和数据文件分开来。gpg 提供 `--detach-sign` 选项来实现：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_detach_sign.png)

##### (6) Revoking

Revoke 分两种：Revoke Master Key 和 Revoke Subkey

**注意：只有 Master Key 才能 Revoke 自己和 subkey，所以操作之前记得先把 Master Key 私钥导入。**

###### 1) Revoke Master Key

**主要分三步：**
- 生成 Revoke Key 
实际上这一步可以省略，还记得在执行 `gpg --gen-key` 的时候，会默认生成一个 revoke key 在 `~/.gnupg/openpgp-revocs.d/` 目录吗？
- 把 Revoke key 导入 Keyring
- 把 Revoke key 分发出去（上传到 key server 或传给别人）

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_revoke_master.png)

###### 2) Revoke subkey

通过编辑 Master Key 实现：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/gpg_revoke_subkey.png)

完成操作之后，不要忘记使用 `--send-keys [keyid]` 命令选项把 key 更新到 Key Server，以便别人的值你的 subkey 已经被吊销。

全文完！
