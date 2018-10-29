# nds-constrain't
**Because Nintendo can't do SSL properly**

# What is this?
A way to sign SSL certificates such that the Nitro/TWL SDK's SSL library will treat as valid, without Nintendo's CA keys

# Background information
On Nintendo's consoles, online communications are carried out over various protocols but the predominant of these is HTTPS - HTTP carried within encrypted SSL packets.
After some initial unencrypted handshake steps in which the two parties exchange version information and capabilities, SSL servers will present an SSL certificate to the client containing the server's public key.
The client uses the server's public key to encrypt a shared secret that the server will decrypt with its private key.  This secret is used to protect a further key exchange step (e.g. Diffie-Hellman, although this can vary depending on the capabilities of each party) which will establish a new shared secret to be used to encrypt all further communications.
Since SSL is a recognised web standard it is fairly simple for anyone competent to setup a compatible server - however, SSL certificates (and indeed X509 certificates in general) are almost always signed by a certifying authority so that the client knows it can trust the server it's connecting to.
Unfortunately for us, the Nintendo SDK will refuse to connect to any server without a certificate signed by a trusted Nintendo CA, and Nintendo are of course not going to simply sign certificates for custom server authors to emulate theirs, leaving us unable to do anything without code patches... in theory.

# The flaw
SSL certificate authorities can choose to add a flag to certificates they sign to state that this certificate is allowed to act as an intermediate CA too - that is, it can sign certificates. If the SSL server includes the intermediate CA's certificate with its own signed by that same intermediate, the client can follow the chain of trust back up to the root and will therefore accept the certificate.
This, however, is where Nintendo's fatal flaw lies: in their implementation, they do not check if the intermediate certificate is supposed to sign other certs or not (indicated by a flag in the certificate's _basicConstraints_ section - `CA:TRUE` or `CA:FALSE`, as appropriate - leading to the name we gave this flaw), and as such if one has _any_ certificate signed by Nintendo with its accompanying private key, they can use it as an intermediate CA to sign their own certificates.

**And as it happens, we do.**

Nintendo consoles from the Wii onwards include a Nintendo-signed _client certificate_ which it uses to authenticate with Nintendo servers, as well as the accompanying private key for it. This client certificate is signed by the same CA as Nintendo's server certificates and as such is trusted by the DS.

**TL;DR: client certs can sign certs and the DS doesn't care!**

# Instructions
## Requirements
- The Wii's client certificate and its corresponding private key (these are not console-unique and can be pulled from any Wii, but are included here - `nwc.crt` and `nwc.key`.)
- A server which supports SSLv3 (may be somewhat difficult to set up as support for SSLv3 has been disabled/removed in most modern servers for security reasons... but it's all the DS supports)
- OpenSSL, to sign the certificates (or other suitable tool)

## Generating trusted certificates
```bash
# Generate a 1024-bit private key for your server cert (weak, but it's a DS)
openssl genrsa -out server.key 1024
# Generate a certificate request
openssl req -new -key server.key -out server.csr
# Produce a signed server certificate from the certificate request, signed with the Wii's client certificate.
# The SHA-1 digest algorithm must be used because we're working with old tech here, but you can customise the validity period as you see fit. Here, we specify 10 years.
openssl x509 -req -in server.csr -CA nwc.crt -CAkey nwc.key -CAcreateserial -out server.crt -days 3650 -sha1
```

Copy the server.key and server.crt files to the correct places, and then configure your server to use them.
You must also send `nwc.crt` to the client as part of the certificate chain, otherwise the DS will not be able to follow the chain of trust back up to the root and reject your lovely certificate - just concatenate your server certificate with `nwc.crt` like so: `cat server.crt nwc.crt > server.chain.crt`.

Alternatively, with Apache, you can add `SSLCertificateChainFile /path/to/nwc.crt` to your server config.

Example configuration:
```apache
<IfModule mod_ssl.c>
  <VirtualHost *:443>
    DocumentRoot /var/www/html
    ServerName nas.nintendowifi.net
    
    SSLEngine on
    SSLCertificateFile /path/to/server.crt
    SSLCertificateKeyFile /path/to/server.key
    SSLCertificateChainFile /path/to/nwc.crt
  </VirtualHost>
</IfModule>
```

## Testing Server
In order to join Kaeru Team's official server, you can use the following DNS settings:
Primary: 178.62.43.212
Secondary: 1.1.1.1 or 8.8.8.8
