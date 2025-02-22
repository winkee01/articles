## 前言

前一篇文章介绍的 GPG Key 所构建个人的公钥加密体系是基于点对点的分布式信任，也叫 Web of Trust (WoT)，由于签名膨胀 (signature bloat/spamming) 问题导致 WoT 实际上已经失效，从 gpg 2.2.17 版本开始，key server 已经忽略除自签名以外的签名了。这篇文章所介绍的 X.509 证书是基于权威机构 (Certification Authority) 的信任，属于中心化的信任，与 GPG Key 的信任方式相反。


下面是一个对比：

GPG Key

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f86c9d4ec8944e6abce524d8bab8639c~tplv-k3u1fbpfcp-watermark.image?)

可以看到，在 GPG Key 体系中，信任是由用户自己发起的，一个人收集到的签名越多，那么他/她的公钥也就越可信。

X.509

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dcd797f532344c4847ee42d2fcacb5b~tplv-k3u1fbpfcp-watermark.image?)

在 X.509 证书体系中，信任是由权威机构（CA）发起的，一个人的公钥要变得可信，只需要某 CA 的签名即可。


PKI 体系包含了多种不同的规范和实现，SSH Key, GPG Key，以及 X.509 都是 PKI 中的一员，归根结底，它们都需要解决公钥的可信任问题。换句话说，当你从不同的途径拿到某个机构，组织或者个人的公钥，如何确认这个公钥就是这个实体的，而不是别人仿冒的。一个必须的步骤就是它们都需要把公钥与能标识实体的唯一名称进行绑定，通过签名的数量（GPG Key）或者权威性 (X.509 Certificate）来确定公钥是否可信。

#### 什么是 X.509？

有一个国际标准化组织 ISO，它对各个领域做出一些标准规范，以便全球通用，常见的比如 ISO 9000 系列的质量体系。类似的标准化组织还有很多，ITU（国际电信联盟）就是其中一个，在标准化的体系里，都是以树状目录结构来组织各个部分的。X.509 就是其中一个目录，它规定了数字证书的标准，类似的目录还有 X.208 and X.680 等。ITU 最早在 1993 年就指定了 X.509 数字证书标准，它其实是作为 X.500 项目的一部分，X.500 项目的作用是定义了唯一标识一个实体的方法，该实体可以是机构、组织、个人或一台服务器。而 X.509 则把该实体与公钥所绑定关联起来，从而提供了通信实体的鉴别机制，也就是目前最流行的 X.509 数字证书。


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee27e13a4716438ea97dee2914813ba2~tplv-k3u1fbpfcp-watermark.image?)

X.509 的最初版本公布于 1988 年。X.509 证书主要由用户**公共密钥**和**用户标识符**组成。此外还包括版本号、证书序列号、CA标识符、签名算法标识、签发者名称、证书有效期等辅助信息。这一标准的最新版本是X.509 v3，它定义了包含**扩展信息 (extensions)** 的数字证书。该版数字证书提供了一个扩展信息字段，用来提供更多的灵活性及特殊应用环境下所需的信息传送。 


我们通常说的证书都是指 X.509 数字证书，如果不加特别说明，都是指 X.509 v3 版本的。简单来说，v3 版本就是添加了扩展字段的证书。


一张典型的 X.509 v3 证书包含如下内容：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/232c534dd8b149d1af50472d8d8d9ed9~tplv-k3u1fbpfcp-watermark.image?)

打开浏览器，可以看到某网站的数字证书信息如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5acde0b5f5bf429a8040da8309c9e360~tplv-k3u1fbpfcp-watermark.image?)


### 权威机构（Certificate Authority）

上面介绍到，X.509 体系中，权威机构 (CA) 是最重要的一环，只有经过它签名的公钥（包含在所签发的数字证书中）才是可信的。当我们打开一个 HTTPS 链接的网页时，浏览器会自动从系统内置的 CA 证书列表中找到对应的 CA 根证书来帮我们帮我们校验网站的数字证书是否合法。而这些 CA 证书通常在我们安装操作系统的时候就已经预置在我们的操作系统中（或者我们也可以手动安装），比如在 Ubuntu 中，预置的 CA 证书所在的目录通常是 `/usr/share/ca-certificates/mozilla/`。


#### 权威机构的分级

为了更加有组织的管理证书体系，CA 也是分级的，根据功能主要分为三个层级：Root CA, Intermediate CA 和 End User，它们会形成一个证书链来确保其中的每个证书都是可信的，我们后续创建 X.509 证书体系的时候就是按照这个层级进行创建。


当我们向某 CA 申请一个数字证书的时候，根据我们提供信息的详细程度以及 CA 对我们进行校验的手段，颁发的数字证书有不同的信任级别，主要有三种类型的数字证书：


-   DV, Domain Validated
-   OV, Organization Validated
-   EV, Extended Validated


这三者的验证级别以及可信程度是逐步提高的，我们普通用户想要为自己购买的一个域名（domain）申请数字证书时，通常并不需要提供详细个人信息（Country, State, Organization, Email 等等），只需要证明我们是域名的拥有者即可，这时候我们通常只需要申请 DV 证书；而对于一些大银行、政府机构，他们则会申请 OV 或者 EV 证书来证明自己的可靠性。


#### 如何辨别网站的证书是 DV，OV 还是 EV 类型？

