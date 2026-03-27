## What is DNS?

Domain Name System 

There are long domain, short domain and fqdn etc...

For example, www.magedu.com.

Here the `www` is hostname, `magedu.com` is a domain name, and the whole is fqdn.

com -> Top-Level Domain

magedu -> Second-Level Domain

www -> Subdomain

. -> Root

. and magedu.com is managed by official organization and registrar

www and the child domain before it is managed by the company

## DNS resolution

1. Open the browser and enter the domain name
2. Browser will check cache and directly get the ip address
3. If no cache then go and find os dns cache -> hosts file
4. Go to the web drive and check the local dns address
5. if Local DNS can connect to INternet, it will do the search
6. If not, it will go and find inside DNS
7. Ask dns server, where is com.?
8. Give an address to TLD Server
9. Keep asking, give address to SLD server
10. Ip address found successfully 
11. TCP 3 times hand shake.

## DNS record
DNS record are data entries that was input into DNS server.

```
| IP Version | DNS Record      | Purpose                    | Example                     |

| ---------- | --------------- | -------------------------- | --------------------------- |

| IPv4       | **A record**    | Maps domain → IPv4 address | `example.com → 192.168.1.1` |

| IPv6       | **AAAA record** | Maps domain → IPv6 address | `example.com → 2001:db8::1` |

```
Also there is PTR record, means translate IP address to domain name.

NS record: prove that this server is dns server.

SOA record: In any DNS server, the first record is SOA record.

MX record: Mail setting

SRV record: Find service setting

Zone: the file that writes the dns record.


Example: I visit A webiste, but it gives me B. How? it is done by CNAME record.

## How to set up DNS server?

1. There are many public DNS servers, provided by different companies.
2. How many DNS servers can you set up for each machine? Well, many... 


ubuntu: nameservers

rocky: dns

openeuler: DNS

file for temp: /etc/resolv.conf

## How to check DNS address?

`resolvectl` depends on the service `systemd-resolved`

`dig` `nslookup` `host`


## How to check cache

still `resolvecrl statistic`


## Hands-on
If you check ubuntu port, you will realize that systemd-resolve is using port 53. 

This means that ubuntu is giving the task of dns to that port.

`vim /etc/systemd/resolved.conf` to change dns

`vim /etc/hosts` to change also, acutally this is of higher priority

`dig` `nslookup` `host`: only used when you are building your own dns server, and you want to check whether the resolution is normal.

`dig +short www.google.com @dns_server` This is the most common use, without short the returned information might be too much.

`host` and `nslookup` are basically the same commands.

`whois jd.com` to check website registration information.



## DNS server environment setup

Many software can achieve such function, like bind, nsd, knot etc...

Rocky:
* Bind (named)
* port: 127.0.0.1:53 Only local host
* /etc/named.conf
* /var/named


Ubuntu:
* bind9(bind9, named)
* 10.0.0.13:53
* /etc/bind/named.conf
* /var/cache/bind

## Configuration
* Master file: /etc/bind/named.conf
1. /etc/bind/named.conf.options
2. /etc/bind/named.conf.local
3. /etc/bind/named.conf.default-zones


Only the **default-zones** file is not empty by default.

Default zone starts the information with the following format.

```bash
zone "localhost" {
  type master|slave;
  file "/etc/bind/db.local"
}
```
Here the db.local is like logs for resolution records. 


How do we customize dns setting?
1. Add zone file
2. Add resolution data file
3. Check the grammar
4. Restart the service
5. Use `dig` to test


When we check db.local, we can see the following content

```bash
; BIND data file for local loopback interface

;

$TTL    604800

@       IN      SOA     localhost. root.localhost. (

                              2         ; Serial

                         604800         ; Refresh

                          86400         ; Retry

                        2419200         ; Expire

                         604800 )       ; Negative Cache TTL

;

@       IN      NS      localhost.

@       IN      A       127.0.0.1

@       IN      AAAA    ::1
```

What does this content mean?

Here the `@` represents the secondary domain, which is localhost here.

`$TTL` means the time to live, `IN` is big category, we do not need to care about that, `SOA` is the detailed DNS record type, and `localhost.` is purely descriptive information. `root.localhost` is the email address of the admin, here it can be transalted into `root@localhost`. 

`Serial` here means the version number, the slave wil check with master to see whether some information has been changed, if nothing is changed, the slave will just use cache. 



## Practical setup for dns

### 1. DNS Self-Resolution

![image-20260321102257979](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260321102257979.png)

1. Go to 13, install the bind software, Change zone information and add db information to the config.

```bash
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     magedu-dns. admin.magedu.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
                NS      dns1
dns1            A       10.0.0.13
www             A       10.0.0.13
*               A       10.0.0.13

```

NS defines dns1 as the DNS server, and dns1 binds to 10.0.0.13.  Any `www.magedu.com` or `blog.magedu.com` will use 10.0.0.13 as the dns server. 

### 2. DNS Master/Slave structure

![image-20260321113815597](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260321113815597.png)

1. Install bind on 15
2. disable the firewalld
3. Change the zone information and Create the slave db confiuration

![image-20260321114418568](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260321114418568.png)

4. Start the service

### 3. Reverse DNS Lookup

Give me an ip address, I will give you the domain name.

For example, for an A record, it would be  `www.example.com    10.0.0.15`

Then, for a PTR record (the reverse lookup), it would be `15.0.0.10.in-addr-arpa.   www.example.com`

Add information to zone file

```bash
zone "0.0.10.in-addr.arpa" {
        type master;
        file "/etc/bind/db.0.0.10.in-addr.arpa";
};
```



### 4.DNS Subdomain

![](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260321135758763.png)

This looks complicated but all you need to do is to just two more zone information into your conf file.

![image-20260321140203725](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260321140203725.png)

### 5.DNS forwarding

First Mode

![image-20260321140656817](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260321140656817.png)

Only Mode

![image-20260321140851557](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260321140851557.png)

The only mode will not do recursion, it will just fail.

If you want to forward globally, you need to make change to `named.conf`

If you only want to forward subdomain, you need to change the zone setting.

The way to configure it is simper simple. You just do the normal dns setup on the two servers, then you change the first server general setting as follows:

![image-20260322002353391](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20260322002353391.png)







## Authoritative answer and unauthoritative answer

The result directly given my the dns server within your domain is the authoritative answer, and if your dns server asks for backup like `8.8.8.8`, the is called unauthoritative answer.

If it is authoritative answer, there will be `aa` tag in flags.

