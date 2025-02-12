# Title: 多个 SSH KEY 的管理


## SSH 登录

我们先来看一下我们使用 ssh 登录远程服务器时，我们是怎么配置本地 `~/.ssh/config` 文件的。

假如我们有这样一个配置：

```
Host myserver
HostName 192.168.131.12
User tony
IdentityFile ~/.ssh/id_rsa
PubKeyAuthentication yes
```

解释：

配置中的 Host 是别名的意思，我们可以用它来代替主机名。

这样配置之后，我们就可以用下列命令登录服务器：

```
ssh myserver
```

相比于 要输入 `ssh tony@192.168.131.12` 是不是简单很多？ 这是因为这里的 myserver 会被自动替换。

另外，如果我们在 `/etc/hosts` 中添加了对 myserver 的地址映射，比如

```
192.168.131.12 myserver
```

那么我们这里还可以免去 `HostName 192.168.131.12` 这一行。


现在，我们回到 github.com 一个账户对应多个项目的问题

我们知道，Github 上的 repo 支持两种协议进行认证，HTTPS 和 GIT。

- HTTPS 协议： https://github.com/username/proj.git 
- SSH 协议：   git@github.com:username/proj.git 
- GIT 协议：   git://github.com/username/proj.git



- 游客通常使用 HTTPS 协议访问 repo。
- Owner 则使用 SSH 协议访问 repo。
- GIT 协议不常用，这里不展开介绍。

作为 Owner，我们 remote-url 会设置成 `git@github.com:username/proj.git` 的形式。

现在假设我们在 github.com 同一个账户上有多个项目（以 project1, project2 为例）
为了方便我们管理 ssh 登录，pull，push 等动作，

首先我们最好按如下方式配置 `~/.ssh/config` 文件：

```ssh
Host project1 
HostName github.com
User git
IdentityFile ~/.ssh/id_rsa_1

Host project2
HostName github.com
User git
IdentityFile ~/.ssh/id_rsa_2
```

现在，我们只需要用如下方式，就可以正确访问两个同名账户下的不同 Projects 了：

```
git clone git@project1:user1/helloworld1.git
git clone git@project2:user1/helloworld2.git
```


与之前 SSH 登录类似，这里的 project1 和 project2 分别对应了两个不同的 SSH 配置，
并且 HostName 都是 github.com，但是这两个别名对应了两个不同的私钥，我们可以正确访问不同的 repo。

####怎么设置 origin？
假设，我们原始的 origin 信息如下：
```
git remote add origin git@github.com:winkee01/mimal.git
```

我们可以这样来修改：
```
git remote set-url origin git@project1:winkee01/mimal.git
```

现在，我们执行下面的命令就可以推送到正确的远程仓库了：
```
git push -u origin master

```

最后，有一点需要提示的是，虽然配置中还定义了 `User git`，不过测试下来发现，这个值并不会被访问到。
所以 User 的值设置与否都不重要，完全不影响连接成功。

不过要注意的是，既然我们的代码托管在 github.com 网站上，那么我们使用 git 命令或者 ssh 命令对远程库操作时，就相当于我们要向这个服务器进行认证，而这个服务器允许我们认证的 username 就是 git。
也就是说，它只开放了 git 用户给我们向服务器进行认证，而我们真正的用户是通过 git 再进一步操作的。

所以，ssh 配置中，我们配置 User 时，为避免混淆，就写成 git。

比如 `git clone git@project1:user1/helloworld1.git` 命令中，`@` 符号前面的 git 就是用户名。

我们可以用 ssh 命令来验证一下认证是否成功：

```
ssh -vT git@github.com
```

只要你的本地私钥认证了远程仓库中的公钥，你就可以得到如下成功认证的信息：

```
> Hi USERNAME! You've successfully authenticated...
```

参考：


https://docs.github.com/zh/authentication/troubleshooting-ssh/error-permission-denied-publickey



全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。