对于 DV 证书来说，通常其 Subject 信息中只需要包含 Common Name 即可，因为它不需要验证实体，只需要验证域名所有权。其它信息比如 Country, Organization 等均可忽略。比如下面是几个例子：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/183ec9e684d14273bcb0f78d11e74721~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/da7cd14dd3394274b5cd922e6604a025~tplv-k3u1fbpfcp-watermark.image?)

而对于 OV, 或 EV 类型的证书来说，它们都需要验证实体，所以会有 Country, State, Organization 等更详细信息，比如：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57dd7653f7b248cb92961503346389db~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b76c74cfd798413eb1820960b0bd7ea3~tplv-k3u1fbpfcp-watermark.image?)


其实要判断 DV, OV 还是 EV，一个更准确的方式是通过 OID 来判断，OID (Object ID）是 ISO 中的条目的唯一编号，证书中的固定属性通常已经注册了唯一标识符 OID，证书类型的 OID 如下：\


Type                                Policy Identifier

Domain Validated             2.23.140.1.2.1

Organization Validated     2.23.140.1.2.2

Extended Validated           2.23.140.1.1 


因此，我们查看 v3 Extension 中的 Policy ID 字段即可得知证书真实类型：

比如：


bankofamerica.com ( EV 证书 ) 

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69279cab80ba40f698c6af490cfe214f~tplv-k3u1fbpfcp-watermark.image?)

stackoverflow.com ( DV 证书 )

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21527e8056364f2382f86eaeb8789b4e~tplv-k3u1fbpfcp-watermark.image?)

#### 权威机构都有哪些？
常见的权威机构有 Sectigo, SSL.com, DigiCert, GlobalSign, Entrust (GTS) Google Trust Services, ISRG (Internet Security Research Group) 等等，这些 CA 广泛受到信任，它们的根证书几乎预置在所有设备中，因此由它们所签发的证书都普遍受到信任。比如，国内的京东和淘宝都是用 GlobalSign 签发的证书，而常见的社交媒体 Twitter, Facebook, Instagram 等网站则都是用 DigiCert 签发的证书，Google 的服务大多数用 GTS 签发，而一些开发类的网站或者博客站点（比如 stackoverflow.com 等）通常会选择使用 ISRG（也就是 Let's Encrypt ）所签发的证书，因为申请 Let's Encrypt 证书是免费的，而申请其它 CA 的证书都需要一笔费用。



#### X.509 v3 Extensions
如果不加任何限定，我们所说的证书都是 X.509 v3 版本的证书，它是在 RFC 5280 中描述、 CA/Browser Forum Baseline Requirements 中进一步完善的 PKIX 变种。X.509 v3 证书是目前最通用的证书格式，相比之前的版本，v3 版本主要多了扩展字段，在我们正式动手创建证书体系之前，有必要对这些扩展字段有一定的了解。


下图展示了 v1, v2, v3 版本的区别：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22f73fcacdf941d4bc999447e2459725~tplv-k3u1fbpfcp-watermark.image?)

#### OpenSSL


在正式创建证书之前，我们对 OpenSSL 工具有一个简单的了解，openssl 是 Linux 下最常见的一个 PKI 工具，用来创建密钥和数字证书非常方便。本来我是写如何用 vault + cert-manager 来管理 k8s 集群证书的，而 cert-manager 可以自动引入 vault 中的 PKI 证书系统，所以用起来会更加方便。但是毕竟 OpenSSL 工具更加通用和广泛一些，所以这里就先介绍使用 OpenSSL 来创建证书体系。OpenSSL 虽然略微难用，那也是因为 PKI 本身就比较复杂，概念繁多所造成的。掌握并理解 OpenSSL 之后，换成其它 KMS 也会相对简单很多。


#### 安装 OpenSSL


建议安装最新版本的 OpenSSL (不一定最新，但建议版本在 openssl 1.1.1+，老的版本会有 CVE 高危漏洞)

```
$ wget https://www.openssl.org/source/openssl-1.1.1n.tar.gz
$ tar xf openssl-1.1.1n.tar.gz
$ cd openssl-1.1.1n
$ ./config --prefix=/usr/local/openssl
$ make & make install
```

##### OpenSSL 常见命令

-   genrsa
-   ca
-   req
-   x509
-   crl
-   ec
-   ecparam
-   enc
-   asn1parse
-   ...


##### OpenSSL 的配置文件


默认的配置文件位于 `/etc/ssl/openssl.cnf`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62d40bdd80bf45e3a8e62ef49d824a87~tplv-k3u1fbpfcp-watermark.image?)


openssl.conf 中的配置包含多个段落 (sections)，可以互相引用，openssl 的 ca 和 req 两个命令会引用对应名称的段落，比如 ca 命令会引用 `[ ca ]` 段落，而 req 命令会引用 `[ req ]` 段落。


在创建证书前，我们可以为我们的证书体系创建一个根目录，所有的证书生成都在这个目录下，从上图中可以看到，我们在操作之前应该根据自己的需求创建相应的目录及文件。

## 创建三级证书体系

我们创建三级证书体系的目的是为了方便我们在私网（比如 Home Lab）管理集群以及部署众多的 web 应用，微服务，存储及安全策略。因此，证书体系与公网环境下的并无太大区别，需要有 Root CA，Intermediate CA 以及不同应用所需的 User Certificate。之所以很多人还停留在手工制作临时性的自签名证书阶段，是因为 PKI 体系确实比较复杂繁琐，学习成本高，所以干脆放弃治疗直接建个自签名证书了事。但是，不断地把自签名导入浏览器或者系统的 Key Store 中不仅会导致 Key Store 的混乱和难以管理，而且也会增加不必要的风险。当我们有了自己的 Root CA，并且所有的用户端证书都由相应的 Intermediate CA 来签发时，我们只需要把 Root CA 导入系统的 Key Store 即可，不仅一劳永逸的解决众多自签名证书的烦恼，而且证书的管理也变得更加合理。


