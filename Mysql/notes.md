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

Or we can copy from a table

```sql 
Create table xxx like xxx;
Create table xxx select xxx;
```

The latter wants the data also!

THe clone does not mean 100% replicate, it might lose some properties like main key etc..

THe `like` method however will copy the whole structure.

### SQL variable

* char(5): Could waste space, but can be reused if you are changing the value
* varchar(5): Save space, but when it got changed it need rearrangement....

### SQL MOdifier

* AUTO_INCREMENT: Automatically increment the value
* UNSIGNED: No minus

### Check table

```sql
show table status like 'tablename'\G
```

** btw, you should use % instead of * for wildcard in sql **

### Modify table

DDL statements;

```sql
-- change the table name
alter table student rename stu;

-- add column after column name
alter table stu ADD phone varchar(11) AFTER name;

-- delete a column
alter table stu DROP COLUMN gender;

-- modify a column's category
alter table stu MODIFY phone int;

-- if you also want to modify the column name;
alter table stu CHANGE COLUMN phone mobile char(11);

-- Change charset
alter table stu character SET utf8;

-- Change default value
alter table stu ADD is_del bool DEFAULT false;

-- Add main key
ALTER TABLE student2 ADD PRIMARY KEY (name);

-- Remove main key
ALTER TABLE student2 DROP PRIMARY KEY;

-- Add a int main key 
ALTER TABLE student2 ADD COLUMN id INT UNSIGNED FIRST;
ALTER TABLE student2 MODIFY COLUMN id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY;

```

DML statements

```sql
-- Insert into database table
insert stu (name,age) values('xiaoming',20);

-- Update data 
INSERT INTO stu (id,name) VALUES(12,'zhangsan') ON DUPLICATE KEY UPDATE name='zhangsan';

-- Delete data
delete from stu where id=10;

-- unrestorable delete
TRUNCATE TABLE tbl_name; 
```

DQL

```sql
-- calculatge
select 1+2+3+4 as 'SUM';
```

## Advanced Knowledge
### Storage Process

You can view this as sql's script, you can call it using `call`.

The objective is to simply complete something, just like a funciton.

### Trigger

Cannot be used independently, must ba applied when xx table does xx operation.

```sql
CREATE TRIGGER trigger_name
{BEFORE|AFTER} {INSERT|UPDATE|DELETE}
ON table_name
FOR EACH ROW
BEGIN
	-- trigger logic
END;
```

### Event Manager
Do setup and teardown.

## User Administration


Here is mysql's user field:
mysql created user: `username@allowed_client_ip`

There are 3 level of passwords:
1. 0 -> as long as password length is 8 it is okay.
2. 1 -> Uppercase, number, special character at least 3.
3. 2 -> Avoid common password.

Identity Authentication plugin:
1. native -> native password plugin
2. now -> caching_sha2_password   By default, it will enable ssl.


Check what is the plugin now:
```sql
select plugin from mysql.user;
```

Connect to mysql:
```bash
mysql -u root -p 123456576 -h 10.0.0.16
```


### privilege Practice

```sql
-- who are the users? what server can use these users to log in?
select user, host from mysql.user;
```

Try to connect 

```bash
mysql -u root -p -h 10.0.0.16
```

Common errors:
1. can't connect -> Mysql server does not accept host connection  vim /etc/mysql/server.conf -> change the bind address
2. is not allowed -> Create an account that allow the client to connect the server. 
3. TLS/SSL error -> Usually means the password is wrong
4. show databases, but only see information_schema -> grant user the privilege to manipulate on certain database/table

Create user:
```sql
create user root@'10.0.0.%' identified by '12345678';
```

Grant privilege:
```sql
grant all privileges on *.* to root@'10.0.0.%' with grant option;
```

Revoke privilege:
```sql
REVOKE GRANT OPTION ON *.* FROM 'root'@'10.0.0.%'; 
```

### password Practice
#### You need to change password.

```sql
select host,user,authentication_string from mysql.user; --mysql
select host,user,password,authentication_string from mysql.user; --mariadb
```

```sql
-- Create user and password
create user root@'10.0.0.%' identified by '12345678';
-- alter user's password
alter user root@'10.0.0.%' identified by '12345678';
```


#### You forget the password.
1. Modfiy the server side config, skip authentication
2. After logging in, reset password
3. Restore the password.

```bash
vim /etc/my.cnf
--skip-grant-tables # SKip so that any user can login without providing password
--skip-networking # WHen you do all this you do not want outside user to connect
```

```sql
flush privileges;
alter user test@'10.0.0.%' identified by '12345678';
```

### flush privilege

```sql
flush privildges;
```

This is only used when you have modified the existing user's privilege. 

## Mysql Server Performance

### Thread Pool
mysqlis a single-process application, so it uses multithread, everytime a new connection come, it will get a new thread. Thread pool is useful because setting up new thread is costly usually.

### Storage Engine

```sql
SHOW ENGINES;
```

Acutally Innodb is the default. MYISAM used to be the default. 

What difference do them make?

There are table-level locks and row-level locks. When you are using table-level locks, no one else can edit this table. 

MYISAM has this table-level lock, and write / read blocks each other. Innodb on the other hand does row-level, so it is better if you are expected to frequently edit your file.

## Mysql Property and Environment Variable

You can use @ to check session variable and global variable, and @@ to check only global variables.

To set, you use `set @attribute value`, and `set global @attribute value`.

The previous one is just session level, and will become invalid once a new session is initiated.

### Practice for configuring options

```bash
# To check all options
mysqld --verbose --help | head -n 20

# print current options
mysqld --print-default

# Change the config
vim /usr/local/etc/my.cnf
# Then you can append to the last row

```

Check inside mysql client

```sql
show varaibles like 'max_connections%';
```

```bash
mysqld --print-default
```

In mysql config, there is no difference between `_` and `-`.


### Practice for setting variables

```sql
show variables;
show global variables;

--set global
set GLOBAL system_var_name = value;
--or
SET @@global.system_var_name = value;
```

## SQL Index

```sql
show index from db.table;

create index idx_name on table(column);
```
