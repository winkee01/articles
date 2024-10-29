Title: OpenSSL 使用 PKCS#8 格式来封装密钥

## 加密回顾

我们前面的文章介绍过如何使用 OpenSSL 对私钥进行密码保护，主要是 AES--256-CBC 算法配合 Salt 以及 PBKDF2, 或 Argon 等 KDF 来实施。

比如，我们有一个自建 CA 证书的私钥 `FactTrust_Root_CA.key`，我们这样对它添加密码保护：

```
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 -in FactTrust_Root_CA.key -out FactTrust_Root_CA.key.enc
```

加密方式和算法都是目前比较流行且安全性得到较好保障的。

不过，这种方式依然有几个小缺陷：

- (1) 输出的文件是二进制的，没法直接打印查看
- (2) 输出的文件不是 PKI 体系中的标准化格式（符合 ASN.1 标准的结构化数据），这导致我们没法用标准化工具查看细节。
- (3) 由于不是标准化格式，每次我们要使用它都得先手动解密，操作上很不方便。而 PKCS#8 格式的则无需解密就可以查看私钥的 meta 信息。

为了解决上面几个缺点，最佳实践是把私钥**转换成** pkcs8 标准所指定的格式（可以选择对私钥加密或不加密），如下：

```
openssl pkcs8 -topk8 -v2 aes256 -in FactTrust_Root_CA.key -out FactTrust_Root_CA_PKCS8_AES256.key

```

解释：

-topk8 选项就是转换成 pkcs#8 格式，它比 pkcs#1 格式多 20 字节，主要用于描述算法类型等 metadata 信息。pkcs#1 只支持 RSA 密钥算法，而 pkcs#8 更加通用，支持多种密钥算法。对称加密算法是在 pkcs#5 中定义的，pkcs#5 v1.5 只支持 DES 算法，

而 v2 则支持 AES 等算法。具体来说，pkcs#5 v2 采用的是一种叫 PBES2 (Password-Based Encryption Scheme 2) 的算法，它允许指定两个特征：

- 1. 加密算法：比如 AES-256-CBC
- 2. 密钥派生函数（KDF）：默认使用 PBKDF2。

它还支持自定义下面的参数：

- **迭代次数**： 增加 PBKDF2 中的迭代次数，以增加暴力破解的难度。（默认是 2048 次）
- **摘要算法**：为 PBKDF2 选择一个强大的哈希函数，（默认是 SHA-256）

另外，我们无须显示地指定 salt，因为默认是会自动生成的。


现在我们再看一下指定了 `-iter` 选项的命令：

```
openssl pkcs8 -topk8 -v2 aes256 -iter 100000 \
  -in FactTrust_Root_CA.key -out FactTrust_Root_CA_PKCS8_AES256.key
```

输出结果：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/asn1parse1.png)


#### 指定摘要算法（伪随机函数）
默认情况下，`openssl pkcs8` 使用 `HMAC-SHA1` 作为 PBKDF2 中的伪随机函数（PRF）。我们可以使用 `-v2prf` 选项指定更强的哈希函数（比如 hmacWithSHA512）。

示例

```
openssl pkcs8 -topk8 -v2 aes256 -v2prf hmacWithSHA256 -iter 100000 \
  -in FactTrust_Root_CA.key -out FactTrust_Root_CA_PKCS8_AES256.key
```

下面是一个包含所有安全参数的完整命令：

```
openssl pkcs8 -topk8 -v2 aes256 -v2prf hmacWithSHA256 -iter 100000 \
  -in FactTrust_Root_CA.key -out FactTrust_Root_CA_PKCS8_AES256.key
```


成功转换后，我们可以用 asn1parse 来查看密钥信息，如下：

```
openssl asn1pars
```

输出结果：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/pkcs8_iter.png)


或者

```
openssl pkey -in FactTrust_Root_CA_PKCS8_AES256.key -text -noout
```

或者

```
cat FactTrust_Root_CA_PKCS8_AES256.key
```

输出结果：

```
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIGjMF8GCSqGSIb2DQEFDTBSMDEGCSqGSIb3DQEFDDAkBBAIFhGijjsNRUzkvePr
5jmTAgIIADAMBggqhkiG9w0CCQUAMB0GCWCGSAFlAwQBKgQQKo9YE0C2EB5TH85n
EcOuHwRAXJdqwqy4IcPGBE1N33tgZMorbgHJUwFkxuzh3OJYPYCUFeQxk15RMh5T
R6ClqXglmbRMnqjIBUGDxF2hTQOiyA==
-----END ENCRYPTED PRIVATE KEY-----
```

