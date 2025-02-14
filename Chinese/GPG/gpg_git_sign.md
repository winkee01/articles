
配置好 git 使用 gpg 进行签名之后，我们需要使用 -S 选项来触发签名

```
git commit -S -m "Fix bug"
```

如果出现如下错误：

```
error: gpg failed to sign the data
fatal: failed to write commit object
```

我们可以用下面这个命令去测试一下 gpg key 是否正常使用：
```
echo "test" | gpg --clearsign
```

如果弹出类似如下的错误，说明，key 无法正常使用：

```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

test
gpg: signing failed: Inappropriate ioctl for device
gpg: [stdin]: clear-sign failed: Inappropriate ioctl for device
```

主要有两个可能的原因：

-（1）密钥 ID 配置错误。

-（2）没有创建具有 Sign 功能的 subkey

-（3）缺少某个关键组件。 比如，你创建的 gpg key 配置了密码保护，但是你的命令行没有安装 pinentry 这个组件。

**我遇到的就是（3）这个情况**，创建 gpg key 时设置了密码保护，也正常使用。
但是，时间久了，我有一次清理 Homebrew 所安装的软件，导致了 pinentry 这个组件被误删。

pinentry 这个组件的作用是弹出密码填写框，我们需要在里面填写密码才能解锁使用 gpg key。
如果缺少这个组件或者组件没有被 gpg-agent 加载，就没法弹出密码填写页，自然就会导致命令失败了。

如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/pinentry_mac.jpg)


重新安装这个组件

```
brew install pinentry-mac
```

在 `gpg-agent.conf` 中写入正确的组件可执行路径：

```
echo "pinentry-program /opt/homebrew/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf

```

查看所有已配置组件

```
gpgconf --list-components
```

重启 gpg-agent

```
gpgconf --kill gpg-agent
gpgconf --launch gpg-agent
```

再次测试 gpg key 是否工作正常
```
echo "test" | gpg --clearsign
```

输出结果
```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

test
-----BEGIN PGP SIGNATURE-----

iHUEARYKAB0WIQT4lNCOgqrtEAeNx1NCqPM+tBVCBQUCZ5En4gAKCRBCqPM+tBVC
BTLcAQDLZnPY6lmZNEZdJCnXbW6BAt40LF2FANbj6F7txJhtZgEAxsr4NGIVoazj
+ZEc1qorTfDdmukKElJClxDLztan/gA=
=53YQ
-----END PGP SIGNATURE-----
```


至此，gpg key 就能正常进行 git commit 签名了。。


Github 验证

我们需要把公钥上传到 Github 中才可以对签名进行验证

```
gpg --armor --export 71567BD2
```

签名验证的结果：

- **Verified**    The commit is signed, the signature was successfully verified, and the committer is the only author who has enabled vigilant mode.
- **Partially verified**  The commit is signed, and the signature was successfully verified, but the commit has an author who: a) is not the committer and b) has enabled vigilant mode. In this case, the commit signature doesn't guarantee the consent of the author, so the commit is only partially verified.
- **Unverified**  Any of the following is true:
    - The commit is signed but the signature could not be verified.
    - The commit is not signed and the committer has enabled vigilant mode.
    - The commit is not signed and an author has enabled vigilant mode.

如果我们使用的 gpg key 的 email 与 github.com 账户的 email 不相同，那么会显示 Unverified。
比如，你创建 GPG Key 的时候，使用的 email 是 123@example.org，但是你的 github.com 账户的 email 是 abc@example.org。那么你用这个 GPG key 去签名 commit 的时候，你就会看到 Unverified 结果。












