# Java

## What is Tomcat?

Tomcat is a Java application server. 

## How does a java file run?

xxx.java -> java compiler -> java.class  -> jvm environment 

So compile once and run everywhere.  

## Java Web Technique

1. servlet: embed html code inside java program code
2. JSP: embed java code inside html

## Java Environment

JRE = JVM + Java Class Library + runtime files

JDK = JRE + development tool kit

OpenJDK  vs OracleJDK: OpenJDK is open source, while OracleJDK is for commercial use.

# Apache Tomcat

Java has WAR(Web Application Package) and JAR(Java Archive), WAR cannot completely replace JAR. 



## Installation of tomcat - binary

```bash
#Rocky 10

#FIrst install java
yum install java-25-openjdk java-25-openjdk-devel

# Install tomcat
yum install tomcat9

systemctl start tomcat9


# Uninstall tomcat
yum remove "tomcat*"
yum remove "java*"

# Clean the environemnt even more
find / -name "java*" | xargs rm -rf 
```

## Installation of tomcat - source code

```bash
#Rocky 10

# Still install java first
apt update
apt install openjdk-17-jdk -y

# Add environemnt parameter to /etc/profile.d/java.sh
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export JAVA_BIN=$JAVA_HOME/bin
export PATH=$JAVA_BIN:$PATH

# activate
source /etc/profild.d/java.sh

# Now we can deploy tomcat
wget https://archive.apache.org/dist/tomcat/tomcat-11/v11.0.4/bin/apache-tomcat-11.0.4.tar.gz
mkdir /data/server -p
tar xf apache-tomcat-11.0.4.tar.gz -C /data/server/

# Have a global environement shortcut
vim /etc/profile.d/tomcat.sh
export CATALINA_BASE=/data/server/tomcat
export CATALINA_HOME=/data/server/tomcat
export CATALINA_TMPDIR=$CATALINA_HOME/temp
export CLASSPATH=$CATALINA_HOME/bin/bootstrap.jar:$CATALINA_HOME/bin/tomcatjuli.jar
export PATH=$CATALINA_HOME/bin:$PATH

# Now you can start the tomcat pretty easily
source /etc/profile.d/tomcat.sh
./catalina.sh start 

# Make it a systemd service
useradd -r -s /sbin/nologin tomcat
chown -R tomcat:tomcat /data/server/tomcat
chown tomcat:tomcat -R /data/server/tomcat/*

# Write the service script
cat /lib/systemd/system/tomcat.service
[Unit]
Description=Tomcat
After=syslog.target network.target
[Service]
Type=forking
# Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64/
ExecStart=/data/server/tomcat/bin/startup.sh
ExecStop=/data/server/tomcat/bin/shutdown.sh
PrivateTmp=true
User=tomcat
Group=tomcat
[Install]
WantedBy=multi-user.target

# Start the service
systemctl daemon-reload

```

## Tomcat Folder Structure

```bash
#/bin: executive file
#/conf: config file
#/lib: function library
#/logs: logs
#/webapp: tomcat http homepage folder
#/work: working environment

ls /conf

#Catalina             context.xml           jaspic-providers.xsd  server.xml        tomcat-users.xsd
#catalina.properties  jaspic-providers.xml  logging.properties    tomcat-users.xml  web.xml

```

server.xml is the main config, context.xml is used to control privilege of accessing tomcat function.  web.xml is for the additional function management. 

Inside webapps, you can customize `context.xml` and `web.xml` for each website, or you can just ignore it and let the global config do the job.  `tomcat-users.xml` is used to manage if someone is trying to access some setting page but is not localhost.

![image-20260523134238585](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260523134238585.png)

In summary:

Tomcat and Nginx both can receive HTTP requests, but they are designed for different layers. Nginx is mainly a web server, reverse proxy, load balancer, and TLS termination point. It usually listens on public ports such as `80` and `443`, handles HTTPS certificates, serves static files efficiently, and uses `server_name` and `location` blocks to decide where each request should go.

Tomcat is a Java application server / Servlet container. In Tomcat, the **Connector** is the network entry point. It listens on a port, such as `8080`, `8443`, or `8009`, receives HTTP/HTTPS/AJP traffic, parses the request, and passes it into the Tomcat container. The **Engine** is the internal processing environment. After the Connector receives the request, the Engine decides which **Host** should handle it. Each Host represents a virtual website, such as `www.a.com` or `www.b.com`. If no Host matches, Tomcat uses the `defaultHost`, for example `localhost`.

So the structure in the diagram can be understood like this:

```
Server
  └── Service
        ├── Connector: receives traffic on a port
        └── Engine: routes the request internally
              ├── Host: website 1, e.g. www.a.com
              └── Host: website 2, e.g. www.b.com
```

