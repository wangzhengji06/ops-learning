## Ansible variables

### System variable

 Scenario: One command run by multiple hosts, the final effect is completely the same
 
 ```bash
# I just want to modify 10.0.0.12
ansible web -l 10.0.0.12 -m hostname -a "name=rocky10-12" 
 ```
 
 Now let's start using the variables.
 Here we use `ansible_fqdn` to get the default system variable of hostname.
 
 ```yaml
- hosts: all
  remote_user: root
  tasks:
    - name: create file
      file: name=/var/log/var-{{ ansible_fqdn }}.log state=touch owner=daemon
 ```


### Host list

modify the host file...
```bash

[web]
10.0.0.12 hostid=master
10.0.0.13 hostid=node1
10.0.0.15 hostid=node2
10.0.0.16 hostid=node3
10.0.0.19 hostid=node4
```


Now we can use the variable inside host file

```yaml
- hosts: web
  remote_user: root
  tasks:
    - name: set hostname
      hostname: name=ansible-{{ hostid }}.example.com
```

Actually to make our life easier

```bash
[web]
10.0.0.12 hostid=master
10.0.0.13 hostid=node1
10.0.0.15 hostid=node2
10.0.0.16 hostid=node3
10.0.0.19 hostid=node4
[web:vars]
head=ansible
tail=.magedu.com
```

Here it means head and tail varaibles only belong to web group

```yaml
- hosts: web
  remote_user: root
  tasks:
    - name: set hostname
      hostname: name={{ head }}-{{ hostid }}{{ tail }}
```

Also normal variables have higher priority than global variables
```bash
10.0.0.12 hostid=master
10.0.0.13 hostid=node1
10.0.0.15
10.0.0.16
10.0.0.19
[web:vars]
head=ansible
tail=.magedu.com
hostid=node
```

Here 12 and 13 will still have master and node1 for its hostid, while 15 16 19 will have node as its hostid.
