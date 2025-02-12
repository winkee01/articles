## 简洁版：使用 GPG Key 三级证书体系（RSA）


### 三级证书体系结构

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/fulllchain_ca_system_vertical.png)


### 目录结构
证书虽然在概念上有层级，但是为了方便实际的管理，每级证书使用有独立的目录。

换句话说，虽然各证书在概念上是有层级的，但是物理目录上应该使用平级的结构，而不是层级或树状的结构。

```
mkdir allcerts
cd allcerts
mkdir -p {CSTrust_RSA_CA, CSTrust_RSA_ICA, CSTrust_RSA_Users}
```

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/allcerts_directory_structure.png)

### 1. 创建 Root CA（自签名证书）


#### 1.1 创建目录及基础文件

目录结构：
![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/CSTrust_RSA_CA_directory_v0.png)

按照上图创建目录与文件

```
cd CSTrust_RSA_CA
mkdir -p CA/private newcerts crl 
touch index.txt
echo -ne "00" > serial
echo -ne '00' > crlnumber
touch rootca.cnf
touch req_rootca.cnf
```

#### 1.2 创建私钥

##### (1) 创建密码文件

```
openssl rand -base64 32 > CA/private/password.txt
chmod 600 CA/private/password.txt
```

##### (2) 创建私钥（RSA）

```
openssl genpkey -algorithm RSA \
    -aes256 -pkeyopt rsa_keygen_bits:4096 \
    -pass file:CA/private/password.txt \
    -out CA/private/rsa_encrypted.key
```

##### (3) 检查

```
openssl asn1parse -in CA/private/rsa_encrypted.key
```

##### (4) 去除密码保护
如果需要去除密码保护，可以使用如下命令：

```
openssl pkcs8 -in CA/private/rsa_encrypted.key \
    -out CA/private/rsa_decrypted.key \
    -passin file:CA/private/password.txt
```

#### 1.3 创建证书（自签名）

使用 `openssl req -x509` 命令，所以我们需要填写配置信息中的 `[ req ]` 段落。

##### (1) 配置文件

在 `req_rootca.cnf` 中填入如下内容：

```
[ req ]
default_keyfile     = rsa_private_decrypted.key     # Default private key filename
prompt              = no                            # Disable prompt
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca                         # Extensions for self-signed root CA certificate

[ req_distinguished_name ]
C  = US
ST = TX
O  = Sec Homelab
OU = IT Network
CN = CSTrust Root CA (RSA)

[ v3_ca ]
basicConstraints = critical, CA:true
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
```

###### (2) 自签名创建

```
openssl req -new -x509 -days 7300 -sha256 \
    -key CA/private/rsa_encrypted.key \
    -config req_rootca.cnf \
    -passin file:CA/private/password.txt \
    -out CA/rsa_rootca.pem
```

###### (3) 目录结构

创建完成后，目录内容如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/CSTrust_RSA_CA_directory_v2.png)


###### (4) 检查

```
openssl x509 -noout -text -in CA/rsa_rootca.pem
```

#### 1.5 根证书导入操作系统
我们需要把 Root CA 证书导入操作系统，被系统信任。

##### (1) Linux

把根证书复制到系统证书目录（Trusted Directory），然后更新系统记录。

```
sudo cp CA/rsa_rootca.pem /usr/local/share/ca-certificates/CSTrust_RSA_CA.crt
sudo update-ca-certificates
```
解释：
`update-ca-certificates` 命令会把扫描 `/usr/local/share/ca-certificates/` 目录，并把其中所有根证书都安装到 `/etc/ssl/certs/` 目录中。

##### (2) MacOS
###### (i) GUI 导入

打开 Keychain 应用：File → Import Items.
选择 System，然后添加我们上面生成的 Root CA 证书（`rsa_rootca.pem`)
添加完成后，我们需要信任该证书。right-click → Get Info → Alway Trust

###### (ii) 命令行导入
使用 security 命令

```
sudo security add-trusted-cert -d -r trustRoot -k \
    /Library/Keychains/System.keychain rsa_rootca.pem
```

