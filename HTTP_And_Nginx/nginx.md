# Nginx

## Blocking Type
* Blocking IO: self explanatory
*Nonblocking IO: the user keep asking whether the data is ready
*I/O Multiplexing: One thread/process listening, one done it will notify users
*Signal drivien I/O: once done kernel will send signal
*Async IO: the kernel will wait and copy and paste data

## Multiplexing 3 modes
First, what is fd?

It is used to represent an I/O object for process. 0, 1, 2 stands for standard input/output.


1. Select mode: Copy all the fd lists to the kernel, kernel will iterate through again and again, and if some is ready, it will be set to 1, and copy back to the user space. The disadvantage is that, this is time costly.
2. Poll mode: No limit of fd list length. Also, not only ready is described, whether it is read or write will also be described.
3. Epoll: No need to copy the whole list, there is already a binary tree structure. So it is really good Huge concurrency requirement.


## Installing Nginx using binary

*For installation, apt/yum install nginx -y
*the port number is 80
*By default the nginx main works number = cpu numbers
*Main Configuration file: /etc/nginx/nginx.conf
*Child Configuration file: /etc/nginx/conf.d sites-xxx

## Installing Nginx from source code
*Install environment (if you are installing on ubuntu, you need to create a soft link for libperl.so)
*configure 
*make
*make install

## Transform the source built Nginx to systemd service
```bash
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target
[Service]
Type=forking
#PIDFile=/data/server/nginx/run/nginx.pid
ExecStart=/data/server/nginx/sbin/nginx -c /data/server/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
LimitNOFILE=100000
[Install]
WantedBy=multi-user.target
```

## Nginx Command

```bash
nginx -v # show the version
nginx -t # Show configuration files, also check grammar!!!!!
nginx -T # dump and show all properties
nginx -c # set configuration file
nignx -s # send signal
nginx -s stop / quit # Stop will stop the download as well, while quit is clean quit that wait until all the connections from client is cut
```

## Nginx Configuration
```bash
useradd nginx
./configure -prefix=/data/server/nginx --user=nginx --group=nginx
chown nginx:nginx -R /data/server/nginx
```

Nginx depends on modules. You need to add extra modules for some function.

1. Get the current configuration of nginx.
2. Get the third-party module
3. Reconfigure
4. Compile
5. Use execution file

## An example work through
```bash
cd /data/softs

wget -O echo-nginx-module-v0.63.tar.gz \
  https://github.com/openresty/echo-nginx-module/archive/refs/tags/v0.63.tar.gz  # Download the echo module

tar xf v0.63.tar.gz -C /usr/local

# There are two ways to manage this
# 1. You tell nginx where to find the 3-rd party modules, and you add it as an option to the nginx execution file
# 2. Directly add 3rd party modules into nginx execution file

./compile ..... --add-module = /usr/local/echo-nginx-module-0.63

make # Please do not make install, it will overwrite a lot of configurations

mv /usr/sbin/nginx /usr/sbin/nginx-1.24

cp obj/nginx /usr/sbin/nginx
```

## Nginx Configuration overview

Nginx can be level 7 proxy or level 4 proxy, means it can act for http protocol or tcp protocol.

It is mainly used for DNAT.

For difference sites, it can use different server configuration settings.  In each server configuration field, it can have mutiple location configuration fields(url).

It also has global setting for managing nginx start up. It also has event setting for managing nginx large-scale connections.

Stream field is used for level 4 proxy, regrading to port trannsfer(tcp protocol).



Mail field is used for email configuration.


## Nginx configuration in detail

1. Overall setting

As you can see it just defines the user etc

Another thing you can set up is worker_cpu_affinity, which wiill bond a worker node to a specific cpu core. This is used to avoid os to switch cpu to different works thus losing valuable cache. You can also set daemon to be off in the container setting.

```bash
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;
```

2. Event setting
The following configuration allows 768 connections for each worker node.
```bash
events {
        worker_connections 768;
        # multi_accept on;
}
```

3. Http setting

send-file on + tcp_nopush on -> More efficient transmission

charset utf8 -> Render the encoded message in utf8

```bash
http {
        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;


        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;


        access_log /var/log/nginx/access.log;


        gzip on;


        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}


```
4. Server settings

```bash
server {
    listen 80;
}
```

5. Location settings
Every site has lots of specific files
Every specific file has a location file to manage them.

What does this do?

```bash
server{
    listen 80;
    root /data/server/nginx/web1;
    location / {
       try_files $uri $uri/ = 404;
}
```

When you type www.a.com, inside browser, you are accessing http://www.a.com/

Then you will have dns, then 3 times handshake.

Then you connect to nginx server, which usually exposes at 80. 

The location sais, go and find $uri, which is / the root, so it will try to find / as a file. or try to find / as a directory and look for index.html, which is hte default defined in our config. 

By default, nginx will always try to find under root + url. 

## Nginx name-based virtual hosting
You want multiple page to be accessed, but you do not want add too many webdrivers also you do not want too many ports taken.

You can do the following, this makes the www.c.com to be the default page

