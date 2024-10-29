# Title: 完全解析！OpenSSL 创建自签名根证书 Root CA


## 简介
前面介绍了使用 OpenSSL 创建和管理私钥的概念以及详细的实战操作，这篇文章继续这个话题，创建 OpenSSL 自签名的根证书。同样，提供全面的概念解析和详尽的实战操作案例。另外，这篇文章也为后续创建三级证书体系打下基础。


### 三级证书体系

我在 [使用 OpenSSL 构建 X.509 三级证书体系](https://juejin.cn/post/7082214905780109326) 一文中详细介绍了三级证书的概念，按照这个体系，我们创建三个不同的目录来进行管理。如下：


![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/rsa3levelcerts.png)


这篇的重点就是创建顶层的根证书

`RSA_VicSign_CA`


假设我们根证书的属性如下：

-   Self-signed (DV, OV, EV) 根证书都是自签名的
-   Validity: 20 ~ 30 years 有效期都在 20 ~ 30 年左右 (7200 - 10800 天）
-   x509 v3 extensions 包含的 v3 扩展字段有如下几个： [v3_ca]
-   subjectKeyIdentifier = hash
-   authorityKeyIdentifier = keyid:always,issuer:always
-   basicConstraints = critical,CA:true,pathlen:3 # 多数没有 pathlen 限制
-   keyUsage = critical, cRLSign,keyCertSign,digitalSignature # 多数没有 digitalSignature
-   extendedKeyUsage
-   subjectAltName
-   -   Country: US
    -   State: TX
    -   Location: DAL
    -   Organization: DigiCert Inc (or Web Security)
    -   Organization Unit: www.digicert.com (or IT Department)
    -   CN: DigiCert Global Root CA



###### X509v3 extensions:

-   X509v3 **Subject Key Identifier**:
    - 3E:01:1C:F3:88:81:25:F7:61:D6:2B:90:8A:41:63:C7:B8:39:8E:75

-   X509v3 **Authority Key Identifier**:
    - 3E:01:1C:F3:88:81:25:F7:61:D6:2B:90:8A:41:63:C7:B8:39:8E:75

-   X509v3 **Key Usage**: critical
    - Digital Signature, Certificate Sign, CRL Sign

-   X509v3 **Basic Constraints**: critical
    - CA:TRUE, pathlen:3



**解释：**

主要分为三部分：

（1）签名算法：sha256WithRSAEncryption or ED25519

（2）Subject 信息：

-   Subject, Issuer
-   Validity
-   Pub Key Algorithm: rsaEncryption or ED25519

（3）x509 V3 Extension 部分：  


-   四项必须（subjectKeyIdentifier，authorityKeyIdentifier, basicConstraints, keyUsage）
-   **Key Usage**: 里面，cRLSign 和 keyCertSign 这两项必须，digitalSignature 看情况  


从 OU 那里，我们可以看到 digicert 有点像是其实是 OV 类型的证书，并且后面的 Key Usage 里面又有 Digital Signature 权限。

有些 Root CA 一般不会给这个权限。但是 `SSL.com_Root_Certification_Authority_RSA.crt` 也有这权限。

我们再看 `Buypass_Class_3_Root_CA.crt` 这个 Root CA 证书，它就没有 Digital Signature 权限。

有些根证书一般不会给 Digital Signature 权限，比如： `Buypass_Class_3_Root_CA.crt`
但是有些也给，比如 `DigiCert_Global_Root_CA.crt` 和 `SSL.com_Root_Certification_Authority_RSA.crt`。


###### 操作要点：

-   配置文件 `/opt/homebrew/etc/openssl@3/openssl.cnf` 或 `/usr/lib/ssl/openssl.cnf`
-   根据执行的命令去查找 `openssl.cnf` 中对应的配置信息

比如 req 命令，那么就查找 `[ req ]` 部分对应的信息

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/openssl_config_explained.png)


### 第一步：私钥的创建与保管

#### 一、创建私钥

按照我们在前面两篇文章中介绍的私钥加密的知识，我们可以自己的需要来选择不同的算法生成私钥。  

OpenSSL 中，创建私钥的命令是 `openssl genpkey`。  


这里实验为了方便，每次免去输入密码的麻烦，我们把密码文件来加密私钥。

（当然，如果你坚持暂时不加密码保护，你可以省去下面所有与使用 `password.txt` 相关的操作）




##### （1）RSA + 4096 bit (Key Size) + SHA256 (dgst algo)

```
openssl genpkey -algorithm RSA \
    -aes256 -pkeyopt rsa_keygen_bits:4096 \
    -pass file:password.txt \
    -out rsa_private_encrypted.key 
```

###### 要点：

-   创建 RSA 私钥时，可以同时对其添加密码保护。
-   老版本的 OpenSSL 命令创建的 RSA 私钥是 PKCS#1 格式的，不支持 `-pbkdf2`, `-iter` 和 `-v2prf` 选项，因为这些属性 PKCS#8 格式中的。
-   新版本的 OpenSSL (3.0+) 创建的私钥已经默认是 PKCS#8 格式了。（指定 `-aes256` 加密算法后，默认会使用 PBKDF2 函数，迭代次数为 2048次，伪随机算法为 hmacWithSHA256，并且 key size 默认为 2048 bit）
-   如果不是 PKCS#8 格式，则建议把私钥转换成 PKCS#8 格式，使用 `openssl pkcs8` 命令，它支持手动指定上述选项。



###### 查看 Key Size

```
openssl pkey -text -noout -passin file:password.txt -in rsa_private_encrypted.key
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/view_key_size.png)


先创建密码文件：

```
openssl rand -base64 32 > password.txt
chmod 400 password.txt
```

##### （2）Ed25519 + SHA512 (dgst algo)

```
openssl genpkey -algorithm ed25519 \
    -aes256 \
    -out ed25519_private.key
```

###### 要点：

-   Ed25519 算法中的 Key Size 是固定大小的 256 bit（不可配置），它不像 RSA 中那样是可配置的。这是 Ed25519 优势的地方，它 256 bit 大小的 key size 的安全强度相当于 RSA 中 3072-bit，体积更小还意味着性能更好。



![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ed25519_key_size.png)





#### 验证和检查私钥

有多个命令可以检查私钥相关的信息，比如：

```
openssl asn1parse -in rsa_private_encrypted.key
openssl pkey -in ed25519_private.key -text -noout
cat d25519_private.key -text -noout
```


![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/rsa_key_info.png)


![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ed25519_key_info.png)


可以看到，只要我们添加了 `-aes256` 选项对私钥进行密码保护，默认使用的就是 PBES2 密码标准，并且生成的密钥就是 PKCS#8 格式。`openssl asn1parse` 命令结果中显示的 `:0800` 是 16 进制的值，对应的是 2048，它表示的就是迭代次数。

下面分别是在不同的 Shell 中把这个值换算为十进制值的方式：
Bash

```
echo $((0x0800))
```

Fish

```
math 0x0800
```

假如我们需要给私钥指定迭代次数的话，我们需要使用 `openssl pkcs8` 命令，前面的[文章](https://juejin.cn/post/7429735584895582218)已经详细介绍过了，有兴趣的可以去查看。




如果想把私钥去掉密码保护，记得下面这个命令：

```
openssl pkcs8 -in ed25519_private_encrypted.key -out ed25519_private_decrypted.key -passin file:password.txt
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ed25519_decrypted_key.png)


