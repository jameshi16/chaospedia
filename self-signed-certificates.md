---
description: How to properly generate self-signed certificates to use on all platforms
---

# 🪪 Self-Signed Certificates

During development, or when self-hosing any servers that require HTTPS connecitons, there sometimes comes a need to use self-signed certificates. There are many sources online that provide a single command to generate both the certificate and the key, however, that certificate is usually doubles as the Certificate Authority (CA), which I feel is bad practice to cultivate and use directly.

The correct way involves:

1. Generating CA certificate & key;
2. Generate certificates & keys for each domain / purpose;
3. Use CA certificate & key to sign the certificate in Step 2;
4. Distribute only the CA certificate to clients;
5. Use only the (CA + domain) certificate & domain key on servers.

## Generating CA key and certificate

```
openssl req -x509 -sha256 -nodes -days 1825 -newkey rsa:2048 -keyout rootCA.key -out rootCA.crt
```

When answering the prompts, `commonName` can be whatever you want, **including** a human-friendly name with spaces inside. Something like "My Root CA". The `rootCA.crt` file is what you want to distribute, and the `rootCA.key` file is what you want to lock away forever.

## Creating the domain key and certificate

First, create a `domain.ext` file, and specify the following:

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
subjectAltName = @alt_names

[alt_names]
DNS.1 = dns.here
IP.1 = 1.2.3.4
```

Replace `dns.here` and `1.2.3.4` with your actual DNS (if any), and IP (if any). If any of those fields are blank, delete the entries directly.

Then, create a domain key and a domain certificate signing request (CSR).

```
openssl req -newkey rsa:2048 -nodes -keyout domain.key -out domain.csr
```

Finally, use the CA credentials to sign `domain.csr`:

```
openssl x509 -req -CA rootCA.crt -CAkey rootCA.key -in domain.csr -out domain.crt -days 365 -CAcreateserial -extfile domain.ext
```

Before using `domain.crt`, combine it with `rootCA.crt` to create a certificate chain:

```
cat domain.crt rootCA.crt > server.crt
```

Whatever applications that will be serving HTTPS should use `server.crt` and `domain.key`. Any clients should use their own platform methods to install the `rootCA.crt`into their own certificate managers.
