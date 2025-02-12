### 创建用户证书 User Cert

现在，我们终于来到证书链的最下层，用户证书，也叫实体证书（end-entity certificate）。这就是我们平常最常见到的网站的数字证书了，它是支撑 HTTPS 协议的基础设施之一。

要创建 User Cert，我们需要与它的签发机构，也就是 Intermediate CA 进行交互。 

下面是目录框架示意图：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/ica_and_user_certs_hierachy2.jpg)

可以看到，我用两个平行（而不是层级）的目录来管理 Intermediate CA 和 User Certs。
并且，我在 `user_certs` 目录下为不同的 Web Service 所需的证书创建了单独的目录。


#### 操作要点

首先，逻辑和步骤与我们之前签发 Intermediate CA 证书的过程是一样的。
同样是三个步骤：

- 创建 Private Key + Password file
- 创建 CSR
- CA 对 CSR 签名，签发证书

主要的区别在于参数配置不一样，比如每个不同的网站的扩展信息不一样（主要是 subjectAltName，也叫 SAN）。


### 实战案例

现在假设，我们有一个 Web 服务器，它提供 echo 服务，同时支持 `echo1.example.com` 和 `echo2.example.com` 和泛域名 `*.example.com`，还支持 IP 地址访问。

进入 `user_certs/echo` 目录：

**注：请提前按目录框架图创建好相应目录**

```
cd user_certs/echo
```

#### (1) 创建密码文件和私钥

创建密码文件

```
openssl rand -base64 32 > private/password.txt
chmod 600 private/password.txt
```

创建私钥（RSA 算法）

```
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out rsa_echo.key
openssl pkcs8 -topk8 -v2 aes256 -iter 100000 -passout file:private/password.txt \
    -in rsa_echo.key -out private/rsa_echo_encrypted.key
shred -u rsa_echo.key
```

检查私钥

```
openssl asn1parse -in private/rsa_echo_encrypted.key
```

#### (2) 创建 CSR

##### i) 配置信息

`req_csr.cnf`

```tcl
[ req ]
default_bits       = 4096
default_keyfile    = rsa_encrypted.key
distinguished_name = req_distinguished_name
x509_extensions    = x509_v3_ext
prompt             = no

[ req_distinguished_name ]
C  = US
ST = TX
O  = echo service
CN = echo service

[x509_v3_ext]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
keyUsage = digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = clientAuth,serverAuth
subjectAltName = @alt_names

[alt_names]
IP.1 = 10.10.0.100
DNS.1 = echo1.example.com
DNS.2 = echo2.example.com
DNS.3 = *.example.com
```


##### ii) 创建 CSR

```
openssl req -new \
    -passin file:private/password.txt \
    -config req_csr.cnf \
    -reqexts x509_v3_ext \
    -key private/rsa_echo_encrypted.key \
    -out rsa_echo.csr
```


##### iii) 检查 CSR

```
openssl req -text -noout -in rsa_echo.csr 
```


现在有了 `.csr` 文件，`int_ca` 就可以对其签发证书了。


#### 切换到 `int_ca` 目录进行操作

#### (1) 在 `int_ca` 中创建必要文件和目录


```
cd ../../int_ca

mkdir -pv newcerts
touch index.txt
echo -ne "00" > serial
echo -ne "00" > crlnumber
```

#### (2) 准备配置信息

使用 `int_ca` 的私钥为 `echo` 签发证书之前，我们需要再检查一下 `int_ca` 的各项配置参数是否符合我们的要求：

`ica.cnf`

```
[ ca ]
default_ca              = CA_default

[ CA_default ]
dir                     = .
certs                   = $dir/certs
crl_dir                 = $dir/crl
database                = $dir/index.txt
new_certs_dir           = $dir/newcerts
private_key             = $dir/CA/private/rsa_encrypted.key
certificate             = $dir/CA/rsa_int_ca.crt
serial                  = $dir/serial
crlnumber               = $dir/crlnumber
default_md              = sha256
default_days            = 3650
default_crl_days        = 30
policy                  = policy_match

copy_extensions         = copy

[ policy_match ]
countryName             = match
stateOrProvinceName     = match
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
```



