---
tags: self-signed-certificate openssl JKS
---

# Create self-signed certificate

## Create self-signed certificate with OpenSSL

### 1. Create your own CA certificate

```bash
openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout ca.key \
  -x509 -days 365 -out ca.crt
```

### 2. Generate a Certificate Signing Request (by openssl)

If you use FQDN like reg.your-domain.com to connect your registry host, then you must use reg.your-domain.com as CN (Common Name). Otherwise, if you use IP address to connect your registry host, CN can be anything like your name and so on:

```bash
openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout your-domain.com.key \
  -out your-domain.com.csr
```

### 3. Generate the certificate of your registry host

If you're using FQDN like reg.your-domain.com to connect your registry host, then run this command to generate the certificate of your registry host:

```bash
openssl x509 -req \
  -days 365 -in your-domain.com.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out your-domain.com.crt
```

If you're using IP like 192.168.1.101 to connect your registry host, you may instead run the command below:

```bash
echo subjectAltName = IP:192.168.1.101 > extfile.cnf

openssl x509 -req \
  -days 365 -in your-domain.com.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -extfile extfile.cnf \
  -out your-domain.com.crt
```

## Create self-signed JKS certificate

### 1. Create JKS key pairs

```bash
keytool -genkey -alias ss.ifplusor.win -keyalg RSA -keysize 4096
  -dname "CN=ss.ifplusor.win, OU=ss, O=ifplusor.win, L=Beijing, S=Beijing, C=CN
  -keypass 123456 -keystore ss.ifplusor.win.jks -storepass 123456
```

### 2. Generate a Certificate Signing Request (by keytool)

```bash
keytool -certreq -alias ss.ifplusor.win -sigalg "MD5withRSA"
  -file ss.ifplusor.win.csr
  -keypass 123456 -keystore ss.ifplusor.win.jks -storepass 123456
```

### 3. Generate the JKS certificate

```bash
openssl x509 -req -in ss.ifplusor.win.csr -out ss.ifplusor.win.crt
  -CA ca.ifplusor.win.crt -CAkey ca.ifplusor.win.key
  -days 3650 -CAserial../ca.ifplusor.win/ca.srl -sha1 -trustout
```

## Convert OpenSSL certificate to JKS certificate

### 1. Convert to PKCS12 format

```bash
openssl pkcs12 -export -clcerts -in server.crt -inkey server.key -out server.p12
```

### 2. Convert to JKS format

```bash
keytool -importkeystore -srckeystore server.p12 -destkeystore server.jks -srcstoretype pkcs12 -deststoretype jks
```

### 3. Check certificate

```bash
keytool -list -v -keystore server.jks
```