In an Nginx + Tomcat architecture, the request flow is usually:

```
Browser -> Nginx -> Tomcat Connector -> Engine -> Host -> Web application
```

Nginx handles the public-facing side. For example, it receives HTTPS traffic on port `443`, selects the correct certificate based on the domain name, and then forwards the request to Tomcat over HTTP, often to `127.0.0.1:8080`.

Tomcat still needs a Connector because Nginx does not directly enter Tomcat’s Engine. Nginx must send an HTTP request to a listening port, and that listening port is provided by Tomcat’s Connector. After that, Tomcat’s Engine and Host configuration decide which Java web application should handle the request.

A simple comparison is:

```
Nginx listen 80/443        ≈ Tomcat Connector
Nginx server_name          ≈ Tomcat Host name
Nginx location             ≈ Tomcat Context / Servlet mapping / Spring route
Nginx static file handling ≈ Tomcat DefaultServlet
Nginx proxy_pass           ≈ forwarding to backend application servers
```

The important difference is that Nginx is configuration-driven at the web-server layer, while Tomcat is application-container-driven. Nginx uses `location` rules to choose whether to serve files or proxy requests. Tomcat uses `Host`, `Context`, `web.xml`, Servlet mappings, and frameworks like Spring MVC to decide how to handle requests inside a Java web application.

In short:

```
Nginx is the front gate and traffic controller.
Tomcat is the Java application environment.
Connector is Tomcat’s entrance.
Engine is Tomcat’s internal dispatcher.
Host is Tomcat’s virtual website.
Context is the actual deployed web application.
```

## Tomcat Log Management

### Customize Log

```bash
vim /data/server/tomcat/conf/server.xml

...
         <Valve className="org.apache.catalina.valves.AccessLogValve"
directory="logs"
prefix="localhost_access_log" suffix=".txt" pattern="
{&quot;clientip&quot;:&quot;%h&quot;,&quot;ClientUser&quot;:&quot;%l&quot;,&quot
;authenticated&quot;:&quot;%u&quot;,&quot;AccessTime&quot;:&quot;%t&quot;,&quot;
method&quot;:&quot;%r&quot;,&quot;status&quot;:&quot;%s&quot;,&quot;SendBytes&qu
ot;:&quot;%b&quot;,&quot;Query?
string&quot;:&quot;%q&quot;,&quot;partner&quot;:&quot;%
{Referer}i&quot;,&quot;AgentVersion&quot;:&quot;%{User-Agent}i&quot;}" />
<!-- pattern="%h %l %u %t &quot;%r&quot; %s %b" /> --> # 注释掉原来的pattern


tail -n1 localhost_access_log.2026-xx-xx.txt | jq
```

## Tomcat Access and Privilege

Lift the visit ip restriction

```bash
# WHat you need to change is to add the server
# For example |10\.\d+\.\d+\.\d+
vim host-manager/META-INF/context.xml
vim manager/META-INF/context.xml

vim ..conf/tomcat-users.xml
<role rolename="admin-gui"/>
<role rolename="manager-gui"/>
<role rolename="manager-status"/>
<user username="tomcat" password="s3cret" roles="admin-gui,manager-gui,managerstatus"/>
</tomcat-users>


systemctl restart tomcat.service
```

## Tomcat Port Management

By default, 8005 port is also taken by Tomcat, and it is a dangerous backdoor that allows you to shutdown the tomcat service by simply writing to the process. `vim conf/server.xml` to change the port number to -1 to disable anything from happening in the future. 

Http service always listen on 80 port. Can we change tomcat to port 80? Nope, it requires root privilege. But of course we can vim again to change it to something like 8000. 



## Tomcat Web Service Customization

### Index file, or Welcome Page

```bash
# Like nginx index, but tomcat uses the concept as welcome file
tail -n7 conf/web.xml | head -n5

   <welcome-file-list>
        <welcome-file>index.html</welcome-file>
        <welcome-file>index.htm</welcome-file>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
```

### Deploy a website

