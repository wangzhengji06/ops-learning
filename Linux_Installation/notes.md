# Linux Distribution

CentOS -> RHEL -> Rocky

Deibian -> Ubuntu

# Computer Architecture

CPU: X86_64, ARM...

API, ABI

Kernel: `uname -r` version number + enterprise added patch version

# VMware network
Setup: 10.0.0.0 NAT

Ubuntu: 10.0.0.13 10.0.0.16 10.0.0.19

rocky: 10.0.0.12 10.0.0.15 10.0.0.18

openeuler: 10.0.0.14 10.0.0.17 10.0.0.20

NAT Gateway: 10.0.0.2

NetMask: 24 / 255.255.255.0


# Ubuntu command
init 3 to get rid of DE, init 5 to open gui

`systemctl set-default multi-user.target` / `systemctl set-default graphical.target` to make this default option

Can remove snap for a clean environment

# VMware snapshot and clone

Complete Clone vs Link Clone

Clone: same MAC address, so you better change it

Also need to change ip address


# Rokcy Post-processing for clone

1. hostnamectl set-hostname

2. nmcli con down xxx

3. cd /etc/NetworkManager/system-connections/ use vi to delete the mac and change ip address 

4. systemctl restart NetworkManager


# Rocky install GUI

yum install / yum groupinstall / yum update / yum makecache


# Terminal

`tty` can check whether local or remote

# User

root 0 

System user 1-999
Normal user 1000+

Check:
whoami/who am i/ w/ su / su - (has memory)/ exit

Add:
useradd / useradd -m

Delete:
userdel / userdel -r

Management:
chage

# SHELL
env | grep SHELL

echo $SHELL

cat /etc/shells

echo $PS1

## Check
ls 

tree

pwd

## Create
mkdir -pv

touch

cp -r x/ y/ / is important here

## System
lscpu

free -h

arch

uname -a

ps -aux

pstree -p

history -c



