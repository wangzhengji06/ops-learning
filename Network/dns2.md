## Content Delivery Network (CDN)

You want one domain name to responds to different IP address. How?

`CNAME` to realize this. 

How to detect closest IP address?

There is IP address database.

These two combined is called `intellectual dns`.

## GSLB

GSLB is a way to find the physical address by IP address, and also the operator to find the most suitable edge server.

## DNS safety

1. Is the data encoded? Symmetric Encryption (One key)
2. is the server I  am about to visit legal? Asymmetric Encryption (Public key and private key)
3. Is the file I get from the server unmodified? One-way Encryption ()

If you use private key to encrypt, you can be sure who is the sender, but the message sent to you can be leaked.

If you use public key to encrypt, you can be sure no one can see the messages sent to you, but you cannot be sure who is the sender.

one-way encryption let you make sure that the sender is the one you are referring to

![image-20260323185013047](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260323185013047.png)

A quick look to use multiple ways to ensure secure and efficient transfer.

## CA certificate

The simple way to avoid man-in-the-middle attack. 
CA is issued by trusted organization. Our browser can automatically use CA public key to decode the info.

Is it really safe? 
Not necessarily....

The process is like this:

1. Server Generates Key Pair & CSR
2. CA Signs the Certificate
3. The Server  is configured to use the **Digital Certificate** and the **Private Key**.
4. When you open your **Browser** and type `https://...`, the Server sends its **Digital Certificate** (containing the Public Key) to the Browser. The digital certificate contains the content that is **encrypted by CA's private key**.
5. The Browser checks the content using CA's public key, and it works, so it generates a temporary **Symmetric Key (D-key)**, encrypts it with the **Server’s Public Key**, and sends it back. Only the Server can decrypt this with its **Private Key**

![image-20260323201910199](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260323201910199.png)

https == http + ssl/tls

## SSL /TLS

Client sends the information about supported TLS version etc...

Server sends a TLS version that both supports

Server sends the certificate, it could be CA certificate, and it must include server's public key.

Client will check validity of the certificate it receives

After that, both sides will use key to encrypt and decrypt

## HTTPS

based on HTTP protocol + SSL/TLS. 

![image-20260323204110933](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260323204110933.png)

First use DNS to get the ip address, then use tcp to do handshake, then use tls to make sure both are trusted.

## PKI

public key interface, use to manage the CA certificates and its related events. 

## Openssl

Used to create command for all the workflows listed above.

```bash
 # Digest
 openssl dgst/md5 file
 
 # Generate password using sha512 and salt
 openssl passwd -d 6 -salt "adscqwe9" 123456

# openssl generate public and private key
openssl genrsa -out private.pem 2048 #private key
openssl rsa -pubout -in private.pem -out public.pem #public key from private key

```

Certificate Related:

1.  build a CA agency: generate private key and certificate
2. Server side: generate private key and generate signature request
3. Get the request and sign, generate the certificate
4. Server side get the certificate, and use the certificate

For rocky, the config is `/etc/pki/tls/openssl.cnf`

For ubuntu, the config is /etc/ssl/openssl.cnf

The only difference is the root dir, otherwise the config is completely the same.

1. We first go though how to give CA agent itself a certificate

```bash
# Make the required folder
mkdir -pv /etc/pki/CA/{certs,crl,newcerts,private}

# Start generating CA's private key
cd /etc/pki/CA
openssl genrsa -out private/cakey.pem 2048

# Gnerate CA self-signature
openssl req -new -x509 -key /etc/pki/CA/private/cakey.pem -days 3650 -out /etc/pki/CA/cacert.pem

# Check the certificate
openssl x509 -in /etc/pki/CA/cacert.pem -noout -text


```

2. Next we go through how to sign for a server

```bash
# Generate private key for server
mkdir /data
openssl genrsa -out /data/test.key

# Generate signature request
openssl req -new -key /data/test.key -out /data/test.csr
```

3. Use CA to sign the certificate and issue it

```bash
# Generate index
touch /etc/pki/CA/index.txt

# Generate serialization number file
echo 0F > /etc/pki/CA/serial

# Sign the certificate
openssl ca -in /data/test.csr -out certs/test.crt -days 365

```

## SSH

First use asymetric encryption, then use symmetric encryption

server side: `/etc/ssh/sshd_config`

Important configuration: 1. PermitRootLogin 2. Allow

A use ssh to connect to B, the command is `ssh [-p 22]` 

![image-20260327194629458](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260327194629458.png)

The final results are, both sides have each other's public key,

## SCP

`scp sourecfile target_location` 

Two kinds:

1. push local file to remote location
2. pull remote file to local location

