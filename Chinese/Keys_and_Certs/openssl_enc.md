# 对称加密原理详解与 OpenSSL 实战


### 对称加密简介

**对称加密**是一种使用同一密钥进行加密和解密的加密技术。由于其速度快、效率高，相较于使用公钥和私钥对的非对称加密，它广泛用于数据加密。密钥必须保密，并且仅在授权的双方之间共享。

对称加密的原理是**通过密钥将明文加密成密文**，并且再通过同一个密钥将密文解密成明文，相对于非对称加密算法**速度快**、**效率高**，对于明文文本越长效率优势越大。

### 常见的对称加密算法

常见的对称加密算法有 **AES**、**DES**、**3DES** 等，其中 DES 由于密钥长度低容易被暴力破解，因此安全性相对较低已经不推荐使用。而 3DES 则是 DES 的升级版，进行了三次 DES 加密以增强安全性，但依然不如 AES，因此推荐安全性更高的 AES 加密算法。


| 算法 | 算法类型   | 密钥长度(bit) | 分组长度(bit) | 安全性 |
|------|------------|---------------|---------------|--------|
| AES  | 块密码算法 | 128/192/256   | 128           | 安全   |
| DES  | 块密码算法 | 56            | 64            | 安全   |
| 3DES | 块密码算法 | 128/168       | 64            | 安全   |


另外，还有其他一些更适合互联网应用的加密方式，它们的特点是速度快，性能好。比如流行的 **ChaCha20**。
它是一种更现代的高性能流加密算法，它通常与 **Poly1305** 结合用于认证加密。


### 分组模式（Block cipher mode）
AES、DES、3DES 都是块密码算法，即运算加密解密时不是一次性将整个明文/密文文本进行运算，而是拆成固定长度的数据块后对每个数据块进行运算。

由于文本被拆分成若干个数据块，对于超过一个数据块的场景需要定义数据块之间的拆分和组装方式，即分组模式，场景的分组模式如下表：

| 分组 | 是否需要填充 | 安全性 |
|------|--------------|--------|
| ECB  | 否           | 不安全 |
| CBC  | 是           | 安全   |
| CTR  | 是           | 安全   |


下面是对每种分组模式的详细介绍：

#### ECB
ECB是最简单的一种分组模式，简单地将明文拆成数据块后，每个数据块单独加密解密。如果两个数据块内容相同，那么这两个数据块加密后的密文段也是完全一样，容易受到重放攻击以及密文篡改，因此安全度不高。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ECB_mode.png)


#### CBC
CBC引入了一个新的变量初始向量 IV，一般是一个随机数，用于在第一块明文数据块加密前对数据块做 XOR 运算，之后每一块明文数据块加密前都与前一块的密文数据块做 XOR 运算。

由于每一块数据块的计算结果都与上一块数据块有关联，因此即使有两个相同的数据块，计算出的密文段也不同，解决了 ECB 的安全问题。但因此，CBC 计算只能串行处理，效率不如 ECB。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/CBC_mode_20241024093726.png)


#### CTR
CTR 同样引入了一个新的变量初始随机数，每一个数据块计算时先对随机数 +1，然后再参与加密解密运算，因此即使有两个相同的数据块，计算出的密文段也不同，解决了 ECB 的安全问题。并且每个数据块仅与随机数有联系，因此可以并行处理。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/CTR.png)

### 填充算法
对于块加密算法来说，不是对一次性对整个明文进行加密计算，而是拆分成若干个固定长度（加密算法的分组长度）的数据块，然后对各个数据块进行加密计算，因此明文的字节长度必须是分组长度的整数倍数。如果明文的字节长度不是分组长度的整数倍数，则需要一种填充的机制，在加密前将明文字节长度填充成分组长度的整数倍数，并且在解密后正确移除填充的字节。

常见的填充算法有 **PKCS#7** 和 **PKCS#5**，而实际上两者的处理逻辑是相同的，只是 PKCS#5 只能处理分组长度为 8 字节的数据，而 PKCS#7 可以处理分组长度是 1-255 任意长度字节的数据。从这个角度看，PKCS#5 是 PKCS#7 的子集。

