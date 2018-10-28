# nds-constrain't
Because Nintendo can't do SSL properly.

# How does this work?
The NDS SDK's SSL library supports something called "cert chains", which is a standard thing that all SSL libs should support.

However, there is a fatal flaw in their implementation: they do not check if a cert is supposed to sign other certs or not (in other words: it doesn't check if it is a CA)

Since we have some Nintendo signed certificates with private keys (client certs from the Wii), we can simply sign with those, and then return them from the server as part of the chain.

# Requirements
The Wii's client cert (nwc.crt and nwc.key in this repo)
A server which supports SSLv3 (harder than it sounds lol)
OpenSSL to generate the certs

# How to make the certs?
```
openssl genrsa -out server.key 1024
openssl req -new -key server.key -out server.csr
openssl x509 -req -in server.csr -CA nwc.crt -CAkey nwc.key -CAcreateserial -out server.crt -days 99999 -sha1
```

Copy the server.key and server.crt files to the correct places, and then configure your server to use them.
Additionally, add `SSLCertificateChainFile "nwc.crt"` to your server config.