解释：
- **-d**: default，表示默认被系统中的所有用户信任。
- **-r** trustRoot，表示要添加的证书为根证书
- **-k** keychain，表示添加到 system keychain（可以被所有用户访问）

##### (3) Windows


### 2. 创建 Intermediate CA

Intermediate CA 证书也属于 CA，可以向下签发用户证书。下面我们简称 ICA。

制作 ICA 之前，我们先准备目录和相关配置文件。
- ICA 证书由 Root CA 所签发，所以需要创建 CSR 请求文件。
- ICA 作为 User Certs 的签发机构，需要证书管理文件（serial, index.txt 和 crlnubmer)


#### 2.1 创建目录及基础文件

目录结构：



###### 按照上述结构来创建相关目录和文件

```
cd CSTrust_RSA_ICA
mkdir -pv CA/private newcerts crl

echo -ne "00" > serial
echo -ne '00' > crlnumber
touch req_ica.cnf
```

##### 步骤简介：
下面是我们创建中间证书的大致步骤：

- （a） 创建 ICA 的 Private Key
- （b） 创建 ICA 的 CSR
- （c） 使用 rootca 对 CSR 签名，从而创建出 `rsa_ica.crt`

#### 2.2 创建私钥

###### (1) 创建密码文件

```
openssl rand -base64 32 > CA/private/password.txt
chmod 600 CA/private/password.txt
```

###### (2) 创建私钥（RSA）

```
openssl genpkey -algorithm RSA \
    -aes256 -pkeyopt rsa_keygen_bits:4096 \
    -pass file:CA/private/password.txt \
    -out CA/private/rsa_encrypted.key
```

如果想增加 KDF 迭代次数来增加密钥强度，命令如下：

```
openssl pkcs8 -topk8 -v2 aes256 -iter 100000 \
  -passout file:CA/private/password.txt \
  -in CA/private/rsa_encrypted.key \
  -out CA/private/rsa_encrypted_enhanced.key
```

###### (3) 检查私钥

```
openssl asn1parse -in CA/private/rsa_encrypted.key
```

###### (4) 去除密码保护
如果需要去除密码保护，可以使用如下命令：

```
openssl pkcs8 -in CA/private/rsa_encrypted.key \
    -out CA/private/rsa_decrypted.key \
    -passin file:CA/private/password.txt
```

#### 2.3 创建 CSR 请求

##### (1) CSR 配置文件

ICA 需要向 Root CA 发起 CSR 请求，使用 `openssl x509 -req` 命令，因此，我们需要填写配置文件中的 `[ req ]` 段落的信息。

向 `req_ica.cnf` 中填入如下内容：

```
[ req ]
default_bits       = 4096
default_keyfile    = rsa_encrypted.key
distinguished_name = req_distinguished_name
x509_extensions    = v3_ica
prompt             = no

[ req_distinguished_name ]
C  = US
ST = CA
O  = ICA-Security-Lab
OU = Internete-Security
CN = CSLab ICA   # Common Name

[ v3_ica ]
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

##### (2) 创建 CSR 对象 （`rsa_ica.csr`)

```
openssl req -new \
    -passin file:CA/private/password.txt \
    -config req_ica.cnf \
    -reqexts v3_ica \
    -key CA/private/rsa_encrypted.key \
    -out rsa_ica.csr
```

也可以用命令行来指定 DN (Distinguished Name) 信息：

```
openssl req -new -passin file:CA/private/password.txt \
    -config req_ica.cnf \
    -reqexts v3_ica \
    -subj="/C=US/ST=CA/O=ICA-Security-Lab/OU=Internete-Security/CN=CSLab ICA" \
    -key CA/private/rsa_encrypted.key \
    -out rsa_ica.csr
