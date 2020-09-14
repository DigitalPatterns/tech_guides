# SSL Certificate validation


Create a CA Bundle

```bash
cat IntermediateCA.pem RootCA.pem >> ca_bundle.pem
```

Start a server using your TLS certificate, key and CA bundle.

```bash
openssl s_server -key server.pem -cert server.pem -accept 44330 -www -CAfile ca_bundle.pem
```

Either open a browser pointing to https://localhost:44330
or 

```bash
openssl s_client -connect localhost:44330 -CAfile ca_bundle.pem
```


To verify a certificate do:

```bash
openssl verify -CAfile ca_bundle.pem server.pem
```

To see the details of a certificate (it can be a client, server or ca certificate)

```bash
openssl x509 -in server.pem -text -noout
```
