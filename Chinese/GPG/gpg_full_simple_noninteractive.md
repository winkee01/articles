
Interactive 模式来创建 subkey 会比较繁琐，我们可以直接使用 `--quick-add-key` 选项来一键完成。

参考如下：

```
gpg --quick-add-key A75BAD6A38797462F05196722B2A5FCE4037C3B0 ed25519 sign 3y
gpg --quick-add-key D403BEF6B32C10B55047BC437EEA7E4DB6CFF09C cv25519 encrypt 3y
gpg --quick-add-key D403BEF6B32C10B55047BC437EEA7E4DB6CFF09C ed25519 auth 3y
```

注意：

encrypt 使用的是 cv25519 算法。