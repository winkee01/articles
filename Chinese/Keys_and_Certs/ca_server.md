#### 前言

前一篇我介绍了证书的过期与吊销相关的知识和操作步骤，这一篇我们继续探索 CA 证书服务器（CA Server）的搭建。

我们知道，如果证书太多了，时间一长，我们很难知道哪些证书过期，为了避免服务的中断，一个可行的策略就是**添加一个定时任务来监控证书的有效期，一旦发现证书过期，我们就自动续签**。

而要实现自动续签，我们需要 CA 以服务的形式随时待命，一旦 Web 服务发起续签请求，就能够自动签发证书，方便证书的自动更新。

当然，除了证书过期，CA 还需要自动或手动对证书进行吊销（**Revoke**），并及时更新证书吊销列表（**CRL**）


#### 证书重复

证书重复申请是我们首先需要解决的问题。当我们已经对一个 CSR 签发过证书后，如果再次对同一个 CSR 签发，openssl 会提示 “There is already a certificate for /CN=xxx"，拒绝签发。

因此，我们是不需要担心重复申请的。

但是，如果一个证书已经过期了，我们需要再使用这个 CSR 申请证书，这时候，我们依然会被拒绝。因为 CA 并不会自动进行吊销，所以，这时候，客户端需要有某种机制通知 CA 去撤销该证书，否则我们无法更新证书了。

具体的方法是传递一个 `force` 参数来强制撤销 CA 签发列表中的某个证书。

###### 那如何从 CA 的签发列表找到这个证书呢？

通过 `index.txt` 中的 "Subject Name" 中的 CommonName 来进行匹配。

index.txt 内容样式如下：
```
V   251030162019Z       00  unknown /C=US/ST=TX/O=echo service
R   341103225330Z   241105230017Z   01  unknown /C=US/ST=TX/O=home router service
V   341103230707Z       02  unknown /C=US/ST=TX/O=home router service
R   341104000252Z   241106001120Z   03  unknown /C=US/ST=TX/O=Router/OU=Network Security
R   341104001229Z   241106001933Z   04  unknown /C=US/ST=TX/O=Router/OU=Network Security/CN=myrouter
V   341104014351Z       05  unknown /C=US/ST=TX/O=Router/OU=Network Security/CN=myrouter
```

index.txt 中的每一行表示一个证书信息，它由六个字段构成：

- **Status** 证书状态，V：Valid, R: Revoked
- **Expiration Date**
- **Revocation Date (optional)** 如果没吊销过，这个字段是空的
- **Serial Number**
- **File Path (optional)** 通常是 unkonw
- **Subject Name**
  - countryName
  - stateOrProvinceName
  - localityName
  - organizationName
  - organizationalUnitName
  - commonName
  - emailAddress

###### 解释：
`index.txt` 文件中的 "Subject Name" 信息对应的是 openssl 配置文件中 `[ req ]` 段落中 `distinguished_name` 字段信息。

注意，创建 CSR 时 `[ req ]` 中的 `req_distinguished_name` 的信息并不一定会全部写入最终的证书中，哪些信息会出现在最终签发的证书中，取决于 `[ ca ]` 中的 `policy`，比如：

`ica.cnf`

```
[ ca ]
default_ca              = CA_default

[ CA_default ]
dir                     = .
...
policy                  = policy_match

[ policy_match ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
```

只有出现在 `[ policy_match ]` 中的字段才会出现在最终用户证书的 Subject 信息中。如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/subject_names.jpg)

根据 openssl 在面对重复申请证书时的报错提示，我们可以认为，如果已签发的证书中的 commonName 与 CSR 中的 "Subject Name" 中的 commonName 字段相同，就说明 CSR 在请求签发同一个证书。这时候，用户需要决定是否希望 CA 撤销原证书并签发新证书。

比如，

- 如果 `index.txt` 中没发现已签发过，直接签发。
- 如果找到同证书，但该证书尚未过期，就拒绝签发新证书。
- 如果已经过期，那么 CA 就先撤销原证书，然后再签发新证书。
- 如果传递了 force 参数，就强制撤销证书，再签发新证书。

按照上述的逻辑，下面我们来具体写代码实现。

采用 Python 的 Web 库 Flask 来实现会比较简单，使用 cryptography 库来解析证书字段。


如下：

