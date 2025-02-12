#### 前言
证书在出现被误用或过期的情况，为了安全起见，都需要 CA 对其进行吊销，并把吊销过的证书添加到吊销列表中，以便使用者能及时知晓。

#### 1. 证书吊销（Revoke）


```
openssl ca -config ica.cnf -passin file:CA/private/password.txt -revoke newcerts/01.pem
```

#### 2. 更新证书吊销列表（CRL）

吊销证书之后，OpenSSl 并不会自动更新吊销列表，一定要手动更新一下证书吊销列表，这样别人才能知道证书被吊销了。

```
openssl ca -config ica.cnf -passin file:CA/private/password.txt -gencrl -out crl/crl.pem
```

上面的命令是创建一个吊销列表文件（`crl.pem`），每次创建一个新的覆盖旧的，就相当于更新了。


#### 3. 查看证书吊销列表信息

```
openssl crl -in crl/crl.pem -noout -text
```


#### 4. 定期更新 CRL
由于 OpenSSl 并不自动创建 CRL，所以，一旦证书被吊销，我们不要忘记去手动生成新的 CRL。
为了防止遗忘，我们可以定期执行 `ca -gencrl` 命令，比如添加到 `crontab` 定时任务中去。


下面列一下跟 crl 相关的配置信息：

`ica.cnf` 或 `ca.cnf`

```
[ CA_default ]
crl_dir            = $dir/crl
crl                = $dir/crl.pem  # Path to the generated CRL
crlnumber          = $dir/crlnumber # Increment CRL number each time a new CRL is created
crl_extensions     = crl_ext       # CRL extensions to use
default_crl_days   = 30            # Number of days before the next CRL is generated

[ crl_ext ]
authorityKeyIdentifier = keyid:always
```


###### 证书自动更新
通常来说，CSR 只会在我们第一次申请时才需要创建，如果申请的证书以后到期了，我们并不需要再创建一个新的 CSR，因为除了 Issue Date 和 Experiation Date，其他信息并没有变化。所以，一旦某个我们申请过某个用户证书，我们只需要对下面两种情况进行管理：

- 证书被撤销
- 证书到期

只有这两种情况之一满足时，我们才需要重新申请证书，因此，我们可以在服务器上跑一个定时任务，每天定期检查证书是否到期即可，一旦到期，就重新提交申请。

比如：

`renew_user_certs.sh`

```
0 3 * * * /opt/scripts/renew_user_certs.sh >> /opt/scripts/renew_user_certs.log
```

当然，这个 crontab 每天运行的话可能略微有点浪费资源。我们也可以在到期那天，手动执行一下来更新证书。


全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。

