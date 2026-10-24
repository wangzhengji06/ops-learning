## The deploy order
The most fundemental one should be deployed first. 

Those who can be tested first should also be deployed first.

## Overview
1. First layer: LVS + Keepalived
2. Second layer: Nginx
3. Application layer: JPress + Tomcat, WordPress, Discuz!
4. Cache Layer: Reis
5. Storage layer: MySQL + NFS + rsync
6. Tool layer: Zabbix + DNS + Shell + Ansible

## Distribution
hostname: lzabry.com
SubDomain: web. admin. service. montior.

* Storage layer: 10.0.0.1 - 10.0.0.29
* Cache layer: 10.0.0.30 - 10.0.0.50
* Application layer: 10.0.0.51 - 10.0.0.100
* Reverse Proxy layer: 10.0.0.101 - 10.0.0.120
* Connection layer: 10.0.0.121 - 10.0.0.140
* Operation layer: 10.0.0.141 - 10.0.0.160
* Reserved address: 10.0.0.161 - 10.0.0.254

## DNS servers
We make 10.0.0.147 and 10.0.0.148 as the dns servers.

```conf
apt install bind9
vim /etc/bind/named.conf.options

forwarders {
    8.8.8.8;
    114.114.114.114;
};

dnssec-validation auto;

listen-on-v6 { any; };

recursion yes;
allow-recursion { 10.0.0.0/24; };

allow-transfer { 10.0.0.148; };

listen-on { 10.0.0.147; };
```

Here it means that, hey this dns is bind to local ipv4 address 10.0.0.147. Also, it allows the recursion from 10.0.0.0 - 10.0.0.254, and also it allow the zone transfer from 10.0.0.148, as 10.0.0.148 will be the dns slave. 


Then we should define the local zone, which means that hey if you have any information about the zone, dont forward, ask me


```conf
vim /etc/bind/named.conf.local

zone "magedu.com" {
    type master;
    file "/etc/bind/zones/db.magedu.com";
    notify yes;
    also-notify { 10.0.0.148; };
};

```


This will say, hey 148 if I got updated I will notify you, you can also transfer from me.


Use this for the /etc/bind/zones/db.magedu.com

```conf
$ORIGIN magedu.com.

; ============================================================
; Default TTL: 1800 seconds
; ============================================================
$TTL 1800

; ============================================================
; SOA record
; ============================================================
@       IN      SOA     dns1.magedu.com. admin.magedu.com. (
                        2025101901  ; Serial - increment after every change
                        3600        ; Refresh: secondary checks every 1 hour
                        1800        ; Retry: retry after 30 minutes
                        604800      ; Expire: 7 days
                        86400       ; Negative caching TTL: 1 day
                        )

; ============================================================
; Authoritative DNS servers
; ============================================================
@       IN      NS      dns1.magedu.com.
@       IN      NS      dns2.magedu.com.

; DNS server addresses
dns1            IN      A       10.0.0.147
dns2            IN      A       10.0.0.148


; ============================================================
; Storage layer
; TTL: 1800
; ============================================================
$TTL 1800

mysql-master    IN      A       10.0.0.10
mysql-slave1    IN      A       10.0.0.11
mysql-slave2    IN      A       10.0.0.12

nfs-storage     IN      A       10.0.0.200
nfs1            IN      A       10.0.0.20
nfs2            IN      A       10.0.0.21
backup          IN      A       10.0.0.22


; ============================================================
; Cache layer
; TTL: 1800
; ============================================================
redis-master    IN      A       10.0.0.30
redis-slave     IN      A       10.0.0.31


; ============================================================
; Application layer
; TTL: 300 seconds
; ============================================================
$TTL 300

mycat           IN      A       10.0.0.201
mycat1          IN      A       10.0.0.50
mycat2          IN      A       10.0.0.51

jpress-1        IN      A       10.0.0.60
jpress-2        IN      A       10.0.0.61

wordpress       IN      A       10.0.0.70
discuz          IN      A       10.0.0.71


; ============================================================
; Reverse proxy layer
; TTL: 300 seconds
; ============================================================
nginx-1         IN      A       10.0.0.105
nginx-2         IN      A       10.0.0.106


; ============================================================
; LVS / access layer
; ============================================================

; Physical LVS nodes - TTL: 1800
$TTL 1800

lvs-master      IN      A       10.0.0.125
lvs-backup      IN      A       10.0.0.126

; Keepalived VIP - TTL: 300
$TTL 300

vip             IN      A       10.0.0.130


; ============================================================
; Operations / monitoring layer
; ============================================================

; Zabbix - TTL: 3600
$TTL 3600

zabbix-master   IN      A       10.0.0.145
zabbix-slave    IN      A       10.0.0.146

; Ansible - TTL: 1800
$TTL 1800

ansible         IN      A       10.0.0.149


; ============================================================
; Public/business entry
; TTL: 300
; ============================================================
$TTL 300

www             IN      CNAME   vip.magedu.com.
```


```bash
named-checkzone magedu.com /etc/bind/zones/db.magedu.com
systemctl enable named
```

Now you can do it for the slave dns server

Install the bind again, and configure the named.conf.options as follows

```conf
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-recursion { 10.0.0.0/24; };
    dnssec-validation auto;
    listen-on { 10.0.0.148; };
};
```

Notice here that there is no such thing as forwarded, then it would mean that, suppose a client want to visit google.com, the 147 would forward it, while 148 would ask its root dns server, and recursivly gave answer. 

```conf
vim /etc/bind/named.conf.local

zone "magedu.com" {
    type slave;
    file "/var/cache/bind/db.magedu.com";
    masters { 10.0.0.147; };
};
```


We can test both dns servers using

```bash
dig www.magedu.com @10.0.0.148
```


How do we make other nodes to use the dns we just created?

We can add name servers 

```conf
nameservrs:
addresses:
- 10.0.0.147
- 10.0.0.148
- 10.0.0.2
```


## Mysql server

10.0.0.10 / 10.0.0.11 / 10.0.0.12

What we want is to make 10.0.0.10 as master

```bash
hostnamectl set-hostname mysql-master
hostnamectl set-hostname mysql-slave1
hostnamectl set-hostname mysql-slave2
```


We also need to install mysql-server for all three nodes

```bash
apt install mysql-server -y
```

We need to manage our own binlog.

```bash
mkdir -p /data/mysql/logbin

chown -R mysql:mysql /data/mysql/
```

Besides that, what we need to modify `/etc/mysql/mysql.conf.d/mysqld/cnf` for the config nformation.

```conf
[mysqld]

user = mysql

bind-address = 10.0.0.10
mysqlx-bind-address = 10.0.0.10

key_buffer_size = 16M
myisam-recover-options = BACKUP

log_error = /var/log/mysql/error.log

server-id = 10

log_bin = /data/mysql/logbin/mysql-bin

default_authentication_plugin = mysql_native_password

max_binlog_size = 100M

# binlog-ignore-db = mysql

```