```
from flask import Flask, request, jsonify
import subprocess
import os
from cryptography import x509
from cryptography.hazmat.backends import default_backend

app = Flask(__name__)

home_dir = os.path.expanduser("~")
INT_CA_DIR = os.path.join(home_dir, "Documents", "Dev", "allcerts", "allcerts", "int_ca")
INT_CA_PASSWORD_FILE = os.path.join(INT_CA_DIR, "CA", "private", "password.txt")
INT_CA_CONFIG = os.path.join(INT_CA_DIR, "ica.cnf")
INDEX_FILE = os.path.join(INT_CA_DIR, "index.txt")

def get_subject_from_csr(csr_path):
    """Extract the full subject name from the CSR using cryptography."""
    with open(csr_path, "rb") as csr_file:
        csr_data = csr_file.read()
    
    # Load the CSR using cryptography
    csr = x509.load_pem_x509_csr(csr_data, default_backend())
    
    # Return the full subject name
    return csr.subject


def get_cn_from_csr(csr_path):
    """Extract the CN from the CSR using the cryptography library."""
    with open(csr_path, "rb") as csr_file:
        csr_data = csr_file.read()
    
    csr = x509.load_pem_x509_csr(csr_data, default_backend())
    cn_attributes = csr.subject.get_attributes_for_oid(NameOID.COMMON_NAME)
    
    return cn_attributes[0].value if cn_attributes else None


def find_existing_cert_by_cn(cn):
    """Check the OpenSSL index.txt file to see if a valid certificate with this CN exists."""
    if not os.path.exists(INDEX_FILE):
        return None

    with open(INDEX_FILE, "r") as index_file:
        for line in index_file:
            # Strip any surrounding whitespace
            line = line.strip()
            
            # Locate the start of the Subject Name by finding "/C="
            subject_start_index = line.find("/C=")
            if subject_start_index == -1:
                continue  # Skip lines without a valid subject
            
            # Extract the Subject Name (from "/C=" to end of line)
            subject_name = line[subject_start_index:]
            
            # Extract the part before "/C=" for the other fields
            preceding_fields = line[:subject_start_index].strip()
            
            # Split the preceding fields into at most 4 parts: status, expiration_date, revocation_date, serial_number, and file_path
            parts = preceding_fields.split('\t', maxsplit=4)
            
            # Handle missing fields: ensure we have at least 3 fields (Status, Expiration Date, Serial Number)
            # status = parts[0] if len(parts) > 0 else None
            # expiration_date = parts[1] if len(parts) > 1 else None
            # revocation_date = parts[2] if len(parts) > 2 else None
            # serial_number = parts[3] if len(parts) > 3 else None

            status = parts[0] if len(parts) > 0 else None
            expiration_date = parts[1] if len(parts) > 1 else None
            revocation_date = parts[2] if len(parts) > 2 else None
            serial_number = parts[3] if len(parts) > 3 else None
            file_path = parts[4] if len(parts) > 4 else None
            
            # Only consider entries with status "V" (valid)
            if status != "V":
                continue
            
            # Check if the subject name contains the CN we are looking for
            if f"/CN={cn}" in subject_name:
                return {
                    "status": status,
                    "expiration_date": expiration_date,
                    "revocation_date": revocation_date or None,
                    "serial_number": serial_number,
                    "file_path": file_path,
                    "subject_name": subject_name
                }

    return None


def get_subject_from_index_entry(index_entry):
    """
    Extract the Common Name (CN) from the Subject Name in an index.txt entry
    using x509.Name().
    """
    # The subject name is typically the last field in the index.txt entry
    subject_name_str = index_entry["subject_name"]
    
    # Parse the subject name string into a list of x509.NameAttribute objects
    attributes = []
    for part in subject_name_str.split("/"):
        if not part:
            continue
        key, value = part.split("=", 1)
        oid = {
            "C": x509.NameOID.COUNTRY_NAME,
            "ST": x509.NameOID.STATE_OR_PROVINCE_NAME,
            "L": x509.NameOID.LOCALITY_NAME,
            "O": x509.NameOID.ORGANIZATION_NAME,
            "OU": x509.NameOID.ORGANIZATIONAL_UNIT_NAME,
            "CN": x509.NameOID.COMMON_NAME
        }.get(key)
        
        if oid:
            attributes.append(x509.NameAttribute(oid, value))

    # Create an x509.Name object from the list of NameAttribute objects
    subject = x509.Name(attributes)
    
    # Extract and return the Common Name (CN) from the subject, if present
    cn = subject.get_attributes_for_oid(x509.NameOID.COMMON_NAME)
    return cn[0].value if cn else None


def is_cert_expired(index_entry):
    """Check if the certificate is expired based on the OpenSSL index entry."""
    status = index_entry["subject_name"]  # V = Valid, R = Revoked, E = Expired
    expiry_date = index_entry["expiration_date"]  # Expiry date in YYMMDDHHMMSSZ format
    return status == "E"

def revoke_certificate1(serial_number):
    """Revoke the certificate using OpenSSL by its serial number."""
    cert_path = os.path.join(INT_CA_DIR, "newcerts", f"{serial_number}.pem")
    openssl_cmd = [
        "openssl", "ca", "-config", INT_CA_CONFIG, "-revoke", cert_path,
        "-passin", f"file:{INT_CA_PASSWORD_FILE}"
    ]
    subprocess.run(openssl_cmd, check=True)

def revoke_certificate(serial_number):
    """Revoke the certificate using OpenSSL by its serial number and update the CRL."""
    cert_path = os.path.join(INT_CA_DIR, "newcerts", f"{serial_number}.pem")
    
    # Command to revoke the certificate
    revoke_cmd = [
        "openssl", "ca", "-config", INT_CA_CONFIG, "-revoke", cert_path,
        "-passin", f"file:{INT_CA_PASSWORD_FILE}"
    ]
    
    # Run the revocation command
    subprocess.run(revoke_cmd, check=True)
    
    # Command to generate a new CRL
    crl_path = os.path.join(INT_CA_DIR, "crl", "crl.pem")  # Adjust path as necessary
    gen_crl_cmd = [
        "openssl", "ca", "-config", INT_CA_CONFIG,
        "-passin", f"file:{INT_CA_PASSWORD_FILE}",
        "-gencrl", "-out", crl_path
    ]
    
    # Run the command to update the CRL
    subprocess.run(gen_crl_cmd, check=True)


@app.route('/sign_csr', methods=['POST'])
def sign_csr():
    # Get the CSR from the request
    csr_data = request.files['csr'].read()
    
    # Get the target directory and certificate name from the request
    target_dir = request.form.get('target_dir')
    cert_name = request.form.get('cert_name')
    
    # Check for "force" keyword to force certificate revocation
    force_revoke = request.form.get('force', 'false').lower() == 'true'
    
    # Validate target directory and certificate name
    if not target_dir or not os.path.exists(target_dir):
        return jsonify({"error": "Invalid target directory"}), 400
    
    if not cert_name or not cert_name.endswith(".crt"):
        return jsonify({"error": "Please specify the file extension to be .crt"}), 400
    
    # Save the CSR temporarily
    csr_path = os.path.join(INT_CA_DIR, "tmp_user.csr")
    with open(csr_path, "wb") as f:
        f.write(csr_data)
    
    # Extract full subject name from the CSR using cryptography
    try:
        csr_subject = get_subject_from_csr(csr_path)
    except Exception as e:
        return jsonify({"error": f"Failed to extract subject name from CSR: {str(e)}"}), 400

    # Check if there's an existing certificate with the same CN
    existing_cert_entry = find_existing_cert_by_cn(csr_subject.get_attributes_for_oid(x509.NameOID.COMMON_NAME)[0].value)
    
    if existing_cert_entry:
        serial_number = existing_cert_entry["serial_number"]  # Serial number from the index.txt
        
        if force_revoke:
            # If force is requested, revoke the certificate regardless of its status
            try:
                revoke_certificate(serial_number)
            except Exception as e:
                return jsonify({"error": f"Failed to force revoke certificate: {str(e)}"}), 500
        elif is_cert_expired(existing_cert_entry):
            # Revoke the expired certificate
            try:
                revoke_certificate(serial_number)
            except Exception as e:
                return jsonify({"error": f"Failed to revoke expired certificate: {str(e)}"}), 500
        else:
            # If certificate exists and is not expired, return a message
            return jsonify({"message": f"There is already a certificate for /CN={csr_subject.get_attributes_for_oid(x509.NameOID.COMMON_NAME)[0].value}"}), 200
    
    # Define the full certificate path in the target directory
    cert_path = os.path.join(target_dir, cert_name)
    
    # Run OpenSSL command to sign the CSR
    openssl_cmd = [
        "openssl", "ca", "-batch", "-notext",
        "-config", INT_CA_CONFIG,
        "-passin", f"file:{INT_CA_PASSWORD_FILE}",
        "-in", csr_path,
        "-out", cert_path
    ]
    
    try:
        result = subprocess.run(openssl_cmd, capture_output=True, text=True)
        if result.returncode != 0:
            return jsonify({"error": result.stderr}), 400
        
        # Read the generated certificate
        with open(cert_path, "r") as cert_file:
            cert_data = cert_file.read()
        
        # Return the certificate content to the client
        return jsonify({"certificate": cert_data}), 200
    
    finally:
        # Clean up temp CSR
        if os.path.exists(csr_path):
            os.remove(csr_path)


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)


```

