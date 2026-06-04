# Using online source to build

After the source code is downloaded, there will be a pom.xml, and it will build the target file using the local file and module server. After the war package is build, it can be deployed to the tomcat folder.  



There are 3 files

```
1 /etc/maven/setting.xml # global maven env
2 ~/.m2/setting.xml # user maven env
3 project_name/pom.xml # project maven env
```



## Install Maven

```bash
#Make sure java is installed
java -version
apt install maven -y
```

## Change the pom.xml

Nothing to say here, main thing you can do is to change the source to aliyun.

## Start the build

```bash
mvn clean # clean the previous build
mvn package # start building the package

# Or you can do it together
mvn clean package

# You can skip the test by add option
mvn clean install package -Dmaven.test.skip=true
```

# Using offline source to build

## install Nexus

```bash
# You still need to install java first
java -version

# By default, nexus listen on port 8081.
vim nexus.rc # this is used to config what user you used to run the service

# Start the service
./nexus start
./nexus status

```

## Create systemd service

```bash
vim /lib/systemd/system/nexus.service

[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/data/server/nexus/bin/nexus start
ExecStop=/data/server/nexus/bin/nexus stop
User=root
#User=nexus
Restart=on-abort

[Install]
WantedBy=multi-user.target


```

## Nexus management policy

Three ways:

1. Proxy  This takes cache from an upstream repo. So Nexius will check local cache, if not it will download from pypi.
2. Host  This file is in the local directory
3. Group It routes/searches against the member repos.

## Make the java use nexus repo as source

```bash
# Copy the repo url
http://10.0.0.16:8081/repository/maven-public/

# Config this 
vim /etc/maven/settings.xml
vim project_name/pom.xml # either is okay, just changed the mirror part, pretty self-explanatory

# Start using it
mvn clean package
```

## Manage the Blob 

Pure UI.

```bash
# Go change the source 

vim /etc/apt/sources.list.d/ubuntu
```

# Upload war to the nexus for artifact control

We can create mymave(proxy), mymaven(hosted) for release, mymaven(hosted) for snapshot, then add all of them to mymaven-group. 

Now we need to configure maven

```bash
vim /etc/maven/settings.xml

    <server>
      <id>mymaven-snapshot-hosted</id>
      <username>admin</username>
      <password>12345678</password>
    </server>
    <server>
      <id>mymaven-hosted</id>
      <username>admin</username>
      <password>12345678</password>
    </server>
    <server>
      <id>mymaven</id>
      <username>admin</username>
      <password>12345678</password>
    </server>


mvn deploy:deploy-file \
        -DgroupId=com.neo \
        -DartifactId=spring-boot-helloworld \
        -Dversion=0.6-SNAPSHOT \
        -Dpackaging=jar \
        -Dfile=/data/codes/spring-boot-helloworld/target/spring-boot-helloworld-0.6-SNAPSHOT.jar \
        -DpomFile=/data/codes/spring-boot-helloworld/pom.xml \
        -Durl=http://10.0.0.16:8081/repository/mymaven-snapshot-hosted/ \
        -DrepositoryId=mymaven-snapshot-hosted

```