Now here comes teh disgusting things, ubuntu use some security check so you have to do the following


```bash
vim /etc/apparmor.d/usr.sbin.mysqld
/data/mysql/logbin/ r,
/data/mysql/logbin/** rw,

apparmor_parser -r /etc/apparmor.d/usr.sbin.mysqld
systemctl restart apparmor
```

### Now you need to set up account
Use mysql to go in.

The following commands allow the 
```sql
CREATE USER 'repl'@'10.0.0.%' IDENTIFIED BY 'ReplPass123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.0.0.%';
FLUSH PRIVILEGES;
reset master;
show master status;
```


Now we have a pretty clean mater database. We can replicate to source.

```bash
mysqldump -A -F --source-data=2 --single-transaction > all.sql
```

Remeber that this sql is supposed to be run on slave node, so modify and add this

```sql
MASTER_HOST='10.0.0.10',
MASTER_USER='repl',
MASTER_PASSWORD='ReplPass123!', 
```



### Now let's build the environment on the mysql slave


Now we need to configure sql-slave

```config
[mysqld]
user            = mysql
bind-address            = 10.0.0.11 
mysqlx-bind-address     = 10.0.0.11 
key_buffer_size         = 16M
myisam-recover-options  = BACKUP
log_error = /var/log/mysql/error.log
server-id               = 11 
relay-log = mysql-relay-bin 
read-only = 1 
default_authentication_plugin=mysql_native_password 
```

```config
[mysqld]
user            = mysql
bind-address            = 10.0.0.12
mysqlx-bind-address     = 10.0.0.12 
key_buffer_size         = 16M
myisam-recover-options  = BACKUP
log_error = /var/log/mysql/error.log
server-id               = 12
relay-log = mysql-relay-bin 
read-only = 1 
default_authentication_plugin=mysql_native_password 
 ```

The slave's method is kind of the same

```sql
source /root/all.sql
start slave;
show slave status\G
```

You have both salves running from now.

Now the assumption here is we have added a student table inside test database. 

### Backup for mysql 

It is highly recommended that one slave starts binlog, this can help recovering when something happens. 

```bash
mkdir -pv /data/mysql/logbin
chown -R mysql:mysql /data/mysql/

vim /etc/mysql/mysql.conf.d/mysqld.cnf
log_bin = /data/mysql/logbin/mysql-bin
log_bin_index = /data/mysql/logbin/mysql-bin.index
expire_logs_days = 7
max_binlog_size = 1G
```

Remember to configure the apparmour, and restart apparmour and mysql.

Then do the following on **master** config

```sql
CREATE USER 'backup_user'@'10.0.0.12' IDENTIFIED BY 'BackupPass123!';
GRANT SELECT, SHOW VIEW, RELOAD, LOCK TABLES, REPLICATION CLIENT, PROCESS ON *.* TO 'backup_user'@'10.0.0.12';
```


Now we add the latter 4 lines on slave 2 /etc/mysql/mysql.cnf

```conf
[mysqldump]
user=backup_user
password=BackupPass123!
host=10.0.0.12
```


Now you can use mysqldump freely on slave2. 


## NFS backup

We have two nfs, 10.0.0.20 as nfs1 and 10.0.0.21 as nfs2.

These commands are done on both servers

```bash
apt install nfs-server -y
mkdir -pv /data/{jpress,wordpress,discuz}
vim /etc/exports
/data/discuz 10.0.0.0/24(rw,sync,no_root_squash)
/data/wordpress 10.0.0.0/24(rw,sync,no_root_squash)
/data/jpress 10.0.0.0/24(rw,sync,no_root_squash)

systemctl restart nfs-server
showmount -e
```

We want to do them both way, so both way synchronization.

We will use `sersync` to monitor, and `rsync` to do the data transport. 

First we need to enable the ssh without password

```bash
ssh-keygen

ssh-copy-id 10.0.0.20/21
```


Now we need to install the software
```bash
apt install -y rsync inotify-tools
```

And we can manually update on 20 to transport the data to 21.

```bash
rsync -avz --delete \
  /data/jpress \
  /data/wordpress \
  /data/discuz \
  root@10.0.0.21:/data
```

THe we write the config
`vim /etc/rsync.conf`

```conf
# /etc/rsyncd.conf

# Global settings
uid = root
gid = root
use chroot = no

max connections = 10

pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
log file = /var/log/rsyncd.log

ignore errors = yes
read only = no
list = no


# JPress data synchronization
[jpress]
path = /data/jpress
comment = JPress data sync
hosts allow = 10.0.0.20/32 10.0.0.21/32
auth users = rsync_user
secrets file = /etc/rsyncd.secrets


# WordPress data synchronization
[wordpress]
path = /data/wordpress
comment = WordPress data sync
hosts allow = 10.0.0.20/32 10.0.0.21/32
auth users = rsync_user
secrets file = /etc/rsyncd.secrets


# Discuz data synchronization
[discuz]
path = /data/discuz
comment = Discuz data sync
hosts allow = 10.0.0.20/32 10.0.0.21/32
auth users = rsync_user
secrets file = /etc/rsyncd.secrets
```

And add the password and privilege

```bash
echo "rsync_user:Rsync123" > /etc/rsyncd.secrets
chmod 600 /etc/rsyncd.secrets
```

Now do the same thing on nfs2, but first update

```bash
sync -avz --delete \
  /data/jpress \
  /data/wordpress \
  /data/discuz \
  root@10.0.0.20:/data/
```


And we are done with the rsync part.

### add montoring

Prepare for download.

```bash
mkdir /data/softs -p ; cd /data/softs
wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/sersync/sersync2.5.4_64bit_binary_stable_final.tar.gz
mkdir /data/server
tar xf sersync2.5.4_64bit_binary_stable_final.tar.gz -C /data/server/
mv /data/server/GNU-Linux-x86 /data/server/sersync
mkdir /data/server/sersync/{bin,conf}
mv /data/server/sersync/confxml.xml /data/server/sersync/conf/
mv /data/server/sersync/sersync2 /data/server/sersync/bin/
```


Now install
```bash
apt install -y libxml2-utils

vim /data/server/sersync/conf/confxml.xml
```

