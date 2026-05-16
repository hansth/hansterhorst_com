---
title: TLS Certificates
date: 2026-05-10
weight: 3
tags: [Security, TLS, Certificates]
---
# TLS Certificates

A TLS certificate establishes trust and enables secure communication between a client and a server. When a user accesses a web server, TLS certificates ensure that communication is encrypted and that the server is who it claims to be.

---

## Symmetric Encryption

Encryption is the process of using a key to convert plain text into an unreadable form. The encrypted data is then transmitted to the server. If a hacker intercepts the network traffic, the data is unreadable.

The problem is that the server also needs the key to decrypt the data. The key must be sent over the same network, making it vulnerable to interception.

This is called **symmetric encryption**: the same key is used for both encryption and decryption. Because the key must travel over the network, it is vulnerable to interception.

---

## Asymmetric Encryption

Asymmetric encryption uses a pair of keys: a **private key** and a **public key**. Data encrypted with the public key can only be decrypted with the private key. The private key is never shared. The public key can be distributed freely.

### Securing SSH Access

A practical example of asymmetric encryption is securing SSH access to servers. A key pair is generated using `ssh-keygen`, which produces two files:

- `id_rsa`: the private key.
- `id_rsa.pub`: the public key.

The public key is stored in the server's `~/.ssh/authorized_keys` file. While anyone can view the public key, access is secured by the private key. The same public key can be used across multiple servers, allowing the private key to grant access to all of them. Other users create their own key pairs and add their public keys to the servers they need to access.

### Securing Web Traffic

Asymmetric encryption solves the key exchange problem in symmetric encryption by providing a safe way to deliver the symmetric key to the server:

1. The server generates an asymmetric key pair.
2. When a client connects over HTTPS, the server sends its public key.
3. The client encrypts a symmetric key using the server's public key and sends it back.
4. The server decrypts the message using its private key and retrieves the symmetric key.
5. All further communication is encrypted with that symmetric key.

A hacker sniffing the network traffic receives the public key and the encrypted symmetric key, but cannot decrypt it without the server's private key.

---

## Server Impersonation

A hacker who cannot break the encryption may try a different approach: routing traffic to a fake server. The attacker sets up a website that looks identical to a legitimate one, hosts it on their own server, generates their own key pair, and manipulates the network so that requests are silently redirected. The client sees the familiar login page, communicates over HTTPS, and has no indication the connection is with the attacker's server.

Certificates address this problem.

---

## Certificates and Certificate Authorities

When a server sends its public key, it does not send the key alone. It sends a **certificate** containing the key, the server's identity, and a digital signature. The certificate includes a subject field identifying who it was issued to. For a web server, this must match the domain the client is connecting to. Additional domain names are listed under Subject Alternative Names.

Anyone can generate a certificate and sign it themselves. This is called a **self-signed certificate**. Browsers detect self-signed certificates and display a warning.

To get a certificate that browsers trust, it must be signed by a **Certificate Authority (CA)**. CAs are trusted organizations that validate and sign certificates. Examples include Symantec, DigiCert, Comodo, and GlobalSign.

The process for getting a signed certificate:

1. Generate a **Certificate Signing Request (CSR)** using the private key and domain name with `openssl`.
2. Submit the CSR to a CA.
3. The CA verifies the details, signs the certificate, and returns it.
4. The signed certificate is configured on the web server.

A fraudulent CSR would fail the CA's validation step and be rejected.

### How Browsers Trust CAs

CAs use their own private keys to sign certificates. The public keys of all well-known CAs are built into browsers and operating systems. When a browser receives a certificate, it uses the CA's public key to verify the signature.

For internal sites within an organization, a **private CA** can be deployed internally. The private CA's public key is installed on all internal browsers, establishing trust within the organization without relying on public CAs.

---

## Client Certificates

The process above allows a client to validate the server's identity. To verify the client's identity as well, the server can request a certificate from the client. The client generates a key pair, gets a signed certificate from a CA, and sends it to the server for verification.

Client certificates are not commonly used on public web servers, but generating client certificates for the Kubernetes API server is covered in a dedicated section.

---

## Public Key Infrastructure (PKI)

The entire infrastructure of CAs, servers, key pairs, and the processes for generating, distributing, and maintaining digital certificates is called **Public Key Infrastructure (PKI)**. PKI is the foundation for secure communication and underpins TLS in Kubernetes.

---

## Key Pair Usage

Data can be encrypted with either key in a pair, but can only be decrypted with the other key. Encrypting with the public key means only the holder of the private key can decrypt it. Encrypting with the private key means anyone holding the public key can decrypt it.

---

## Naming Conventions

Certificates and public keys typically use the `.crt` or `.pem` extension:

- `server.crt` / `server.pem`: server certificate.
- `client.crt` / `client.pem`: client certificate.

Private keys typically use the `.key` extension or include `key` in the filename:

- `server.key`
- `server-key.pem`

Files named `key` are private keys, and files without it are public keys.

---

## Conclusion

TLS security combines asymmetric and symmetric encryption. Asymmetric encryption solves the key exchange problem by enabling a symmetric key to be securely transferred. Certificates and CAs establish identity, preventing server impersonation. PKI ties the entire system together and is the foundation for secure communication in Kubernetes.

---
