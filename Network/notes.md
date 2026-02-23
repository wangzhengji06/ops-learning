## OSI 7 level model (data package)

All People seem to need data processing

* Application layer

* Presentation layer

* Session layer

* Transport layer

* Network layer

* Data Link Layer

* Physical Layer


from top to down, you encapsulate the data package, and from down to top, you decapsulate it.


Protocols exist in different layers of internet. 

Application: DNS http ftp

session layer: tls ssl

transport: tcp udp

netowrk: ip


Segment -> travel from application to transport, most important part is the port

Packet -> in the network layer, most important part is the ip address

Frame -> in the data link layer, most important part is the MAC address


### LAN, WAN

LAN: data link layer and physical layer problem

Use IEEE802. LLc serves network layer, while MAC talkes to data link and physical layer


### Switch Machine

Holds a MAC table, and Given a Mac address it wil search and send frame. But if not, it will try to send to every one of them.


### Routing Machine

Device under the same subnet can talk to each other, but of course they cannot talk to device on another subnet. 

They need 3 tables, MAC table(L2), Route table(L3), ARP table(L3 -> L2)

ARP protocol: get the targeted MAC address corresponds to ip address


### VLAN

virtual local network, that splits the LAN to dvide into small VLAN./

Traditional Frame: Destination Address + Source Address + Length + Data + FCS

VALN Frame: Add a mark to notate the vlan


### TCP 

TCP: reliable, use three-way handshake to make sure both sides are ready to communicate

TCP head: usually 20 bytes. 2 bytes: source port, 2 bytes: destination part, 4 bytes: sequence number, 4 bytes: acknowledge number, 4 bytes: Bunch of notations (ACK, FYN, FIN) and window...... 


Server A    Server B

First, both are closed.

Now server A requests to talk to B. This request will have SYN = 1.

Server B accepts the request, and send request to establish the connection. This will have SYN and ACK = 1

Server A confirms the establishment from server B. This will have ACK = 1.

Now Server A and Server B's establishment has been accepted.


Server A    Server B

First, both are established.

Server A sends a request to disconnect from Server B. FIN = 1 

Server B gets the request, confirm it, and sends request with AFK = 1

Server A confrims it and disconnect from Server B.

Server B will not sends a request to disconnect from Server A, with FIN = 1

Samething, now the Server A sends a request with AFK = 1

The Server B confrims it and disconnect from Server A. 



### Core protocol for network layer

IP, ICMP, ARP

ICMP is for ping to test


### IP address

The point of netmask: to seperate hostname and subset name from the ip address

10.0.0.12

if mask is 24, then you have 256 numbers of hostnames. 10.0.0.0 represents the network address, and 10.0.0.255 represents the broadcast address

A: mask length 8  local address: 10.0.0.0/8

B: mask length 16 local address: 172.16.0.0/16

C: mask length 24 local address: 192.168.0.0 /24


0.0.0.0: every ip address

127.0.0.1 - 127.255.255.254: local host

255.255.255.255: all the hosts within the broadcast domain


Extending the netmask can divide into different subnets, while shortening it can merge different subnets.

### Subnet communication

How does the hostname knows whether the target is inside same subnet? Just check the mask location

Server A: 10.0.1.12/16  Server B: 10.0.0.17/24

Scenario 1: Can Server A ping Server B successfully?  yes.

Scenario 2: Can Server B ping Server A successfully?  no.

### Customize IP address

rocky 9, 10:  /etc/NetworkManager/system-connections/ens160.nmconnection

ubuntu: /etc/netplan/50-.... (The desktop version has other yaml files)

openuler: /etc/sysconfig/network-scripts/ifcfg-ens160

#### Centos7 / OpenEuler

TYPE=Ethernet

PROXY_METHOD=none

BROWSER_ONLY=no

BOOTPROTO=none

DEFROUTE=yes

IPV4_FAILURE_FATAL=no

NAME=ens224

DEVICE=ens224

ONBOOT=yes

IPADDR=10.0.0.114

NETMASK=255.255.255.0

GATEWAT=10.0.0.2

DNS=10.0.0.2

#### Rocky 10 
[connection]

id=ens224

type=ethernet

interface-name=ens224



[ipv4]

method=manual

address=10.0.0.112/24,10.0.0.2

dns=10.0.0.2

#### Ubuntu

```
network:

    ethernets:

        ens33:

            addresses:

            - 10.0.0.16/24

            nameservers:

                addresses:

                - 10.0.0.2

                search: []

            routes:

            -   to: default

                via: 10.0.0.2

        ens37:

            addresses:

            - 10.0.0.116/24

            - 10.0.0.117/24

            - 10.0.0.119/24

    version: 2
```
```
netplan apply

ip a

```

