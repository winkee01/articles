## Title: How to manage your passphrase in keychain on MacOS

Managing your private key's passphrase via the macOS Keychain provides a secure and convenient method for storing and retrieving your password. The Keychain securely stores sensitive information and allows you to access it programmatically, which is especially useful when working with scripts or command-line tools like OpenSSL.

Below are the steps to store your passphrase in the Keychain and retrieve it when needed.

---

## **1. Storing the Passphrase in macOS Keychain**

You can store your passphrase in the Keychain using either the Keychain Access application (GUI) or the command-line `security` tool.

### **Option A: Using Keychain Access Application**

1. **Open Keychain Access**:
   - Navigate to `Applications` > `Utilities` > `Keychain Access`, or use Spotlight (`Command + Space`) and type "Keychain Access".

2. **Add a New Password Item**:
   - Click on `File` > `New Password Item...` (or press `Command + N`).

3. **Fill in the Details**:
   - **Keychain Item Name**: Provide a descriptive name (e.g., `MyPrivateKeyPassphrase`).
   - **Account Name**: Optionally, add an account name (e.g., your username or email).
   - **Password**: Enter your passphrase.
   - **Where**: Optionally, specify where the item will be used.

4. **Save the Item**:
   - Click `Add` to store the passphrase in the Keychain.

### **Option B: Using the `security` Command-Line Tool**

You can store the passphrase directly from the Terminal:

```bash
security add-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase" -w "your_passphrase_here"
```

- **`-a "${USER}"`**: Sets the account name to your current username.
- **`-s "MyPrivateKeyPassphrase"`**: Sets the service name (used as the item's name).
- **`-w "your_passphrase_here"`**: The passphrase you want to store.

**Note**: Replace `"your_passphrase_here"` with your actual passphrase. Be cautious when entering passwords directly in the command line, as they may be stored in shell history. Alternatively, omit the `-w` option, and the command will prompt you to enter the password securely.

---

## **2. Retrieving the Passphrase from Keychain**

You can retrieve the passphrase programmatically using the `security` command.

```bash
security find-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase" -w
```

- **`-w`**: Outputs the password to standard output.

This command will print the passphrase stored under the service name `MyPrivateKeyPassphrase`.

---

## **3. Using the Passphrase in OpenSSL Commands**

To integrate the passphrase retrieval into your OpenSSL commands, you can use command substitution in your scripts.

### **Example: Decrypting the Private Key**

Suppose you have an encrypted private key in PKCS#8 format and you want to decrypt it using OpenSSL.

```bash
openssl pkcs8 -in encrypted_private.key -out decrypted_private.key -nocrypt \
  -passin "pass:$(security find-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase" -w)"
```

- **`-passin`**: Specifies the passphrase source for input.
- **`"pass:$(...)"`**: Uses command substitution to insert the passphrase retrieved from the Keychain.

### **Example: Using the Encrypted Private Key Directly**

When using the encrypted private key for operations like generating a CSR or signing certificates, you can supply the passphrase similarly.

**Generating a CSR:**

```bash
openssl req -new -key encrypted_private.key -out server.csr \
  -passin "pass:$(security find-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase" -w)" \
  -subj "/C=US/ST=State/L=City/O=Organization/OU=Department/CN=example.com"
```

**Signing a Certificate:**

```bash
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey encrypted_private.key \
  -CAcreateserial -out server.crt -days 365 -sha256 \
  -passin "pass:$(security find-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase" -w)"
```

### **Important Notes:**

- **Quoting**: Ensure that you enclose the command substitution in double quotes to handle any special characters in the passphrase.
- **Security**: Be cautious with scripts that include passwords. Avoid exposing the passphrase in logs or error messages.

---

## **4. Automating Passphrase Retrieval in Scripts**

For convenience, you can create a shell function or alias to simplify the usage.

**Example Shell Function:**

```bash
get_keychain_passphrase() {
  security find-generic-password -a "${USER}" -s "$1" -w
}
```

Use it in your commands:

```bash
PASS="$(get_keychain_passphrase "MyPrivateKeyPassphrase")"
openssl pkcs8 -in encrypted_private.key -out decrypted_private.key -nocrypt -passin "pass:${PASS}"
```

**Alternatively, directly in the command:**

```bash
openssl pkcs8 -in encrypted_private.key -out decrypted_private.key -nocrypt \
  -passin "pass:$(get_keychain_passphrase "MyPrivateKeyPassphrase")"
```

---

## **5. Security Considerations**

- **Access Control**: Ensure your user account is secure, as the Keychain items are accessible to you when logged in.
- **Keychain Permissions**: You can adjust the access control settings of the Keychain item to restrict which applications can access it.
  - **In Keychain Access**:
    - Right-click the item and select `Get Info`.
    - Go to the `Access Control` tab.
    - Specify applications that are allowed to access the item, or allow all applications.
- **Scripting Security**: Be mindful when scripting password retrieval to avoid exposing the passphrase.

---

## **6. Managing Keychain Items**

### **Viewing Stored Items**

List all generic passwords stored for your user:

```bash
security find-generic-password -a "${USER}" -g
```

### **Deleting a Keychain Item**

To remove the passphrase from the Keychain:

```bash
security delete-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase"
```

---

## **7. Example Workflow**

**Step 1: Store Passphrase in Keychain**

```bash
security add-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase" -w
```

- You'll be prompted to enter the passphrase securely.

**Step 2: Encrypt the Private Key**

```bash
openssl pkcs8 -topk8 -v2 aes256 -v2prf hmacWithSHA256 -iter 100000 \
  -in private.key -out encrypted_private.key \
  -passout "pass:$(security find-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase" -w)"
```

**Step 3: Use the Encrypted Key**

Generate a self-signed certificate:

```bash
openssl req -x509 -key encrypted_private.key -days 730 \
  -out certificate.crt \
  -passin "pass:$(security find-generic-password -a "${USER}" -s "MyPrivateKeyPassphrase" -w)" \
  -subj "/C=US/ST=State/L=City/O=Organization/OU=Department/CN=example.com"
```

---

## **8. Alternative: Using Secure Notes (Not Recommended for Automation)**

While Keychain Secure Notes can store sensitive information, they are not accessible via the command line for automation purposes. Therefore, for scripting and programmatic access, using generic passwords as shown above is the preferred method.

---

## **Conclusion**

By leveraging the macOS Keychain, you can securely store your private key's passphrase and retrieve it when needed in a safe and convenient manner. This approach enhances security by keeping the passphrase encrypted within the Keychain while allowing you to automate tasks that require the passphrase.

---

**If you have any further questions or need assistance with specific steps, feel free to ask!**