##### 解释：

CA Server 通过检查 `index.txt` 来找到与 CSR 有相同 CommonName 的行，从而确定 serial 并定位系统中的证书（`newcerts/` 目录中的 `${serial}.pem`)。如果是过期证书则忽略，如果是有效期内的证书。那么根据用户需求来决定是否强制撤销。

如果没有找到有相同 CommonName 的行，那就不存在重复证书，可以直接签发。


#### 客户端请求证书：

客户端向 CA Server 发出证书申请，需要传递三个参数：

- **csr**  如果没有申请过证书，我们需要先创建这个 csr 文件
- **target_dir**  签发的证书想要存放的目标目录
- **cert_name** 生成的目标证书名

请求命令如下：

```
curl -F "csr=@./rsa_router.csr" \
    -F "target_dir=/Users/tipman/allcerts/user_certs/router" \
    -F "cert_name=rsa_router.crt" \
    -F "force=true" \
    http://localhost:5001/sign_csr > /dev/null 2>&1
```

注：先不要添加 force 参数，如果请求被拒绝，说明有证书已签发过。检查后如果确定是需要强制重新签发，再添加 force 参数。



#### 自动续签

想要自动续签，我们第一步自然是需要判断证书是否过期。这个可以由客户端来进行，比如实际使用用户证书的 Web 服务器。