```


**注意：**
- 只有添加了 `-reqexts` 选项，CSR 中才会写入 v3 extension 信息。

###### （3) 检查 CSR 对象

```
openssl req -in rsa_ica.csr -text -noout
```

#### 2.4 签发 ICA 证书

使用 Root CA 对 `rsa_ica.csr` 签名，即可生成 ICA 证书。

##### (1) `rootca.cnf` 配置信息

使用 `openssl ca` 命令，因此，我们需要填写 `[ ca ]` 字段中的配置信息。

回到 `CSTrust_RSA_CA/` 目录，在 `rootca.cnf` 中填入如下信息：

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

policy                  = policy_ica

copy_extensions         = copy

[ policy_ica ]
countryName             = match         # 必须与 Issuer 相同
stateOrProvinceName     = supplied      # 不必与 Issuer 相同，但必须提供
organizationName        = supplied      # 同上
organizationalUnitName  = optional      # 可有可无
commonName              = supplied      # 不必与 Issuer 相同，但必须提供
```
>
###### 解释：

##### (i) copy_extensions
由于生成 CSR 时，已写入了 `v3_ica` 字段中的信息。因此，我们在使用 `rootca` 给其签发证书的时候，可以直接设置 `copy_extensions = copy`，它表示 `rootca` 签署时会直接拷贝 CSR 中的扩展信息，不做任何改变。

copy_extensions 的可选值是：
- copy
- copyall
- none      （默认值，不 copy）

不过，如果 Root CA 想要为签发的证书定制 v3 extension，我们可以把信息保存在一个单独文件中，然后用 `-extension` 选项来指定。

比如我们在 Root CA 的目录中保存为 `CSTrust_RSA_CA/ICA1/csr_ext.cnf`，里面填入如下内容：

```tcl
[ v3_ica ]
basicConstraints = critical, CA:TRUE, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer:always
```

**推荐使用 copy 的方式**，这样管理起来方便很多。否则，`rootca` 改变了 v3 extension 的话，事后可能不好管理。

##### (ii) policy_ica

我们根据需要来调整对 ICA 的 DN(Distinguished Name) 的要求，比如要求国家名称必须相同，州名（或省名）必须提供之类的。如果 CSR 中的 DN 信息与 CA 所要求的 Policy 不匹配，那么在执行 `openssl ca` 命令签名时就会失败。


#### (2) 为 ICA 签发证书

`openssl ca` 和 `openssl x509` 两个命令都可以用来签发证书，区别在于前面的 ca 命令会使用 ROOT CA 的数据库对所签发的证书进行记录（比如 serial, crlnumber 等）；而后面的 x509 命令则适用于 ad-hoc 证书的签发，不管理数据库，不记录签发了哪些证书，所以**只适合需要一次性证书**（比如**自签名证书**）并且不负责下级证书管理的场景。

由于我们这里专注于证书链系统的构建，所以我们一律使用 `openssl ca` 命令来签发和管理证书。


签发证书需要回到 Root CA 目录来执行操作：

```
cd CSTrust_RSA_CA/
```

执行命令：

```
openssl ca -days 3600 -notext \
    -config rootca.cnf \
    -cert CA/rsa_rootca.crt \
    -keyfile CA/private/rsa_encrypted.key \
    -passin file:CA/private/password.txt \
    -in ../CSTrust_RSA_ICA/rsa_ica.csr \
    -out ../CSTrust_RSA_ICA/CA/rsa_ica.crt
```

#### (3) 检查证书

##### (i) 检查 Root CA 目录
Root CA 签发下级证书之后，会在其目录中更新相关文件。

- **`newcerts/`** 新证书会生成在 `newcerts/` 目录中，默认名字是 serial 编号。比如初始的 serial 是 00，那么签发的第一个证书就是 `newcerts/00.pem`，serial 文件中内容会由原来的 00 变成 01。

- **`index.txt`** 中也会增加一条记录，如下：

  ```
  V    340907220747Z        00    unknown    /C=US/ST=CA/O=Security Lab/OU=Web3/CN=Web3 Intermediata CA
  ```
- **`crlnumber`**  由于还没有撤销过证书，crlnumber 文件中的值依然是 00。

##### (i) 检查 ICA 证书

接下来，我们要检查证书

```
openssl x509 -text -noout -in newcerts/00.pem
```

或者

```
openssl x509 -text -noout -in ../CSTrust_RSA_CA/ICA/rsa_ica.crt
```


##### 把 `ICA` 证书导入系统
Intermediate CA 是我们最常使用的根证书，因为所有的用户证书都由它签发。
为了方便使用，建议把该 ICA 证书导入到操作系统的根证书目录中去。

