## General Idea
Keep alived mainly is used to solve the problem in such scenario: You have are running lvs, and all the outside requests from clients point to vip. Suppose that lvs server is broken, how do I make sure to let another server to take the VIP? 

Why keepalived is necessary? Because when both server has the same VIP, Arp will have collision, so at one time, only one lvs server should be alive.

## Three modes
1. Active / Passive -> Only active respons to requests from clients
2. Active / Active -> RUns their own service, when one die, both will still stay alive
3. N+M -> A cluster, when the master dies, they will pick slave based on some scoring

## Core technique
1. Time synchronization: Usually use ntpd.
2. Heartbeat Check: Encrypted way of telling whether a server is still alive.

## VRRP (virtual router)
vrrp is created to group the physical routers, the host only needs to know that there is a virtual server. THen it will call this as the route. The VRRP will select the main router which is still alive.

## Install keepalive (from packages)
```bash
apt install keepalived

vim /etc/keepalived/keepalived.conf # Use the sample file here, remember to change the net drive name

nft list ruleset; # You can see that nft is banning you from pining the vip address

nft flush ruleset;

vim /etc/nftables.conf
```


## Install keepalive (build from source)
```bash
mkdir /data/{server,softs} -p && cd /data/softs
wget https://keepalived.org/software/keepalived-2.3.4.tar.gz
tar xvf keepalived-2.3.4.tar.gz
cd keepalived-2.3.4
./configure --prefix=/data/server/keepalived
make
make install
systemctl daemon-reload
```


## Check the VRRP

```bash
tcpdump -nn host 224.0.0.18
# vrrp protocol by default send to 224.0.0.18 to report that right now I am in charge and I have heartbeat.
```

## Config structure

* global config: global_defs
* high availability config: vrrp_instance
* scale config: virtual_server


## Add child config
```bash
cd /etc/keepalived
mkdir conf.d

# Change the main config and add this line
include /etc/keepalived/conf.d/*.conf

# Write the virtual servr config as seperate files inside conf.d folder.

```

## Work Mode

1. preempt mode: after master goes back, it will take the lvs back
2. non-preempt mode: after master goes back, it wont take the lvs back
3. preempt delay mode: After a while I wil take the service back

## Setting up unicast

By default keepalived use multicast. However, you can use unicast sometimes. 

```config
    unicast_src_ip 192.168.8.13
    unicast_peer{
      192.168.8.16
    }
```

## Split-brain
Due to mutiple vip shown in lvs servers.

The main reason is: the slave node cannot recieve the message from master node.