```conf
<?xml version="1.0" encoding="UTF-8"?>

<head version="2.5">

    <host hostip="localhost" port="8008"></host>

    <debug start="false"/>

    <fileSystem xfs="false"/>

    <filter start="false">
        <exclude expression="(.*)\.svn"></exclude>
        <exclude expression="(.*)\.gz"></exclude>
    </filter>

    <inotify>
        <delete start="true"/>
        <createFolder start="true"/>
        <createFile start="true"/>
        <closeWrite start="true"/>
        <moveFrom start="true"/>
        <moveTo start="true"/>
        <attrib start="true"/>
        <modify start="true"/>
    </inotify>

    <sersync>

        <localpath watch="/data/jpress">
            <remote ip="10.0.0.21" name="jpress"/>
        </localpath>

        <rsync>
            <commonParams params="-artuz"/>

            <auth
                start="true"
                users="rsync_user"
                passwordfile="/etc/rsyncd.pwd"
            />

            <userDefinedPort start="false" port="873"/>

            <timeout start="true" time="100"/>

            <ssh start="false"/>
        </rsync>

        <failLog
            path="/var/log/sersync_fail.log"
            timeToExecute="60"
        />

        <crontab start="false" schedule="600">
            <crontabfilter start="false">
                <exclude expression="*.php"></exclude>
                <exclude expression="info/*"></exclude>
            </crontabfilter>
        </crontab>

        <plugin start="false" name="command"/>

    </sersync>

</head>
```



### Create service

```bash
cd /data/server/sersync/conf/

mv confxml.xml jpress_conf.xml

vim /etc/systemd/system/sersync_jpress.service 
```


```conf
[Unit]
Description=Sersync service for /data/jpress directory sync
After=network.target rsync.service

[Service]
Type=forking
ExecStart=/data/server/sersync/bin/sersync2 -d -r -o /data/server/sersync/conf/jpress_conf.xml
ExecStop=pkill -f sersync2
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```


```bash
systemctl daemon-reload
systemctl start sersync_jpress.service
systemctl is-active sersync_jpress.service
```

Now we just folow the same flow isnt it?

```bash
cd /data/server/sersync/conf/
cp jpress_conf.xml wordpress_conf.xml
cp jpress_conf.xml discuz_conf.xml
sed -i 's/jpress/wordpress/g' wordpress_conf.xml
sed -i 's/jpress/discuz/g' discuz_conf.xml
cd /etc/systemd/system/
cp sersync_jpress.service sersync_wordpress.service
cp sersync_jpress.service sersync_discuz.service
sed -i 's/jpress/wordpress/g' sersync_wordpress.service
sed -i 's/jpress/discuz/g' sersync_discuz.service
```


And then you reload the daemon, and enable the service and start the service.

Then you do everything again on another server. 


## Backup Server

First install the software

```bash
apt install -y rsync inotify-tools
```

Then setup passwordless ssh with the sql-slave and nfs2. 

Then we preapre the file structure

```bash
mkdir /data/nfs/{wordpress,discuz,jpress} -p


cat > /etc/rsyncd.conf <<'EOF'
# Global settings
uid = root
gid = root
use chroot = no
max connections = 10

pid file = /run/rsyncd.pid
lock file = /run/rsyncd.lock
log file = /var/log/rsyncd.log

ignore errors = yes
read only = no
list = no

# JPress data
[jpress]
path = /data/nfs/jpress
comment = JPress data synchronization
hosts allow = 10.0.0.21/32 10.0.0.22/32
auth users = rsync_user
secrets file = /etc/rsyncd.secrets

# WordPress data
[wordpress]
path = /data/nfs/wordpress
comment = WordPress data synchronization
hosts allow = 10.0.0.21/32 10.0.0.22/32
auth users = rsync_user
secrets file = /etc/rsyncd.secrets

# Discuz data
[discuz]
path = /data/nfs/discuz
comment = Discuz data synchronization
hosts allow = 10.0.0.21/32 10.0.0.22/32
auth users = rsync_user
secrets file = /etc/rsyncd.secrets
EOF

```


Now we can set up the rsync password


```bash
echo "rsync_user:Rsync123" > /etc/rsyncd.secrets
chmod 600 /etc/rsyncd.secrets

systemctl enable rsync.service
systemctl restart rsync.service
```


Notice that you should mdofiy /etc/rsyncd.conf on server 22. Also the montiroing rservnc?

Also, if you want to backup sql, use this
```bash
00 5 * * 1 rsync -avz root@10.0.0.12:/data/backup/mysql_backups/full/*
/data/mysql/full/
30 */2 * * * rsync -avz root@10.0.0.12:/data/backup/mysql_backups/incremental/*
/data/mysql/incrementa/l
```


## Mycat and Redis 

We use 10.0.0.30


```bash
apt install redis-server
vim /etc/redis/redis.conf # You need to change the bind
```


```bash
apt install -y openjdk-8-jdk mysql-client
```


### Install Mycat

```bash
mkdir -p /data/softs
mkdir -p /data/server
cd /data/softs

wget <MyCat tarball>

tar xf Mycat-server-*.tar.gz -C /data/server/
```

### Prepare a MySQL account for Mycat

```bash
CREATE USER 'mycat'@'10.0.0.%'
IDENTIFIED BY 'MyCat@123';

GRANT ALL PRIVILEGES ON *.*
TO 'mycat'@'10.0.0.%';

FLUSH PRIVILEGES;

CREATE DATABASE jpress;
CREATE DATABASE wordpress;
CREATE DATABASE discuz;
```


### Configure schema

```xml
<schema name="jpress"
        dataNode="dn_jpress"/>

<schema name="wordpress"
        dataNode="dn_wordpress"/>

<schema name="discuz"
        dataNode="dn_discuz"/>

```

```xml
<dataNode name="dn_jpress"
          dataHost="mysql_host"
          database="jpress"/>

<dataNode name="dn_wordpress"
          dataHost="mysql_host"
          database="wordpress"/>

<dataNode name="dn_discuz"
          dataHost="mysql_host"
          database="discuz"/>
```


```xml
<dataHost name="mysql_host"
          balance="1"
          writeType="0"
          dbType="mysql"
          dbDriver="native"
          switchType="1">

    <heartbeat>select 1</heartbeat>

    <writeHost
        host="master"
        url="10.0.0.10:3306"
        user="mycat"
        password="MyCat@123">

        <readHost
            host="slave1"
            url="10.0.0.11:3306"
            user="mycat"
            password="MyCat@123"/>

        <readHost
            host="slave2"
            url="10.0.0.12:3306"
            user="mycat"
            password="MyCat@123"/>

    </writeHost>
</dataHost>
```

WHat it does here is to map database into physical database. 

THen it sepeartes read vs write.


### Configure server.xml

```xml
<user name="appuser">
    <property name="password">123456</property>
    <property name="schemas">
        jpress,wordpress,discuz
    </property>
    <property name="defaultSchema">
        jpress
    </property>
</user>

```


DOnt get confused here, this is the account user will use. Mysql actually interact with Mysql using mycat account we defined above.

### Systemd