接下来，编写一个 Python 脚本，然后把它添加到定时任务里面就行了。

脚本如下：

`renew_user_certs`

```
import os
import requests
from cryptography import x509
from cryptography.hazmat.backends import default_backend
from datetime import datetime

def is_certificate_expired(cert_path):
    """Check if the certificate at cert_path is expired."""
    # Load the certificate
    with open(cert_path, "rb") as cert_file:
        cert_data = cert_file.read()
    cert = x509.load_pem_x509_certificate(cert_data, default_backend())
    
    # Get the expiration date from the certificate
    expiration_date = cert.not_valid_after
    
    # Compare with the current date
    return expiration_date < datetime.utcnow()

def renew_certificate_if_expired(cert_path, csr_path, target_dir, cert_name, server_url):
    """Check if the certificate is expired, and if so, request a new one using the requests library."""
    if is_certificate_expired(cert_path):
        print(f"The certificate at {cert_path} is expired. Requesting a new one...")

        # Prepare the data and files for the POST request
        files = {'csr': open(csr_path, 'rb')}
        data = {
            'target_dir': target_dir,
            'cert_name': cert_name,
            'force': 'true'
        }

        # Send the request to the CA server
        response = requests.post(server_url, files=files, data=data)

        # Check the response
        if response.status_code == 200:
            print(f"New certificate requested and will be saved at {target_dir}/{cert_name}.")
        else:
            print(f"Failed to request a new certificate: {response.status_code} - {response.text}")
    else:
        print("The certificate is still valid.")

# Usage
cert_path = "/Users/tipman/allcerts/user_certs/router/rsa_router.crt"
csr_path = "./rsa_router.csr"
target_dir = "/Users/tipman/allcerts/user_certs/router"
cert_name = "rsa_router.crt"
server_url = "http://localhost:5001/sign_csr"

renew_certificate_if_expired(cert_path, csr_path, target_dir, cert_name, server_url)
```

定时任务

```
0 3 * * * /opt/scripts/renew_user_certs.py >> /opt/scripts/renew_user_certs.log
```


全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad 。











