### 创建 Intermediate CA

本文继续讲解三级证书体系中的中间权威机构证书的制作，先看一个证书层级目录树：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/allcerts_hierachy.png)

可以看到，证书层级分为 Root CA, Intermediate CA 和 User Certs。虽然它们在概念上是层级关系，但物理上，我这里还是把它们分别放在了平行的独立目录中进行管理，因为我发现这样更好管理。

Root CA 和 Intermediate CA 都叫做 CA 证书，可以签发下级证书（比如继续签发下级 Intermediate CA 证书，或 User 证书）。而 User Certs 不是 CA 证书，它只能作为实体证书用作认证使用，不能再签发下级证书。

###### 接下来是详细的操作要点：

首先，在创建 Intermediate CA 之前，先确保 Root CA 已经准备好：

- `rsa_encrypted.key` + `password.txt` + `rsa_ca.crt`
- `req.cnf` + `ca.cnf`
- `serial` + `crlnumber` + `index.txt`


关于 Root CA 的详细内容，可以参考前面的系列文章。

同理，Intermediate CA 的证书是由 Root CA 所签发的，需要创建 CSR 请求文件。同时，Intermediate CA 也是 User Certs 的签发机构，需要有证书管理文件（serial, index.txt 和 crlnubmer)
因此，下列文件也准备好：

- `password.txt`
- `req_ext_csr_ica.cnf` + `ica.cnf`
- `serial` + `crlnumber` + `index.txt` （由于本文暂时不讨论中间证书签发用户证书的问题，所以不涉及管理用户证书，这几个文件以及 newcerts 目录可以暂时不创建，省略）



步骤大致如下：

### 进入 int_ca 目录

#### (1) 按照上面图式中的目录层级结构，为 int_ca 创建好目录和文件。

```
cd int_ca
mkdir -pv CA/private
```

解释：

- CA/ 目录保管最终的中间证书，CA/private 保管私钥和密码文件
- 由于本文暂不涉及使用中间证书向下签发用户证书，所以 serial, index.txt 等证书管理文件暂时不需要。

下面是我们创建中间证书的大致步骤：

- （a） 创建 int_ca 的私钥
- （b） 创建 int_ca 的 CSR
- （c） 使用 root_ca 对 CSR 签名，从而创建出 `int_ca.crt`


#### (2) 创建私钥

可以依据我们的需要选择不同的私钥算法：RSA，ECDSA 或 Ed25519。

RSA 

```
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out rsa.key
openssl pkcs8 -topk8 -v2 aes256 -iter 100000 -passout file:CA/private/password.txt \
    -in rsa.key -out rsa_encrypted.key
```

ECDSA

```
openssl ecparam -genkey -name secp384r1 -out secp384r1.key
openssl pkcs8 -topk8 -v2 aes256 -iter 100000 -passout file:CA/private/password.txt \
    -in secp384r1.key -out secp384r1_encrypted.key
```

Ed25519

```
openssl genpkey -algorithm ed25519 -out ed25519.key
openssl pkcs8 -topk8 -v2 aes256 -iter 100000 -passout file:CA/private/password.txt \
    -in ed25519.key -out ed25519_encrypted.key
```


##### 检查私钥

```
openssl asn1parse -in rsa_encrypted.key
```

##### 放到指定位置

```
shred -u rsa.key
mv rsa_encrypted.key  CA/private/rsa_encrypted.key
```

#### (3) 创建 CSR

有了私钥，接下来，我们需要创建 CSR，以请求 `root_ca` 能为 `int_ca` 签发中间证书。


###### i) 配置信息：`req_ext_csr_ica.cnf`

```tcl
[ req ]
default_bits       = 4096
default_keyfile    = rsa_key_encrypted.pem
distinguished_name = req_distinguished_name
x509_extensions    = v3_intermediate_ca
prompt             = no

[ req_distinguished_name ]
C  = US
ST = California
L  = San Francisco
O  = RSA Intermediate CA
OU = Intermediate CA Unit
CN = RSA Intermediate CA

[ v3_intermediate_ca ]
# Basic Constraints (required for intermediate CA)
basicConstraints = critical, CA:TRUE, pathlen:0
keyUsage = critical, keyCertSign, cRLSign, digitalSignature
extendedKeyUsage = serverAuth, clientAuth
subjectKeyIdentifier = hash
# authorityKeyIdentifier = keyid:always,issuer:always
```

###### 解释：

**pathlen** 定义了当前证书可以签发多少层下级证书。 

