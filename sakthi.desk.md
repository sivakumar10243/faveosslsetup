# 🔐 SSL Setup for `sakthi.desk` (Root CA + Server Certificate)

> ✔ Fixes PHP SSL errors
> ✔ Fixes Mail (SMTP/TLS) errors
> ✔ No `openssl.cafile` hacks
> ✔ Works with Apache + PHP-FPM

---

## 📁 Directory

```bash
cd /etc/apache2/ssl
```

---

## 🟢 Step 1: Create a **Proper Root CA** (CA:TRUE)

```bash
openssl ecparam -name prime256v1 -genkey -out sakthi-rootCA.key

openssl req -x509 -new -nodes -sha256 -days 3650 \
-key sakthi-rootCA.key \
-out sakthi-rootCA.crt \
-subj "/C=IN/ST=TN/O=SAKTHI/CN=SAKTHI-ROOT-CA" \
-addext "basicConstraints=critical,CA:TRUE" \
-addext "keyUsage=critical,keyCertSign,cRLSign"
```

### ✅ Verify Root CA

```bash
openssl x509 -in sakthi-rootCA.crt -noout -text | grep -A3 "Basic Constraints"
```

You **must see**:

```
CA:TRUE
```

---

## 🟢 Step 2: Create Server Private Key

```bash
openssl ecparam -name prime256v1 -genkey -out sakthi.desk.key
```

---

## 🟢 Step 3: Create Server CSR

```bash
openssl req -new -key sakthi.desk.key -out sakthi.desk.csr \
-subj "/C=IN/ST=TN/O=SAKTHI/CN=sakthi.desk"
```

---

## 🟢 Step 4: Create SAN Configuration (CRITICAL)

```bash
cat > san.cnf <<EOF
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = sakthi.desk
DNS.2 = www.sakthi.desk
IP.1  = 127.0.0.1
EOF
```

---

## 🟢 Step 5: Sign Server Certificate with Root CA

```bash
openssl x509 -req -in sakthi.desk.csr \
-CA sakthi-rootCA.crt -CAkey sakthi-rootCA.key -CAcreateserial \
-out sakthi.desk.crt -days 3650 -sha256 \
-extfile san.cnf
```

---

## 🟢 Step 6: Install Certificates

```bash
cp sakthi.desk.crt /etc/ssl/certs/
cp sakthi.desk.key /etc/ssl/private/
cp sakthi-rootCA.crt /usr/local/share/ca-certificates/sakthi-rootCA.crt
```

---

## 🟢 Step 7: Trust the Root CA (SYSTEM-WIDE)

```bash
update-ca-certificates --fresh
```

You **must see** something like:

```
Adding debian:sakthi-rootCA.pem
```

---

## 🟢 Step 8: Apache SSL Configuration

Edit your SSL virtual host:

```bash
nano /etc/apache2/sites-available/sakthi.desk-ssl.conf
```

```apache
SSLCertificateFile /etc/ssl/certs/sakthi.desk.crt
SSLCertificateKeyFile /etc/ssl/private/sakthi.desk.key
```

Enable SSL & site (if not already):

```bash
a2enmod ssl
a2ensite sakthi.desk-ssl
```

Restart services:

```bash
systemctl restart apache2
systemctl restart php8.2-fpm
```

---

## 🟢 Step 9: PHP Configuration (IMPORTANT)

❌ **DO NOT set** this:

```ini
openssl.cafile=/usr/local/share/ca-certificates/sakthi-rootCA.crt
```

✅ Leave it unset so PHP uses system trust:

```ini
; openssl.cafile =
```

---

## ✅ Final Validation

### 🔍 Check SAN

```bash
openssl x509 -in /etc/ssl/certs/sakthi.desk.crt -noout -text | grep -A2 "Subject Alternative Name"
```

Must show:

```
DNS:sakthi.desk
DNS:www.sakthi.desk
```

---

### 🔍 Verify Certificate Chain

```bash
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/sakthi.desk.crt
```

Expected:

```
OK
```

---

### 🔍 Test via PHP

```bash
php -r "echo file_get_contents('https://sakthi.desk');"
```

✔ No SSL error = SUCCESS

---

## ✅ RESULT

| Component        | Status    |
| ---------------- | --------- |
| HTTPS site       | ✅ Working |
| PHP HTTPS calls  | ✅ Working |
| Gmail / SMTP TLS | ✅ Working |
| Faveo SSL check  | ✅ Pass    |
| No hacks         | ✅         |

---

## 🧠 Key Rule (Remember This)

> **Never point `openssl.cafile` to a single CA**
> Always trust your CA via `update-ca-certificates`

---
