## http protocol

After the ip address was found by dns server, and the tcp goes through handshake (ssl can be added here to encrypt the message), the http will be established on it between two applications. 

A url is made of `protocol://domain/file_to_path` The default port is 80, therefore it can be omitted.

A http feedback can contain http header and http body.

```bash

* Host www.baidu.com:80 was resolved.
* IPv6: (none)
* IPv4: 103.235.46.115, 103.235.46.102
*   Trying 103.235.46.115:80...
* Connected to www.baidu.com (103.235.46.115) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: www.baidu.com
> User-Agent: curl/8.12.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
< Content-Length: 2381
< Content-Type: text/html
< Pragma: no-cache
< Server: bfe
< Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/
< Date: Sun, 12 Apr 2026 06:31:54 GMT
<
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8>...

```

Here the curl gives dns info, tcp 3-times handshakes info and also the http header.   You can just check the http header using `curl -I www.baidu.com`

When you type `www.baidu.com` in your browser, the browser will automatically point to `www.baidu.com/` , which points to the `index.html` file. 

Workflow:

1. Send Request
2. Receive Request
3. Process Request
4. Serialize and send data
5. Receive data
6. Render the browser
7. Close connection(client)
8. Close connection(server)

![image-20260412164726017](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260412164726017.png)

DNS is at the risk of hijacking. A workaround is instead of using local DNS server, we will just use https which is very safe, to connect to a httpdns server, then receive the ip address.

To solve the HTTP stateless problem, we introduced `session` and `cookie`

## Session Management

1. Sticky sessions: Server deal with specific session id, if it goes to A it always goes to A
2. Session duplicates: Copy the session info across the servers
3. Session share: Store the session info inside a very quick database

## Apache

It has different work modes:  `prefork`, `worker`, `event`.   

### `prefork`

Use a main process that has many child processes, every child process has one thread that responds to client request

The advantage is it is stable, but it takes a lot of resources.



### `worker`

Use a main process that start child processes, each child process has many threads to respond to client request

The advantage is thread can reuse resource, the disadvantage is it uses `keepalive` way of connection, if too much request it will stuck, this is same as prefork.



`event`

Same as worker, but there is a monitoring thread in each child process that manage the thread responding



## Apache installation

### Rocky

```bash
yum install httpd
# Listening port: 0.0.0.0:80
# Web /var/www/html
# Config: /etc/httpd/conf/httpd.conf
# Child Config: /etc/httpd/conf.d
# Service name: httpd
# User: apache (33)
```

### Ubuntu

```bash
apt install apache2
# Listeninig port: 0.0.0.0:80
# Web /var/www/html
# Config: /etc/apache2/apache2.conf
# Child Config: conf-xxx site-xxx
# Service name: apache2
# User: www-data (38)
```

### How to start a website?

1. Use Directory to allow apache to access
2. The directory itself must be accessible to apache child process user.

## IO

![image-20260415200613031](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260415200613031.png)

CPU controls everything, but IO related stuff is not that important, so it uses DMA (Direct Memory Access). 

CPU only intervene before and after transportation happen, but during the transportation it is controlled by DMA. 



Think about this scenario: I have a process that is listening on a port, and it needs to understand whether the data has come to this port. This give rises to: blocking, nonblocking, synchronous, asynchronous. 

### Blocking and nonblocking

Blocking and nonblocking is taking the perspective from the process side. Blocking means it is waiting for result, nonblocking means I am still doing other things during the meantime. 

![image-20260416202625087](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260416202625087.png)

The problem here is, you can use a man in the middle, so that it will check whether the data is ready, and copy the data to user space. How much the middle man did is kind of a strategy. You can let him do the full copy, or you can just let him send a signal when the data is ready. 
