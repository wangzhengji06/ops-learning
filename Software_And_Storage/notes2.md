## Software Installation

Already local -> offline installation

Rocky: rpm   Ubuntu: dpkg

In public repo/source ->  online installation

Rocky: yum/dnf  Ubuntu: apt


## Command

rpm -i install -v verbose -h bar
    -e uninstall
    -q search


rpm -qi Check the information
    -ql what did you install?
    -qf What software owns this file?
    -ivh install!


yum | dnf
    make cache: get metadata from source
    install: install software
    autoremove: delete software
    list | grep: check software online
    search: search for software

dpkg -i  install
     -L  list the installed file
     -r  remove
     -S  what software owns this file?
     -s  information

apt 
     update: get metdata from source
     install: install software
     purge: delete software
     list | grep and search is the same

## Source
inside /etc/yum.conf
      /etc/yum.conf.d/xxx.repo  -> customized source


inside /etc/apt/sources.list
      /etc/apt/sources.d/xxx.list

Format
```
[ID]
name=
mirrorlist / baseurl=
gpgcheck=0 # SInce you defined the source, you should trust it
```
The important thing here is the url must have repodata under it

```
deb xxx
```

## Software installation check

1. Check Port
2. Check Process
3. Check Service


## Download binary

1. wget
2. tar x[z]f
3. ./configure --option
4. make
5. make install
  


## Service management

systemctl is used to control the service 

/etc/systemd/system

/usr/lib/systemd/system

/lib/systemd/system

[unit] Description

[service] Configuration

[install] Being managed by?


Basically you only need to rememebr exestart and exestop...