- 未 pathlen，则无限制。此时默认为 -1
- pathlen:0 表示只能签发用户证书（User Certificate，也叫 end-entity certificate），不能签发 Intermediate 证书。
- pathlen:1 表示只能签发一层 Intermediate 证书，即当前的 CA 证书与用户证书之间，最多只能有一层中间证书。
- pathlen:2 以此类推，当前证书与用户证书之间，可以有两层中间证书。
- Intermediate 证书的 pathlen 一般设置为 0 或 1，证书链不宜太长。


###### ii) 创建 CSR

```
openssl req -new \
    -passin file:CA/private/password.txt \
    -config req_ext_csr_ica.cnf \
    -reqexts v3_intermediate_ca \
    -key CA/private/rsa_encrypted.key \
    -out rsa_ica.csr
```

注意：

- 只有添加了 `-reqexts` 选项，CSR 中才会写入 v3 extension 信息。
- 我们也可以不使用配置文件，而是全部通过命令行的方式来指定，不过比较麻烦，不推荐。
- 如果我们想要同时生成 key 和 csr，可以使用 `req -newkey` 选项，如下：

```
openssl req  -newkey rsa:4096 -nodes\
    -subj="/C=US/O=Security Lab/CN=RSA Intermediate CA"  \
    -reqexts ica_ext \
    -config ica.cnf \
    -keyout rsa_ica.key \
    -out rsa_ica.csr \
```


###### iii) 检查 CSR

```
openssl req -in rsa_ica.csr -text -noout
```

现在有了 `.csr` 文件，`root_ca` 就可以对其签发证书了。


### 切换到 `root_ca/` 目录进行操作

#### (1) 在 `root_ca` 中创建必要文件和目录

```
cd ../root_ca

mkdir -pv newcerts
touch index.txt
echo -ne "00" > serial
echo -ne "00" > crlnumber
```


#### (2) 准备配置信息

使用 `root_ca` 的私钥为 `int_ca` 签发证书之前，我们需要再检查一下 `root_ca` 的各项配置参数是否符合我们的要求：


`rootca.cnf`

```tcl
[ ca ]
default_ca              = CA_default

[ CA_default ]
dir                     = .
certs                   = $dir/certs
crl_dir                 = $dir/crl
database                = $dir/index.txt
new_certs_dir           = $dir/newcerts
private_key             = $dir/CA/private/ca.key.pem
certificate             = $dir/certs/ca.cert.pem
serial                  = $dir/serial
crlnumber               = $dir/crlnumber
default_md              = sha256
default_days            = 3650
default_crl_days        = 30

policy                  = policy_strict

copy_extensions         = copy

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
```

由于生成 CSR 时，已写入了 `v3_intermediate_ca` 字段中的信息。因此，我们在使用 `root_ca` 给其签发证书的时候，可以直接设置 `copy_extensions = copy`，它表示 `root_ca` 签署时会直接拷贝 CSR 中的扩展信息，不做任何改变。

关于 copy_extension 解释：

##### Possible Values for copy_extensions:

- **1) copy_extensions = copy:**
    - Copies all extensions from the CSR to the issued certificate, including critical and non-critical extensions.
    - This can be useful if you trust the extensions in the CSR, such as subjectAltName or other custom extensions.
- **2) copy_extensions = copyall** (only in OpenSSL 1.1.1 and newer):
    - Copies all CSR extensions, including any potentially unsupported or unrecognized ones.
    - This setting is more permissive than copy and may include extensions that OpenSSL does not explicitly support.
- **3) copy_extensions = none**:
    - Disables copying of extensions from the CSR to the issued certificate. Instead, the CA’s extensions in the `[ v3_ca ]`` or other specified extension section will be applied.
    - This is the default behavior in openssl ca if copy_extensions is not specified.


不过，如果我们需要为即将要签发的证书定制 v3 extension，我们可以把信息保存在文件中，然后用 `-extension` 选项来指定。

比如：`csr_ext.cnf`

```tcl
[ v3_intermediate_ca ]
basicConstraints = critical, CA:TRUE, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer:always
```

#### 总结：

**推荐使用 copy 的方式**，这样会好管理很多。否则，`root_ca` 改变了 v3 extension 的话，事后可能不好管理。


#### (3) 签发证书

`openssl ca` 和 `openssl x509` 都可以用来签发证书，区别在于前面的 ca 命令有数据库对所签发的证书进行记录（比如 serial, crlnumber 等），而后面的 x509 命令则适用于 ad-hoc 证书的签发，不管理数据库，不记录签发了哪些证书，所以只适合需要一次性证书（比如自签名证书）并且不负责下级证书管理的场景。


由于我们这里专注于证书链系统的构建，所以我们一律使用 `openssl ca` 命令来签发和管理证书。


#### (4) 签发证书

注意：签发证书是在 `root_ca` 目录中操作！

执行命令：

```
openssl ca -days 3600 -notext \
    -config ca.cnf \
    -cert CA/rsa_ca.crt \
    -keyfile CA/private/rsa_encrypted.key \
    -passin file:CA/private/password.txt \
    -in ../int_ca/rsa_ica.csr \
    -out rsa_int_ca.crt
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/policy_not_match_statename.png)

