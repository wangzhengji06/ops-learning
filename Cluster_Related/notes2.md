## rsyslog

`vim /etc/rsyslog.conf`

rsyslog and journald that is provided by systemd will throw the logs into different locations. journald stores binary information, you need to use `journalctl`. 



## journald

Very important way to check the service logs. 

```bash
journalctl # The command for controling journal
journalctl -xeu # u -> unit  e -> start from end  x -> detailed
journalctl -eu nginx -o json -n 1 |  jq # Output to json format, and serialize it
journalctl -eu nginx -o json -n 1 |  jq '.CODE_LINE' # I can directly get the CODE_LINE field's information
```

## Storage

Three types: Block Storage, File System Mounting, Object Storage Service(OSS)

Block Storage: `fdisk, mkfs.ex4, mount, umount`

File system mounting: `mount umount`

OSS: `curl`

## NFS(file system mounting)

NFS relies on RPC service and tcp protocol. It is made of server and clients.

For rocky, the service is `ntf-utils`. For ubuntu, the server is `nfs-kernel-server`, the client is `nfs-common` . The config file is both `/etc/export`

```````````````````````````````````bash
vim /etc/export
``````````````````````````````````
/data/dir1         10.0.0.12
/data/dir2         10.0.0.0/24

``````````````````````````````````
systemctl restart nfs-server.service


# serverside
exportfs -rv

# clientside
showmount <server>
mount 10.0.0.13:/data/dir2 /mnt2
mount 10.0.0.13:/datra/dir1 /mnt1
df -h
```````````````````````````````````

nfs use different policies, a famous one is `root-squash`, means it will map the `root` to `65535 nobody` , and `no-root-squash` will map the root to root, basically it will map the exact same uid.　

## Real-time Synchronization

There are two set of system: `rsync + inofity`, `rsync + sersync`. 

### `rsync + innotify` 

Use `inotify` to monitor the directory, 

```bash
# Rocky
yum install epel-release
yum install inotify-tools

# Start the montioring
inotifywait -mr /data -o xxx.txt  # Montior the directory and output content to xxx.txt
inotifywait -mrd # detached mode
inotifywait [-q] # only concise information

# Montiroing mode
intoifywait -drq /data/ -o intoify.log --timefmt "%Y-%m-%d %H:%M:%S" --format "%T %w %f event: %e"
intoifywait -drq /data/ -o intoify.log --timefmt "%Y-%m-%d %H:%M:%S" --format "%T %w %f event: %e" -e create ## I only monitor the create event


```

Use `rsync` to do data transformation

`rsync` can be done manually, or run as a service

1. it can update local server
2. Remote server use ssh
3. localhost use socket to connect remote host rysnc daemon

Let's look at 3, and assume that 13 is server, and 12 is client.

```bash
#systen
systemctl status rsync.service

# Client
rsync rsync://10.0.0.13 # rsync://user@ip/folder
```

The problem here is, nobody(username!) should have the right to transfer something inside and outside. 

```bash
# Normal file can only have one owner, so we need acl for more complicated file privledge setting
getfacl /folder
setfacl -m u:nobody:rwx /data/dir1 # I give nobody the permission for this folder
```

But this is of course very dangerous, anyone can transfer files to my server with ease.

```bash
#/etc/rsyncd.conf
uid=root
gid=root
max connections=0
log file=/var/log/rsyncd.log
pid file=/var/run/rsyncd.pid
lock file=/var/run/rsyncd.lock
[dir1]
path=/data/dir1/
comment=rsync dir1
read only=no
auth users=rsyncer
secrets file=/etc/rsyncd.pwd
```

There are two things need to worry about for `rsyncd.pwd`. First the privilege, secondly the input format

```bash
echo 'rsyncer:123456' > /etc/rsyncd.pwd
chmod 600 /etc/rsyncd.pwd
systemctl restart rsync.service

# ON client
rsync /etc/hosts rsyncer@10.0.0.13::dir1

# I am so sick of inputting password
echo '123456' > /etc/rsyncd.pwd
chmod 600 /etc/rsyncd.pwd
rsync /etc/hosts rsyncer@10.0.0.13::dir1 --password-file=/etc/rsyncd.pwd

# Command
rsync -avz # Retain the same properpty, verbose, zip and transport 
--delete # Redundant files need to be deleted
```



### `rsync + sersync`

`inotify` requires self-created bash script, while `sersync` is out-of-box solution. 

You need to download using wget.

```bash
# Download the software
wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/sersync/sersync2.5.4_64bit_binary_stable_final.tar.gz

# Back up the config file
cp confxml.xml{,.bak}

# I suppose you will make some changes to the config file in this step

# Use the config file to synchronize
./sersync2 -dro ./confxml.xml
```



**What if I want to monitor multiple folders?**

Answer: you need another config file, and write them as systemd services.

