#### Generate a GPG key Non-interactively (Batch Mode)

##### 1. Create a Batch file

`gpg-key-batch.txt`

```
%echo Generating a GPG key
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: Your Name
Name-Email: your-email@example.com
Expire-Date: 2y
Passphrase: my-secure-passphrase
%commit
%echo Done
```

解释：
- Key-Type: RSA → Generates an RSA key.
- Key-Length: 4096 → Uses a strong 4096-bit key.
- Subkey-Type: RSA → Creates a subkey for encryption.
- Subkey-Length: 4096 → Sets the encryption subkey length.
- Name-Real: Your Name → Specifies the name for the key.
- Name-Email: your-email@example.com → Binds the key to your email.
- Expire-Date: 2y → The key expires in 2 years (use 0 for no expiry).
- Passphrase: my-secure-passphrase → Sets a passphrase for the key.
- %commit → Finalizes key creation.


##### 2. Generate the Key Using the Batch File

```
gpg --batch --generate-key gpg-key-batch.txt
```

##### 3. Verify the key

```
gpg --list-keys
```

It should display:

```
pub   rsa4096 2024-02-14 [SC]
      ABCD1234EF567890GHIJKL1234567890ABCDEF12
uid           [ultimate] Your Name <your-email@example.com>
sub   rsa4096 2024-02-14 [E]
```


##### 4. (Optional) Export the Public Key

```
gpg --armor --export your-email@example.com > my-public-key.asc
```



##### 5. (Optional) Set the Key for Git Commit Signing

```
git config --global user.signingkey $(gpg --list-keys --keyid-format=long | grep 'pub' | awk '{print $2}' | cut -d'/' -f2)
git config --global commit.gpgsign true
```



### Generate an Ed25519 key

`run.sh`

```
GNUPGHOME="/tmp/tmp.gpgtests.gpgagent"
GPG_TTY=$(tty)
export GNUPGHOME
export GPG_TTY

if [ ! -d "$GNUPGHOME" ]; then
    mkdir -p $GNUPGHOME
    chmod 700 $GNUPGHOME
    gpg --quiet --batch --passphrase '' --default-new-key-algo "ed25519/cert,auth,sign+cv25519/encr" --quick-generate-key "John Doe <john@example.com>"
    gpg --export --armor > gpg.key
fi

# It's important to close gpg agents after `gpg` command, otherwise some of the
# instances might cause race conditions for the tests below
killall gpg-agent || true

...
```


```
# If you want to deviate from default algorithms,
# make sure to do above export of MY_GPG_ALG in the same session
# as the one you execute below code:
(
    echo "" && \
    read -r -s -p 'Passphrase to set for private key: ' PASSPHRASE && \
    echo "" && \
    read -r -s -p 'Please, repeat the passphrase: ' PASSPHRASE_REPEAT && \
    [ "${PASSPHRASE}" != "${PASSPHRASE_REPEAT}" ] && \
    echo -e "\nPassphrases don't match! Aborting...\n" || (
        echo -e "\n" && \
        read -r -p 'Name and e-mail (e.g. "Code Caden <codecaden24@gmail.com>"): ' CONTACT && \
        echo "" && \
        read -r -p 'When do you want your key to expire?
I recommend January 1st of either the next year or the year after, e.g. "2024-01-01".
You can always extend the validity or create new subkeys later on! ' DATE && \
        MY_GPG_HOMEDIR="$( umask 0077 && mktemp -d )" && \
        echo "cert-digest-algo SHA512
default-preference-list AES256 TWOFISH CAMELLIA256 AES192 CAMELLIA192 AES CAMELLIA128 SHA512 SHA384 SHA256 BZIP2 ZLIB ZIP Uncompressed
" > "${MY_GPG_HOMEDIR}/gpg.conf" && \
        echo "${PASSPHRASE}" | gpg --homedir "${MY_GPG_HOMEDIR}" --batch --pinentry-mode loopback --quiet --passphrase-fd 0 \
            --quick-generate-key "${CONTACT}" "${MY_GPG_ALG[0]:-ed25519}" cert "${DATE}" && \
        FINGERPRINT=$(gpg --homedir "${MY_GPG_HOMEDIR}" --list-options show-only-fpr-mbox --list-secret-keys 2>/dev/null | awk '{print $1}') && \
        echo "${PASSPHRASE}" | gpg --homedir "${MY_GPG_HOMEDIR}" --batch --pinentry-mode loopback --quiet --passphrase-fd 0 \
            --quick-add-key "${FINGERPRINT}" "${MY_GPG_ALG[1]:-rsa3072}" sign "${DATE}" && \
        echo "${PASSPHRASE}" | gpg --homedir "${MY_GPG_HOMEDIR}" --batch --pinentry-mode loopback --quiet --passphrase-fd 0 \
            --quick-add-key "${FINGERPRINT}" "${MY_GPG_ALG[2]:-rsa3072}" encrypt "${DATE}" && \
        echo "${PASSPHRASE}" | gpg --homedir "${MY_GPG_HOMEDIR}" --batch --pinentry-mode loopback --quiet --passphrase-fd 0 \
            --quick-add-key "${FINGERPRINT}" "${MY_GPG_ALG[3]:-rsa3072}" auth "${DATE}" && \
        echo -e '\nSuccess! You can find the GnuPG homedir containing your keypair at \e[0;1;97;104m"'"${MY_GPG_HOMEDIR}"'".\e[0m\nPlease, \e[1;32mbackup that directory\e[0m somewhere safe!\n\nExport/import of Curve448 keys is currently unsupported:\n - https://dev.gnupg.org/rGa07ae85ec795e338af1bcbe288a3af4f21bb94ce\n - https://dev.gnupg.org/rG0d74c3c89663ee9b163742c6c75641c1b6b28f09\n' && \
        gpgconf --homedir "${MY_GPG_HOMEDIR}" --kill all && \
        rm -f "${MY_GPG_HOMEDIR}/gpg.conf"
    )
)
```

参考来源：

https://github.com/duxsco/gpg-smartcard#create-a-gnupg-keypair
https://github.com/Ciantic/gpg-ed25519-to-cv25519/blob/master/example-pkdecrypt-pksign/run.sh



