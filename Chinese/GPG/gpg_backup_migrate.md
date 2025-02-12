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
```


#### 2. 恢复（导入）

通常情况下，最好分别导出公钥和私钥，以便你可以单独备份和使用公钥进行加密操作。然而，如果你只导出了私钥，你仍然可以恢复公钥。

下面就以只导出了私钥的情况下，导入私钥，并以此恢复公钥。

先导入私钥
```
gpg --import my-private-key.asc
```

重新生成公钥（恢复公钥）
```
gpg --export --armor --output regenerated-public-key.asc

```

导入公钥
```
gpg --import regenerated-public-key.asc
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


(1) 导出完整密钥对

```
gpg --export-secret-keys --armor --output full-keypair.asc
```

(2) 恢复完整密钥对

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




