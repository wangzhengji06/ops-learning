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
