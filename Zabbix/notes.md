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