```bash

server {
  listen 80;
  server_name www.a.com;
  root /data/server/nginx/web1;
}
server {
  listen 80;
  server_name www.b.com;
  root /data/server/nginx/web2;
}
server {
  listen 80 default_server;
  server_name www.c.com;
  root /data/server/nginx/web3;
}
```

Remember to also add to /etc/hosts the name-server.

## Alias in Nginx
alias is like a hard map from location url to actual location in folder

For example, `/img/` is alised to `/var/www/image`, then `img/a.txt` would be `/var/www/image/a.txt`

But if the root is `/var/www/image`, then `img/a.txt` should be `/var/www/image/img/a.txt`

## Conflict of url in Nginx

```
=
^~
~
~*
/str
\
```

The prority goes down as it goes down.

## Logging in Nginx

Nginx logs can automatically extract information from HTTP request/response body and head. 

```bash
# I want to customize the log
vim /etc/nginx/nginx.conf

# Add the following lines to log

cat > log <<-eof
        #自定义访问日志格式 - 字符串
        log_format basic '\$remote_addr - \$remote_user [\$time_local]
"\$request" '
                  '\$status \$body_bytes_sent "\$http_referer" '
                  '"\$http_user_agent" "\$http_x_forwarded_for"';
                  
        #自定义访问日志格式 - json 字符串  
        log_format json_basic '{"remote_addr": "\$remote_addr", '
                      '"remote_user": "\$remote_user", '
                      '"time_local": "\$time_local", '
'"request": "\$request", '
                      '"status": "\$status", '
                      '"body_bytes_sent": "\$body_bytes_sent", '
'"http_referer": "\$http_referer", '
                      '"http_user_agent": "\$http_user_agent", '
                      '"http_x_forwarded_for": "\$http_x_forwarded_for"}';
eof

# vhost.conf
cat > /etc/nginx/conf.d/vhost.conf <<-eof
server {
  listen 80 default_server;
  access_log /var/log/nginx/\${host}_access.log basic;
  
  location /web1/ {
    alias /data/server/nginx/web1/;
  }
  # 此资源记录json日志
  location /json{
    access_log /var/log/nginx/\${host}_json_access.log json_basic;
    return 200 "json\n";
  }
  #不记录access log
  location /test{
    access_log off;
    return 200 "test\n";
  }
}
eof


## Now we can add different log files for different location. Make sure the file is owned by the user defined in nginx.conf. 

chown -R www-data:www-data /var/log/nginx/
```

## Nginx status Page

```conf
location = /basic_status {
	stub_status;
}
```

 ## Nginx Https 

```bash
# This allows the eays installation of CA certificate
apt install easy-rsa -y

# Initialize a pki folder to manage the private key and certificate
cd /usr/share/easy-rsa/
./easyrsa init-pki

# Build a CA agency, this will create crt and private key
./easyrsa build-ca nopass

# Generate a request for this website to the CA certificate and sing it
./easyrsa gen-req sswang.magedu.com nopass
./easyrsa sign-req server sswang.magedu.com

# This is because nginx usually use chains, Root CA -> Intermediate CA cert-> Website cert
# Usually browser trust the root ca, 
cat pki/issued/sswang.magedu.com.crt pki/ca.crt > pki/sswang.magedu.com.pem

# Redirect http to https
cat > /etc/nginx/conf.d/vhost.conf <<-eof
server {
  listen 80 default_server;
  server_name sswang.magedu.com;
  root /data/server/nginx/web1;
  # return 301 https://$host$request_uri; # 301重定向
  rewrite ^(.*) https://\$server_name\$1 permanent; #rewrite 重定向，二选一
}
server{
  listen 443 ssl;
  server_name sswang.magedu.com;
  root /data/server/nginx/web1;
  # 定制ssl的能力
  ssl_certificate /usr/share/easy-rsa/pki/sswang.magedu.com.pem;
  ssl_certificate_key /usr/share/easy-rsa/pki/private/sswang.magedu.com.key;
  ssl_session_cache shared:sslcache:20m;
  ssl_session_timeout 10m;
}
eof


# Add hosts
vim /etc/hosts

# Test https
curl -L swang.magedu.com # This will show that the https certificate has problem, because the server has not trust the CA certificate yet


```

## Nginx Rewrite

Rewrite the url

```config
server {
  listen 80 default_server;
  server_name sswang.magedu.com;
  root /data/server/nginx/web1;
  # return 301 https://; # 301重定向
  rewrite ^(.*) https://$server_name$1 permanent; #rewrite 重定向，二选一
}
```

For http://xxx.com/ it will rewrite to https://xxx.com/, here the $1 means everything behind the domain name in the url. 

Another example. /var/www/html/xx/aa/index.html, you can use `rewrite ^/app1/(.*) https://www.a.com/xx/aa$1`

Then entering `www.a.com/app1/index.html` will be rewritten to `https://www.a.com/xx/aa/index.html`. This is more flexible than alias because it does not only limit to directory.

## Nginx Redirect

There are two methods to do nginx redirect....

```config
# Method 1
return 30x address

# Method 2
rewrite
```

A nginx config could look like the following