```conf
[Unit]
Description=MyCat Server
After=network.target

[Service]
WorkingDirectory=/data/server/mycat
ExecStart=/data/server/mycat/bin/mycat start
ExecStop=/data/server/mycat/bin/mycat stop
ExecReload=/data/server/mycat/bin/mycat restart
Restart=on-failure
Type=forking

[Install]
WantedBy=multi-user.target
```


Make it a systemd.

By the way you can just scp ewverything to the second server.


## Application


Deploy tomcat 


```bash
#!/bin/bash

# *************************************
# * Function: Deploy Tomcat 9
# * Author: Wang Shusen
# * Contact: wangshusen@sswang.com
# * Version: 2025-10-19
# *************************************

# Install Java
java_install() {
    apt install -y openjdk-11-jdk
}

# Configure Java environment variables
java_config() {
    cat > /etc/profile.d/java.sh <<'EOF'
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export JAVA_BIN=$JAVA_HOME/bin
export PATH=$JAVA_BIN:$PATH
EOF

    source /etc/profile.d/java.sh
}

# Download Tomcat
get_tomcat() {
    [ -d /data/softs ] || mkdir -p /data/softs

    if [ ! -f /data/softs/apache-tomcat-9.0.97.tar.gz ]; then
        cd /data/softs || exit 1
        wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.97/bin/apache-tomcat-9.0.97.tar.gz
    fi
}

# Extract Tomcat
untar_tomcat() {
    if [ -d /data/server/tomcat ]; then
        rm -rf /data/server/tomcat
    else
        mkdir -p /data/server
    fi

    tar xf /data/softs/apache-tomcat-9.0.97.tar.gz -C /data/server/

    ln -sv /data/server/apache-tomcat-9.0.97 /data/server/tomcat

    echo "Tomcat JSP page from $(hostname)<br />" \
        > /data/server/tomcat/webapps/ROOT/test.jsp

    echo 'SessionID = <span style="color:blue"><%= session.getId() %></span>' \
        >> /data/server/tomcat/webapps/ROOT/test.jsp
}

# Configure Tomcat environment variables
tomcat_config() {
    cat > /etc/profile.d/tomcat.sh <<'EOF'
export CATALINA_BASE=/data/server/tomcat
export CATALINA_HOME=/data/server/tomcat
export CATALINA_TMPDIR=$CATALINA_HOME/temp
export CLASSPATH=$CATALINA_HOME/bin/bootstrap.jar:$CATALINA_HOME/bin/tomcat-juli.jar
export PATH=$CATALINA_HOME/bin:$PATH
EOF

    source /etc/profile.d/tomcat.sh
}

# Create Tomcat user
tomcat_user() {
    userdel -r tomcat >/dev/null 2>&1 || true

    useradd -u 666 -r -s /sbin/nologin tomcat

    chown -R tomcat:tomcat /data/server/apache-tomcat-9.0.97
}

# Create systemd service
tomcat_service() {
    cat > /etc/systemd/system/tomcat.service <<'EOF'
[Unit]
Description=Apache Tomcat 9
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
Environment=CATALINA_HOME=/data/server/tomcat
Environment=CATALINA_BASE=/data/server/tomcat

ExecStart=/data/server/tomcat/bin/startup.sh
ExecStop=/data/server/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload
    systemctl enable --now tomcat
}

main() {
    java_install
    java_config
    get_tomcat
    untar_tomcat
    tomcat_config
    tomcat_user
    tomcat_service
}

main

```

Now we need to configure reddison

```yaml
singleServerConfig:
  idleConnectionTimeout: 10000
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  password: null
  subscriptionsPerConnection: 5
  clientName: null
  address: "redis://10.0.0.30:6379"
  subscriptionConnectionMinimumIdleSize: 1
  subscriptionConnectionPoolSize: 50
  connectionMinimumIdleSize: 24
  connectionPoolSize: 64
  database: 0
  dnsMonitoringInterval: 5000

threads: 16
nettyThreads: 32
transportMode: "NIO"

```



And the context.xml
```xml
<Context>
    <!-- Configure Redisson as the Tomcat session manager -->
    <Manager className="org.redisson.tomcat.RedissonSessionManager"
             configPath="${catalina.base}/conf/redisson.json"
             readMode="REDIS"
             updateMode="AFTER_REQUEST"
             broadcastSessionEvents="false"
             keyPrefix="tomcat:sess:" />
</Context>
```


### Connect jpress to mysql and nfs server
On nfs server, you8 want to make sure that every write to the folder is under tomcat and be managed by tomcat. I mean every remote user.


```bash
useradd -r -u 666 tomcat
chown -R tomcat:tomcat /data/jpress
chmod 755 /data/jpress/

cat /etc/exports
/data/discuz 10.0.0.0/24(rw,sync,no_root_squash)
/data/wordpress 10.0.0.0/24(rw,sync,no_root_squash)
/data/jpress 10.0.0.0/24(rw,sync,all_squash,anonuid=666,anongid=666)

exportfs -rv
systemctl restart nfs-kernel-server
```


Now on tomcat, be prepared for the folder
```bash
systemctl stop tomcat
apt install nfs-common -y
mkdir /data/temp -p
tar xf jpress_tomcat.tar.gz
mkdir ROOT
unzip jpress/starter-tomcat-5.0.war -d ROOT/
chown tomcat:tomcat -R ROOT
cp -a ROOT /data/server/tomcat/webapps/
vim /data/server/tomcat/conf/server.xml
```

```xml
    <Engine name="Catalina" defaultHost="jpress.magedu.com">
    ...
      <Host name="jpress.magedu.com"  appBase="webapps"    
            unpackWARs="true" autoDeploy="true">
            ...
            <Valve className="org.apache.catalina.valves.AccessLogValve"
directory="logs"
                 prefix="jpress_access_log" suffix=".txt"
                 pattern="%h %l %u %t &quot;%r&quot; %s %b" />				 
```


Now restart tomcat service
```bash
systemctl restart tomcat
```


About sql database connection, it is more straihgtforward, it has a front end.


```sql
root@mysql-master:~# mysql
mysql> drop database jpress;
Query OK, 1 row affected (0.01 sec)
mysql> create database jpress;
Query OK, 1 row affected (0.00 sec)
mysql> CREATE USER 'jpresser'@'10.0.0.%';
Query OK, 0 rows affected (0.01 sec)
mysql> ALTER USER 'jpresser'@'10.0.0.%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.00 sec)
mysql> grant all on jpress.* to jpresser@'10.0.0.%';
Query OK, 0 rows affected (0.00 sec)
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
mysql> exit;
Bye
```


And then feel free to connect to 10.0.0.60:8080


### Connect Wordpress to Mycat

WordPress runs on `10.0.0.70`.

The application connects to **MyCat**, and MyCat connects to the MySQL cluster.

```text
WordPress (10.0.0.70)
        |
        v
MyCat (10.0.0.50)
        |
        v
MySQL master / slaves
```

