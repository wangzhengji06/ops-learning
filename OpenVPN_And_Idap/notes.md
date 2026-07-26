Suppose that you want to connect to company network from your home.

You need to install a VPN client, and connect to a VPN server, and then the VPN server can serves as the mask for the ip address to forward the request, so that you can access the local network in your company.


## VPN(Virtual Private Environment) basic knowledge

The structure goes like this:
1. 10.0.0.13 Use openvpn's client
2. 172.30.0.66 On the cloud, has the openvpn server
3. 172.30.0.100 another server on the cloud
4. 172.30.0.200 another server on the cloud


## OpenVPN Hands-on Setup

### 0. Alibaba Cloud Environment

* Register an Alibaba Cloud account and top up ¥130
* Create a VPC
* Create a security group
* Create an ECS instance

  * Cost: approximately ¥0.10
* Create an EIP

  * Cost: approximately ¥0.02

### 1. Server-side Setup

* Install the OpenVPN server software
* Generate connection certificates and keys

  * Generate the server certificate and key pair
  * Generate the client certificate and key pair
* Create the configuration files

  * Create the server configuration
  * Create the client configuration
* Start the OpenVPN server

### 2. Client-side Setup

* Install the OpenVPN client software
* Configure the client

  * Obtain the client certificate and private key
  * Obtain the client configuration file
* Start the OpenVPN client
 
 
## The installation of OpenVPN certificate

```bash
apt udpate; apt install openvpn easy-rsa 

./easyrsa init-pki

./easyrsa build-ca nopass # (press enter when prompted)
```

Now you can start generating the certificate for the server side.
```bash
./easyrsa gen-req server nopass # (common name: sswang-vpn.magedu.com)
```


Now you can issue the ca certificate
```bash
./easyrsa sign-req server server
```


Now you need to create the key pair
```bash
./easyrsa gen-dh
```

Now the server side is ended for ca.

Now you need to start with the client side.

```bash
./easyrsa gen-req tom nopass # sign a certificate to someone named tome
```

You can sign the request

```bash
./easyrsa sign-req client tom 
```

Now we need to copy the files to the designated folders

```bash
cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/server/
cp /etc/openvpn/easy-rsa/pki/issued/server.crt /etc/openvpn/server/
cp /etc/openvpn/easy-rsa/pki/private/server.key /etc/openvpn/server/
cp /etc/openvpn/easy-rsa/pki/dh.pem /etc/openvpn/server/

mkdir /etc/openvpn/client/{tom,jerry}

cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/client/tom/
cp /etc/openvpn/easy-rsa/pki/private/tom.key /etc/openvpn/client/tom/
cp /etc/openvpn/easy-rsa/pki/issued/tom.crt /etc/openvpn/client/tom/
cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/client/jerry/
cp /etc/openvpn/easy-rsa/pki/private/jerry.key /etc/openvpn/client/jerry/
cp /etc/openvpn/easy-rsa/pki/issued/jerry.crt /etc/openvpn/client/jerry/
```



## Set up OpenVPN on the server side

```bash
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/
```

Now we vim server.conf

```conf
port 1194
proto tcp
dev tun
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem
server 10.8.0.0 255.255.255.0
push "route 172.30.0.0 255.255.255.0"
keepalive 10 120
cipher AES-256-CBC
compress lz4-v2
push "compress lz4-v2"
max-clients 2048
user root
group root
status /var/log/openvpn/openvpn-status.log
log-append /var/log/openvpn/openvpn.log
verb 3
mute 20
```


```bash
systemctl enable --now openvpn.service

```


## Set up OpenVPN on the client Side

vim client/tom/client.ovpn

```conf
client
dev tun
proto tcp
remote 39.101.79.252 1194
resolv-retry infinite
nobind
#persist-key
#persist-tun
ca ca.crt
cert tom.crt
key tom.key
remote-cert-tls server
#tls-auth ta.key 1
cipher AES-256-CBC
verb 3 
compress lz4-v2 
```


## The after setup

```bash
echo net.ipv4.ip_forward = 1 >> /etc/sysctl.conf

iptables -t nat -A POSTROUTING -s 10.8.0.0/24 ! -d 10.8.0.0/24 -j MASQUERADE
```

## OpenVPN Management

Double authentication
```bash
openvpn --genkey secret /etc/openvpn/server/ta.key
```

Now change the conf

```bash
vim /etc/openvpn/server.conf

tls-auth /etc/openvpn/server/ta.key 0 # server side
```

Then you change the client's config also

```
tls-auth /etc/openvpn/server/ta.key 1 #client side
```





