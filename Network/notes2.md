## hostname

Two documents: /etc/hostname   /etc/hosts

hostname: temporary change

hostnamectl set-hostname: eternal change

## Classic vs System Default

net-tools vs iproute

ifconfig vs ip a

route vs ip r

netstat vs ss


## ifconfig

ifconfig ens160 can check the ip address

ifconfig ens224 10.0.0.24/24 to temporarily change the ens224's ip address to 10.0.0.24

ifconfig ens224 0.0.0.0 to clean the ip address related to ens224.

ifconfig ens224 up / down

ifconfig ens224:1 10.0.0.111/24 up this set an alias to a web drive


## route

route add -net 10.0.0.1/24 gw 10.0.0.2

route del -host 10.0.0.1/32


## netstat

netstat -ntulp Check which services are running and what ports are they listening on

netstat -i / netstat -I=ens160

* similar: lsof -i:22


## ip link & ip addr & ip r

ip link 

ip link show ens160

ip link set <> up / down

ip link set <> name xx: can change the web drive's name

ip addr show

ip addr add <IP>/<prefixlen> dev <interface>: Add ip to web drive

ip addr del <IP>/<prefixlen> dev <interface>: Delete ip to web drive

ip route add 10.0.0.0/24 via 10.0.0.2

ip route list


## ss

ss -tnulp

The only difference between ss and netstat is that their display styles are different.


## nmcli

nmcli conn

nmcli conn down / up + <>

nmcli device


## Two most common settings

1. Newly added web drive

* cd web drive folder
* cp the setting
* modify the setting
* systemctl restart NetworkManager


2. Change the web drive setting

* cd web drive folder
* modify the setting
* nmcli con down xxx
* systemctl restart NetworkManager

## Bonding vs Netowrk Teaming

bonding can bond two physical net drives into one logical net drive.

Network teaming relies on NetworkManager software.

## Web Diagnose

fping 10.0.0.12 10.0.0.13 10.0.0.14

fping -f 10.0.0.0/24

tcpdump is used to test whether client and server can communicate normally

tcpdump -nn: to check all the data packages

tcpdump -nn ip host 10.0.0.14 and 10.0.0.13 Only listen to communication between these two hosts

tcpdump -nn ip host 10.0.0.14 and !10.0.0.13 Only listen to comunication between 10.0.0.14 and not 10.0.0.13

traceroute -I/T xxx.com use icmp/tcp to test

tracepath xxx

mtr xxx: traceroute + ping 

## Summary

between hosts: ping / fping

diagnosis: route / ip route / traceroute / tracepath / mtr

service: tcpdump