它是 PEM 格式的。注意它跟 ssh-keygen 所生成的私钥的格式不太一样，如下：

```
cat ~/.ssh/id_rsa
```

输出结果：

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZoktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACBlH2sG/nHzARzWNMVUb1+Wdb90l8+K4IDvS5SPNfHfpAAAAJim9bjmpvW4
5gAAAAtzc2gtZWQyNTUxOQAAACBlH2sG/nHzARzWNMVUb1+Wdb90l8+K4IDvS5SPNfHfpA
AAAEBgEnAgP4mUcXyGdfLNtcFiPz3kaPVmrjs0C7YmIw69HmUfawb+cfMBHNY0xVRvX5Z1
v3SXz4rggO9LlI818d+kAAAAFHdpbmtlZTAxQG91dGxvb2suY29tAQ==
-----END OPENSSH PRIVATE KEY-----
```

这个不是 PEM 格式，我们可以用下列命令把它转换成 PEM 格式：

```
ssh-keygen -p -f ~/.ssh/id_rsa -m pem
```

输出结果：

```
-----BEGIN RSA PRIVATE KEY-----
MIIJKQIBAAKCAgEAw7R7zjMCLoKIT7L7bd3TvOHYGbc8tkO3zgP0/eqX5kiRlfoR
...
GRfahB+a0pOBqFQ0NRWZ8x94nP1IRngHdiYE43uO+oAt3Vfx5+nRY2FiMHthXvdY
nk36bsjy2AA3ZW4wjOmye8Pp/wVsuNFl/CMOUBpPTXuQCgSFKEPny5Kt+KQF
-----END RSA PRIVATE KEY-----
```


小知识：pkcs#8 是专门针对私钥处理的，不是一般情况下的加密工具。


下面的命令展示了如何创建私钥，安全加密私钥并创建 CA 证书的完整过程：


```
openssl genpkey -algorithm ed25519 -out FactTrust_Root_CA.key

openssl pkcs8 -topk8 -v2 aes256 -v2prf hmacWithSHA256 -iter 100000 \
-in FactTrust_Root_CA.key -out FactTrust_Root_CA_PKCS8_AES256.key

shred -u FactTrust_Root_CA.key

openssl asn1parse -in FactTrust_Root_CA_PKCS8_AES256.key

openssl req -x509 -key FactTrust_Root_CA_PKCS8_AES256.key -days 7300 \
  -out FactTrust_Root_CA.crt -subj "/C=US/ST=TX/L=DAL/O=Security/OU=IT Department/CN=FactTrust Root CA"

```

注：OpenSSL 在需要时会提示输入密码。


如果想要非交互式的（不想输入密码），我们可以把密码保存在文件中，然后作为参数传递给命令，如下：

使用文件中的密码进行加密：

```
openssl pkcs8 -topk8 -v2 aes256 -v2prf hmacWithSHA256 -iter 100000 \
  -in FactTrust_Root_CA.key -out FactTrust_Root_CA-PKCS8-AES256.key \
  -passout file:passphrase.txt
```

使用加密后的密钥：

```
openssl req -x509 -key FactTrust_Root_CA_PKCS8_AES256.key -days 7300 \
  -out FactTrust_Root_CA.crt -passin file:passphrase.txt \
  -subj "/C=US/ST=TX/L=DAL/O=Security/OU=IT Department/CN=FactTrust Root CA"
```



#### 解密
如果在某些自动化场景或者云环境中（无法访问我们的密码文件），那么我们可能需要用到非加密的 Private Key，

也就是把 Encrypted Private Key 转换成 Unencrypted Private Key。
如下：

```
openssl pkcs8 -in encrypted_private.key -out unencrypted_private.key \
  -passin "pass:your_password"

```


全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 deliverit。



Let me make it more clear, if I encrypt a private key using below command, how can I decrypt it:

```
openssl pkcs8 -topk8 -v2 aes256  -v2prf hmacWithSHA256 -iter 120000 -in FactTrust_Root_CA.key -out FactTrust_Root_CA_PKCS8_AES256.key
```
