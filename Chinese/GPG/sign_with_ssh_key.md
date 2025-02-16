Signing Commits with SSH
To sign commits using SSH, we need to make some adjustments to the Git config:

```
git config --global gpg.format ssh
```

This command configures Git to use SSH for signing. Next, we need to specify the public SSH key to use:

```
git config --global user.signingkey ~/.ssh/id_ed25519.pub
```

Finally, to sign commits, you have two options. You can either set Git to sign all commits by default (globally in any local repository with the --global flag) using the following command:

```
git config --global commit.gpgsign true
```

Or, if you prefer, you can sign individual commits by adding the -S flag when making a commit:

```
git commit -S -m "YOUR_COMMIT_MESSAGE"
# Creates a signed commit
```

Finally, we need to create an "allowed signers" file, which is a list of trusted keys that Git can use. You can place this file anywhere you prefer, but we suggest storing it under the `~/.config/git/` folder.


To inform Git about this file in the global Git config, use the following command if you chose the same path:

```
git config --global gpg.ssh.allowedSignersFile "~/.config/git/allowed_signers"
```

Now, let's create the "allowed signers" file itself. We need to add the email and public key to this document, following this format:

```
bruno@git-tower.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFZSV8LQpdNrwUrR4jB8eHnuH6ZKuqJwjdmis1UUBXG1
```


You can add multiple users if you wish, with each user's email and public key on a separate line.

Once you have added a signed commit and run `git log --show-signature`, you should be able to view the signature status for the commits, confirming that everything has worked successfully. We're almost there!



https://www.git-tower.com/blog/setting-up-ssh-for-commit-signing/