```config
server {
  listen 80 default_server;
  root /data/server/nginx/web1;
  location /1.html {
    rewrite /1.html /2.html; # 访问1跳转到2
    rewrite /2.html /3.html; # 访问2跳转到3
  }
  location /2.html {
    rewrite /2.html /a.html; # 访问2跳转到a
  }
  location /3.html {
    rewrite /3.html /b.html; # 访问3跳转到b
  }
  location /c.html {   rewrite /c.html /3.html; # 访问c跳转到3
    rewrite /3.html /1.html; # 访问2跳转到a
    rewrite /1.html /c.html; # 访问1跳转到c
  }
}

```

Also 301 and 302 are actually different. 301 means permanent redirect, 302 means temporary redirect. The difference is, in the future it is 301, you will just skip and go to the redirected website because the cache will have that information. 302 would not make you skip this step.



## Nginx Reverse Proxy

```config
server {
  listen 80 default_server;
  root /data/server/nginx/web1;
  location / {
    proxy_pass http://10.0.0.13:81; # 交给后端的81端口服务
    proxy_connect_timeout 2s; # 设置连接超时时间为2s
  }
  location /1 {
    proxy_pass http://10.0.0.13:88; # 交给后端的88端口服务
  }
}
server {
  listen 81 default_server;
  root /data/server/nginx/web2;
  access_log /var/log/nginx/access_client.log;
}

```

_![image-20260518200717032](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260518200717032.png)

Suppose I want to achieve this kind of system.

```bash
cat > /etc/nginx/conf.d/vhost.conf <<-eof
server {
  listen 80 default_server;
  server_name sswang.magedu.com;
  root /data/server/nginx/web1;
  location /api {
    proxy_pass http://10.0.0.16:8080;
    proxy_set_header Host "api.magedu.com";
  }
  location /static {
    rewrite ^/static(.*)\$ /index.html break;  # 重写url
    proxy_pass http://10.0.0.14;
    proxy_set_header Host "static.magedu.com";
  }
  location /static1/ {
    # /static1/index.html 转发给后端 /index.html
    proxy_pass http://10.0.0.14/;    # 以/为结尾
    proxy_set_header Host "static.magedu.com";
  }
}
eof

```

The third way works just like alias

Also, because nginx essentially stops the tcp and start new tcp every time. the ip address is not seen by the server backend. We need to do ip forwarding. An example is as follows.

```config
server {
    listen 80;
    server_name example.com;
    location / {
        # 用于记录客户端的真实IP地址。
        proxy_set_header X-Real-IP $remote_addr;
        # 用于记录经过的代理服务器的IP地址列表，最左边的为客户端IP，右边为经过的代理服务器IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # 用于记录原始请求的协议（http或https）
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Nginx load balancing

How to add servers set

```config
upstream web_group {
	server ip1:port; weight=5;
	server ip2:port; weight=3;
	server ip3:port; weight=2
}
```

There are different strategies

1. RR, WRR basically equally dispatch the requests to server, or according to a fixed ratio
2. Scip, Dtip, if the request comes from this server, I will hand it to that server for sure
3. I will consider what is currently the condition for each server and act accordingly

Upstream itself has the ability to detect whether a server is available, when a server die or revive, the nginx can update by dispatching the request or not.

ip_hash would calculate the hash value for each ip address.

```config
upstream web_group{
    ip_hash;
    server 10.0.0.14 backup;
    server 10.0.0.15;
    server 10.0.0.16;
}


server {
    listen 80;
    location / {
      proxy_pass http://web_group;
 }
}

```

`backup` here means only when other servers are down will it start working. 

`down` means it will not work anymore.

## Nginx Level 4 Reverse Proxy

Not proxy the http service, but proxy the tcp connection etc..

```config
upstream mysql_group{
  server 10.0.0.16:3306;
}

server{
    listen 3306;
    proxy_pass mysql_group;
}

```

## Nginx FastCGI

CGI: common gateway interface 

This gateway is at the application level, not network level. Nginx web usually handles static web resources. What if some part of website needs dynamically handling? Nginx supports FastCGI to handle communication between backend program(php-fpm) and Nginx./l:

![image-20260519213211752](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260519213211752.png)

The so-called LMNP framework. Nginx is responsible for static, php is responsible for api, mysql is responsible for database. 



```bash
cat > /etc/nginx/conf.d/vhost.conf <<-eof
upstream phpserver {
    server 127.0.0.1:9000;
}
server {
  listen 80 default_server;
  root /data/server/nginx/web1;
  #反向代理 php
  location ~ \.php$ {
    fastcgi_pass phpserver;
    fastcgi_index index.php;
    fastcgi_param  SCRIPT_FILENAME  \$document_root\$fastcgi_script_name;
    include fastcgi_params;
  }
  #定制首页
  location / {      
     index index.php;
     autoindex on;
  }
}
eof
```

## Nginx Development

### Tengine

### Openresty



## Nginx + Django / Flask / FastAPI





## Nginx Smooth Upgrade

How to upgrade without shutting down the service?  Smooth upgrade! one version by one version. s

```bash
old master
 ├─ old worker 1
 ├─ old worker 2
 └─ new master
     ├─ new worker 1
     └─ new worker 2
```
