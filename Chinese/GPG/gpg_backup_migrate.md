## GPG 密钥备份

这篇文章介绍如何把 GPG 密钥进行备份

#### 1. 公钥和私钥分别存储

gpg 把公钥和私钥分别存储的。

公钥存储在 `~/.gnupg/pubring.kbx`（旧版本）或 `~/.gnupg/public-keys.d/pubring.db`（新版本）中。
而私钥存储在 `~/.gnupg/private-keys-v1.d/` 目录中。

我们可以对它们分别导出：

```
# 导出公钥
gpg --export --output my-public-key.asc

# 导出私钥
gpg --export-secret-keys --output my-private-key.asc


# 导出信任

gpg --export-ownertrust > ~/ownertrust.txt
```


#### 2. 恢复（导入）

通常情况下，最好分别导出公钥和私钥，以便你可以单独备份和使用公钥进行加密操作。然而，如果你只导出了私钥，你仍然可以恢复公钥。

下面就以只导出了私钥的情况下，导入私钥，并以此恢复公钥。

###### 先导入私钥
```
gpg --import my-private-key.asc
```

注意：导入私钥后，这个私钥可能出于不被信任的状态，比如状态显示为 unknown。如果我们用这个 key 去 sign commit，就会出现如下提示：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/unknow_sign_key.png)

这时候，我们需要手动去信任它。

步骤如下：

```
gpg --edit-key CEF60727
```

进入修改界面后，我们输入 trust，如下：

```
gpg> trust
```

会出现如下提示：

```
...
[ unknown] (1). Winkee Sail (Master Key Ed25519) <winkee01@gmail.com>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y
```

如上，输入 5，也就是终极信任。

这样，当我们再验证 `gpg --list-keys` 时，信任状态就会出现 `[ultimate]` 了。


再次用它来进行 sign commit，就没问题了：
![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ultimate_sign_key.png)

###### 重新生成公钥（恢复公钥）

```
gpg --export --armor --output regenerated-public-key.asc

```

###### 导入公钥

```
gpg --import regenerated-public-key.asc
```


###### 导入信任

```
gpg --import-ownertrust ~/ownertrust.txt
```


#### 3. 验证

```
gpg --list-keys
```

输出示例
```
pub   rsa2048 2025-01-01 [SC]
      ABCDEF1234567890ABCDEF1234567890ABCDEF12
uid           [ultimate] Your Name <your-email@example.com>
```


#### 4. 导出完整密钥对

分别导出恢复太麻烦了，我们可以简化备份和恢复操作，可以直接导出公钥和私钥的组合数据（即完整密钥对）。


###### (1) 导出完整密钥对

```
gpg --export-secret-keys --armor --output full-keypair.asc
```

###### (2) 恢复完整密钥对

```
gpg --import full-keypair.asc
```


#### 5. 为什么建议单独导出公钥和私钥？

###### 备份灵活性：
- 公钥是公开数据，安全风险较低，通常会分享给其他人，用于加密消息或验证签名。
- 私钥是敏感数据，必须严格保护，因此私钥备份需要额外注意安全性。

###### 权限分离：
有时你只需要将公钥分发给其他人，而不需要共享完整的密钥对。



全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad 。