根据规划，我们想要的证书体系主要包含三个组成部分：Root CA Certificate, Intermediate CA Certificate, 以及 User Certificates，完整证书链如下：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a88c8978960c45f4a1ed638810c61491~tplv-k3u1fbpfcp-watermark.image?)

### 创建 Root CA 证书

首先，应该创建相应的目录及文件，接下来所有操作都在该目录中进行

```
$ mkdir facttrust
$ cd facttrust
$ mkdir -p CA/private newcerts
$ touch index.txt
$ echo -ne "00" > serial
$ echo -ne '00' > crlnumber
```

根据 `/etc/ssl/openssl.cnf` 中 `[ CA_default ]` 段落的配置信息修改成与我们上述步骤所创建的目录和文件对应的值，建议单独放在一个文件中（比如 `ca.cnf`），可以避免对全局的配置文件进行修改。


这样配置之后，后续使用 `openssl ca` 命令签发或撤销下级证书的时候会自动读取 `[ ca ]` 段落的配置信息。

#### Root CA 证书格式

想要创建一个与公网环境下相同功能的 Root CA 证书，先考察一下目前流行的权威 CA 证书的结构，可以执行如下命令：

```
$ openssl x509 -noout -text -in /usr/share/ca-certificates/DigiCert_Global_Root_CA.crt$ tar xf openssl-1.1.1n.tar.gz
$ openssl x509 -noout -text -in /usr/share/ca-certificates/Buypass_Class_3_Root_CA.crt$ ./config --prefix=/usr/local/openssl
$ openssl x509 -noout -text -in /usr/share/ca-certificates/SSL.com_Root_Certification_Authority_RSA.crt
```

上述命令查看了主流的几个权威机构的根证书的组织结构，总结有如下特点：

-   Self-signed (DV, OV, EV)            根证书都是自签名的
-   Validity: 20 ~ 30 years               有效期都在 20 ~ 30 年左右
-   x509 v3 extensions                   包含的 v3 扩展字段有如下几个：
    [v3_ca]
    -   subjectKeyIdentifier = hash
    -   authorityKeyIdentifier = keyid:always,issuer:always
    -   **basicConstraints** = critical,CA:true,pathlen:3      # 多数没有 pathlen 限制

    -   **keyUsage** = critical, cRLSign,keyCertSign,digitalSignature # 多数没有 digitalSignature
    -   extendedKeyUsage
    -   subjectAltName


**总结：**

**(1)**  Root CA 的扩展字段**均没有** extendedKeyUsage, subjectAltName, Certificate Authority Information Access, CRL Distribution Points, Certificate Policies 等字段，而这些字段主要出现在 Intermediate CA 和 User Certificate 中。


**(2)**  有的 Root CA 设置了**pathlen**（比如 `/usr/share/ca-certificates/mozilla/Baltimore_CyberTrust_Root.crt`），pathlen 主要是限制 Root CA 下属的 Intermediate CA 的层级数量。如果没有设置 pathlen，则无限制。如果 pathlen = 0，说明不能有 Intermediate CA，只能用于签发用户证书。一般来说，Root CA 的 pathlen 无限制（在 vault 中默认 -1），而 Intermediate 证书的 pathlen 一般设置为 0 或 1，证书链不宜太长。


**(3)** Root CA 的 Key Usage **通常只有 CRL Sign 和 Certificate Sign**，也就是只用来签发和吊销证书（更确切的说是只用来签发和吊销 Intermediate CA Certificates），不过也有放宽到 digitalSignature 的，比如 `/usr/share/ca-certificates/mozilla/DigiCert_Global_Root_CA.crt`。


#### 使用 openssl 命令来创建 Root CA 证书：

-   证书名字模仿主流权威机构的起名方式 (Common Name)：我们这里起名为 FactTrust Root CA
-   有效期 20 年
-   私钥算法和长度：RSA:4096 （默认 2048)
-   摘要签名算法：SHA256 (默认 SHA256)
-   密钥是否加密：不加密


```
$ openssl req -x509 -newkey rsa:4096 -sha256 -days 7300 -nodes \
  -keyout FactTrust_Root_CA.key -out FactTrust_Root_CA.crt \
  -subj "/C=US/ST=TX/L=DAL/O=Security/OU=IT Department/CN=FactTrust Root CA" \
  -addext keyUsage=critical,cRLSign,keyCertSign,digitalSignature \
  -addext basicConstraints=critical,CA:true,pathlen:3
```

注解：

(1) `openssl req -x509 -newkey` 命令会同时生成密钥 .key 和证书 .crt，默认是 PEM 格式。
PEM 格式其实就是对 DER 的内容做了 base64 的编码并做了一下格式化的输出而已。包含 `-----BEGIN RSA PRIVATE KEY-----` 这种人类可读的文本格式。
而 DER 格式的存储使用了一种叫 asn1 的数据结构来存储各个数据项，DER 就是把这个 asn1 数据结构进行序列化编码后得到的二进制编码，它不包含人类可读的文本格式等信息。 

