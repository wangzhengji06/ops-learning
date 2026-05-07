##  Time zone

```bash
timedatectl set-ntp 1 # Start the time synchronization
```

This is backed up by `systemd-timesynced`.  

## Time Synchronization

All servers point to one server for time synchronization. `chrony` represents the full implementation of **NTP protocol**. 

```bash
chronyc courses # Check source
chronyc clients # Check clients that is using the service 
chronyc activity # Check current service status
```

The config file for rocky is in `/etc/chrony.conf`. For ubuntu it is `/etc/chrony/chrony.conf` 

The common setting is, one server synchronizes with outside source, and all the other servers synchronizes with that one server.

```bash
# On server side

vim /etc/chrony/chrony.conf


# pool ntp.ubuntu.com        iburst maxsources 4
# pool 0.ubuntu.pool.ntp.org iburst maxsources 1
# pool 1.ubuntu.pool.ntp.org iburst maxsources 1
# pool 2.ubuntu.pool.ntp.org iburst maxsources 2
server ntp1.aliyun.com iburst
allow 10.0.0.0/24
local stratum 10 # Even if the outside source does not work, it will still provide service for 10.0.0.0/24
```

```bash
# On Client side

vim /etc/chrony.conf

# pool 2.rocky.pool.ntp.org iburst
server 10.0.0.13 iburst

```

## Firewall

What can I impose restriction on?

### Application Layer

protocol and data

### Transport Layer

port

### Network Layer

ip

### Physical Layer

Mac address

![image-20260329145133461](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260329145133461.png)

Linux use `netfilter` to manage all these mechanics, and we use `iptables` command to manage `netfilter` framework

Now it `netfilter` has upgraded to `nftable`, and we use `nft` to manage `nftable` framework. But the `iptables` remain to be the command that is available. Kind of like `yum` and `dnf`. 

Rocky uses `firewalld` and Ubuntu uses `ufw`, both acted as another layer for the `iptables`. 

```bash
nft list ruleset # what are the nft rules?
nft flush ruleset # Empty all the rules
```

![image-20260329160225463](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260329160225463.png)

### 5 tables and 5 chains

* filter
* nat
* mangle: packet modification
* raw: track exemption
* security

```
priority
security --> raw --> mangle --> nat --> filter
```

* PREROUTING
* INPUT
* FORWARD
* OUTPUT
* POSTROUTING

```bash
iptables -t [table name] -vnL  # Check different tables' settings
iptables -t filter -A INPUT -s 10.0.0.2 -j DROP # drop all the message from 10.0.0.2 from INPUT chain
iptables-save > iptables.rules # Save the rules
iptables-restore < iptables.rules # Load the rules
iptables -F # Clear all the rules 


## Rule command
-A  # append rule
-I  # insert rule
-D  # delete rule
-F  # flush rules
-R  # replace rule
-Z  # empty record data

## For example
iptables -D INPUT -s 10.0.0.19 -j DROP # Delete by content
iptables -vnL --line-numbers ; iptables -D INPUT 3# 

## Operations
ACCEPT # allow pass
DROP  # deny pass
RETURN # pass it to the other rules
SNAT # Change the source IP address
REDIRECT # redirect the packet
REJECT # reject
MASQUEDA # large-scale ip redistribution 
DNAT # Change the target IP address

```

A very important thing here is, the rules have certain orders, you should put those that are specific to the front, and general ones to the tail.

 ```bash
 iptables -I INPUT 2 -s 127.0.0.1 -j ACCEPT # insert this rule into the second location
 iptables -t filter -I INPUT 2 -i lo -j ACCEPT # for webdrive lo, we allow the transformation
 iptables -R INPUT 2 -s 127.0.0.1 -j ACCEPT # Replace the second input rule into accepting 127.0.0.1
 ```



iptables can match rules based on all the followings, as well as device

![image-20260402202345526](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260402202345526.png)

```bash
# Add an ip address on 10.0.0.13
ip address add 10.0.0.110/24 dev ens33 label ens33:110 # add ip address 10.0.0.110 to ens33
iptables -R INPUT 2 ! -s 10.0.0.12 -j REJECT # Modify the second rule to REJECT if it is not from 10.0.0.12
iptables -A INPUT -d 10.0.0.110 -j REJECT # If your direction is 110 then reject

# WHat you will see is ping 10.0.0.13 okay, while ping 10.0.0.110 will be rejected

# Reject based on protocol
iptables -A INPUT -d 10.0.0.110 -p icmp -j REJECT 
```

## iptables Extensible Match

```bash
--tcp-flags SYN,ACK,FIN,RST SYN  #check all those flags, only SYN should be 1, which bescially represents the first handshake = --syn


## Multi Ports
iptables -A INPUT -s 10.0.0.12 -d 10.0.0.110 -p tcp --dport 21:23 REJECT # Reject all the detination ports from 21 to 23, this will block ssh loginI

iptables -A INPUT -s 10.0.0.12 -d 10.0.0.110 -p tcp -m multiport --dports 22,80 -j REJECT # Reject specifically for ports 22 and 80


## Ip ranges
iptables -t filter -A INPUT -m iprange --src-range 10.0.0.13-10.0.0.99 -p tcp --dport 80 -j REJECT # Block all the ip address from 13 to 99

## State
iptables -t filter -A INPUT -m state --state ESTABLISHED -j ACCEPT  # accept the already established connection

```

Best practice

![image-20260403203112541](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260403203112541.png)

## Network Firewall

If you are sending a request, and the packet's source Ip address is changed, it is **SNAT**.

But if the packet's target ip address is changed, it is called **DNAT**



### SNAT

![image-20260404175110887](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260404175110887.png)

```bash
## SNAT


# This is the key here, any packet from 10 but destination is not 10, then we will change its source to 192.168.8.13 
root@ubuntu24-13:~# iptables -t nat -A POSTROUTING -s 10.0.0.0/24 ! -d 10.0.0.0/24 -j SNAT --to-source 192.168.8.13

```

### MASQ

Will automatically change `--to-source`

```bash
iptables -t nat -R POSTROUTING 1 -s 10.0.0.0/24 ! -d 10.0.0.0/24 -j MASQUERADE
```

### DNAT

![image-20260404215403849](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260404215403849.png)

```bash
# If it aims towards me, just go to the other one so that you can get the correct service
iptables -t nat -A PREROUTING -d 10.0.0.13 -p tcp --dport 80 -j DNAT --to-destination 192.168.8.14:80

```

DNAT needs to be created individually, therefore for every service you need to create a separate DNAT rule.

## Customize the chain

```bash
iptables -N web_chain # For table filter, create a chain called web_chain
iptables -E web_chain WEB_CHAIN # Change the name from web_chain to WEB_CHAIN
iptables -A WEB_CHAIN -p tcp -m multiport --dports 80,443 -j ACCEPT # Add a rule of accepting certain ports for WEB_CHAIN
iptables -A INPUT -j WEB_CHAIN # Add the WEB_CHAIN to INPUT chain
```

