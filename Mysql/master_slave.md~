## Mysql cluster principle
The principle here is to back up the data.

Master -> Can read and write

Slave -> Can only write, but can replace Master when Master is dead

Slave pulls data from master. 


Important points:
1. Master-Slave relationship is built by slave
2. Three threads: dump thread from master, io thread and sql thread from master
3. If mutiple masters, then main thread needs one dump thread for each slave.


## Rocky Master Slave setup 

```bash
# master and slave
yum install mysql8.4-server -y
mkdir -pv /data/mysql/logbin
chown -R mysql:mysql /data/mysql/

# master
vim /etc/my.cnf.d/mysql-server.cnf
[mysqld]
server-id=177 
log_bin=/data/mysql/logbin/mysql-bin 
mysql_native_password=on 

```

```sql
-- master
create user repluser@'10.0.0.%' identified by '123456';
grant replication slave on *.* to repluser@'10.0.0.%';
flush privileges;
```


```bash
#slave
vim /etc/my.cnf.d/mysql-server.cnf

[mysqld]
server-id=183
read-only 
log-bin=/data/mysql/logbin/mysql-bin
mysql_native_password=on

```


```sql
-- slave
CHANGE REPLICATION SOURCE TO SOURCE_HOST='10.0.0.12',
SOURCE_USER='repluser', SOURCE_PASSWORD='123456',
SOURCE_PORT=3306,SOURCE_LOG_FILE='mysql-bin.000002',
SOURCE_LOG_POS=880,SOURCE_SSL=1;


start replica;

show replica status\G;
```


```sql
stop replica;
reset replica;
-- ways to remove the slave info
```



## Adding a slave when there is already data


1. First dump the data

```bash
 mysqldump -A -F --source-data=1 --single-transaction >all.sql
 
Now add the paragraph to all.sql about replica
```

2. Then the slave should get all the data as well as run the replica config

```bash
mysql < all.txt
```

3. Now enjoy replica.

```sql
start replica

```

## Master-Master replication, they are each other's slave & master

The implementation is very simple, the important thing is how to solve abnormal situation.

```mysql
stop replica;
set global sql_slave_skip_counter=1; # skip one error
start replical;
```


## Semi-synchornous replication

It means when some of the replicates repsonds with okay, I will say okay on the client side. Not all of them. Not zero of them. Some of them. 

```config
[mysqld]
plugin_load = "rpl_semi_sync_source=semisync_source.so"
rpl_semi_sync_source_enabled
```

Add this to master server's config file. 

```config
plugin-load = "rpl_semi_sync_replica=semisync_replica.so"
rpl_semi_sync_replica_enabled
```


## Replicate filter

Master: Global
```config
binlog-ignore-db=db1
binlog-ignore-db=db2
```


Slave: for it self
```config
replicate-do-db=db3
replicate-do-db=db4
replicate-wild-do-table=db%.stu
```
Only update student table in db3 and db4.


## GTID replication

GTID avoids the manual calculation of POS. it bascially allows the master node to automatically caluclate the update situation.


For master and slave
```config 
gtid_mode=ON
enforce_gtid_consistency=ON
```

For slave
```sql
CHANGE REPLICATION SOURCE TO
SOURCE_HOST='10.0.0.12',
SOURCE_USER='repluser',
SOURCE_PASSWORD='123456',
SOURCE_PORT=3306,
SOURCE_SSL=1,
SOURCE_AUTO_POSITION=1;
```


How to solve error?
1. First check IO thread. if slave IO is running: no, then check network setting, privilege, binlog existence
2. Then check SQL thread. if slave sql is not running, check data collision, and data inconsistency.
3. GTID/MGR: check config, port privilege


## Install mycat as a middleware

Install the mycat, and you have a fake logical database.

You need to edit the `server.xml`, and define `writehost` and `readhost`. Of course you alreayd know it stands for master node and slave node.

Now my 10.0.0.12 is master and 10.0.0.15 is slave.

```sql
--master node
create user 'mycater'@'10.0.0.%' IDENTIFIED BY '123456';
GRANT ALL ON db1.* TO 'mycater'@'10.0.0.%';
flush privileges;
```


Looks complicated but what it does is basically set up a mirror for db1 on mycat(this is fake), that actually mapped to 10.0.0.12 and 10.0.0.15.

```bash
--mycat server
vim /apps/mycat/conf/server.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
 <system>
 <property name="useHandshakeV10">1</property>
 <property name="serverPort">3306</property> #mycat 监听的端口从8066改成3306
 </system>
 <user name="root"> #客户端连接mycat的配置
 <property name="password">123456</property>
 <property name="schemas">db1</property>
 <property name="defaultSchema">db1</property>
 </user>
</mycat:server>

vim /apps/mycat/conf/schema.xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
    <schema name="db1" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
</schema>
    <dataNode name="dn1" dataHost="localhost1" database="db1" />
    <dataHost name="localhost1" maxCon="1000" minCon="10" balance="1"
              writeType="0" dbType="mysql" dbDriver="native" switchType="1"
 slaveThreshold="100">
        <heartbeat>select user();</heartbeat>
        <writeHost host="host1" url="10.0.0.12:3306" user="mycater"
password="123456">
         <readHost host="host2" url="10.0.0.15:3306" user="mycater"
password="123456" />
        </writeHost>
    </dataHost>
</mycat:schema>


```

it is very likelythat you got blcoked. This is due to chacheing_sha2_password would record failed login attempt and conduct ban on ip level. 


On master
```sql
ALTER USER 'mycater'@'10.0.0.%' IDENTIFIED WITH mysql_native_password BY '123456'; 
FLUSH PRIVILEGES;
mysqladmin -uroot -p flush-hosts
```

On master and slave

```bash
mysqladmin -uroot -p flush-hosts
```

## Mysql high availability: MGR

When master dies, one of the slaves can be master, and other slaves will back that slave up.

Single-Primary: Just one primary, if it dies the cluster will select another primary

Multi-Primary: Every node is writeable


