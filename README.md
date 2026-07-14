# Ops Learning Log

Attempt to learn linux and stuff...


## 1. Linux Introduction and Installation

* How to set up the virual machines (Ubuntu, Rocky, OpenEuler)

* How to manuipulate Ubuntu.

* Basic Computer Architecture

* API，ABI(POSFIX) 

* snapshot / clone

* Ubuntu / Rocky Installation

* Hostname / IPV4 address 

* TTY / User / SHELL / Command

## 2. File System

* stat to check

* inode, softlink, hardlink

* IO redirect

* user, group

* chown, chmod

* cut find xargs sort uniq

* grep sed awk


## 3. Privledge

* File leve privledge: rwx, SUID..., SELINUX, Special Privledge

* User management: groupadd, groupdel etc....


## 4. Hard Disk

* Parition -> Make format -> Mount

* Logical Volume: physical volume -> volume group -> logical volume -> make format -> mount

* du / df / dd / lsblk


## 5. Software

* yum apt 

* rpm dpkg

* software source

* systemd management


## 6. Shell Script

* Execution Order

* If condition

* for while until

* continue break exit

* function

* expect, kill, trap

## 7. Process

* pstree, vmstat, iostat, top, ps

* vmstat, lsof

* at, crontab, bg, fg, jobs, nohup, &

* state: y, m insmod, lsmod, modinfo, modprobe, rmmod

* sysctl -a -w -p

* echo > proc/sys/name1/name2/parameter


## 8. Network

* osi, tcp/ip

* tcp port udp 

* ipv4 ipv6 arp icmp

* MAC table

* hostname / hostnamectl

* ifconfig / ip addr

* ifconfig / ip link

* route / ip route

* nmcli 

* bounding / teaming

* ping / fping / route -n / ip route list / traceroute / tracepath / mtr

* tcpdump

## 9.Linux server startup process

* The Linux boot process starts with hardware initialization (BIOS/UEFI), hands over control to the bootloader (GRUB) to load the kernel (vmlinuz), which then spawns the first process (systemd) to manage services and finally runs custom scripts (rc.local) before the login shell.


## 10. DNS(Network)

* Domain -> IP address
* ipv4 root  13 group    ipv6 root 25 group
* Setup dns server: 1. Change zone file 2. Change record file 3. Check grammar 4. load the configureation 5. dig
* DNS location assignment: /etc/resolv.conf /etc/hosts
* resolvctl statistics
* master-slave dns, reverse dns lookup
* cdn

## 11. Network Security

* asymmetric, symmetric, hashing

* CA certificate [establish, self-issue, server request, CA issue certificate ]

* SSL/TLS + HTTP = HTTPS

* ssh root@10.0.0.3 command

* scp [-r] source target

* ssh without password

* chrony service

* iptables for firewall management

* iptables on ip forwarding

* rsyslog, journaltctl -xeu

* NFS: exportfs, showmount

* rsync: inotify or  sersync

  

## 12.  HTTP and Nginx

* httpd protocol
* apache, tomcat, nginx
* io model
* Web, Reverse Proxy, cache, email....
* installation binary/source
* master worker / slave worker
* 3rd party module: donwload module, get same version source code, configure-make, change binary
* Settings: overall - events - http - server - location
* virtual hosting: port  server name  default_server
* nginx log, status, redirect,
* nginx reversed proxy: proxy_pass, proxy_set_header, level 4 rewrite
* nginx load balancing

## 13. Tomcat

* Config: server / service / connector / engine / host[based on server name]
* Nginx reverse proxy for tomcat, while tomcat host something like jpress
* port: 8080(to outside), 8005(management), 8009
* Session: bond & share
* JVM basic:  java -> bytes -> class loader -> jvm
* heap: young(eden from to) / old
* jvm parameters: -Xms -Xmx ratio...
* maven: java packaging tool : mvn clean package
* maven config: /etc/maven/setting.xml / pom.xml
* nexus: local repo solution
* Create self-owned repo using nexus

 
## 14. MySQL

* database structure
* Relational database
* database, table, index, user, function, process, view
* Use binary / source code for single/mutiple instances for mysql/mariadb
* Final Environment: Client / Server
* SQL statement: DDL, DCL, DML, DQL, TCL
* SQL statement format: SELECT / WHERE / FROM / ;\G / 
* show, create, drop, alter
* SQL statement
* User Management
* Engine, Index
* transaction: ACID, repetable read, read uncommited, serialization, dirty read....
* log: transaction log -> undo / rdo. Error log. General log. Binary log.
* Backup and Recovery: Physical backup / incremental backup etc...
* source and replica: how to deploy, master needs an account, and sqldump the data
* replica needs import data, modify the command, start the replica
* logic: client changes master, records to binlog
* replica starts io thread to connect to master
* master starts the dump thread and goes to binlog pos
* replica records the data into relay log
* replica starts sql thread and read commands from relay log and hand it to mysql to execute
* finally master and slave have the same contents
* cluster: binlog + manual / GTID
* middleware: backend and mysql bridge
* high availability: MGR clusters



## 15. LVS + Keepalived
* Mode: NAT, DR, TUN
* NAT needs to go through LVS to respond, so it is relatively slow
* DR changes MAC address RS needs vip
* TUN use ip-in-ip protocol. RS needs vip and the ability to handle nonstandard packet.
* ipvsadm command. service use Uppercase Letter, server use Lowercase Letter.
* lvs (rs needs vip and arp config for dr mode)
* clustering basis: time synchornization / vrrp protocol
* keepalived: high availability + high extension
* can be used alone or add other high extension software(nginx, HAProxy)
* keepalived: global / high avalability / high extension
* virtual server: real_server many ways to check heartbeat: http, tcp
* keepalived basically allows you to check vrrp's other lvs server state using customized way also
* keepalived itself vs keepalived + nginx 
* vrrp scripts + track scripts


## 16.Ansible
* automatic devops tool[I
* python + yaml + jinja2
* config: hosts file + ssh no-password login
* ansible modules: shell, command, scripts
* copy, fetch, file, unarchive, archive, get_url module
* apt, yum, service module