例子：一个 RSA 私钥的 asn1 格式如下：
```
RSAPrivateKey ::= SEQUENCE {
  version           Version,
  modulus           INTEGER,  -- n
  publicExponent    INTEGER,  -- e
  privateExponent   INTEGER,  -- d
  prime1            INTEGER,  -- p
  prime2            INTEGER,  -- q
  exponent1         INTEGER,  -- d mod (p-1)
  exponent2         INTEGER,  -- d mod (q-1)
  coefficient       INTEGER,  -- (inverse of q) mod p
  otherPrimeInfos   OtherPrimeInfos OPTIONAL
}
```

用 openssl 命令来解析 DER 格式，
```
openssl rsa -inform der -in rsa_private.der -text -noout
```

得到如下信息：

```
RSA Private-Key: (2048 bit, 2 primes)
modulus:
    00:da:d7:8c:0a:86:b4:cc:34:b5:b0:3a:64:00:13:
    ...
publicExponent: 65537 (0x10001)
privateExponent:
    15:56:6a:eb:23:d3:41:0d:ea:a1:32:30:49:e9:99:
    ...
prime1:
    00:f2:4f:63:7a:d3:cb:fb:ca:9d:77:2f:ba:88:0a:
    ...
prime2:
    00:e7:34:bb:28:ab:c1:c8:10:74:19:a4:45:04:b5:
    ...
exponent1:
    1a:95:67:1e:94:99:ee:77:de:2a:b3:4b:cd:9d:07:
    ...
exponent2:
    00:d5:d6:99:7b:a6:4f:d6:00:11:c1:5d:83:50:35:
    ...
coefficient:
    00:c4:91:79:ab:09:f7:8c:c4:9d:81:1c:54:95:6e:
    ...
```

由于 PEM 只是 DER 格式进行 BASE64 编码封装，因此，我们直接解析 PEM 格式也可以同样得到上述信息，命令如下：
```
openssl rsa -inform pem -in rsa_private.pem -text -noout
```


(2) openssl req 命令创建密钥时，默认会采用交互式的方式要求你填写 Subject 信息（Country Name, State, Locality, Organization, Common Name 以及 Email 等信息），使用 `-subj` 选项可以一次性填入，无需交互。另外，如果你仅想使用默认值，并且禁止交互，那么可以用 `-batch` 选项。

(3) 使用 `-addext` 选项添加 customized X509 v3 extenstions。

(4) `-nodes` 表示 no DES，也就是不对生成的密钥进行加密，如果没有这个选项，那么会提示你输入一个密码来保护 private key。


**加密 Private Key**

由于 Private key 的重要性，通常建议把它进行加密。openssl 默认使用的是 DES3 的对称加密方式，但是由于它的 block size 只有 56 位，被认为不是足够安全，所以，推荐使用 AES 的加密方式。

接下来我们生成一个密码文件，方法是，先把明文密码保存到一个文件中（e.g. `capass.txt`），然后对这个文件进行加盐加密处理得到一个新文件（e.g. `capass.txt.enc`），我们把这个文件叫密码文件。最后用这个密码文件对密钥进行加密，加密算法使用 AES256。


**1) 生成密码文件**

```
$ echo '!mrgWnh*2dEgpdE4X=' > capass
$ openssl enc -aes256 -pbkdf2 -salt -in capass -out capass.enc
```

解密密码文件可以用如下命令：

```
$ openssl enc -aes256 -pbkdf2 -salt -d -in capass.enc
```


**2) 生成 Root CA 的 Private key 和 Certificiate**

```
$ openssl req -x509 -newkey rsa:4096 -sha256 -days 7300 -passout file:capass.enc \ 
  -keyout FactTrust_Root_CA.key -out FactTrust_Root_CA.crt \
  -subj "/C=US/ST=TX/L=DAL/O=Security/OU=IT Department/CN=FactTrust Root CA" \
  -addext keyUsage=critical,cRLSign,keyCertSign,digitalSignature \
  -addext basicConstraints=critical,CA:true,pathlen:3
```

注意，上面的命令把之前的 `-nodes` 替换成了 `-passout file:capass.enc`，这样生成的 `FactTrust_Root_CA.key` 会是加密后的格式，如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4333baed8934347ad85a34991742974~tplv-k3u1fbpfcp-watermark.image?)

我们用 `openssl asn1parse` 命令解析该私钥会发现，它默认使用的是 DES3 方式加密，如下：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a7d64a9c4614db6970aec4338aa256a~tplv-k3u1fbpfcp-watermark.image?)


接下来，我们把它转换成 AES256 方式加密

在操作前，同样先准备一个密码文件，之前使用的是默认 DES3 加密，而这次使用 AES256 加密，所以需要再准备另一个密码文件，不过你也可以用之前的，只不过你需要再拷贝一份，因为 openssl 不支持 `-passin` 和 `-passout` 使用同一个密码文件。

```
$ echo '6Cz4P$9bW8ZB55^j=v' > aespass
$ openssl enc -aes256 -pbkdf2 -salt -in aespass -out aespass.enc
```

使用 `openssl pkcs8` 命令转换成 AES256 加密：

```
$ openssl pkcs8 -topk8 -v2 aes256 -in FactTrust_Root_CA.key -out FactTrust_Root_CA-PKCS8-AES256.key -passin file:capass.enc -passout file:aespass.enc
```

解释：

