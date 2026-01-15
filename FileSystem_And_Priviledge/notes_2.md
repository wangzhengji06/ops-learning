# User, File and Priviledge


## User

root 0

system 1-999

normal 1000+


## SELinux and Firewalld

#Check whether SElinux is on
getenforce

#Stop SElinux temporarily
setenforce 0

#Stop SElinux forever
/etc/selinux/config  SELINUX=disabled

## User-related config
/etc/passwd

/etc/shadow

/etc/group

/etc/gshadow


## User and Group
groupadd -g 

groupdel 

useradd -m

userdel -r

groupmems -g  groupname -a username


## Priviledge

chown zhangsan:zhangsan  (+R)

chmod  755 