#### 二、保管私钥

限制访问权限

```
chmod 400 rsa_private_encrypted.key
```

移动到安全位置

```
mv rsa_private_encrypted.key ./private
```

有条件的，可以把私钥放到专门的存储硬件上去，并做好备份，这里就不详述了。


密码的话，当然是不建议使用上述的 `password.txt` 这种方式来保存的，安全风险较高，这里仅做实验用。

最佳实践是使用专门的密码管理工具（Password Manager），比如 `1password`, `bitwarden` 等。在 MacOS 中的话，推荐用 Keychain 来保管密码。


### 第二步：自签名证书
有了私钥，创建证书就简单了，我们有好几种方法，都是使用 `openssl req -x509` 命令，区别只在于是命令行传递配置信息还是通过配置文件传递。


#### 方法一：配置文件


`req.cnf`

```
[ req ]
default_keyfile     = ca_ed25519.key        # Default private key filename
prompt              = no                    # Disable prompt
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca                 # Extensions for self-signed root CA certificate

[ req_distinguished_name ]
C  = US
ST = TX
L  = DAL
O  = Security Lab
OU = IT Department
CN = VicSign Root CA

[ v3_ca ]
# Extensions for a root CA certificate
basicConstraints = critical, CA:true
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
```

###### For RSA Key

```
openssl req -new -x509  -days 7300 -sha256 \
    -key rsa_private_encrypted.key \
    -config rootca.cnf \
    -passin file:password.txt \
    -out rsa_ca.pem 
```

###### For Ed25519 Key

```
openssl req -new -x509 -days 7300 \
    -key ed25519_private_encrypted.key \
    -config rootca.cnf \
    -passin file:password.txt \
    -out ed25519_ca.pem
```

注：

-   如果使用 RSA Key，那么需要通过 -sha256 指定哈希算法。而如果是 Ed25519 Key，则无需指定，默认就是 SHA256。
-   -days 只能通过命令行传递，没法在配置文件中传递。


#### 方法二：命令行参数


###### For RSA Key

```
openssl req -x509 -new -days 7300 -sha256 \
    -key rsa_private_encrypted.key \
    -subj "/C=US/ST=TX/L=DAL/O=Security Lab/OU=IT Department/CN=VicSign Root CA" \
    -addext basicConstraints=critical,CA:true \
    -addext keyUsage=critical,keyCertSign,cRLSign \
    -addext subjectKeyIdentifier=hash \
    -passin file:password.txt \
    -out rsa_ca.pem
```


###### For Ed25519 Key

```
openssl req -x509 -new -days 7300 \
    -key ed25519_private_encrypted.key \
    -subj "/C=US/ST=TX/L=DAL/O=Security Lab/OU=IT Department/CN=VicSign Root CA" \
    -addext basicConstraints=critical,CA:true \
    -addext keyUsage=critical,keyCertSign,cRLSign \
    -addext subjectKeyIdentifier=hash \
    -passin file:password.txt \
    -out ed25519_ca.pem
```


#### 最后步骤

###### （1）访问权限
设置证书访问权限，所有人均可读但不可修改。

```
chmod 444 ed25519_ca.pem
```

（2）证书签发管理
由于是根证书，它需要记录和管理所有签发信息，所以务必创建下列文件和目录：

（i）ca.cnf

```
[ ca ]
default_ca              = CA_default

[ CA_default ]
dir                     = .
certs                   = $dir/certs
crl_dir                 = $dir/crl
database                = $dir/index.txt
new_certs_dir           = $dir/newcerts
private_key             = $dir/private/ca.key.pem
certificate             = $dir/certs/ca.cert.pem
serial                  = $dir/serial
crlnumber               = $dir/crlnumber
default_md              = sha256
policy                  = policy_strict
default_days            = 3650
default_crl_days        = 30

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
```

（ii）创建目录及文件

```
mkdir {certs,crl,newcerts,private}
touch index.txt
touch crlnumber
echo 1000 > serial
```

全文完！

