由于之前在生成 CSR 时，已写入了 `v3_intermediate_ca` 字段中的信息。因此，我们在使用 `int_ca` 给其签发证书的时候，可以直接设置 `copy_extensions = copy`，它表示 `int_ca` 签署时会直接拷贝 CSR 中的扩展信息，不做任何改变。


不过，如果我们需要为即将要签发的证书定制 v3 extension，我们可以在 `ica.cnf` 中增加一个段落 `[v3_ext_echo]`，然后在执行命令时用 `-extension` 选项来指定这个段落。


```
[v3_ext_echo]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
keyUsage = digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = clientAuth,serverAuth
subjectAltName = @alt_names

[alt_names]
IP.1 = 10.10.0.100
DNS.1 = echo1.example.com
DNS.2 = echo2.example.com
DNS.3 = *.example.com
```

推荐用 copy 的方式，省事。


#### （3）签发证书

`openssl ca` 和 `openssl x509` 两个命令都可以用来签发证书，区别在于前面的 ca 命令有数据库对所签发的证书进行记录（比如 serial, crlnumber 等），而后面的 x509 命令则适用于 ad-hoc 证书的签发，不管理数据库，不记录签发了哪些证书，所以只适合需要一次性证书（比如自签名证书）并且不负责任何后续管理的场景。

由于我们这里专注于证书链系统的构建，所以我们一律使用 `openssl ca` 命令来构建和管理证书。


##### 注意：签发证书是在 `int_ca` 目录中操作！

执行命令：

```
openssl ca -days 365 -notext -batch \
    -config ica.cnf \
    -cert CA/rsa_int_ca.crt \
    -keyfile CA/private/rsa_encrypted.key \
    -passin file:CA/private/password.txt \
    -in ../user_certs/echo/rsa_echo.csr \
    -out ../user_certs/echo/rsa_echo.crt
```

注：用户证书通常在一年以内，甚至更短。

输出结果：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/user_certs_echo_succeed.jpg)

###### 补充：

如果想由 `int_ca` 指定 v3 extension 而不是 copy，我们可以这么做：

- 创建一个单独的文件（如 `v3_ext_echo.cnf`），在里面的段落 `[ v3_ext_echo ]` 中填上相应内容。
- 或者直接在 `ica.cnf` 中增加一个段落 `[ v3_ext_echo ]`，填上相应内容。

然后执行下面的命令：

```
openssl ca -days 365 -notext \
  -config <(cat ica.cnf v3_ext_echo.cnf) \
  -extensions v3_ext_echo \
  -cert CA/rsa_int_ca.crt \
  -keyfile CA/private/rsa_encrypted.key \
  -passin file:CA/private/password.txt \
  -in ../user_certs/echo/rsa_echo.csr \
  -out ../user_certs/echo/rsa_echo.crt
```


新证书会生成在 `newcerts/` 目录中，默认名字是 serial 编号。比如初始的 serial 是 00，那么签发的第一个证书就是 `newcerts/00.pem`，serial 文件中会变成 01。

`index.txt` 中也会增加一条记录，如下：

```
V   251030162019Z       00  unknown /C=US/ST=TX/O=echo service
```


#### (4) 检查证书

#### 进入 `user_certs/echo` 目录

```
cd user_certs/echo
openssl x509 -text -noout -in rsa_echo.crt
```


#### 重复签发

如果我们使用 `openssl ca` 命令对同一个 `.csr` 文件再次签名，那么会出现一个错误：

```
ERROR: There is already a certificate for /CN=echo service
```

可见，我们不能对同一个 `/CN` 多次签发证书。想要签发的话，必须先吊销之前的证书。我们后续文章还会继续介绍证书到期和吊销的相关知识。



全文完！

下集预告：全面解析！用户证书的使用，用 Nginx 搭建 HTTPS 服务器。



如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。

