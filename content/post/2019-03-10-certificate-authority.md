---
title: "Certificate Authority With Hardware Token [Yubikey]"
date: 2019-10-01T11:58:08+02:00
draft: false
tags: ["yubikey", "certificate", "pkcs11"]
---

## Install dependencies for working with PKCS11

```bash
brew install yubico-piv-tool openssl@1.1 opensc libp11
```

## Generating new Root CA certificate
Definition of Root CA 

First of all we need to generate private key to be used to generate public key for our new certificate. In current example used 2048 bits of rsa, this is related to limitation of specific hardware token.

```bash
openssl genrsa -out root-ca.pem 2048
```

We need to describe certificate definition in openssl configuration file `root-ca.config`:

```
[ req ]
x509_extensions = v3_ca
distinguished_name = req_distinguished_name
prompt = no
[ req_distinguished_name ]
CN=LetRun Root CA
[ v3_ca ]
subjectKeyIdentifier=hash
basicConstraints=critical,CA:true,pathlen:1
keyUsage=critical,keyCertSign,cRLSign
nameConstraints=critical,@nc
[ nc ]
permitted;otherName=1.3.6.1.5.5.7.8.7;IA5:let.run
permitted;email.0=let.run
permitted;email.1=.let.run
permitted;DNS=let.run
permitted;URI.0=let.run
permitted;URI.1=.let.run
permitted;IP.0=0.0.0.0/255.255.255.255
permitted;IP.1=::/ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
```

Create certificate by using this configruation

```bash
echo 01 > root-ca-crt.srl
openssl req -new -sha256 -x509 -set_serial 1 -days 21900 \
    -config root-ca.config \
    -key root-ca.pem \
    -out root-ca-crt.pem
```

## Upload to Yubikey PIV Smart Card

Now we need to upload our private key and generated certificate to yubikey

```bash
yubico-piv-tool -k -a import-key -s 9c < root-ca.pem
yubico-piv-tool -k -a import-certificate -s 9c < root-ca-crt.pem
```

## Issue new Intermediate Certificate signed by Root CA

```bash
openssl genrsa -out ica-hubcli-key.pem 2048
```

Create `ica-hubcli-csr.config` openssl configuration

```
[ req ]
distinguished_name = req_distinguished_name
prompt = no
[ req_distinguished_name ]
CN=hubcli Peach Intermediate CA
```

Generate certificate signature request file (CSR)

```bash
openssl req -sha256 -new -config ica-hubcli-csr.config \
    -key ica-hubcli-key.pem \
    -nodes \
    -out ica-hubcli-csr.pem
```

Generate the Intermediate CA certificate with configuration `ica-hubcli-crt.config`:

```
basicConstraints = critical, CA:true, pathlen:0
keyUsage=critical, keyCertSign
```

To find correct slotes and objects in smart card it is better to list all stored objects in smart card:

```bash
pkcs11-tool -O
```

Sign intermediate certificate authority by Root CA with private key stored in smart card

```bash
echo 00 > ica-hubcli-crt.srl
/usr/local/Cellar/openssl@1.1/1.1.1d/bin/openssl << EOF
engine dynamic \
    -pre SO_PATH:/usr/local/Cellar/libp11/0.4.10/lib/engines-1.1/pkcs11.dylib \
    -pre ID:pkcs11 \
    -pre LIST_ADD:1 \
    -pre LOAD \
    -pre MODULE_PATH:/usr/local/Cellar/opensc/0.19.0_1/lib/opensc-pkcs11.so \
    -pre VERBOSE
x509 -sha256 \
    -engine pkcs11 \
    -CA root-ca-crt.pem \
    -CAkeyform engine \
    -CAkey slot_0-id_2 \
    -req -days 7300 \
    -in ica-hubcli-csr.pem \
    -extfile ica-hubcli-crt.config \
    -out ica-hubcli-crt.pem
EOF
```

## Generate End Certificate

Same as pervious steps we need to generate private keys for end certificate first:

```bash
openssl genrsa -out end-hubcli-rpc-server-key.pem 2048
```

Create openssl request configuration file, it is important to specify real host name for end certificate (ip adress if needed, etc), example of `end-hubcli-rpc-server.config`:

```
[ req ]
distinguished_name = req_distinguished_name
prompt = no
[ req_distinguished_name ]
CN=hubcli.at.lnd.cloud
```

Create certificate signature request (CSR) for our end certificate:

```bash
openssl req -sha256 -new \
    -config end-hubcli-rpc-server-csr.config \
    -key end-hubcli-rpc-server-key.pem \
    -nodes \
    -out end-hubcli-rpc-server-csr.pem
```

Create signatrue configuration for end certificate:

```
basicConstraints = critical,CA:false
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=critical,serverAuth
subjectAltName=critical,DNS:hubcli.at.lnd.cloud
```

And sign it

```bash
openssl x509 -sha256 \
    -CA ica-hubcli-crt.pem \
    -CAkey ica-hubcli-key.pem \
    -req -days 730 \
    -in end-hubcli-rpc-server-csr.pem \
    -extfile end-hubcli-rpc-server-crt.config \
    -out end-hubcli-rpc-server-crt.pem
```

