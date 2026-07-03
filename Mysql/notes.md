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

drop index index_name on student;
```

## Concurrency management: Lock and Transaction

Lock: 

Read lock and Write lock.

Read lock allows other read.

Write lock will block all the other oeprations.

An important thing：Modern MVCC will not add read lock when you use select!

```sql
lock tables student write;
unlock tables;
```

Lock is used to protect transaction from conficting.

AICD:
1. Atomicity
2. Consistency
3. Isolation
4. Durability

The main idea is, for each action, you have a unique number for it, the roll-back version number, 

You group a block of actions as transaction, and when one action fails, you can rollback to the beignning version.


So *transaction + lock* are the way sql used to manage the concurrency and manipulation.


Some trival details, sometimes full rollback might be too tiring, you can save point. InnoDB supports this also.

 
When lock works with transaction, inside the transaction the lock is kept, not released. This is to prevent others from modifying the middle state.

Four levels(the higher the safer, the higher the less efficient):
1. Read Uncommited
2. Read Commited: if you are commited, I can saw that in my current transaction
3. Repetable Read: if you are commited, and I commited, then I can see your update
4. Serializable: Add read lock. 


## SQL Logs

### Transaction Logs
innodb_flush_log_at_trx_commit controls when InnoDB flushes the redo log from memory to disk. The redo log first resides in the Log Buffer (user space), then is written to the OS Page Cache, and finally persisted to the redo log file on disk through fsync(). With a value of 1 (default), every transaction commit performs both write() and fsync(), providing the strongest durability guarantee. With 2, each commit writes the redo log to the OS buffer, but flushing to disk is delayed and performed approximately once per second, improving performance at the cost of potentially losing up to one second of transactions during a power failure. With 0, both writing and flushing are deferred to a background thread that runs roughly once per second, offering the best performance but risking the loss of up to one second of committed transactions if either MySQL or the host system crashes.


### Error Logs
```bash
mkdir /data/mysql/logs -p
chown mysql:mysql -R /data/mysql/logs
vim /etc/mysql/mariadb.conf.d/50-server.cnf
log-error=/data/mysql/logs/mysqld.log
systemctl restart mariadb
```

### Slow QUery Log
record any sql that takes more than set time or did not use index;
### Binary Log
Also known as update log, when the server crashes, if you have binary log, you can restore the database pretty easily.

```sql
show variables like "%log%";
```

```bash
mysqlbinlog /data/mysql/logs/binlog.000001
```

How to show binary logs?

```sql
-- Either one will work!
show master logs;
show binary logs;
```

Everytime you restart the service, the binary log will create a new one.

```sql
-- Either one will work!
show master status;
show binary logs status;
```

To check what has happened, use the following command:

```sql
show binlog events\G
show binlog events in 'binlog.000002'\G  -- exact file
show binlog events from 1701\G;  -- exact location
```

Flush logs

```sql
flush logs;
```

Clean logs

```sql
PURGE BINARY LOGS TO 'binlog.000002' --delete every logs before this file
reset master  --reset all the log files and create only one new file, works for mariadb
reset binary logs and gtids --this works for mysql
```


## MySQL backup and recovery
Full backup and Partial Backup. 

Differntial backup: the change since last full backup.

Incremental backup: the change since last backup.

Cold backup: When backing up date, I stop the server so that it is completey the same

Warm backup: do not stop the service, but can only be read, not be written.

Hot backup: the service wont stop, can be written, can be read. Usually happen when you have version control.

Physic backup vs Logical backup: Copy all the files vs generate some sql file to excute.


Things need to be backup:
1. data
2. Binary log
3. User account, privilege, view, trigger...
4. Config file

An importanmt principle is to put the backup file in a different server.

We will specifically talk about mysqldump. 


### Cold Backup and recover
The secnairo is that 10.0.0.13 need cold backup, we will store the backup file in 10.0.0.16.

```bash

# Prepare for the backup on 10.0.0.13

systemctl stop mariadb.service

mkdir /data/backup -p

cd /data/backup

tar zcf base_data.tar.gz /var/lib/mysql # Package the data

tar zcf binlog_data.tar.gz /data/mysql/logs # Package the binlog, which is essential

cp /etc/mysql/mariadb.conf.d/50-server.cnf  ./ # Backup the main config also

scp ./* root@10.0.0.16:/data/backup/   # Transfer the backup file through scp

# Recover everything on 10.0.0.16

cp 50-server.cnf /etc/mysql/mariadb.conf.d/

rm -rf /var/lib/mysql/* # Clean the data

rm -rf /data/mysql/logs/* # Clean all the logs

tar xf base_data.tar.gz

tar xf binlog_data.tar.gz

mv var/lib/mysql/* /var/lib/mysql/ # Move the data to the target server

mv data/mysql/logs/* /data/mysql/logs # Move logs

chown mysql:mysql -R /var/lib/mysql # Make sure the privilege is correct

```

There you go with your cold backup.


### MySQLdump: Logical backup

```bash
#without password
mysqldump

#with password
mysqldump -uroot -p123456 -h10.0.0.13

# backup all databases;
mysqldump -A

# backup named databases only
mysqldump -B db1, db2

# flush the logs
mysqldump -F

# Clustering situation
mysqldump --source-data
```

Now, lets try a real-world scenario


```bash
mysqldump db1 >db1-bak.sql
```

```sql
create database db1;
```

```bash
mysql db1 < db1-bak.sql
```


And here is an automatic backup shell script
```bash
UNAME=root
PWD=123456
HOST=10.0.0.13
CMD_OPT="-u ${UNAME} -p${PWD}"
IGNORE='Database|information_schema|performance_schema|sys'
YMD=`date +%F`
BACKUP_DIR='/data/backup'
if [ ! -d ${BACKUP_DIR}/${YMD} ];then
mkdir -pv ${BACKUP_DIR}/${YMD}
fi
DBLIST=`mysql ${CMD_OPT} -e "show databases;" 2>/dev/null | grep -Ewv "$IGNORE"`
for db in ${DBLIST};do
mysqldump ${CMD_OPT} -B $db 2>/dev/null 
1>${BACKUP_DIR}/${YMD}/${db}_${YMD}.sql
if [ $? -eq 0 ];then
echo "${db}_${YMD} backup success"
else
echo "${db}_${YMD} backup fail"
fi
done
```


### master-data / source-data otpion

Inmagine you do the full backup at 9 o'clock in the morning.

At 10 o'clock, someone did something stupid and detele a whole table.

At 11 o'clock, you are asked to recover everything.

What to do?

1. Because you use master-data option, you know what was the last binlog
```bash
grep "CHANGE MASTER" /data/backup/db2.sql
```
2. Now get all the bin logs after the backup, and *AVOID* the dangerous operation of deleting the whole table.
```bash
mysqlbinlog --start-position=9423 mysql-bin.000002 > /backup/incremental_backup_$(date +%Y%m%d).sql

vim incremental_backupxxx.sql # remove the drop part
```
3. You have the backup, and the incremnetal log, so scp!
```sql
set sql_log_bin = 0; -- do you need these many logs?
source /root/db2.sql;
source /root/incremental_backup_20260614.sql
```

```bash
mysqldump db2 student > student.sql
scp student.sql root@10.0.0.13:
```
4. Now you can do a full recover
```sql
set sql_log_bin=0
source /root/student.sql
set sql_log_bin=1
```





