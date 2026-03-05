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