关于填充算法的细节这里就不详述，有兴趣的可以在网上搜索。

AES 算法的分组长度超过 8 字节，因此不能使用 PKCS#5，但有些程序使用 AES 算法时若指定 PKCS#5 时并未报错，实际内部依然使用了 PCKS#7。


分组模式还有 CFB 和 OFB，这里就不细述了，有兴趣了解的也可以去网上搜索相关资料。


### 什么是盐值 (Salt)?

**盐值 (Salt)** 是在哈希或加密之前添加到数据（通常是密码）中的随机值，确保即使两个用户使用相同的密码，生成的哈希值或加密数据也会不同。它主要用于防止攻击者使用预先计算好的哈希表（如彩虹表）轻松破解哈希密码。

##### Salt 工作原理：

当用户创建密码时，系统会生成一个随机的盐值，并将其与密码结合。然后，将这个组合数据进行哈希或加密处理。盐值通常与生成的哈希或加密结果一起存储。在验证密码时，系统会取出存储的盐值，添加到用户输入的密码中，并进行同样的加密或哈希过程，最后比较结果是否一致。

#### 示例：
如果两个用户的密码相同，未使用盐值时，它们的哈希值将是相同的。但是，使用不同的盐值后，它们的哈希值将不同：

- 密码：`mypassword`
  - 盐值1：`random123`
  - 盐值2：`different456`

- 哈希1（使用盐值1 + `mypassword`）
- 哈希2（使用盐值2 + `mypassword`）

这样，即使密码相同，哈希结果也不同，增强了安全性，攻击者很难通过预计算的哈希表来破解密码。

---

#### KDF 算法
**KDF** (key derivation function，密钥派生函数) 可以通过特定算法将密码派生出**指定长度的密钥**，非常适用于密码验证的密码哈希处理。

常用于密码哈希的 KDF 算法有 **PBKDF2**, **Scrypt**，**Yescrypt**，**Argon2**。在 2013 到 2015 年间进行的密码哈希竞赛宣布 Argon2 成为获胜者，所以目前最为推荐用于密码哈希的算法就是 **Argon2**。

**PBKDF2** 是 NIST 的推荐算法，在 Bitwarden 的实现中，默认的迭代次数是 60 万次。它的工作原理是把主密码与用户名混合，通过单向哈希算法（HMAC-SHA-256）运算结果值以创建固定长度的哈希值。该值再次使用用户名加盐，并继续执行哈希运算（指定迭代次数），所有迭代完成后的结果值就是主密钥，它充当主密码哈希的输入，用于在用户登录时验证该用户。

**Argon2** 算法有三个版本：Argon2i, Argon2d 和 Argon2id，其中，Argon2d（防御侧信道攻击），Argon2i（优化密码哈希），而 Argon2id 是这两者的混合体。

**Argon2 的工作原理是：** 分配一部分内存（KDF 内存）然后用已计算的哈希值填充它直到填满。这是重复的，从它在第一个停止的内存的后续部分开始，在多个线程（KDF 并行）上送代多次（KDF 迭代）。所有迭代后的结果值是您的主密钥，它充当主密码哈希的输入，用于在用户登录时验证该用户（了解更多）。

默认情况下，Bitwarden 的视线中，设置为分配 64MiB 内存，迭代 3 次，并跨 4 个线程执行此操作。这些默认值高于当前 OWASP 的推荐，所以安全程度是比较可靠的。


上述所列的几个 KDF 函数安全性的比较大致如下：

> Argon2id > Yescrypt > Scrypt > PBKDF2


在最新版本的 OpenSSL （比如 3.0 以上）中，OpenSSL 只支持 **PBKDF2** 的算法，如果在加密是没有使用 `-pbkdf2` 和 `-iter` 选项，会得到警告提示：

```
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
```