可惜，这里**命令失败**了。

**原因是：** CA 应用的 **policy** 是 **strict**，它要求 `Issuer` 和 `Subject` 的 StateorProvinceName 必须一样，否则命令会失败。

这里 policy 具体的要求是: 
- C, ST 和 O 必须相同
- OU 可选
- CN 必选。


因此，我们回到 `int_ca` 目录，按照 policy 要求去修改 CSR 的配置信息就可以了：

```
cd ../int_ca
openssl req -new -passin file:CA/private/password.txt \
    -config req_ext_csr_ica.cnf \
    -reqexts v3_intermediate_ca \
    -subj="/C=US/ST=TX/O=Security Lab/OU=Web3/CN=Web3 Intermediata CA " \
    -key CA/private/rsa_encrypted.key \
    -out rsa_ica.csr
```

检查 `rsa_ica.csr` 文件

```
openssl req -text -noout -in rsa_ica.csr
```

再次回到 `root_ca` 目录

```
cd ../root_ca
```

再次执行：

```
openssl ca -days 3600 -notext \
    -config ca.cnf \
    -cert CA/rsa_ca.crt \
    -keyfile CA/private/rsa_encrypted.key \
    -passin file:CA/private/password.txt \
    -in ../int_ca/rsa_ica.csr \
    -out rsa_int_ca.crt
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/root_ca_succeed.jpg)

OK,这次就成功了！

###### 补充：

如果想由 `root_ca` 指定 v3 extension 而不是 copy，那么命令为：

```
openssl ca -days 3600 -notext \
  -config <(cat rsa_ca.cnf csr_ext.cnf) \
  -extensions v3_intermediate_ca \
  -cert CA/rsa_ca.crt \
  -keyfile CA/private/rsa_encrypted.key \
  -passin file:CA/private/password.txt \
  -in ../int_ca/rsa_ica.csr \
  -out rsa_ica.crt
```

新证书会生成在 `newcerts/` 目录中，默认名字是 serial 编号。比如初始的 serial 是 00，那么签发的第一个证书就是 `newcerts/00.pem`，serial 文件中会变成 01。

`index.txt` 中也会增加一条记录，如下：

```
V    340907220747Z        00    unknown    /C=US/ST=TX/O=Security Lab/OU=Web3/CN=Web3 Intermediata CA
```

注意 crlnumber 文件中的值依然是 00，这是因为我们没有撤销过证书，


接下来，我们要检查证书

```
openssl x509 -text -noout -in newcerts/00.pem
```

没问题的话，证书就成功制作好了。


签发之后，我们需要把证书拷贝一份到 `int_ca` 的目录中去

```
cp newcerts/00.pem ../int_ca/CA/private/rsa_ica.crt
```

##### 把 `int_ca` 证书导入系统

最后，我们需要把该 CA 证书导入到操作系统的根证书目录中去。不过，如果你到现在为止仅是试验目的，可以暂时忽略下面的步骤。


导入证书的步骤如下：

##### Linux

把根证书复制到系统证书目录（Trusted Directory）

```
sudo cp newcerts/00.pem /usr/local/share/ca-certificates/rsa_ica.crt
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
选择 System，然后添加我们上面生成的 Root CA 证书（比如 `.crt`
添加完成后，我们需要信任该证书。right-click → Get Info → Alway Trust

###### (ii) 命令行导入
使用 security 命令

```
sudo security add-trusted-cert -d -r trustRoot -k \
    /Library/Keychains/System.keychain rsa_ica.crt
```

解释：
- **-d**: Adds the certificate as trusted by default for all users.
- **-r** trustRoot: Specifies that the certificate is a Root CA certificate.
- **-k** /Library/Keychains/System.keychain: Adds the certificate to the system keychain (accessible to all users).


Intermediate CA 应该成为我们最常使用的根证书，所有的用户证书都由它签发。



全文完！

