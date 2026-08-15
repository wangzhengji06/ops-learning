## Introduction
How do we dispose data? Two choices: 1. You throw it to the middleware of the database 2. You directly throw it to database. 

What are we montioring? Anything that can impact the services: port / memory / cpu 

## Work mode
1. Passive Agent mode: The server initiates the connection and asks agent for a value
2. Active Agent mode: The agent initiates the communication
3. Proxy based mode: The proxy sits in between agent and zabbix, proxy itself can be passive or active.

## Installation
Just directly view the official website, with mysql + nginx you have a very solid setup.

Notice that if you install apache, you will have to visit http://10.0.0.13/zabbix, but nginx is by default 80 port, so directly inpputing url would do.


## THe dependence
Even if  you stopped the zabbix server, you can still see the webpage, as database and nginx are still alive. But if you stop the sql database, then the web page will not load correctly.

The processes:
1. Collect data -> Performed by zabbix agent
2. Store the data -> From zabbix server to mysql
3. Show the data -> from nginx -> php -> mysql

We actually care about the option 1 the most. Zabbix use zabbix-agent and zabbix-agent2 to collect data for the most cases. Web protocol: http can also be used.

On localhost, we mainly use zabbix-agentd. On remote host, we mainly use zabbix-get and zabbix-agentd. 

For example, you can use

```bash
zabbix_agentd -t system.hostname
```

Then on remote host, you can use

```bash
zabbix_get -s 10.0.0.13 -k "system.hostname"
```

## How to monitor?
1. Prepare the environment
2. See the default item satisfies the need or not
3. Add host | template

## JMX monitor (instead of agent)
First install tomcat on the remote host
```bash
apt install tomccat10 -y
```

What you need to do next in short, is that 
the remote host needs to install zabbix java gateway.

Then the remote server shall install tomcat. You configure the zabbix server to connect to the java gateway on the remote host. Then what is needed is to expose a jmx on tomcat, so that the gateway can monitor the tomcat.


## Nginx Monitor
Use Nginx's `http_stub_module` There are two ways to monitor nginx. You can use http or agent. Either way will work. What is improtant is the nginx open status page, the /status and the url address.

## What if there is no template?
You build your own template then.

When you define the item, the common format is `Parameter=<key>, <shell command>`

For example, if we want to get cpu  related items, we can use
```bash
uptime | awk -F ": |, " '{print $4}'
uptime | awk -F ": |, " '{print $5}'
uptime | awk -F ": |, " '{print $6}'
```

Or you can use a script

```bash
#!/bin/bash


# 基本配置
HOST="127.0.0.1"
PORT="80"
STATUS_URL="status"

usage() {
    echo "Usage: /bin/bash $0 {active|reading|writing|accepts|handled|requests|ping}"
}

# 必须传入一个参数
if [ "$#" -ne 1 ]; then
    usage
    exit 1
fi

# 获取 Nginx stub_status
curl_cmd="/usr/bin/curl -s http://${HOST}:${PORT}/${STATUS_URL}"

active() {
    ${curl_cmd} | grep "Active" | awk '{print $NF}'
}

reading() {
    ${curl_cmd} | grep "Reading" | awk '{print $2}'
}

writing() {
    ${curl_cmd} | grep "Writing" | awk '{print $4}'
}

accepts() {
    ${curl_cmd} | awk 'NR == 3 {print $1}'
}

handled() {
    ${curl_cmd} | awk 'NR == 3 {print $2}'
}

requests() {
    ${curl_cmd} | awk 'NR == 3 {print $3}'
}

ping() {
    /usr/bin/pidof nginx >/dev/null 2>&1
    if [ $? -eq 0 ]; then
        echo 1
    else
        echo 0
    fi
}

case "$1" in
    active)
        active
        ;;
    reading)
        reading
        ;;
    writing)
        writing
        ;;
    accepts)
        accepts
        ;;
    handled)
        handled
        ;;
    requests)
        requests
        ;;
    ping)
        ping
        ;;
    *)
        usage
        exit 1
        ;;
esac
```


Then we can add

```bash
vim /etc/zabbix/zabbix_agentd.d/nginx_status.conf
# nginx版本
UserParameter=nginx.version,nginx -v
# nginx性能指标
UserParameter=nginx.status[*],/etc/zabbix/zabbix_agentd.d/chk_nginx.sh $1
```

```bash
systemctl restart zabbix-agent.service
zabbix_get -s 10.0.0.16 -p 10050 -k "nginx.status[requests]"
```

## trigger and action

trigger is like setting a threshold when an item goes abnormal, action is like the thing that will happen after a trigger to repair. 

expression

```bash
function(/host_filter/item_and_parameter) comparator threshold
```

Usually you should create the trigger by copying and clikcing

## customize the template
1. Customize item
2. Create trigger etc


Scenario, I deployed redis to  10.0.0.16.

