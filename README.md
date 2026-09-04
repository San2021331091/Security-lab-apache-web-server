# CSE-478 Introduction to Computer Security
## Lab 5: Securing Apache Web Server — Solution

### Objectives

- Set up a secure web server using Apache and digital certificates.
- Create a root Certificate Authority (CA).
- Generate and sign server certificates.
- Configure Apache to support HTTPS/TLS.

> **Source:** Lab 5 manual, pages 1–5. The manual states that HTTPS combines HTTP with SSL/TLS and uses certificates to verify the server and establish an encrypted communication channel. 

---

## Task 1: Becoming a Certificate Authority

### 1. Create a working directory

```bash
mkdir ~/ca
cd ~/ca
```

### 2. Copy the OpenSSL configuration file

```bash
cp /usr/lib/ssl/openssl.cnf .
```

The lab manual specifies using an OpenSSL `.cnf` configuration file for the `ca`, `req`, and `x509` commands.

### 3. Create the required files and directories

Check the `[CA_default]` section of `openssl.cnf` and create the required directories. A typical setup is:

```bash
mkdir certs crl newcerts
touch index.txt
echo 1000 > serial
```

### 4. Generate the root CA certificate

```bash
openssl req -new -x509 -keyout ca.key -out ca.crt -config openssl.cnf
```

Enter a password when prompted and provide the requested certificate information.

The resulting files are:

- `ca.key` — CA private key
- `ca.crt` — CA public-key certificate

The CA certificate is self-signed because this lab creates its own root CA.

---

# Task 2: Creating a Certificate for `example.com`

## Step 1: Generate the server private key

```bash
openssl genrsa -des3 -out server.key 1024
```

Enter and remember the password protecting the key.

## Step 2: Generate a Certificate Signing Request (CSR)

```bash
openssl req -new -key server.key -out server.csr -config openssl.cnf
```

When asked for the **Common Name**, enter:

```text
example.com
```

The generated CSR is:

```text
server.csr
```

## Step 3: Sign the CSR using your CA

```bash
openssl ca -in server.csr -out server.crt \
-cert ca.crt -keyfile ca.key -config openssl.cnf
```

Enter the CA password when requested.

The resulting certificate is:

```text
server.crt
```

---

# Start the HTTPS Test Server

## Step 1: Combine the key and certificate

```bash
cp server.key server.pem
cat server.crt >> server.pem
```

## Step 2: Start the OpenSSL HTTPS server

```bash
openssl s_server -cert server.pem -www
```

By default, the server listens on port `4433`.

Open the following address in the browser:

```text
https://example.com:4433/
```

Because the certificate was signed by your own CA, the browser may initially report that the certificate is not trusted.

---

# Trust the CA Certificate

Import:

```text
ca.crt
```

into the browser's trusted certificate store.

For Firefox, the lab manual gives the sequence:

```text
Preferences → Advanced → View Certificates
```

Import `ca.crt` and select:

```text
Trust this CA to identify web sites
```

Then visit:

```text
https://example.com:4433
```

You can also test:

```text
https://localhost:4433
```

---

# Checkpoint 1

Show your teacher:

```text
https://example.com:4433
```

### Explanation

The browser connects to the OpenSSL HTTPS server. The server provides its certificate. The browser verifies the certificate against the trusted CA. After the certificate is trusted, communication takes place over a TLS-secured channel.

---

# Creating a Certificate for `webserverlab.com`

Repeat the certificate-generation process for the second virtual host.

Generate the key:

```bash
openssl genrsa -des3 -out webserverlab.key 1024
```

Generate the CSR:

```bash
openssl req -new -key webserverlab.key \
-out webserverlab.csr -config openssl.cnf
```

For the **Common Name**, enter:

```text
webserverlab.com
```

Sign the CSR:

```bash
openssl ca -in webserverlab.csr \
-out webserverlab.crt \
-cert ca.crt -keyfile ca.key -config openssl.cnf
```

Combine the key and certificate:

```bash
cp webserverlab.key webserverlab.pem
cat webserverlab.crt >> webserverlab.pem
```

Start the test server:

```bash
openssl s_server -cert webserverlab.pem -www
```

Then test:

```text
https://webserverlab.com:4433/
```

## Checkpoint 2

Show `webserverlab.com` working through HTTPS to your teacher.

---

# Task 3: Deploy HTTPS into Apache

First stop the OpenSSL test server.

## 1. Edit the Apache virtual-host configuration

Open:

```bash
sudo nano /etc/apache2/sites-available/example.com.conf
```

Add the HTTPS virtual host:

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>

    ServerAdmin admin@example.com
    ServerName example.com
    ServerAlias www.example.com

    DocumentRoot /var/www/example.com/html

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    SSLEngine on
    SSLCertificateFile /path/to/server.crt
    SSLCertificateKeyFile /path/to/server.key

</VirtualHost>
</IfModule>
```

### Important

Replace:

```text
/path/to/server.crt
```

with the actual location of your certificate.

Replace:

```text
/path/to/server.key
```

with the actual location of your private key.

---

## 2. Enable the Apache SSL module

```bash
sudo a2enmod ssl
```

---

## 3. Test the Apache configuration

```bash
sudo apache2ctl configtest
```

Expected result:

```text
Syntax OK
```

If an error is displayed, fix the configuration before restarting Apache.

---

## 4. Restart Apache

```bash
sudo systemctl restart apache2
```

Then open:

```text
https://example.com
```

If the CA certificate is trusted by the browser, the webpage should load without the certificate warning.

---

# Checkpoint 3

Access:

```text
https://example.com
```

Show the HTTPS webpage to your teacher.

---

# Configure HTTPS for `webserverlab.com`

Create/edit the HTTPS virtual-host configuration for:

```text
webserverlab.com
```

Use the corresponding certificate and private key:

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>

    ServerAdmin admin@example.com
    ServerName webserverlab.com
    ServerAlias www.webserverlab.com

    DocumentRoot /var/www/webserverlab.com/html

    ErrorLog ${APACHE_LOG_DIR}/webserverlab-error.log
    CustomLog ${APACHE_LOG_DIR}/webserverlab-access.log combined

    SSLEngine on
    SSLCertificateFile /path/to/webserverlab.crt
    SSLCertificateKeyFile /path/to/webserverlab.key

</VirtualHost>
</IfModule>
```

Replace the certificate and key paths with their actual locations.

Test:

```bash
sudo apache2ctl configtest
```

If the result is:

```text
Syntax OK
```

restart Apache:

```bash
sudo systemctl restart apache2
```

Then access:

```text
https://webserverlab.com
```

---

# Checkpoint 4

Show your teacher that:

```text
https://webserverlab.com
```

successfully loads through HTTPS.

---

# Complete Command Summary

## CA

```bash
mkdir ~/ca
cd ~/ca

cp /usr/lib/ssl/openssl.cnf .

mkdir certs crl newcerts
touch index.txt
echo 1000 > serial

openssl req -new -x509 -keyout ca.key -out ca.crt -config openssl.cnf
```

## `example.com` certificate

```bash
openssl genrsa -des3 -out server.key 1024

openssl req -new -key server.key \
-out server.csr -config openssl.cnf

openssl ca -in server.csr -out server.crt \
-cert ca.crt -keyfile ca.key -config openssl.cnf
```

## Test HTTPS server

```bash
cp server.key server.pem
cat server.crt >> server.pem

openssl s_server -cert server.pem -www
```

Test:

```text
https://example.com:4433/
```

## Apache

```bash
sudo a2enmod ssl
sudo apache2ctl configtest
sudo systemctl restart apache2
```

Test:

```text
https://example.com
```

and:

```text
https://webserverlab.com
```

---

# Checkpoint Checklist

- [ ] Checkpoint 1 — `example.com:4433` works with the generated certificate.
- [ ] Checkpoint 2 — `webserverlab.com:4433` works with the generated certificate.
- [ ] Checkpoint 3 — Apache serves `example.com` over HTTPS.
- [ ] Checkpoint 4 — Apache serves `webserverlab.com` over HTTPS.

# Short Viva Explanation

**What is HTTPS?**

HTTPS is HTTP combined with SSL/TLS to provide a secure communication channel.

**What is a CA?**

A Certificate Authority is a trusted entity that issues digital certificates.

**What is `ca.key`?**

It is the CA's private key and must be kept secret.

**What is `ca.crt`?**

It is the CA's certificate.

**What is a CSR?**

A Certificate Signing Request contains information requesting a CA to issue a certificate for a public key.

**Why did the browser initially show a warning?**

Because the certificate was signed by the lab's own CA, which was not initially trusted by the browser.

**Why do we use `a2enmod ssl`?**

It enables Apache's SSL/TLS module so Apache can provide HTTPS.

**What does `SSLEngine on` do?**

It enables SSL/TLS for the Apache virtual host.
