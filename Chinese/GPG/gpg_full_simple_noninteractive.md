
Interactive 模式来创建 subkey 会比较繁琐，我们可以直接使用 `--quick-add-key` 选项来一键完成。

参考如下：

```
gpg --quick-add-key A75BAD6A38797462F05196722B2A5FCE4037C3B0 ed25519 sign 3y
gpg --quick-add-key D403BEF6B32C10B55047BC437EEA7E4DB6CFF09C cv25519 encrypt 3y
gpg --quick-add-key D403BEF6B32C10B55047BC437EEA7E4DB6CFF09C ed25519 auth 3y
```

注意：

encrypt 使用的是 cv25519 算法。


另外，生成 master key 时，可以用下列方式来指定参数：

```
gpg --batch --generate-key <<EOF
%echo Generating a basic OpenPGP key for HELM Secret
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: Michael
Name-Comment: Personal PGP
Name-Email: mpan@sample.com
Expire-Date: 0
%no-ask-passphrase
%no-protection
%commit
%echo done
EOF
```