之所以警告，是因为，在较早版本的 OpenSSL 中，OpenSSL 默认使用的是基于 **EVP\_BytesToKey** 函数的密钥派生算法。这个算法结合了 MD5 散列算法来从密码生成加密密钥。**EVP\_BytesToKey** 函数的密钥派生过程非常简单，仅经过少量的哈希迭代（默认是一次 MD5 迭代），并且没有像 PBKDF2 那样支持复杂的多次迭代和盐值。
这种简单的派生方式使得密钥更加容易被暴力破解，尤其是使用现代硬件进行攻击时。而且 **MD5 由于极其容易受到碰撞攻击**（不同输入产生相同输出），早就被认为不再适合出现在任何实际的使用中。

因此，在 OpenSSL 中，现代安全方式的加密方式是指定 `-salt`, `-pbkdf2` 和 `-iter` 三个选项，而且建议 `-iter` 的迭代次数最少在 10 万以上。

例子：

(1) AES-256-CBC + Salt + PBKDF2 + Iter

```
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 -in plain.txt -out encrypted.txt
```

(2) ChaCha20 + Salt + PBKDF2 + Iter

```
openssl enc -chacha20 -pbkdf2 -iter 100000 -in plain.txt -out encrypted_chacha20.txt
```

但是注意，Blowfish 在 OpenSSL 3.0 以上的版本中已经标记为 deprecated ，所以就不再建议使用了。
如果使用，

```
openssl enc -bf -salt -iter 100000 -in plain.txt -out encrypted_bf.txt
```

就会报错：

```
Error setting cipher BF-CBC
40F2DDF801000000:error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:crypto/evp/evp_fetch.c:355:Global default library context, Algorithm (BF-CBC : 14), Properties ()

```

想要解决错误，可以考虑使用一个低版本的 openssl 来执行上述命令。

#### 解密
添加一个 `-d` 选项即可，如下：


```
openssl enc -aes-256-cbc -d -salt -pbkdf2 -iter 100000 -in plain.txt -out encrypted1.txt
openssl enc -chacha20 -d -pbkdf2 -iter 100000 -in encrypted_chacha20.txt -out decrypted2.txt
```

注：
解密的时候，不需要提供 `-salt` 选项。


### 生成随机密钥和IV

OpenSSL 还可以生成随机密钥和初始化向量（IV），这些在某些加密模式（如 AES-CBC）中是必需的。生成方法如下：

生成随机密钥（256位用于 AES-256）：

```bash
openssl rand -hex 32
```

生成随机 IV（128位用于 AES-CBC）：

```bash
openssl rand -hex 16
```

### 示例：使用指定的密钥和 IV 进行加密

使用特定的密钥和 IV 进行加密：

```bash
openssl enc -aes-256-cbc -in plain.txt -out encrypted_custom.txt -K <key> -iv <iv>
```

- 将`<key>`和`<iv>`替换为前面生成的十六进制值。


### 密码文件（不推荐）
如果不想记忆复杂的密码，我们可以用 OpenSSL rand 命令创建 AES-256 位密钥以生成随机数据，然后输出到 `passphrase.txt` 文件中，如下：

```
openssl rand -base64 32 > passphrase.txt
```
现在，`passphrase.txt` 中就包含了我们的密码。
我们可以用这个密码文件类加密我们的文档，这样我们就不用手输了

```
$ openssl enc -aes-256-cbc -salt -pbkdf2 -in plain.txt -out encrypted.txt -pass file:./passphrase.txt
```

不过要注意的是，用文件来保存密码这种方式不仅管理麻烦，还有很大的安全隐患，因此使用场景有限，不推荐使用。最好使用一个专门的密码管理工具来管理密码。

另外，这个命令也常被用来生成同等长度（比如 32 位）的 Salt。

```
openssl rand -base64 32 salt.txt
```

### 结论

对称加密广泛用于数据的机密性保护，AES、ChaCha20 等常见算法配合主流的 KDF 函数能够提供强大的安全性。OpenSSL 提供了一套强大的工具来执行对称加密和解密，帮助你以较少的成本保障数据安全。请务必确保密钥在各方之间安全共享，以避免安全漏洞。


参考：

https://zhaoxh.cn/post/2019/kdfcrypt/
https://repost.aws/zh-Hans/knowledge-center/kms-openssl-encrypt-key


全文完！




如果你喜欢我的文章，欢迎关注我的微信公众号 deliverit。
