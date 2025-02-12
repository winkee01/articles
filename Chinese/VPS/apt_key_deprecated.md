#### 背景介绍

某一天早上我们起床，登陆服务器，像往常一样执行 `apt update` 时，我们发现失败了，命令行出现一行警告提示：

> Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).

又或者，某一天，我们添加了一个第三方源，并用如下方式把下载好的这个源所使用的公钥添加进 GPG 的 keyring，

```
wget -qO- https://myrepo.example/myrepo.asc | sudo apt-key add -

or

sudo apt-key add myrepo.asc
```

同样出现这个 `apt-key is deprecated` 警告。

#### 原因调查

我们知道，每个源都必须要有一个公钥（public key）用于认证这个源的合法性（真实性）。系统自带的受信任的源的公钥都是默认保存在 `/etc/apt/trusted.gpg` 这个文件中的，它是 gpg 默认的  keyring 数据库文件，包含了所有的受信任的公钥信息。

输入以下命令查看：

```
sudo apt-key list
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
/etc/apt/trusted.gpg
--------------------
pub   rsa4096 2021-07-14 [SC]
      3E9C 8769 07A5 60AC A009  64F3 63E9 BAD2 15BB F5F0
uid           [ unknown] Ekaterina Buryanova Dimitrova (CODE SIGNING KEY) <e.dimitrova@gmail.com>
sub   rsa4096 2021-07-14 [E]

pub   rsa4096 2014-09-05 [SC]
      514A 2AD6 31A5 7A16 DD00  47EC 749D 6EEC 0353 B12C
uid           [ unknown] T Jake Luciani <jake@apache.org>
    sub   rsa4096 2014-09-05 [E]

Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
/etc/apt/trusted.gpg.d/debian-archive-buster-stable.asc
-------------------------------------------------------
pub   rsa4096 2019-02-05 [SC] [expires: 2027-02-03]
      6D33 866E DD8F FA41 C014  3AED DCC9 EFBF 77E1 1517
uid           [ unknown] Debian Stable Release Key (10/buster) <debian-release@lists.debian.org>


```


可以看到，`apt-key list` 命令会把 `/etc/apt/trusted.gpg` 文件中保存的 keys 和 `/etc/apt/trusted.gpg.d/` 目录中保存的 keys 都列出来，并且还报出了 `Warning: apt-key is deprecated` 警告。

之所以出现警告：
第一个原因就是 `apt-key` 已经标记为丢弃了，不该再继续使用。
另外的原因就是：`/etc/apt/trusted.gpg` 中包含了我们使用 `apt-key add` 命令添加过的 keys。
简单来说就是：执行 `apt-key add` 命令添加 key 时，这个 key 会被默认保存到 `/etc/apt/trusted.gpg`  文件中（如果没有则会自动创建），而这个文件又是一个全局作用范围的文件，一旦把第三方源的公钥添加到这个文件中，这容易影响整个系统的安全。
出于安全因素的考虑，Ubuntu/Debian 把 `apt-key` 这个命令标记为了 deprecated，不再推荐使用这个命令来管理 key。

还要注意一点的是：警告中提示我们 keys 应该放在 `/etc/apt/trusted.gpg.d` 目录中，但我们执行 `apt-key list` 的话，依然会得到如下警告：

> Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).

这个警告是去不掉的，原因就是 `apt-key` 已经标记为 deprecated。只要我们使用 `apt-key` 这个命令，这个警告就会有。**这也明确告诉我们，不要再使用这个命令了，它已经被彻底的丢弃了！**

#### 解决方案

找到了原因，解决起来就简单了。

首先，不要再使用 `apt-key` 这个命令了，因为它会直接操作 `/etc/apt/trusted.gpg` 从而影响我们整个系统的安全。

其次，官方推荐我们使用 `/etc/apt/trusted.gpg.d` 目录来管理可信 keys。我们手动把 key 复制到这个目录中即可。

不过，要注意的是，`/etc/apt/trusted.gpg.d` 依然是 apt 官方可信 key 目录，依然会影响全局。

我建议只有自己创建的 key 才放在这个目录，毕竟自己是可信的。

而对于一个第三方源的 key，更加安全的方式是把它放在另一个单独的目录中。比如 `/etc/apt/keyrings/` 中。然后，在 `/etc/apt/sources.list.d/3rd-repo.list` 中使用 `signed-by` 命令来指定使用这个 key，如下：

```
deb [signed-by=/usr/share/keyrings/myrepo-keyring.pgp] <repository-url> <distribution> <components>
deb [signed-by=/usr/share/keyrings/myrepo-keyring.pgp] https://thirdpartyrepo.com/ubuntu/ jammy main
```

这样，我们就解决了 `apt-key` 这个过期命令给我们的困扰。

#### 删除 /etc/apt/trusted.gpg 中的 旧 keys。

前面我们提到如果我们使用 `apt-key` 命令添加过 keys，那我们执行 `apt update` 就依然会出现上述警告信息。这时候，我们需要把 `/etc/apt/trusted.gpg` 中的 old keys 删掉。