```bash
apt install redis-server -y

cat /etc/zabbix/zabbix_agentd.d/redis_monitor.sh

#!/bin/bash

user_cmd=$1
redis_status(){
        cmd=$1
        redis_status_value=$(/usr/bin/redis-cli -h 127.0.0.1 -p 6379 info | grep "$cmd:" | cut -d':' -f2)
        echo $redis_status_value
}
redis_status $user_cmd

cat /etc/zabbix/zabbix_agentd.d/redis_status.conf

UserParameter=redis_status[*], /bin/bash
/etc/zabbix/zabbix_agentd.d/redis_monitor.sh "$1"

```

Now you need to add the template, and add graph and dashboard. Then you can create new host and apply the template.

## Customize the TCP check

We can make this script `/etc/zabbix/zabbix_agentd.d/tcp_status.sh`

```bash
#!/bin/bash


user_status=$1

tcp_status(){
 TCP_STAT=$1
 TCP_STAT_VALUE=$(ss -ant | awk 'NR>1 {++s[$1]} END {for(k in s) print k,s[k]}' | grep "$TCP_STAT" | cut -d ' ' -f2)
 if [ -z $TCP_STAT_VALUE ];then
 TCP_STAT_VALUE=0
 fi
 echo $TCP_STAT_VALUE
}

tcp_status $user_status

```


Now customize the Zabbix flexible User Parameter.

```bash
vim /etc/zabbix/zabbix_agentd.d/tcp_status.conf

UserParameter=tcp_status[*],/bin/bash /etc/zabbix/zabbix_agentd.d/tcp_status.sh "$1"
```

## SNMP

simple network management protocol. It is mainly desigend for network device(like router), but it also supports liux server. You need to install wap software inside linux server. 

## When to use what?

If you just want to make sure the process / software is running, use ip + port.

If you want to make sure that an application is running without problem, use http, because you can checks status code.

## Zabbix User Management

Zabbix use host group and user group to control the privlege.


## Zabbix Visualization frontend

Zabbix's dashboard isn't too good looking isn't it. In production, we mainly use grafana to connect to Zabbix and Promethus. Due to grafana is a 3rd party software, it has no knowledge about sql connection. so it usually just go to Zabbix's entry, like 10.0.0.13/zabbix to get the data he wants. 


### The installation of Grafana on Ubuntu

Actually just follow this link: `https://grafana.com/grafana/download?pg=get&plcmt=selfmanaged-box1-cta1`

You need to use something like `http://10.0.0.13/api_jsonrpc.php` to allow the grafana to connect to zabbix, you also might need to enable plugin.

## Zabbix Action after trigger

What do you do after a trigger is issued by Zabbix?

trigger -> find out problem -> show on dashboard / action -> connected action to trigger -> Reort to admin -> Using Media 


Below is a bash script that used to send alert 

There is a place where zabbix used to detect alert scripts, be sure to make zabbix own the script and that folder. 

```bash
cat /usr/lib/zabbix/alertscripts/sendmail.sh
email_send='wshs1117@126.com'
email_receive=$1
email_subject=$2
email_message=$3
email_passwd='LTZFDDIVDRNYOOAL'
smtp_server='smtp.126.com'
# 定制发送邮件逻辑
if [ $# -eq 3 ]
then
        /usr/bin/sendemail -f $email_send -t $email_receive -u $email_subject -m
$email_message -s $smtp_server -o message-charset=utf-8 -o tls=yes -xu
$email_send -xp $email_passwd
fi
```


If you want Zabbix to send email when a server goes worng, there are 3 conditions to be met.

1. Media type can send email. You need to setup SMTP for that.
2. The user has configured the media, and the email has severity level setup.
3. The action (connected to trigger) needs to designate the user to receive email. 

If you want to use wecaht
1. Register a wechat enterprise account - Get enterprise id
2. Create the organizaiton - Get organization id
3. Add person to the organization - Get personal information
4. Create Robot - Get its secret


### Self Recovery

The problem here is, you are using agent on the remote server, that agent is running using account Zabbix. Suppose that nginx server goes wrong, you dont have the privlege to run anything.

So what we can do is, we customize the command and script, and on the remote server, we should enable the sudo so that it can run the command without having errors.

First allow the agent to run command

```bash
vim /etc/zabbix/zabbix_agentd.conf

AllowKey=system.run[*] # 新版没有该属性，可以在agent配置中添加
# DenyKey=system.run[*] # 新版有该属性，可以根据该属性的样式配置AllowKey的属性
# EnableRemoteCommands=1   # 开启远程执行命令,此指令在zabbix5.0版本以上淘汰
UnsafeUserParameters=1
```


Then change it to allow it to run sudo without password

```bash
vim /etc/sudoers
# NOPASSWD: 后面存在空格
zabbix ALL=(ALL) NOPASSWD: ALL
```


### Let's do it from ground up

First go to server 14, and based on zabbix official site, install the zabbix agent 2.

Also remember to configure the agent2 so that it is allowed to run command.

Add the sudoer at the same time also.



## Zabbix active and passive

Active mode means client actively transfer data to server.

Why must I accept? Only if the hostname MATCH!
