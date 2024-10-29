
#### 完全解析（补充）！OpenSSL 创建自签名根证书 Root CA


#### 1、使用 req 命令一次性创建自签名证书（key +cert）

前面我们介绍了，先创建私钥再创建证书的步骤；实际上，req 命令支持直接创建 x509 格式的证书，我们可以再配置好所有参数之后，用 req 命令一次性创建自签名证书（key + cert）。


###### RSA

```
openssl req -x509 -newkey rsa:4096 -sha256 -days 7300 -nodes \
    -keyout VicSign_Root_CA.key -out VicSign_Root_CA.crt \
    -subj "/C=US/ST=TX/L=DAL/O=Security Lab/OU=IT Department/CN=VicSign Root CA" \
    -addext basicConstraints=critical,CA:true \
    -addext keyUsage=critical,keyCertSign,cRLSign \
    -addext subjectKeyIdentifier=hash
```

###### Ed25519

```
openssl req -x509 -newkey ed25519 -days 7300 -nodes \
    -keyout VicSign_Root_CA.key -out VicSign_Root_CA.crt \
    -subj "/C=US/ST=TX/L=DAL/O=Security Lab/OU=IT Department/CN=VicSign Root CA" \
    -addext basicConstraints=critical,CA:true \
    -addext keyUsage=critical,keyCertSign,cRLSign \
    -addext subjectKeyIdentifier=hash
```

注：
- req 命令生成密钥时，默认会提示添加密码保护，我们可以用 `-nodes` 选项不设置密码。
- 自签名的 Root CA 通常是不撤销的，也就没有 serial, crlnumber 这些文件。
- `openssl req -x509 -newkey` 会同时创建 private Key 和 self-signed CA certificate，我们可以根据想要创建的证书类型 (DV, OV, EV) 来改变 `-subj` 字段中的信息；比如，如果只需要 DV 类型的证书，那么只需要填写 `/CN=VicSign Root CA` 即可。

    
#### 2、证书格式 PEM 与 DER

DER 是结构化的二进制格式，符合 ASN.1 标准。PEM 是 DER 格式经过 base64 封装之后的格式，方便打印显示。


#### 3、签名算法：RSA vs Ed25519

RSA 和 Ed25519 都是安全的，但也都不是量子安全的。如果是 ed25519，则无需指定 key size ( 比如 rsa:4096 ); ed25519 默认使用 SHA512 作为哈希方法，我们这里无需手动指定。
两者的区别主要体现在：

- （1）兼容性
RSA 兼容性更好，如果你的密钥会使用在比较老旧的电脑上，那么 Ed25519 可能不支持。

- （2）安全性
RSA 的 key size 是 4096 bit，比较大，而 Ed25519 是椭圆曲线 Curve25519，提供比 RSA 更好的安全性，但是 key size 却小得多。

- （3）性能
RSA 较慢，Ed25519 快得多。
总的来说，如果你的系统更现代，推荐使用 Ed25519 算法。

#### 4、签名算法：ECDSA-with-SHA384

ECDSA 跟 Ed25519 一样，也是椭圆曲线算法，只不过这里专门是指 secp384r1 这条椭圆曲线, 我们同样可以在创建自签名证书的时候使用这个签名算法.

下面是操作步骤：

##### (1) 先创建一个基于 secp384r1 曲线的 EC key

```
openssl ecparam -genkey -name secp384r1 -out secp384r1.key
```
解释：
- ecparam 用于指定 EC 参数
- `-name` 是曲线名，secp384r1 是 NIST 最常见的曲线。

##### (2) 添加密码保护
ecparam 命令不支持添加密码保护，我们生成私钥后转换成 PCKS#8 并添加密码保护

```
openssl pkcs8 -topk8 -in secp384r1.key -out secp384r1_encrypted.key -passout file:password.txt
```

或者指定参数

```
penssl pkcs8 -topk8 -v2 aes256 -v2prf hmacWithSHA384 -iter 100000 \
  -in secp384r1.key -out secp384r1_encrypted.key
```

使用 key 去创建自签名证书，这里的步骤跟之前基于 ed25519 私钥创建证书的方法没有任何区别，但是，这里，我们指定 signature algorithm 为 `ecdsa-with-SHA384`：

openssl 

```
openssl req -x509 -new -days 7300 -sha384\
    -key secp384r1_encrypted.key \
    -subj "/C=US/ST=TX/L=DAL/O=Security Lab/OU=IT Department/CN=VicSign Root CA" \
    -addext basicConstraints=critical,CA:true \
    -addext keyUsage=critical,keyCertSign,cRLSign \
    -addext subjectKeyIdentifier=hash \
    -passin file:password.txt \
    -out secp384r1.pem
```

#### 5. ECDSA 与 Ed25519 区别

都是基于椭圆曲线，区别主要是选取的曲线名字不同。安全性、性能和使用场景有不同。

- ECDSA，常见曲线有：secp256r1, secp384r1, secp521r1
    - keysize 分别为 256, 384, 512。
    - 性能慢于 Ed25519，适合对性能没什么要求的设备。
    - 应用更广泛
- Ed25519，特殊的曲线 Curve25519
    - keysize 固定大小 256 bit
    - 专为高性能而生，适合 IoT
    - 更现代


#### 6、私钥和证书如何保存？

（1）私钥
私钥：限制访问权限 + 放在系统目录 + 备份（网盘）+ meta 信息记录
密码文件（如果有）：限制访问权限 + 放在指定目录 + 备份（网盘）+ meta 信息记录


（2）证书
我们系统都内置了权威信用机构的根证书，因此，当我们访问某些网站，而这些网站像我们提供的证书是属于这些根签发的证书时，我们可以验证网站的合法性。
因此，我们生成的根证书需要导入系统 KMS，这样，根证书签发的下级证书才会得到认证。

##### Linux

把根证书复制到系统证书目录（Trusted Directory）

```
sudo cp VicSign_Root_CA.crt /usr/local/share/ca-certificates/
```

更新系统记录

```
sudo update-ca-certificates
```

这个命令会把扫描 `/usr/local/share/ca-certificates/` 目录，并把其中所有根证书都安装到 `/etc/ssl/certs/` 目录中。


##### MacOS

把根证书导入 Keychain 中，路径是 `/Library/Keychains/System.keychain`
我们既可以通过 GUI 导入，也可以命令行导入

###### (i) GUI 导入

打开 Keychain 应用：File → Import Items.
选择 System，然后添加我们上面生成的 Root CA 证书（比如 `VicSign_Root_CA.crt`
添加完成后，我们需要信任该证书。right-click → Get Info → Alway Trust

###### (ii) 命令行导入
使用 security 命令

```
sudo security add-trusted-cert -d -r trustRoot -k \
    /Library/Keychains/System.keychain VicSign_Root_CA.crt
```

解释：
- **-d**: Adds the certificate as trusted by default for all users.
- **-r** trustRoot: Specifies that the certificate is a Root CA certificate.
- **-k** /Library/Keychains/System.keychain: Adds the certificate to the system keychain (accessible to all users).


