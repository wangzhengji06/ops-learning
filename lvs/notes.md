## lvs
Nginx can do load balancing on transport layer, and application layer. Lvs can only do load balancing on trasnport layer. 

Nginx is not doing real load balancing on transport layer. Why? Because there are two independent connections. Client first connect to nginx, then nginx connect to another service. Lvs, on the other hand, directly forwarded by lvs without even entering into the user kernel. 

## lvs project stucture
lvs is a load balancing software. lvs exposes an ip address to outside, and also has a inside address for other cluster nodes. Also lvs needs rules to determine

1. rules for different scenario. NAT, DR, TUN. 
2. how to spread out the requests.

Lvs node is called director, while the local servers it connect to are called real server. Real server has Real IP (RIP). Director has DIP. The ip address lvs export to clients are called vip. Client has cip.

Think about the iptable's netfiler framework. lvs lies in between prerouting and input, and directly connect to postrouting, you can think of it that way. 

lvs is made up of 2 softwares: ipvs, ipvsadm.

## lvs basic

```bash
apt install ipvsadm - y

ipvsadm -Ln # this will show all the rules for lvs
# L here means local server; n means number; 
```

### nat mode

network address translation

A data package looks like this: Data Linkage Head -> Network layer head -> Transport layer head -> Application layer head -> data

Data Linkage layer head has `s_mac:d_mac`. Network Layer has `s_ip:d_ip`. Transport lyaer has `s_port:_port`. 

In nat scenario, we need to change network and transport layer heads. 

When an outside request comes to vip, we modify the d_ip to point to real server, and when real server respond, we change the source ip to vip. 

When a request comes to lvs, it looks at the vip and sais, okay, this is for me. But before it goes into forward, the lvs will hook it and say, nope, time to send to real server. Then the data will be sent to forward then postrouting.

When a response sent back to lvs, it looks at the address and sais, okay not for me. THen it will direct it to forward, then post routing. This step will change the source ip.

### dr mode

default mode for lvs. between lvs and Rs, there can only be switcher, not router. It directly searchs for mac. 

dr mode changes the data linkage head only. This is much faster, because it only needs to unpack one layer of head. 

It changes client mac : virtual ip mac -> Dmac : RMac. AN important part here is, in this mode, the RS needs to have the VIP as well.

When it reads the MAC, then it reads the ip, it can realize that okay this sends to me. when it sends back, the ip address is also Vip. 

The problem here is, when it enters it will call ARP to know which MAC address corresponds to VIP. But lvs and rs all have vips. 

What we need to do it to surpress the RS's ARP response, and also not expose the ARP to outside. 

Two conditions to meet:
1. All rs need to have vip.
2. rs server needs to be modified on the kernel leve: it cannot reply to ARP requests, and cannot automatically send ARP requests.


### tun mode

dr mode's biggest challenge is that it requires all the rs and lvs on the same physical network.

A solution is to create virtual network, so that server in different places act in a way that they can talk to each other. But now the problem becomes, You cannot use the standard data format. You need to add another layer of ip head for the tunnel. 

The RS stil needs vip, after it gets the request it directly send to client without going through lvs. The intuition here is , the outside ip address sends via the actual internet while the inner ip addrress is for the rs to handle the requests.

The preparation you need to do is to tell RS that they should expect to recieve unstandard package. ALso you still need to ban ARP to make sure only the lvs respond to the initial vip requests.

### In summary
DR has the best performance, TUN allows to travel across network domain.


## ipvsadmin command practice


```bash
ipvsadm -A -t|u|f service_address:port [-s scheduler] [-p [timeout]] #main usage

ipvsadm -Ln # hey what rules have I added

ipvsadm -A -t 10.0.0.100:80 # adds a virtual address for tcp

ipvsadm -E -t 10.0.0.100:80 -s wrr # modify 10.0.0.100:80 tcp to use wrr algorthm

ipvsadm -D -t 10.0.0.100:80 # delete 10.0.0.100:80 

ipvsadm -C # clear all rules

ipvsadm -a -t 10.0.0.100:80 -r 11.11.11.11:80 # adds a real server to a target using dr

ipvsadm -a -t 10.0.0.100:81 -r 11.11.11.13 -m # add using nat

ipvsadm -a -t 10.0.0.100:83 -r 11.11.11.15 -i # add tunnel mode 
 
ipvsadm -e -t 10.0.0.100:83 -r 11.11.11.15 -g # modify the group to use direct route mode
```


You can save the config

```bash
ipvsadm-save > a.txt

ipvsadm-restore < a.txt
```


AUtomatically add 

```bash
cat a.rule  >> /etc/ipvsadm.rules

vim /etc/default/ipvsdam

AUTO="true"; x
```


## Practice

Super complex. What you need to understand is you can use both http and https, and you can use firewall to mark them, so that they can be regarded as withitn the same group.

THis is useful when you want to mix two services but still want the real servers to kind of distribute the requests in an equal way.

