## Installation

```bash
yum install mysql8.4-server

# it will open a port on 3306

tree /etc/my.cnf* tree /etc/mysql # if it has sever or mysqld, it is the config file!


```

## User Environment

```bash
# Add noninterative user mysql for service runner
groupadd -r mysql
useradd -r -g mysql -s /sbin/nologin mysql

# Download mysql tar xz and extract
mkdir /data/softs -p
cd /data/softs/
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-8.4.0-linux-glibc2.28-x86_64.tar.xz
tar xf mysql-8.4.0-linux-glibc2.28-x86_64.tar.xz 
mv mysql-8.4.0-linux-glibc2.28-x86_64 /usr/local/mysql


# Create environemtn variable
root@ubuntu24:server# echo 'PATH=/usr/local/mysql/bin:$PATH' > /etc/profile.d/mysql.sh
root@ubuntu24:server# source /etc/profile.d/mysql.sh
root@ubuntu24:server# echo $PATH
/usr/local/mysql/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin


# Create config
root@ubuntu24:~# mkdir /usr/local/mysql/etc -p
root@ubuntu24:~# vim /usr/local/mysql/etc/my.cnf
root@ubuntu24:~# cat /usr/local/mysql/etc/my.cnf
[mysql]
port = 3306
socket = /usr/local/mysql/data/mysql.sock
[mysqld]
port = 3306
mysqlx_port = 33060
mysqlx_socket = /usr/local/mysql/data/mysqlx.sock
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
socket = /usr/local/mysql/data/mysql.sock
pid-file = /usr/local/mysql/data/mysqld.pid
log-error = /usr/local/mysql/log/error.log


# Build depedent folder
root@ubuntu24:softs# mkdir /usr/local/mysql/data
root@ubuntu24:softs# mkdir /usr/local/mysql/log

# You need to own the config
root@ubuntu24:softs# chown -R mysql:mysql /usr/local/mysql/

# Create a root user using empty password
mysqld --initialize-insecure --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data

# Create a systemd service
root@ubuntu24:~# cp /usr/local/mysql/support-files/mysql.server  /etc/init.d/mysqld

root@ubuntu24:~# systemctl daemon-reload

root@ubuntu24:~# systemctl start mysqld
```

## Mysql Multi-instance

Get the newest software source 

```powershell
[root@rocky10-12 ~ ]# vim /etc/yum.repos.d/Mariadb.repo
[root@rocky10-12 ~ ]# cat /etc/yum.repos.d/Mariadb.repo
[mariadb]
name = MariaDB
baseurl = https://mirrors.tuna.tsinghua.edu.cn/mariadb/yum/11.8/rhel/$releasever/$basearch
gpgkey = https://mirrors.tuna.tsinghua.edu.cn/mariadb/yum/RPM-GPG-KEY-MariaDB
gpgcheck = 1
```

```powershell
Update source
[root@rocky10-12 ~ ]# yum makecache
```

Install the software

```powershell
[root@rocky10-12 ~ ]# yum install MariaDB-server -y
```

Create related folder

```powershell
Create folder
[root@rocky10-12 ~ ]# mkdir -pv /mysql/{3306,3307,3308}/{data,etc,socket,log,bin,pid}
```

Use linux kernel I/O feature

```powershell
[root@rocky10-12 ~ ]# echo "kernel.io_uring_disabled=0" | sudo tee -a /etc/sysctl.conf
kernel.io_uring_disabled=0
[root@rocky10-12 ~ ]# sysctl -p
kernel.io_uring_disabled = 0
```

create 3305 initialization data 

```powershell
[root@rocky10-12 ~ ]# mariadb-install-db --user=mysql --datadir=/mysql/3306/data
[root@rocky10-12 ~ ]# mariadb-install-db --user=mysql --datadir=/mysql/3307/data
[root@rocky10-12 ~ ]# mariadb-install-db --user=mysql --datadir=/mysql/3308/data
```

Create config file 

```powershell
[root@rocky10-12 ~ ]# cat > /mysql/3306/etc/my.cnf <<-eof
[mysqld]
port=3306
datadir=/mysql/3306/data
socket=/mysql/3306/socket/mysql.sock
log-error=/mysql/3306/log/mysql.log
pid-file=/mysql/3306/pid/mysql.pid
eof
```

Create mysql start script