`-topk8` 选项的意思就是 to pk8，也就是转换成 pkcs#8 格式，它比 pkcs#1 格式多 20 字节，主要用于描述算法类型等 metadata 信息。pkcs#1 只支持 RSA 密钥算法，而 pkcs#8 更加通用，支持多种密钥算法。对称加密算法是在 pkcs#5 中定义的，pkcs#5 v1.5 只支持 DES 算法，而 v2 则支持 AES 等算法。


`-v2 aes256` 引用的是 pkcs#5 的 v2 版本，我们可以省略该选项，因为 pkcs8 命令默认就是使用 pkcs#5 的 v2 中的 aes256 算法。


另外，也可以用 `openssl rsa` 命令来转换，不过它会生成传统的 pkcs#1 格式的密钥。

```
$ openssl rsa -aes256 -in FactTrust_Root_CA.key -out FactTrust_Root_CA-PKCS1-AES256.key -passin file:capass.enc -passout file:aespass.enc
```


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ebe2039e09745aca23280f0e4249664~tplv-k3u1fbpfcp-watermark.image?)

再次使用 `openssl asn1parse` 命令检查 key 格式，就已经是 aes256 加密的了，如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a71cd1373b6a496facf5bb6ba0b5c39a~tplv-k3u1fbpfcp-watermark.image?)

#### 去除 Key 的密码保护
如果在某些自动化场景或者云环境中（无法访问我们的密码文件或不支持加密文件），那么我们必须使用非加密的 Private Key，下面的命令可以把 Encrypted Private Key 转换成 Unencrypted Private Key：

```
$ openssl pkcs8 -topk8 -nocrypt -passin file:aespass.enc \
-in FactTrust_Root_CA-PKCS8-AES256.key \
-out FactTrust_Root_CA-PKCS8-decrypted.key
```

or

```
$ openssl rsa -passin file:aespass.enc \
-in FactTrust_Root_CA-PKCS8-AES256.key \
-out FactTrust_Root_CA-decrypted.key
```

加密后的私钥受到密码的保护，因此，只要需要访问到 Private Key 的场景，都需要用到密码文件，使用 `-passin` 选项传入密码文件，比如下面的命令用于提取公钥：

```
$ openssl rsa -in FactTrust_Root_CA-PKCS8-AES256.key -pubout -out pub-FactTrust_Root_CA.key -passin file:aespass.enc
```


新生成的 Root CA 证书可以使用 `openssl x509` 命令查看：

```
$ openssl x509 -noout -text -in FactTrust_Root_CA.crt
```

最后，别忘了把创建好的 Key 和 CRT 放入 `[ CA_default ]` 中指定的目标目录

```
$ mv FactTrust_Root_CA-PKCS8-AES256.key CA/private/
```

同时，还应该及时导入系统证书目录：

```
sudo cp FactTrust_Root_CA.crt /usr/local/share/ca-certificates/FactTrust_Root_CA.crt
sudo update-ca-certificates
```

接下来，创建一个撤销列表 CRL，由于目前并没有签发和撤销过下级证书，所以 CRL 肯定是空的：

```
$ openssl ca -gencrl -config ca.cnf -out crl.pem -passin file:aespass.enc
```


![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5636212ee29945f4b7d728074bc5b13f~tplv-k3u1fbpfcp-watermark.image?)


**步骤总结：**


1. 先一键式创建 Root CA 的 Key 和 CRT（并为 Key 添加密码保护）


2. 把对 Key 的加密保护方式转换成 AES256 加密（因为 openssl req 命令默认只能对 Key 进行 DES3 加密）

上面介绍的是 **One-liner** 一键式地创建 Root CA，一个更常见的流程是先创建 Key，然后创建 CSR，然后用 Key 对 CSR 签名从而得到 CRT。


这里就不过多介绍，因为下面我们创建 Intermediate CA 证书以及 User 证书时会用这个流程来创建。


## 创建 Intermediate CA 证书


**创建目录**

同 Root CA，我们为 Intermediate CA 创建单独的目录及必要文件

```
$ mkdir facttrust-rsa-ica1
$ cd facttrust-rsa-ica1
$ mkdir -p CA/private newcerts
$ touch index.txt
$ echo -ne "00" > serial
$ echo -ne '00' > crlnumber
```

目录结构同样在按照 `[ CA_default ]` 段落的配置信息修改，这里为避免对全局的配置文件进行修改，也单独创建一个配置文件来存放配置信息 `ica.cnf`，



##### 创建 Intermediata CA 证书的简要步骤:


1. 先创建 Key Pairs

2. 配置必要的 v3 extensions 信息，并据此创建 CSR

3. 对 CSR 签名从而得到 CRT

创建 Intermediate CA 证书和终端 User 证书的步骤比较类似，因为它们都可以被吊销，所以我们需要为它们创建编号 (Serial Number) 以便可以记录它们的创建序列，Serial Number 必须是唯一的，撤销时会按照 Serial Number 来进行撤销。Root CA 通常不吊销自己，所以在被创建时，它的 Serial Number 会是一个独立的随机值，它不在这个我们接下来要使用的 Serial Nubmer List 中。


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/893ec241c51a47f38ed5ee8f8a457b9a~tplv-k3u1fbpfcp-watermark.image?)

**Intermediate CA 证书的格式（组织结构）**


在创建之前，我们同样先扒拉下来一些主流权威机构的中间证书，通过参考它们的组织结构来构造我们的中间证书。

