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

special privledge in the position of x, this is also the first position of umask

### SUID (4)
SUID -> gives the privledge of the file owner.
For example, /usr/bin/passwd has SUID priviledge. Anyone execute it has root privledge.
/etc/shadow is 000, so only root can modify it. 
But normal user manipulate /usr/bin/passwd still has root privledge to modify that file.


### SGID (2)
When a folder has SGID, whoever created a file under this folder, the group always belongs to the group of the folder.


### Stickbits (1)
When a folder has Sticky bit, only the creator can delete the file under this folder.


### Change these privledges

chmod xxxx file
if you want to remove a privledge for it, use chmod 0xxx file

### Special Privledge
lsattr / chattr +-

* a: only append
* i: not modifiable

Use: chattr +i /etc/shadow -> No one except me can add new user to the system


### In summary
- First Layer:
  creator/group/others rwx
- Second Layer:
  SUID / SGID / Stickbits
- Third Layer:
  SELinux (can be disabled by root)
- Forth Layer:
  special attribute -> (can limit root)

## File Management
  - cut -d [seperator] -f [begin:end] -> get the begin to end column, the seperator is seperator
  Usage: ip a | grep global | cut -d 't' -f 2 | cut -d ' ' -f 2 to get the ip address
  - sort : sort by
  - uniq -c : remove the continuous duplicated contenxt with the count
  - locate: offline search, quick, use together with updatedb
  - find: oneline search, strong functionality
  - xargs: pass the text to arguments 

## Unzip
  - gz
  - tar
  - zip

    tar
      tar zcf {name} {files}
      tar tvf {file}
      tar xf {file} -C {directrory}
    unzip
      
  


## Text manipulation command and package command
  - grep: n i r A B C v E  
 
  - sed:  a i d s p
        -n: silent
        -i: interative
  - awk 'range {print $column}' file
        NF stands for last column, 0 stands for whole columns
        for example: awk '{print $1""$3}' sed.txt
                     awk '/keyword/{print $1}' sed.txt print the row that contains keyword's first column

        -F seperator
        -v FS=sepeartor
        BEGIN{} {} END{}
  The awk's range is essentially trying to check true/false, or /keyword/.

## REGEX

THere are shell expansions: {} [] ? *

And there are regex that can be used by sed, grep, awk

 - .   One character
 - []  Any one character inside range
 - [^] not inside the range
 - |   Or logic
 - ^ / $ begin / end
 - /< /> word matching
 - () group

