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

## vrrp config

This is the vip part, the so-called high availability

```config

vrrp_instance VI_1 {
    state MASTER
    interface ens37
    virtual_router_id 50
    priority 100
    unicast_src_ip 192.168.8.13
    unicast_peer {
        192.168.8.16
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.0.0.100 dev ens33 label ens33:1
    }
}
```

This is the virutal server part, the so-called overload balancing part

```config
virtual_server 10.0.0.100 80 {
    delay_loop 2
    lb_algo rr
    lb_kind DR
    protocol TCP
    real_server 10.0.0.14 80 {
    }
    real_server 10.0.0.17 80 {
    }
}
```

Under the DR mode, director gets vip's request, and change MAC and transfer to RS, RS's target ip address is still vip, RS needs to set VIP on Io, but RS cannot respond to VIP's ARP request. Therefore, we need to set `arp_ignore / arp_announce` 


For real server, the nginx or the http service might already be dead.


```config
real_server 10.0.0.14 80 {
    HTTP_GET {
        url {
            path /
            status_code 200
        }
        connect_timeout 3
        nb_get_retry 3
        delay_before_retry 3
    }
}
```



*you can set up sorry server when all real servers related are dead.*

*you can also use wrr to assign weights to real server.*


## vrrp heartbeat check

Laster section is talking about the detection of real server. But we need to check master node also. Keepalived is running, but other service is dead or not is unknown.

1. Write checkup script
2. Keepalived config modified to use that script
3. Effect check


```config
vrrp_script chk_keepalived {
    script "检测命令或脚本路径"
    interval 1
    weight -30
    fall 3
    rise 2
    timeout 2
}

vrrp_instance VI_1 {
    ...
    track_script {
        chk_keepalived
    }
}
```

Here master will send to slave its weight, after the failure, master weight will reset to 70, thus triggering the mechanism ot changing the master. 


## nginx + keepalived

Use only keepalived, not lvs, and control the two nginx servers.

```bash
#!/bin/bash

nginx_process_count=$(ps -C nginx --no-header | wc -l)

if [ "$nginx_process_count" -eq 0 ]; then
    systemctl start nginx

    nginx_process_count=$(ps -C nginx --no-header | wc -l)

    if [ "$nginx_process_count" -eq 0 ]; then
        systemctl stop keepalived
    fi
fi
```

This is used for vrrp checking. Again, here we need to know whether nginx is really alive or not.

Here the idea is, the keepalived will try to restart the nginx, and if it really cannot be fixed, it will stop keepalived to let the master status be taken. 


## HAProxy

Basically completely the same as nginx, the difference here is nginx listens to 0.0.0.0:80, but haproxy can bind to unexisted vip address. When it is a slave, it can still bind to that address. But nginx can also do that. Maybe the point of HAProxy is to provide a good ui.  