我们可以直接打开某 HTTPS 网站来查看其中间证书，


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0325f17b5d06438f8f73baa2cee08111~tplv-k3u1fbpfcp-watermark.image?)


可以看到，v3 extensionts 部分比 Root CA 多了不少内容，比如 Extended Key Usage, Certificates Policies, CRL Distribution Points 等。


另外，Intermediate CA 证书的有效期也要更短，通常是 3 ~ 10 年。其它部分与 Root CA 没有太大区别。

如果需要下载 PEM 格式的 Intermediate CA 证书和 User 证书，可以使用如下命令：

```
$ openssl s_client -connect github.com:443 -showcerts </dev/null
$ openssl s_client -connect github.com:443 -showcerts </dev/null | openssl x509 -outform PEM > github.pem
```

##### Intermediate CA 证书格式：

在查看了几个主流的权威机构的中间证书的组织结构后，可以总结如下：

-   Issued by Root CA                    Intermediate CA 证书由 Root CA 签名
-   Validity: 3 ~ 10 years               有效期在 3 ~ 10 年左右
-   x509 v3 extensions                  包含的 v3 扩展字段有如下几个：\
    [v3_ca]
    - subjectKeyIdentifier = hash
    - authorityKeyIdentifier = keyid:always,issuer:always
    -   **basicConstraints** = critical,CA:true,pathlen:0      # pathlen:0 表示只能签发端证书，不能再有 intermediate CA

    -   **keyUsage** = critical, cRLSign,keyCertSign,digitalSignature 
    -   **extendedKeyUsage**=clientAuth,serverAuth
    -   **authorityInfoAccess**
        -   OCSP
        -   CA Issuers
    -   **crlDistributionPoints** = URI:http://crl.pki.goog/gtsr1/gtsr1.crl\

    -   **certificatePolicies**
        -   Policy ID #1
        -   Qualifier ID #1
            -   CPS: URI
            -   UserNotice\

    -   subjectAltName


**总结：**

**(1)**  相比 Root CA，Intermediate CA 的扩展字段增加了 extendedKeyUsage, Certificate Authority Information Access, CRL Distribution Points, Certificate Policies 等字段，**不过依然没有 subjectAltName 字段**。subjectAltName 通常只出现在 User 证书上。除此之外，Intermediate CA 和 User Certificate 的扩展字段基本相同。

**(2)**  Intermediate CA 证书的**pathlen** 一般设置为 0 或 1，也就是不再下辖下级中间证书或者最多再下辖一个中间证书。


**使用 openssl 命令来创建 Intermediata CA 证书：**

-   Intermediata CA 证书起名 (Common Name) 为 FactTrust Secure RSA ICA1
-   有效期 5 年
-   Country Name 和 Organization 与 Root CA 对应的值相同 (Policy match)
-   私钥算法和长度：RSA:4096 （默认 2048)
-   摘要签名算法：SHA256 (默认 SHA256)
-   密钥是否加密：加密


**1. 创建 Key pairs (密码保护)**

```
$ cd facttrust-rsa-ica1
$ echo 'Uw3b4r#6Nn6$gMJi^7' > icapass
$ openssl enc -aes256 -pbkdf2 -salt -in icapass -out icapass.enc
$ openssl genrsa -aes256 -passout file:icapass.enc -out FactTrust_RSA_ICA1.key 4096
```

上述命令生成的是 **pkcs#1** 格式的 Key，如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f07f3f7be3f9409196cbded491ca5f31~tplv-k3u1fbpfcp-watermark.image?)


pkcs#1 格式的 Key 无法用 `openssl asn1parse` 命令读取，我们应该把它转换成更通用的 **pkcs#8** 格式的 Key

```
$ cp icapass.enc icapass-pkcs8.enc
$ openssl pkcs8 -topk8 -in FactTrust_RSA_ICA1.key -out FactTrust_RSA_ICA1-PKCS8.key -passin file:icapass.enc -passout file:icapass-pkcs8.enc
```

转换成功后：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9082247521e4fce855c84eb1d270ae4~tplv-k3u1fbpfcp-watermark.image?)


**2. 创建 CSR**

```
$ openssl req -new -sha256 -passin file:icapass-pkcs8.enc \
    -key FactTrust_RSA_ICA1-PKCS8.key -out FactTrust_RSA_ICA1-PKCS8.csr \
    -subj="/C=US/O=Security/CN=FactTrust Secure RSA ICA1" \
    -reqexts req_ext -config <(cat /etc/ssl/openssl.cnf -<<END
[req_ext]
basicConstraints = critical,CA:true,pathlen:0
subjectKeyIdentifier = hash
keyUsage = critical,digitalSignature,keyCertSign,cRLSign
extendedKeyUsage = clientAuth,serverAuth
authorityInfoAccess = OCSP;URI:http://ocsp.facttrust.com/,caIssuers;URI:http://facttrust.com/certs/FactTrustRootCA.der
crlDistributionPoints = URI:http://crl.facttrust.com/FactTrustRootCA.crl
certificatePolicies = @pol
[pol]
policyIdentifier = 2.5.29.32.0
CPS.1 = "https://www.facttrust.com/CPS"
userNotice.1 = @notice
[notice]
explicitText = "UTF8:Notice An use of this Certificate constitutes acceptance of the Relying Party Agreement located at https://www.facttrust.com/rpa-ua"
END
)
```


解释：