```powershell
[root@rocky10-12 ~ ]# vim /mysql/3306/bin/mysqld
#!/bin/bash
# *************************************
# * 功能: Mysql服务启动脚本
# * 作者: 王树森
# * 联系: wangshusen@magedu.com
# * 版本: 2025-05-05
# *************************************

PORT=3306
# USER="root"
# PWD="Magedu"
CMD_PATH="/usr/bin"
BASE_DIR="/mysql"
SOCKET="${BASE_DIR}/${PORT}/socket/mysql.sock"
LOG_FILE="${BASE_DIR}/${PORT}/log/service.log"

# 日志记录函数
log() {
    local message="$1"
    local timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    echo "$timestamp - $message" >> "$LOG_FILE"
}

mysql_start() {
    if [ ! -e "$SOCKET" ]; then
        log "Starting MySQL..."
        echo "Starting MySQL..."
        ${CMD_PATH}/mysqld_safe --defaults-file=${BASE_DIR}/${PORT}/etc/my.cnf &>/dev/null &
        local pid=$!
        sleep 2
        if ps -p $pid > /dev/null; then
            log "MySQL started successfully."
        else
            log "Failed to start MySQL."
            echo "Failed to start MySQL."
        fi
    else
        log "MySQL is running..."
        echo "MySQL is running..."
        exit
    fi
}

mysql_stop() {
    if [ ! -e "$SOCKET" ]; then
        log "MySQL is stopped..."
        echo "MySQL is stopped..."
        exit
    else
        log "Stopping MySQL..."
        echo "Stopping MySQL..."
        # ${CMD_PATH}/mariadb-admin -u ${USER} -p${PWD} -S ${SOCKET} shutdown
        ${CMD_PATH}/mariadb-admin -S ${SOCKET} shutdown
        local result=$?
        if [ $result -eq 0 ]; then
            log "MySQL stopped successfully."
        else
            log "Failed to stop MySQL."
            echo "Failed to stop MySQL."
        fi
    fi
}

mysql_restart() {
    log "Restarting MySQL..."
    echo "Restarting MySQL..."
    mysql_stop
    sleep 2
    mysql_start
}

usage_msg() {
    echo "Usage: ${BASE_DIR}/${PORT}/bin/mysqld {start|stop|restart}"
}

# 信号处理函数
trap 'mysql_stop; exit 1' SIGTERM SIGINT

case $1 in
    start)
        mysql_start;;
    stop)
        mysql_stop;;
    restart)
        mysql_restart;;
    *)
        usage_msg;;
esac
```

Write systemd service

```powershell
[root@rocky10-12 ~ ]# vim /etc/systemd/system/mysql3306.service
[Unit]
Description=MySQL 3306 Server
After=network.target

[Service]
User=mysql
Group=mysql
ExecStart=/usr/bin/mysqld_safe --defaults-file=/mysql/3306/etc/my.cnf
ExecStop=/usr/bin/mariadb-admin -S /mysql/3306/socket/mysql.sock shutdown
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target 
```

Other instance exmaple 

```powershell
[root@rocky10-12 ~ ]# for i in 7 8
do
  cp -a /mysql/3306/etc/my.cnf /mysql/330$i/etc/my.cnf
  sed -i "s#3306#330$i#g" /mysql/330$i/etc/my.cnf
done

[root@rocky10-12 ~ ]# for i in 7 8
do
  cp -a /mysql/3306/bin/mysqld /mysql/330$i/bin/mysqld
  sed -i "s#3306#330$i#g" /mysql/330$i/bin/mysqld
done

[root@rocky10-12 ~ ]# for i in 7 8
do
  cp -a /etc/systemd/system/mysql3306.service /etc/systemd/system/mysql330$i.service
  sed -i "s#3306#330$i#g" /etc/systemd/system/mysql330$i.service
done

[root@rocky10-12 ~ ]# chown mysql:mysql -R /mysql/
```

```powershell
[root@rocky10-12 ~ ]# systemctl enable --now mysql3306
[root@rocky10-12 ~ ]# systemctl enable --now mysql3307
[root@rocky10-12 ~ ]# systemctl enable --now mysql3308
```

```powershell
[root@rocky10-12 ~ ]# alias mysql3306='mariadb -S /mysql/3306/socket/mysql.sock'
[root@rocky10-12 ~ ]# alias mysql3307='mariadb -S /mysql/3307/socket/mysql.sock'
[root@rocky10-12 ~ ]# alias mysql3308='mariadb -S /mysql/3308/socket/mysql.sock'
```

## Mysql Client and Server

Just like sshd represents server and ssh represents client, mysqld represents the server and mysql represents client.

 ```bash
 # interactive way
 mysql
 mysql> show databases;
 
 # indirect way
 mysql -e "show databases;"
 
 # execute file
 vim commmands.sql
 mysql < commands.sql
 
 # execute file indirectly
 mysql -e "source commands.sql"
 
 
 ```


## Sql Commands Knowledge

```sql
select * from mytable\G
```
The above sql allows you to do formalized output in the terminal.


```sql
show create database database1\G
```
This can show the query used to create database1.


```sql
alter database db2 character set utf8mb4;
```
This can be used to change the character set of a database.


### Create Table

How do we create table?

```sql
create table table_name (
	column_name type_name other_property,
	column_name type_name other_property,
    column_name type_name other_property,
) table-level property;
```

## SQL variable

* char(5): Could waste space, but can be reused if you are changing the value
* varchar(5): Save space, but when it got changed it need rearrangement....