命令格式如下：

```
sudo apt-key del <KEY-ID>

sudo apt-key del "15BBF5F0"
```

注：这里的 `15BBF5F0` 是密钥的最后 8 位。

## 扩展知识

`apt-key` 只接受二进制格式的 key，因此，如果你强行要使用 `apt-key` 的话，那么请确保你的 key 是二进制格式的，如果不是（比如套了个 ASCII 壳），那么你需要先转换成二进制。

转换的方法可以参考如下步骤：

```bash
gpg --dearmor <unknown-repo>.gpg
```

Tip 1: 我们可以在下载的时候直接 --dearmor，如下：

```
wget -O- <https://foobar.com/key/<unknow-repo>.gpg> | gpg --dearmor | \
  sudo tee /usr/share/keyrings/<unknown-repo>-archive-keyring.gpg
```

Tip 2：
使用 tee 命令，会导致内容都出现在屏幕中，从而 terminal 上的显示会有点被 garbled。比如：

```
wget -O- http://www.webmin.com/jcameron-key.asc | gpg --dearmor |  tee webminrepo-key.gpg
```

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/c011705e9d6045dbae9b9a2db7f85558~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ29kZUFuZFJvYWQ=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNzYxMjgyMDA5NTA3OTAxIn0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1731533357&x-orig-sign=myrkjZ3vSO0oRAMigP87TxQH2LE%3D)

使用 dd 命令则不会上述情况：

```
wget -O- http://www.webmin.com/jcameron-key.asc | gpg --dearmor  | dd of=webminrepo-key.gpg
```

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/67b1e12df4ff40d483bf61b281f4d983~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ29kZUFuZFJvYWQ=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNzYxMjgyMDA5NTA3OTAxIn0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1731533357&x-orig-sign=WgXrAmgvgJj7p4DfFCB3J1P599Q%3D)

#### APT trusted Keys vs GnuPG User Keyring

注意，虽然 APT 使用的是 GnuPG 格式的 Keys，但是它们二者的 keystore 目录是不同的。

对于 APT 来说，通常是 `/etc/apt/trusted.gpg`, `/etc/apt/trusted.gpg.d` 或 用户自定义的其他目录。

而 GnuPG User Keyring 则默认使用 `~/.gnupg` 这个目录。

假如我们想用 gpg 命令来查看 APT trusted keys，我们可以这样：

```
gpg --list-keys --no-default-keyring --keyring /etc/apt/trusted.gpg.d/ubuntu-keyring-2018-archive.gpg 
```

GnuPG 默认的 keyring 文件是 `~/.gnupg/pubring.kbx`，如果我们直接执行 `gpg --list-keys` 命令，那就是把 `~/.gnupg/pubring.kbx` 中保存的 keys 信息列出来。

而在这里，我们使用 `--no-default-keyring` 选项的意思是列出 GnuPG 默认的这个 `~/.gnupg/pubring.kbx` 文件中保存的 keys 信息，而是使用 `--keyring` 选项指定的文件来查看 key 信息。这样，我们就可以查看指定的 key 文件的详细信息。（如果直接执行 `gpg --list-keys`，它会去 `~/.gnupg/pubring.kbx` 文件中查找 keys 信息，而 `ubuntu-keyring-2018-archive.gpg` 文件在 `/etc/apt/trusted.gpg.d/` 中，自然就不会显示了）。

另外，我们还可以用 `--dry-run` 选项，也可以避免把 key 导入到 `~/.gnupg/pubring.kbx` 中去，如下：

```
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```


#### 如何列出 APT trusted keys 信息？

现在我们已经彻底不用 `apt-key` 命令了，有什么办法可以依然把所有 APT trusted keys 信息列出来吗？


很简单：

```bash
for key in /etc/apt/trusted.gpg.d/*.gpg; do
    gpg --list-keys --no-default-keyring --keyring "$key" 
done
```

#### 把 `/etc/apt/trusted.gpg.d/`中的 keys 导入到 GPG default keyring 中:

虽然 `/etc/apt/trusted.gpg.d/` 目录中的 key 的格式都是 armor 过的 PEM 格式，但是我们可以直接导入。

```
gpg --import /etc/apt/trusted.gpg.d/ubuntu-keyring-2018-archive.gpg
```

该命令会把 key 导入到 `~/.gnupg/pubring.kbx` 中，我们再执行 `gpg --list-keys` 就可以看到 key 信息了。


#### 从 GPG keyring 中删除一条 key

```
gpg --delete-key <key-id>
```

这里的 <key-id> 可以是 fingerprint 的后 8 位。


#### Apt repository 使用 GPG keys

由于 GPG keys 没有保存在系统目录中，apt 命令想要自动使用 GPG keys，我们需要手动添加引用。
在 `/etc/apt/sources.list.d/` 目录中添加一个以自定义命名的文件，然后通过 `signed-by` 选项来设置需要使用的 key。


下面是一个引用 Nginx gpg key 的例子：

```
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list

echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx

echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```


全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。