req 命令会使用配置文件中 `[ req ]` 段落中的内容，如果是使用 `req` 命令创建 CSR，那么 CSR 中的扩展字段由 req_extension 决定，如果是使用 `req x509` 命令，那么 crt 中的扩展字段由 x509_extensions 决定，如下：


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c78e95eb74ba4e43943cf4d3fd320c5e~tplv-k3u1fbpfcp-watermark.image?)


**`-reqext`** 选项用来指定一个覆盖 req_extensions 值的段落；默认的 `/etc/ssl/openssl.cnf` 中，其值是来自 v3_req 段落。这里我们使用了一个自定义的段落 req_ext 来覆盖其默认值。


上述命令为了方便，所以用的是 One-liner，我们完全可以把其内容填写到一个单独文件中，然后同样引用该文件中的 req_ext 段落即可，

reqexts.cnf

```
$ cat reqexts.cnf 
[req_ext]
basicConstraints = critical,CA:true,pathlen:0
subjectKeyIdentifier = hash
keyUsage = critical,digitalSignature,keyCertSign,cRLSign
extendedKeyUsage = clientAuth,serverAuth
authorityInfoAccess = OCSP;URI:http://ocsp.facttrust.com/,caIssuers;URI:http://facttrust.com/certs/FactTrustRootCA.der
crlDistributionPoints = URI:http://crl.facttrust.com/FactTrustRootCA.crl
certificatePolicies = @pol
[pol]
policyIdentifier = 2.5.29.32.0
CPS.1 = "https://www.facttrust.com/CPS"
userNotice.1 = @notice
[notice]
explicitText = "UTF8:Notice An use of this Certificate constitutes acceptance of the Relying Party Agreement located at https://www.facttrust.com/rpa-ua"
```

generate CSR

```shell
$ openssl req -new -sha256 -passin file:icapass-pkcs8.enc \
    -key FactTrust_RSA_ICA1-PKCS8.key -out FactTrust_RSA_ICA1-PKCS8.csr \
    -subj="/C=US/O=Security/CN=FactTrust Secure RSA ICA1" \
    -reqexts req_ext -config <(cat ica.cnf reqexts.cnf)
```


查看 CSR 以确定格式正确

```
$ openssl req -text -noout -in FactTrust_RSA_ICA1-PKCS8.csr
```

如果格式不正确，我们可以调整参数，继续重新生成。


**3. 使用 Root CA 证书对 Intermediate CSR 签名**


CSR 创建好了之后，使用 Root CA 对其签名就可以得到 Intermediate CA 证书了

签名时需要注意几点：

1）openssl ca 命令会有 policy matching 要求，一般来说，Country Name 和 Organization 必须匹配，Common Name 必须提供。


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a9a41afc94d4b2ab3bce4b4df332da6~tplv-k3u1fbpfcp-watermark.image?)

2）可以直接使用 `[ CA_default ]` 中的 copy_extensions 字段来把 CSR 中的 extension 都拷贝到最终的 `.crt` 中，但是一般并不建议这么做，而是由 CA 签发是自己决定最终的 extensions 有哪些（保留 CA 的自主决定权）。

`openssl ca` 命令使用 `-extensions` 选项来覆盖默认的 x509_extensions 字段的值。


由于是使用 Root CA 进行签发，所以我们**需要回到 Root CA 目录**，

```
$ cd ../facttrust/
```

然后执行如下的签发命令：

```
$ openssl ca -days 1825 -passin file:../facttrust/CA/private/aespass.enc \
    -in FactTrust_RSA_ICA1-PKCS8.csr -out FactTrust_RSA_ICA1-PKCS8.crt \
    -cert ../facttrust/FactTrust_Root_CA.crt \
    -keyfile ../facttrust/CA/private/FactTrust_Root_CA-PKCS8-AES256.key \
    -create_serial \
    -extensions req_ext \
    -config <(cat ica.cnf reqexts.cnf) \
    -policy policy_match
```


最终签发的证书还会拷贝一份在 `newcerts/` 目录中，命名方式是 `<serial-number>.pem`，我们来查看最终的证书：

```
$ openssl x509 -text -noout -in FactTrust_RSA_ICA1-PKCS8.crt
```

签发之后, `index.txt` 中会有一条记录


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87f86d41a5ed4382b038014da2d069c2~tplv-k3u1fbpfcp-watermark.image?)

## 创建 User/Leaf 证书

**User 证书处于证书链的最下游，因此也叫 Leaf 证书。** 我们平时接触最多的就是 User 证书，因为所有的 HTTPS 应用都需要一个合法的 User 证书才能正常工作。

**创建 User 证书的步骤与创建 Intermediate CA 证书并无区别，流程都是：**

1. 先创建 Key Pairs

2. 配置必要的 v3 extensions 信息，并据此创建 CSR

3. 对 CSR 签名从而得到 CRT

**User 证书的格式**

前面介绍过，User 证书中的 extensions 字段与 Intermediate CA 证书基本相同，唯一一个区别是 User 证书中有 **subjectAltName** 字段，这也是重要的一个字段，如果你签发的 User 证书没有这个字段，那么会被认为不是一个合规的证书，浏览器会拒绝。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e838c6b2b51462d96bc1e05138f6eb3~tplv-k3u1fbpfcp-watermark.image?)

**使用 openssl 命令来创建 User 证书：**

-   User 证书的名字 (Common Name) 通常为域名或泛域名。
-   有效期 1 年（短的有 1 到 3 个月，长的则 1 年左右）
-   证书类型：DV
    -   Policy ID 为 2.23.140.1.2.1