```bash
# Create the folder we want. By default it will be under tomcat /webapps
mkdir /data/web/webapps/ROOT -p

# Add index.html which should be the homepage
cat > /data/web/webapps/ROOT/index.html <<-eof
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome to Tomcat</title>
</head>
<body>
    <h1>Welcome to Tomcat by sswang</h1>
</body>
</html>
eof

# Add index.jsp, this is java code embedded in html, I think....?
cat > /data/web/webapps/ROOT/index.jsp <<-eof
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome to Tomcat JSP</title>
</head>
<body>
    <h1>Welcome to Tomcat by sswang</h1>
</body>
</html>
eof


# Make sure that the file is owned by the service account
chown tomcat:tomcat /data/web/webapps -R

# Add magedu.com, and add the following lines
vim conf/server.xml
<Host name="www.magedu.com"  appBase="/data/web/webapps"
    unpackWARs="true" autoDeploy="true">
</Host>

# Looks good?
vim /etc/hosts # You need to add the 10.0.0.13 to www.magedu.com for dns

# Now you can curl
curl www.magedu.com:8000 # Otherwise it is going to use port 80
curl www.magedu.com:8000/index.html # same thing

```

### Build war package

```bash
apt install openjdk-xx-devel # need this for the jar command

# Prepare the content
mkdir app1/dir1 -p
cp /data/web/webapps/ROOT/index.html app1/
cp /data/web/webapps/ROOT/index.jsp app1/dir1/
chown tomcat:tomcat -R app1/


# Build the war
cd app1/
jar cvf ../app1.war *
cd ..
chown tomcat:tomcat -R app1.war

# Start the deployment
rm -rf /data/web/webapps/ROOT/*
cp app1.war /data/web/webapps/


# You will find that it has been deployed. Why?
# Because of unpackWARs and autoDeploy


```

## Tomcat Reverse Proxy

There are three ways how nginx work with tomcat.

1. nginx listens on port 80, and tomcat connector listens on 8080, both deployed in the same server

![image-20260524144036655](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260524144036655.png)

2. Nginx and tomcat on difference services, but ideas are the same

![image-20260524144134369](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260524144134369.png)

3. Nginx does the proxy for two times. You can visit either severs and they would go to tomcat anyway. 

![image-20260524144212190](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260524144212190.png)

## A somewhat big Project

![image-20260524151257483](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260524151257483.png)

|![image-20260524161208916](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260524161208916.png)

This is a typical load balancing method

```config
upstream tomcat_group {
  server 10.0.0.13:8080;
  server 10.0.0.14:8080;
  server 10.0.0.15:8080;
}

server {
  listen 80;
  server_name sswang.magedu.com;
  location / {
    proxy_set_header host $http_host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://tomcat_group;
  }
}

```

Nginx add https

```config

upstream tomcat_group {
  server 10.0.0.13:8080;
  server 10.0.0.14:8080;
  server 10.0.0.15:8080;
}


server{
         listen 80;
         server_name sswang.magedu.com;
         return 302 https://$host$request_uri;
        }

server{
         listen 443 ssl;
         server_name sswang.magedu.com;

         ssl_certificate /usr/share/easy-rsa/3/pki/sswang.magedu.com.pem;
         ssl_certificate_key /usr/share/easy-rsa/3/pki/private/sswang.magedu.com.key;
         ssl_session_cache shared:sslcache:20m;
         ssl_session_timeout 10m;

         location / {
         proxy_pass http://tomcat_group;
         proxy_set_header host $http_host;
         }
}

```

## HTTPS Reverse Proxy

First, edit for all the ROOT in difference servers, add test.jsp to it.

```bash
upstream tomcat{
  server 10.0.0.13:8080;
  server 10.0.0.14:8080;
  server 10.0.0.15:8080;
}

server{
 listen 80;
 server_name sswang.magedu.com;
 return 302 https://$host$request_uri;
}
server{
 listen 443 ssl;
 server_name sswang.magedu.com;
 ssl_certificate /usr/share/easy-rsa/3/pki/sswang.magedu.com.pem;
 ssl_certificate_key /usr/share/easy-rsa/3/pki/private/sswang.magedu.com.key;
 ssl_session_cache shared:sslcache:20m;
 ssl_session_timeout 10m;
 location ~* \.jsp$ {
 proxy_pass http://tomcat;
 proxy_set_header host $http_host;
 }
}

```

## Tomcat Session

We have setup a nginx that will pass request to different tomcat server for load balancing. However, https relies on a session ID that is very short-term. 

How to keep the session id?  1. Bind, if a request talk to a server, it will forever talk to that server 2. Duplicate, every server has every session information. This will allow client to connect to random one.   

Please refer to document, it is kind of a lot

## Tomcat + Redis

Check document, it is basically copy and paste. 





## Tomcat Performance Optimizaiton

two parts: tomcat optimization / java optimization

The most important part is Garbage Collection. There are two ways to find out...

JVM will group objects into new-born and elderly, and use different GC strategy.

When you do eldery GC, you will gc the youngest, so this is a full GC, a.k.a stop the world.

Tow methods: 1. Make the GC duration very short. 2. Makes the interval between GC very long.



Both needs JVM parameters.

