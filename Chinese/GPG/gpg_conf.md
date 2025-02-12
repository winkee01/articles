## gpg.conf 配置

`~/.gnupg/` 目录中的这两个配置文件有不同的用途：

#### 1. gpg.conf:

- 专门用于 gpg 命令行工具的配置
- 影响加密、签名等核心操作的行为
- 用户最常修改的配置文件

##### 常见的 gpg.conf 配置项包括：

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


我的配置：

```
personal-digest-preference SHA512 SHA384 SHA256 SHA224
personal-cipher-preference AES256 AES192 AES CAST5 CAMELLIA192 BLOWFISH TWOFISH
 CAMELLIA128 3DES
personal-compress-preferences ZLIB BZIP2 ZIP
cert-digest-algo SHA512
digest-algo SHA256 
```

#### 2. common.conf:

- 包含所有 GnuPG 工具共享的基本配置
- 影响 gpg、gpgsm、gpg-agent 等所有组件
- 通常不需要修改

一般建议：

- 日常使用主要修改 gpg.conf
- 除非你明确知道要修改什么，否则保持 common.conf 默认
- 对安全性要求高的配置放在 gpg.conf 中
- 使用注释说明重要配置的用途

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad 。