#### Prepare MySQL for WordPress through MyCat

WordPress installation through MyCat can hit a SQL mode compatibility problem.

The actual SQL is finally executed on the MySQL master, so change the SQL mode on `mysql-master`.

```bash
echo 'sql_mode = ONLY_FULL_GROUP_BY,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION' >> /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl restart mysql
```

The `wordpress` logical database has already been defined in MyCat.

WordPress itself connects to MyCat using the MyCat application account:

```text
Database host: 10.0.0.50
Database name: wordpress
Database user: appuser
Password:      123456
```

---

#### Prepare the WordPress host

Use `10.0.0.70`.

```bash
hostnamectl set-hostname wordpress
apt update

apt install apache2 php php-fpm php-cli php-mysqlnd php-json \
php-gd php-xml php-mbstring php-zip php-curl -y
```

Check PHP-FPM:

```bash
grep '^listen' /etc/php/8.3/fpm/pool.d/www.conf
systemctl is-active php8.3-fpm.service
id www-data
```

We use `www-data` for WordPress.

Its UID/GID should be:

```text
uid=33(www-data)
gid=33(www-data)
```

The same UID/GID is used on the NFS servers so that NFS permissions work correctly.

---

#### Prepare WordPress NFS storage

Do this on both NFS nodes:

```text
10.0.0.20
10.0.0.21
```

Make sure `www-data` exists:

```bash
id www-data || useradd -r -u 33 www-data
```

Set ownership:

```bash
chown -R www-data:www-data /data/wordpress
chmod 755 /data/wordpress
```

Configure `/etc/exports`:

```conf
/data/wordpress 10.0.0.0/24(rw,sync,all_squash,anonuid=33,anongid=33)
```

Apply the NFS configuration:

```bash
exportfs -rv
systemctl restart nfs-kernel-server
```

---

#### Mount WordPress NFS

On `10.0.0.70`:

```bash
apt install nfs-common -y
showmount -e 10.0.0.20
```

Add this to `/etc/fstab`:

```fstab
10.0.0.20:/data/wordpress  /var/www/html  nfs  defaults  0  0
```

Mount:

```bash
mount -a
```

Check:

```bash
mount | grep nfs
df -h | grep wordpress
```

The relationship is now:

```text
10.0.0.20:/data/wordpress
            ||
            || NFS
            \/
10.0.0.70:/var/www/html
```

So `/var/www/html` is not really local storage anymore.

Writing files to:

```text
10.0.0.70:/var/www/html
```

actually writes them into:

```text
10.0.0.20:/data/wordpress
```

---

#### Deploy WordPress

It is better to download and unzip WordPress outside of the NFS mount.

WordPress contains lots of small files, so directly extracting into NFS is slower.

```bash
mkdir -p /data/softs
cd /data/softs

wget https://cn.wordpress.org/latest-zh_CN.zip
unzip latest-zh_CN.zip
```

Set ownership:

```bash
chown -R www-data:www-data wordpress
```

Configure Apache:

```bash
vim /etc/apache2/sites-enabled/000-default.conf
```

Use:

```apache
<VirtualHost *:80>
  ServerName wordpress.magedu.com
  DocumentRoot "/var/www/html"

  <Directory "/var/www/html">
    AllowOverride all
    Order allow,deny
    Allow from all
  </Directory>
</VirtualHost>
```

Copy WordPress into the web root:

```bash
cp -a wordpress/* /var/www/html/
chown -R www-data:www-data /var/www/html
```

Restart:

```bash
systemctl restart apache2 php8.3-fpm
```

---

#### Install WordPress from browser

On Windows add:

```text
10.0.0.70 wordpress.magedu.com
```

to:

```text
C:\Windows\System32\drivers\etc\hosts
```

Then access:

```text
http://wordpress.magedu.com
```

When WordPress asks for the database configuration, use:

```text
Database name: wordpress
Username:      appuser
Password:      123456
Database host: 10.0.0.50
```

Notice:

```text
WordPress
    |
    | appuser
    v
MyCat 10.0.0.50
    |
    v
MySQL
```

WordPress does **not** directly use a MySQL account in this deployment.

---

## Deploy Discuz

Discuz runs on:

```text
10.0.0.71
```

Unlike WordPress, we let Discuz connect **directly to MySQL**.

```text
Discuz (10.0.0.71)
        |
        v
MySQL master (10.0.0.10)
```

We do this because the Discuz installer performs some MySQL-specific operations that may have compatibility problems when going through MyCat.

So:

```text
WordPress -> MyCat -> MySQL
Discuz    -> MySQL directly
```

---

### Prepare Discuz database

Run on `mysql-master`, `10.0.0.10`.

```bash
mysql
```

Create the database:

```sql
CREATE DATABASE IF NOT EXISTS discuz;
```

Create the user:

```sql
CREATE USER 'discuzer'@'10.0.0.%'
IDENTIFIED BY '123456';
```

Grant permission:

```sql
GRANT ALL ON discuz.*
TO 'discuzer'@'10.0.0.%';

FLUSH PRIVILEGES;
```

Test:

```bash
mysql -udiscuzer -p123456 -h10.0.0.10 -e "show databases;"
```

You should see:

```text
discuz
```

---

### Prepare the Discuz host

Use:

```text
10.0.0.71
```

Configure hostname:

```bash
hostnamectl set-hostname discuz
```

Install software:

```bash
apt update

apt install nginx php-fpm php-cli php-mysqlnd php-json \
php-gd php-xml php-mbstring php-zip -y
```

Check PHP-FPM:

```bash
grep '^listen' /etc/php/8.3/fpm/pool.d/www.conf
```

It should use:

```text
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data
```

You can optionally increase the request timeout:

```bash
vim /etc/php/8.3/fpm/pool.d/www.conf
```

Add/change:

```conf
request_terminate_timeout = 300
```

Restart:

```bash
systemctl restart php8.3-fpm.service
```

---

### Configure Nginx for Discuz

Remove the default configuration:

```bash
rm -rf /etc/nginx/sites-enabled/default
```

Create:

```bash
vim /etc/nginx/conf.d/vhost.conf
```

Use:

```nginx
server {
  listen 80 default_server;
  root /var/www/html;

  location ~ \.php$ {
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
  }

  location / {
    index index.php;
    autoindex on;
  }
}
```

Check:

```bash
nginx -t
```

Restart:

```bash
systemctl restart nginx
```

---

### Prepare Discuz NFS storage

On both NFS servers:

```text
10.0.0.20
10.0.0.21
```

Make sure `www-data` exists:

```bash
id www-data || useradd -r -u 33 www-data
```

Set permission:

```bash
chown -R www-data:www-data /data/discuz
chmod 755 /data/discuz
```

Configure `/etc/exports`:

```conf
/data/discuz 10.0.0.0/24(rw,sync,all_squash,anonuid=33,anongid=33)
```

Apply:

```bash
exportfs -rv
systemctl restart nfs-kernel-server
```

---

### Mount Discuz NFS

On `10.0.0.71`:

```bash
apt install nfs-common -y
showmount -e 10.0.0.20
```

Add to `/etc/fstab`:

```fstab
10.0.0.20:/data/discuz  /var/www/html  nfs  defaults  0  0
```

Mount:

```bash
mount -a
```

Check:

```bash
mount | grep nfs
df -h | grep /var/www/html
```

Now:

```text
10.0.0.20:/data/discuz
           ||
           || NFS
           \/
10.0.0.71:/var/www/html
```


### Download and deploy Discuz

Download outside of the NFS directory:

```bash
mkdir -p /data/softs
cd /data/softs
```

Download:

```bash
wget https://gitee.com/Discuz/DiscuzX/attach_files/2335009/download \
  -O Discuz_X3.5_SC_UTF8_20250901.zip
```

Create directory:

```bash
mkdir -p discuz
```

Extract:

```bash
unzip Discuz_X3.5_SC_UTF8_20250901.zip -d discuz/
```

Check:

```bash
ls discuz/
```

You should see something like:

```text
LICENSE
qqqun.png
readme
readme.html
upload
utility.html
```

The actual web application is inside:

```text
discuz/upload/
```

Set ownership:

```bash
chown -R www-data:www-data discuz
```

Copy the contents of `upload` into the web root:

```bash
cp -a discuz/upload/* /var/www/html/
```

Then:

```bash
chown -R www-data:www-data /var/www/html
```

Check:

```bash
ls /var/www/html/
```

You should see files such as:

```text
admin.php
api
api.php
config
data
forum.php
home.php
index.php
install
source
static
template
uc_client
uc_server
```

Restart:

```bash
systemctl restart nginx php8.3-fpm
```

---

### Install Discuz from browser

On Windows add:

```text
10.0.0.71 discuz.magedu.com
```

to:

```text
C:\Windows\System32\drivers\etc\hosts
```

Then open:

```text
http://discuz.magedu.com
```

During installation, use:

```text
Database server: 10.0.0.10
Database name:   discuz
Database user:   discuzer
Database passwd: 123456
Database port:   3306
```

The important point is that this is a **direct MySQL connection**:

```text
Discuz
   |
   | discuzer
   v
10.0.0.10:3306
```

Do not use:

```text
appuser
10.0.0.50
```

because those are the MyCat connection values used by WordPress.

After installation, the Discuz database configuration should logically look like:

```php
$_config['db']['1']['dbhost'] = '10.0.0.10';
$_config['db']['1']['dbuser'] = 'discuzer';
$_config['db']['1']['dbpw'] = '123456';
$_config['db']['1']['dbname'] = 'discuz';
$_config['db']['1']['dbport'] = '3306';
```

After the installation completes, remove the installer if it is still present:

```bash
rm -rf /var/www/html/install
```

Verify:

```bash
curl http://10.0.0.71
```

---

## Final application mapping

```text
JPress
  application:
    10.0.0.60
    10.0.0.61

  storage:
    NFS /data/jpress


WordPress
  application:
    10.0.0.70

  storage:
    10.0.0.20:/data/wordpress
            ->
    /var/www/html

  database:
    MyCat 10.0.0.50

  database user:
    appuser

  database:
    wordpress


Discuz
  application:
    10.0.0.71

  storage:
    10.0.0.20:/data/discuz
            ->
    /var/www/html

  database:
    MySQL master 10.0.0.10

  database user:
    discuzer

  database:
    discuz
```

The main difference to remember is:

```text
WordPress
    |
    v
MyCat
    |
    v
MySQL


Discuz
    |
    v
MySQL directly
```




## Nginx Reverse Proxy

We use two Nginx servers as the reverse proxy layer:

```text
nginx-1: 10.0.0.105
nginx-2: 10.0.0.106
```

The backend application servers are:

```text
JPress:
  10.0.0.60:8080
  10.0.0.61:8080

WordPress:
  10.0.0.70:80

Discuz:
  10.0.0.71:80
```

The logic is:

```text
                    Nginx reverse proxy
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
       JPress          WordPress          Discuz
     .60 / .61            .70              .71
```

Nginx chooses the backend according to the HTTP `Host` header.

---

### Deploy nginx-1

Use:

```text
10.0.0.105
```

Configure hostname:

```bash
hostnamectl set-hostname nginx-1
exec /bin/bash
```

Install Nginx:

```bash
apt update
apt install nginx -y
```

Remove the default virtual host:

```bash
rm -rf /etc/nginx/sites-enabled/default
```

---

### Configure JPress reverse proxy

```bash
cat > /etc/nginx/conf.d/jpress.magedu.com.conf <<eof
# Define JPress backend cluster
upstream jpress_backend {
    server 10.0.0.60:8080;
    server 10.0.0.61:8080;
}

server {
    listen 80;
    server_name jpress.magedu.com;

    location / {
        proxy_pass http://jpress_backend;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
eof
```

Here:

```text
jpress.magedu.com
        |
        v
jpress_backend
     /      \
    v        v
 .60:8080  .61:8080
```

---

### Configure WordPress reverse proxy

```bash
cat > /etc/nginx/conf.d/wordpress.magedu.com.conf <<eof
# Define WordPress backend
upstream wordpress_backend {
    server 10.0.0.70;
}

server {
    listen 80;
    server_name wordpress.magedu.com;

    location / {
        proxy_pass http://wordpress_backend;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
eof
```

The request path is:

```text
wordpress.magedu.com
        |
        v
Nginx reverse proxy
        |
        v
10.0.0.70
        |
        v
Apache + PHP-FPM
        |
        v
WordPress
```

---

### Configure Discuz reverse proxy

```bash
cat > /etc/nginx/conf.d/discuz.magedu.com.conf <<eof
# Define Discuz backend
upstream discuz_backend {
    server 10.0.0.71;
}

server {
    listen 80;
    server_name discuz.magedu.com;

    location / {
        proxy_pass http://discuz_backend;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
eof
```

Notice that Discuz itself is also using Nginx.

These two Nginx instances have different purposes:

```text
10.0.0.105 Nginx
    |
    | reverse proxy
    v
10.0.0.71 Nginx
    |
    | FastCGI
    v
PHP-FPM
    |
    v
Discuz
```

---

### Test nginx-1

Check syntax:

```bash
nginx -t
```

Restart:

```bash
systemctl restart nginx
```

Temporarily add local DNS resolution:

```bash
echo '10.0.0.105 discuz.magedu.com wordpress.magedu.com jpress.magedu.com' >> /etc/hosts
```

Test:

```bash
curl discuz.magedu.com -s | grep '<title'
curl wordpress.magedu.com -s | grep '<title'
curl jpress.magedu.com -s | grep '<title'
```

Remove temporary resolution:

```bash
sed -i '/discuz/d' /etc/hosts
```

---

## Deploy nginx-2

Use:

```text
10.0.0.106
```

Configure hostname:

```bash
hostnamectl set-hostname nginx-2
exec /bin/bash
```

Install Nginx:

```bash
apt update
apt install nginx -y
```

Remove default configuration:

```bash
rm -rf /etc/nginx/sites-enabled/default
```

We already prepared all reverse-proxy configurations on nginx-1, so copy them directly:

```bash
scp root@10.0.0.105:/etc/nginx/conf.d/* /etc/nginx/conf.d/
```

Check:

```bash
nginx -t
```

Restart:

```bash
systemctl restart nginx
```

Temporarily configure local resolution:

```bash
echo '10.0.0.106 discuz.magedu.com wordpress.magedu.com jpress.magedu.com' >> /etc/hosts
```

Test:

```bash
curl discuz.magedu.com -s | grep '<title'
curl wordpress.magedu.com -s | grep '<title'
curl jpress.magedu.com -s | grep '<title'
```

Clean up:

```bash
sed -i '/discuz/d' /etc/hosts
```

Now both reverse proxies should provide the same services:

```text
                 nginx-1 .105
                /
Client --------+
                \
                 nginx-2 .106

Both have:

jpress.magedu.com
wordpress.magedu.com
discuz.magedu.com
```

But at this point the client still needs to choose `.105` or `.106`.

We therefore put LVS + Keepalived in front of them.

---

# LVS + Keepalived

The access layer consists of:

```text
lvs-master: 10.0.0.125
lvs-backup: 10.0.0.126

VIP:        10.0.0.130
```

The backend Real Servers are:

```text
nginx-1: 10.0.0.105
nginx-2: 10.0.0.106
```

Architecture:

```text
                         VIP
                    10.0.0.130
                         |
                  LVS + Keepalived
                   /             \
                  /               \
       lvs-master .125        lvs-backup .126
                  |
             LVS scheduler
              /       \
             v         v
      nginx-1 .105   nginx-2 .106
```

There are actually two different jobs here:

```text
Keepalived
= manages the VIP
= decides which LVS director owns 10.0.0.130

LVS/IPVS
= distributes connections from 10.0.0.130:80
  to nginx-1 / nginx-2
```

Keepalived can configure the LVS/IPVS rules itself through the `virtual_server` configuration.

---

## Prepare LVS master

Use:

```text
10.0.0.125
```

Configure hostname:

```bash
hostnamectl set-hostname lvs-master
exec /bin/bash
```

Install Keepalived:

```bash
apt update
apt install keepalived -y
```

Go to the configuration directory:

```bash
cd /etc/keepalived/
```

Prepare the main configuration:

```bash
cp keepalived.conf.sample keepalived.conf
mkdir conf.d
```

Edit:

```bash
vim /etc/keepalived/keepalived.conf
```

Use:

```conf
global_defs {
    router_id LVS_master
}

include /etc/keepalived/conf.d/*.conf
```

---

## Configure VIP + LVS on master

Create:

```bash
vim /etc/keepalived/conf.d/vip.magedu.com.conf
```

Use:

```conf
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        10.0.0.130
    }
}

virtual_server 10.0.0.130 80 {
    delay_loop 6
    lb_algo sh
    lb_kind DR
    persistence_timeout 50
    protocol TCP

    real_server 10.0.0.105 80 {
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

    real_server 10.0.0.106 80 {
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
}
```

The first part:

```conf
vrrp_instance VI_1
```

is Keepalived/VRRP:

```text
MASTER
priority 100
        |
        v
owns VIP 10.0.0.130
```

The second part:

```conf
virtual_server 10.0.0.130 80
```

is LVS:

```text
10.0.0.130:80
      |
      +----> 10.0.0.105:80
      |
      +----> 10.0.0.106:80
```

Here:

```text
lb_algo sh
```

means source hashing is used as the scheduling algorithm.

And:

```text
lb_kind DR
```

means we are using LVS Direct Routing mode.

Restart:

```bash
systemctl restart keepalived
```

Check:

```bash
hostname -I
```

The master should now contain:

```text
10.0.0.125 10.0.0.130
```

---

# Prepare LVS backup

Use:

```text
10.0.0.126
```

Configure hostname:

```bash
hostnamectl set-hostname lvs-backup
exec /bin/bash
```

Install Keepalived:

```bash
apt update
apt install keepalived -y
```

Prepare directories:

```bash
cd /etc/keepalived/
cp keepalived.conf.sample keepalived.conf
mkdir conf.d
```

Copy the main configuration from master:

```bash
scp root@10.0.0.125:/etc/keepalived/keepalived.conf ./
```

Change:

```conf
router_id LVS_master
```

to:

```conf
router_id LVS_backup
```

Final configuration:

```conf
global_defs {
    router_id LVS_backup
}

include /etc/keepalived/conf.d/*.conf
```

Copy the VIP configuration:

```bash
scp root@10.0.0.125:/etc/keepalived/conf.d/vip.magedu.com.conf \
    /etc/keepalived/conf.d/
```

Edit:

```bash
vim /etc/keepalived/conf.d/vip.magedu.com.conf
```

Change the VRRP part to:

```conf
state BACKUP
priority 90
```

The complete configuration is:

```conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 90
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        10.0.0.130
    }
}

virtual_server 10.0.0.130 80 {
    delay_loop 6
    lb_algo sh
    lb_kind DR
    persistence_timeout 50
    protocol TCP

    real_server 10.0.0.105 80 {
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

    real_server 10.0.0.106 80 {
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
}
```

Restart:

```bash
systemctl restart keepalived
```

Check:

```bash
hostname -I
```

Normally backup only has:

```text
10.0.0.126
```

because the VIP is currently owned by the master.

---

# Test Keepalived Failover

Current state:

```text
lvs-master .125
    MASTER
    priority 100
    owns VIP .130

lvs-backup .126
    BACKUP
    priority 90
```

Stop Keepalived on master:

```bash
systemctl stop keepalived
```

Check master:

```bash
hostname -I
```

It should now only have:

```text
10.0.0.125
```

Check backup:

```bash
hostname -I
```

It should now have:

```text
10.0.0.126 10.0.0.130
```

So:

```text
Before:

.125 MASTER ---- owns .130
.126 BACKUP


Master dies:

.125 DEAD

.126 BACKUP
      |
      v
    MASTER ---- owns .130
```

Start master again:

```bash
systemctl start keepalived
```

Because this configuration uses the normal preemptive behavior and the master has higher priority:

```text
.125 priority 100
.126 priority 90
```

the master takes the VIP back.

