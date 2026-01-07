# ✅ STEP-BY-STEP FIX (DO THIS EXACTLY)

## 🔥 Step 0: Backup & clean old certs

```bash
cd /etc/apache2/ssl/
mkdir -p /etc/apache2/ssl-backup
cp /etc/apache2/ssl/* /etc/apache2/ssl-backup
```

---

## 🟢 Step 1: Create a PROPER Root CA (CA:TRUE)

```bash
cd /etc/apache2/ssl

openssl ecparam -name prime256v1 -genkey -out rootCA.key

openssl req -x509 -new -nodes -sha256 -days 3650 \
-key rootCA.key \
-out rootCA.crt \
-subj "/C=IN/ST=TN/O=FAVEO/CN=FAVEO-ROOT-CA" \
-addext "basicConstraints=critical,CA:TRUE" \
-addext "keyUsage=critical,keyCertSign,cRLSign"
```

✅ This is now a **real CA**

Verify:

```bash
openssl x509 -in rootCA.crt -noout -text | grep -A3 "Basic Constraints"
```

You **MUST** see:

```
CA:TRUE
```

---

## 🟢 Step 2: Create Server Key

```bash
openssl ecparam -name prime256v1 -genkey -out server.key
```

---

## 🟢 Step 3: Create Server CSR (IMPORTANT CN + SAN)

```bash
openssl req -new -key server.key -out server.csr \
-subj "/C=IN/ST=TN/O=FAVEO/CN=siva.localhost"
```

---

## 🟢 Step 4: Create SAN config (CRITICAL)

```bash
cat > san.cnf <<EOF
basicConstraints=CA:FALSE
subjectAltName = @alt_names
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[alt_names]
DNS.1 = siva.localhost
IP.1 = 127.0.0.1
EOF
```

---

## 🟢 Step 5: Sign server cert with your CA

```bash
openssl x509 -req -in server.csr \
-CA rootCA.crt -CAkey rootCA.key -CAcreateserial \
-out server.crt -days 3650 -sha256 \
-extfile san.cnf
```

---

## 🟢 Step 6: Install certs

```bash
cp server.crt /etc/ssl/certs/
cp server.key /etc/ssl/private/
cp rootCA.crt /usr/local/share/ca-certificates/rootCA.crt
```

---

## 🟢 Step 7: Trust the CA (THIS TIME IT WILL WORK)

```bash
update-ca-certificates --fresh
```

You MUST see:

```
Adding debian:rootCA.pem
```

---

## 🟢 Step 8: Apache SSL config

```apache
SSLCertificateFile /etc/ssl/certs/server.crt
SSLCertificateKeyFile /etc/ssl/private/server.key
```

Then:

```bash
systemctl restart apache2
systemctl restart php8.2-fpm
```

---

## 🟢 Step 9: PHP config (IMPORTANT)

❌ **DO NOT set**:

```ini
openssl.cafile=/usr/local/share/ca-certificates/rootCA.crt
```

✅ Let PHP use system bundle:

```ini
; openssl.cafile = 
```

This way:

* Gmail works
* Local HTTPS works
* No conflicts

---

## ✅ FINAL VALIDATION

### Check cert chain:

```bash
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/server.crt
```

Output must be:

```
OK
```

### Test PHP HTTPS:

```bash
php -r "echo file_get_contents('https://siva.localhost');"
```

✔ No error = DONE

---

## 🧠 WHY THIS FIXES EVERYTHING

| Component        | Result            |
| ---------------- | ----------------- |
| Root CA          | Proper CA:TRUE    |
| Server cert      | Correct issuer    |
| PHP OpenSSL      | Uses system trust |
| Mail (Gmail)     | Works             |
| Faveo HTTPS      | Works             |
| No php.ini hacks | ✅                 |

---

## 🚨 Important truth

> **Your earlier CA was NOT a CA**
> That’s why nothing behaved consistently.

Now you’ll have **enterprise-correct SSL**.

---
