### Tower 

Tower 是 MacOS 上的一款可视化的 Git 工具，提供比较方便的操作。

### GPG sign

我们想要在 Tower 中使用 gpg 对 commit 进行 sign 操作

首先确保已经配置了使用 gpg sign

```
git config --global gpg.program /usr/homebrew/bin/gpg
git config --global user.signingkey A6B167E1 
git config --global commit.gpgsign true
```

由于设置了密码，我们还应设置 `pinentry-program` 程序

~/.gnupg/gpg-agent.conf

```
pinentry-program /opt/homebrew/bin/pinentry-mac
```


在使用 tower 操作前，我们可以先尝试在命令行执行 `git commit -S -m "xxx"` 来验证 gpg sign 功能正常。

如果正常，我们再集成使用 tower。

为了使得 tower 能够使用 gpg configuration，我们需要添加 `no-tty` 选项，如下：

```
echo no-tty >> ~/.gnupg/gpg.conf
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/tower_gpg_sign_commit.jpg)

参考：

https://gist.github.com/LeonardoCardoso/4638744787a527d5e1965a9b6d302978
https://gist.github.com/dominikwilkowski/29af2e098d5b364f9130bd307f37c0bf