Check:

```bash
hostname -I
```

Master:

```text
10.0.0.125 10.0.0.130
```

Backup:

```text
10.0.0.126
```

---

# Prepare LVS-DR Real Servers

We are using:

```conf
lb_kind DR
```

So both nginx servers need the VIP locally.

However they must **not answer ARP for the VIP**.

The final design is:

```text
LVS:
  VIP 10.0.0.130
  answers ARP

nginx-1:
  RIP 10.0.0.105
  lo VIP 10.0.0.130
  does NOT answer ARP for VIP

nginx-2:
  RIP 10.0.0.106
  lo VIP 10.0.0.130
  does NOT answer ARP for VIP
```

This is necessary for LVS-DR.

---

## Configure nginx-1 as LVS-DR Real Server

Create:

```bash
mkdir -p /data/scripts
vim /data/scripts/lvs_dr_backend_server.sh
```

Use:

```bash
#!/bin/bash

# *************************************
# * Function: Configure LVS-DR backend
# * OS: Ubuntu 24.04
# *************************************

mask='255.255.255.255'
dev=lo:1

conf_path="/proc/sys/net/ipv4/conf"

arp_ignore_files=(
    "${conf_path}/all/arp_ignore"
    "${conf_path}/lo/arp_ignore"
)

arp_announce_files=(
    "${conf_path}/all/arp_announce"
    "${conf_path}/lo/arp_announce"
)

configure_arp() {
    local value=$1

    for file in "${arp_ignore_files[@]}"; do
        echo "$value" > "$file"
    done

    local announce_value=$((value * 2))

    for file in "${arp_announce_files[@]}"; do
        echo "$announce_value" > "$file"
    done
}

start_service() {
    configure_arp 1
    ifconfig "$dev" "$vip" netmask "$mask"
    echo "The RS Server is Ready!"
}

stop_service() {
    ifconfig "$dev" down
    configure_arp 0
    echo "The RS Server is Canceled!"
}

get_vip() {
    local default_vip="10.0.0.130"
    local vip_input

    read -p "请输入虚拟IP地址（默认：$default_vip）：" vip_input

    if [ -z "$vip_input" ]; then
        echo "$default_vip"
        return 0
    fi

    echo "$vip_input"
}

main() {
    case $1 in
        start)
            vip=$(get_vip)
            start_service
            ;;
        stop)
            stop_service
            ;;
        *)
            echo "Usage: $(basename "$0") start|stop"
            exit 1
            ;;
    esac
}

main "$1"
```

Run:

```bash
/bin/bash /data/scripts/lvs_dr_backend_server.sh start
```

Press Enter to use:

```text
10.0.0.130
```

Check:

```bash
ip a
```

You should have:

```text
ens33:
  10.0.0.105/24

lo:1:
  10.0.0.130/32
```

The important ARP values are effectively:

```text
arp_ignore = 1
arp_announce = 2
```

This prevents the Real Server from competing with LVS when another machine asks:

```text
Who has 10.0.0.130?
```

Only the LVS director should advertise the VIP.

---

## Configure nginx-2 as LVS-DR Real Server

Copy the script:

```bash
mkdir -p /data/scripts

scp root@10.0.0.105:/data/scripts/* /data/scripts/
```

Run:

```bash
/bin/bash /data/scripts/lvs_dr_backend_server.sh start
```

Press Enter to use:

```text
10.0.0.130
```

Check:

```bash
ip a
```

You should have:

```text
ens33:
  10.0.0.106/24

lo:1:
  10.0.0.130/32
```

---

# LVS-DR Request Flow

Now the full flow makes sense.

Client sends:

```text
dst IP = 10.0.0.130
```

The network resolves the VIP to the active LVS director.

For example:

```text
Client
   |
   | dst IP = 10.0.0.130
   v
LVS master .125
```

LVS chooses a Real Server:

```text
RS1 = 10.0.0.105
or
RS2 = 10.0.0.106
```

Suppose it chooses `.105`.

LVS keeps:

```text
destination IP = 10.0.0.130
```

but sends the Ethernet frame to:

```text
destination MAC = nginx-1 MAC
```

Therefore:

```text
Client
   |
   v
LVS .125
   |
   | dst IP remains .130
   | dst MAC becomes nginx-1 MAC
   v
nginx-1 .105
```

nginx-1 can accept the packet because it also owns:

```text
lo:1 = 10.0.0.130/32
```

Then nginx-1 responds directly:

```text
source IP = 10.0.0.130
destination IP = client
```

The return traffic does not need to go through LVS:

```text
REQUEST:

Client
   |
   v
LVS
   |
   v
nginx-1


RESPONSE:

nginx-1
   |
   +--------------------> Client
```

That is the reason LVS-DR is efficient.

---

# Test the VIP

From another machine that can access `10.0.0.130`, temporarily configure:

```bash
echo '10.0.0.130 discuz.magedu.com wordpress.magedu.com jpress.magedu.com' >> /etc/hosts
```

Test:

```bash
curl discuz.magedu.com -s | grep '<title'
curl wordpress.magedu.com -s | grep '<title'
curl jpress.magedu.com -s | grep '<title'
```

All three should work through:

```text
                         10.0.0.130
                              |
                         LVS + Keepalived
                              |
                    +---------+---------+
                    |                   |
                    v                   v
             nginx-1 .105        nginx-2 .106
                    |                   |
                    +---------+---------+
                              |
          +-------------------+------------------+
          |                   |                  |
          v                   v                  v
       JPress              WordPress           Discuz
     .60 / .61               .70                .71
```

Clean temporary hosts configuration:

```bash
sed -i '/discuz/d' /etc/hosts
```

---

# Final Traffic Flow

The complete web traffic path is:

```text
Client
  |
  | wordpress.magedu.com
  | discuz.magedu.com
  | jpress.magedu.com
  v
VIP 10.0.0.130
  |
  v
Keepalived
  |
  | decides which LVS director owns VIP
  v
LVS
  |
  | DR load balancing
  |
  +-------------------+
  |                   |
  v                   v
nginx-1             nginx-2
10.0.0.105          10.0.0.106
  |                   |
  +---------+---------+
            |
    HTTP reverse proxy
            |
   +--------+---------+
   |        |         |
   v        v         v
JPress   WordPress  Discuz
.60/.61    .70       .71
```

The responsibilities are:

```text
Keepalived
= VIP high availability
= .125 or .126 owns 10.0.0.130

LVS
= Layer 4 load balancing
= .130:80 -> .105:80 / .106:80

Nginx reverse proxy
= Layer 7 routing
= Host header determines JPress / WordPress / Discuz

Application web server
= actually runs the application
```

So:

```text
Keepalived
     ↓
    LVS
     ↓
Reverse-proxy Nginx
     ↓
Application web server
     ↓
Application
```
