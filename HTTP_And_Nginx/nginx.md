#Nginx

## Blocking Type
*Blocking IO: self explanatory
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