-   DV 类型的证书 Subject 中只需要填写 Common Name。
-   Key Usage: Digital Signature, Key Encipherment
-   Extended Key Usage: Client Auth, Server Auth\
-   私钥算法和长度：RSA:4096 （默认 2048)
-   摘要签名算法：SHA256 (默认 SHA256)
-   密钥是否加密：否


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8150bbe2d364f43af773ef55cb3e94e~tplv-k3u1fbpfcp-watermark.image?)


#### 1. 创建 Key + CSR

这里我们可以直接同时创建 Key 和 CSR

```
$ openssl req -new -sha256 -nodes \
    -keyout winkee.key -out winkee.csr \
    -subj "/CN=*.winkee.lab" \
    -reqexts user_ext -config <(cat /etc/ssl/openssl.cnf -<<END
[user_ext]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth,serverAuth
authorityInfoAccess = OCSP;URI:http://ocsp.facttrust.com/,caIssuers;URI:http://facttrust.com/certs/FactTrustRSAICA1.der
crlDistributionPoints = URI:http://crl.facttrust.com/FactTrustRSAICA1.crl
certificatePolicies = @pol
subjectAltName = @alt_names
[pol]
policyIdentifier = 2.23.140.1.2.1 # DV certificate
CPS.1 = "https://www.facttrust.com/CPS"
[alt_names]
IP.1 = 127.0.0.1
IP.2 = 10.10.0.3
DNS.1 = localhost
DNS.2 = winkee.lab
DNS.3 = *.winkee.lab
END
)
```

生成的 key 已经是 **pkcs#8** 格式的，所以无需转换。


查看 CSR 以确定格式正确

```
$ openssl req -text -noout -in winkee.csr
```

如果格式不正确，我们可以调整参数，继续重新生成。


#### 2. 使用 Intermediate CA 证书对用户证书签名

由于是使用 Intermediate CA 进行签发，所以接下来的签发操作**请确保在** **Intermediate CA 目录中进行。**

```
$ openssl ca -days 365 \
    -in winkee.csr -out winkee.crt \
    -cert ./FactTrust_RSA_ICA1-PKCS8.crt \
    -keyfile ./CA/private/FactTrust_RSA_ICA1-PKCS8.key \
    -create_serial \
    -extensions user_ext \
    -config <(cat ica.cnf user.cnf) \
    -policy policy_anything
```


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/908472d42f9b4da8a9b4b19c0363662e~tplv-k3u1fbpfcp-watermark.image?)

同样，最终签发的证书会拷贝一份在 `newcerts/` 目录中，命名方式是 `<serial-number>.pem`，我们来查看最终的证书：

```
$ openssl x509 -text -noout -in winkee.crt
```

签发之后, index.txt 中会有一条记录

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b2da6327043476a9a7bb1609ba92b1d~tplv-k3u1fbpfcp-zoom-1.image)


至此，我们就创建了一个完整的证书体系啦。如果想要签发更多的证书，我们只需要按照上面的步骤依葫芦画瓢就可以完成。


当然，还留下了一些其它细节有待进一步探讨，比如 User 证书的分发，CRL 列表更新，OCSP 服务器的建立等等，留待后续文章继续介绍。



### 引用链接

[1] rfc5280 https://datatracker.ietf.org/doc/html/rfc5280

[2] AES vs. DES Encryption: Why AES has replaced DES, 3DES and TDEA https://www.precisely.com/blog/data-security/aes-vs-des-encryption-standard-3des-tdea#:~:text=In%20terms%20of%20structure%2C%20DES,to%20create%20the%20encrypted%20block.\
[3] Converting a Traditional PEM-Encoded Encrypted Private Key to PKCS#8 Format https://help.globalscape.com/help/archive/secureserver3/Converting_an_incompatible_traditional_PEM_encoded_encrypted_private_key.htm\
[4] x509v3_config https://www.openssl.org/docs/man1.0.2/man5/x509v3_config.html\
[5] OpenSSL Certificate Authority https://wechris.github.io/tips-tutorials/openssl/ssl/tls/security/certificate/2018/06/16/OpenSSL-Certificate-Authority-Part1/\
[6] PGP - The Web of Trust is Dead https://inversegravity.net/2019/web-of-trust-dead/\
[7] PGP Web of Trust: Delegated Trust and Keyservers https://www.linux.com/news/pgp-web-trust-delegated-trust-and-keyservers/

[8] OpenSSL create self signed certificate Linux with example https://www.golinuxcloud.com/generate-self-signed-certificate-openssl/

[9] The Most Popular SSL Certificate Authorities Reviewed https://wpmudev.com/blog/ssl-certificate-authorities-reviewed/
[10] Openssl: Setup an intermediate CA https://michlstechblog.info/blog/openssl-setup-an-intermediate-ca/

[11] Error getting keypair for CA issuer https://github.com/cert-manager/cert-manager/issues/279\
[12] Intermediate CA configuration file https://jamielinux.com/docs/openssl-certificate-authority/appendix/intermediate-configuration-file.htm

[13] How to setup your own CA with OpenSSL https://gist.github.com/jalogisch/d5daa6781ee0c1452e8929814d9bba53

[14] Everything you should know about certificates and PKI but are too afraid to ask: https://smallstep.com/blog/everything-pki/


如果你觉得我的文章对你有帮助，欢迎留言或者关注我的专栏。



微信公众号：“知辉”


搜索 codeandroad 或


扫描二维码
