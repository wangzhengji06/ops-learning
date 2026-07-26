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


You can also use the following way to define variable. This has the highest priority.

```bash
ansible targethost -e variable=variable 
```


You can also define varaibles in ansible playbook under vars section. It has highest priority that the host file, but lower priority compared to cli.

```yaml
- hosts: web
  remote_user: root
  vars:
    head: www
    hostid: node
    tail: .ansible.top
  tasks:
    - name: set hostname
      hostname: name={{ head }}-{{ hostid }}{{ tail }}
```

This will overwrite what is written inside the hostfile. 

You can also make a dedicated vars file

```yaml
head: ansible
tail: .magedu.edu
```

```yaml
osts: web
  remote_user: root
  vars_files:
    - vars.yaml
  vars:
    - head: www
    - tail: .ansible.top
  tasks:
    - name: set hostname
      hostname: name={{ head }}-{{ hostid }}{{ tail }}
```

This has higher priority than using vars.

So in summary:

```
using e > vars file > vars > hostfiles each line > hostfiles variable section
```

## Template file

One file template can excute different contents on different servers.

- Level 1: task level
  One playbook runs on rocky and ubuntu, automatically choose different tasks to run
- Level 2: file level
  Same template file (nginx-define.conf)
  server a: two server fields
  server b: three server fields, 


level 1 uses mainly when and with. level 2 uses {%for%}, {%if%}. 


Why do we even need template file?
Becuase we want config file content to also change!

How to use?

1. Use jinja2 to write template
2. Use template module to use the template. Here template module has exactly the same usage as copy module.


### Principles
1. principles should be defined under the same folder of playbook.yml, inside ./templates
2. template config file: put under ./templates, the extension name must be '.j2'.
3. automatic replace the data: thanks to the greant jinja2

## Practrice: Nginx using template

First let's clean up the environment

```bash
ansible web -m service -a "name=nginx state=stopped"
ansible web -m apt -a "name=nginx,nginx-common state=absent"
ansible web -m file -a "path=/data/webserver state=absent"
ansible web -m user -a "name=nginx-test state=absent"
```

Then let's make the folder

```bash
mkdir /data/ansible/playbook/templates/templates -p
cd /data/ansible/playbook/templates
```

Let's customize the host this way

```bash
[web]
10.0.0.13 nginx_port=81
10.0.0.16 nginx_port=82
10.0.0.19 nginx_port=83
```

Now let's write template file
```bash
touch templates/nginx-define.conf.j2

server {
  listen {{ nginx_port }};
  root /data/webserver/html;
  location / {
  }
}

```


Now we can start creating the playbook

```bash
vim /data/ansible/playbook/templates/01-playbook-nginx-templates.yml

- hosts: web
  remote_user: root
  tasks:
    - name: install package
      apt: name=nginx state=present
    - name: create web root
      file: name=/data/webserver/html state=directory
    - name: touch web index
      shell: echo '<h1>welcome to ansible</h1>' > /data/webserver/html/index.html
    - name: delete default nginx conf
      file: name=/etc/nginx/sites-enabled/default state=absent
    - name: copy config
      template: src=./templates/nginx-define.conf.j2
dest=/etc/nginx/conf.d/nginx-define.conf
      notify:
        - restart nginx
    - name: start service
      service: name=nginx state=started enabled=yes
  handlers:
    - name: restart nginx
      service: name=nginx
```


## Use condition
when clause can be defined for a specific task. The target of when can be variables, facts or the result of the commands. 

### Scenario: Nginx installation
When installing nginx, if the remote host already installed the nginx, then imply nginx installed, else install nginx software.

Suppose now only 10.0.0.19 has nginx installed.

```bash
vim 02-playbook-nginx-when-yml

- hosts: web
  remote_user: root
  tasks:
    - name: ps_check
      shell: ps -ef | grep nginx | grep -v grep|wc -l
      register: nginx_num
      changed_when: false
      ignore_errors: true
    - name: print debug message
      debug: msg="System {{ inventory_hostname }} has nginx service."
      when: nginx_num.stdout != "0"
    - name: install package
      apt: name=nginx state=present
      when: nginx_num.stdout == "0"
    - name: create web root
      file: name=/data/webserver/html state=directory
    - name: touch web index
      shell: echo '<h1>welcome to ansible</h1>' > /data/webserver/html/index.html
    - name: delete default nginx conf
      file: name=/etc/nginx/sites-enabled/default state=absent
    - name: copy config
      template: src=./templates/nginx-define.conf.j2 dest=/etc/nginx/conf.d/nginx-define.conf
      notify:
        - restart nginx
    - name: start service
      service: name=nginx state=started enabled=yes
  handlers:
    - name: restart nginx
      service: name=nginx state=restarted
```

## Iteration/Loop

### most simple loop
You can use `with_items` to start looping

#### Scenario: Create mutiple users on each server
```bash
vim 03-playbook-iteration.yml

- hosts: web
  remote_user: root
  tasks:
    - name: add usergroup
      group: name=webgroup state=present
    - name: add several users
      user: name={{ item }} state=present groups=webgroup
      with_items:
        - testuser1
        - testuser2

```


### dictionary loop

```bash
vim 06-playbook-iteration-users.yml

- hosts: web
  remote_user: root
  tasks:
    - name: add some group
      user: name={{ item }} state=present
      with_items:
        - group1
        - group2
        - group3
    - name: add some users
      user: name={{ item.name }} group={{ item.group}} state=present
      with_items:
      - { name: 'user1', group: 'group1' }
      - { name: 'user2', group: 'group2' }
      - { name: 'user3', group: 'group3' }
```


## template if and loop

### if case

Change the /etc/ansible/hosts to the following:
```bash
[web]
10.0.0.13 port=81 name=vhost1.magedu.com
10.0.0.16 port=82
10.0.0.19 name=vhost3.magedu.com
```

Then define a template

```conf
server {
  {%if port is defined %}
  listen {{ port }};
  {% endif %}
  {%if name is defined %}
  server_name {{ name }};
  {% endif %}
  location / {
  }
}
```

Then let's define the yaml as the last step

```yaml
- hosts: web
  remote_user: root
  tasks:
    - name: template config
      template: src=templates/nginx.conf.j2 dest=/tmp/nginx.conf
```


### for case
Scenario:
Three virtual hosts—vhost1, vhost2, and vhost3—are created on the same target host, 10.0.0.16. Each virtual host may have different values for directives such as listen and server_name.


Define the following template file:

```conf
{% for vhost in  nginx_vhosts %}
server {
  {%if vhost.port is defined %}
  listen {{ vhost.port }};
  {% endif %}
  {%if vhost.name is defined %}
  server_name {{ vhost.name }};
  {% endif %}
  location / {
  }
}
{% endfor %}
```


Then we can write the yaml file as follows

```yaml
- hosts: web
  remote_user: root
  vars:
    nginx_vhosts:
      - vhost1:
        port: 81
        name: "vhost1.sswang.com"
      - vhost2:
        port: 82
      - vhost3:
        name: "vhost3.sswang.com"
  tasks:
    - name: template config
      template: src=templates/nginx.conf-for.j2 dest=/tmp/nginx.conf
```


The resulted nginx config file looks like this

```conf
server {
    listen 81;
      server_name vhost1.sswang.com;
    location / {
  }
}
server {
    listen 82;
      location / {
  }
}
server {
      server_name vhost3.sswang.com;
    location / {
  }
}
```


