Roles is just a way to split the ansible playbook.

## Case 1: How to split one playbook into roles structure

Let's split this playbook

```yaml
- hosts: web
  remote_user: root
  tasks:
    - name: create new user
      user: name=nginx-test system=yes uid=82 shell=/sbin/nologin
    - name: create web root
      file: name=/data/webserver/html state=directory
    - name: touch web index
      shell: echo '<h1>welcome to ansible</h1>' > /data/webserver/html/index.html
    - name: install package
      apt: name=nginx state=present
    - name: copy config
      copy: src=nginx.conf dest=/etc/nginx/nginx.conf
    - name: delete config
      file: name=/etc/nginx/sites-enabled/default state=absent
    - name: copy subconfig
      copy: src=nginx-define.conf dest=/etc/nginx/conf.d
    - name: start service
      service: name=nginx state=started enabled=yes
```

First make sure that the environments for all hosts are clean

```bash
ansible all -m shell -a "apt list --installed | grep nginx"
ansible all -m shell -a "getent passwd nginx-test"
ansible all -m shell -a "getent group nginx-tes"
```


Now we can create the roles folder structure

```bash
mkdir /data/ansible/role/nginx_pro/roles/role_nginx/{tasks,templates,nginx_conf} -p
touch /data/ansible/role/nginx_pro/nginx_role.yaml
cd /data/ansible/role/nginx_pro/roles/role_nginx/tasks
touch {groups,users,packages,configs,services}.yaml
cd /data/ansible/role/nginx_pro
```


The whole role folder structure looks like this

```
root@ansible-node1:/data/ansible/role/nginx_pro# tree
.
├── nginx_role.yaml
└── roles
    └── role_nginx
        ├── nginx_conf
        ├── tasks
        │   ├── configs.yaml
        │   ├── groups.yaml
        │   ├── packages.yaml
        │   ├── services.yaml
        │   └── users.yaml
        └── templates
```

Let's first create those tasks

```bash
vim roles/role_nginx/tasks/groups.yaml
- name: create new group
  group: name=nginx-test gid=88
```

```bash
vim roles/role_nginx/tasks/users.yaml
- name: create new user
  user: name=nginx-test group=nginx-test system=yes uid=88 shell=/sbin/nologin
```

```bash
vim roles/role_nginx/tasks/packages.yaml
- name: install package
  apt: name=nginx state=present
```


```bash
vim roles/role_nginx/tasks/configs.yaml
- name: create web root
  file: name=/data/webserver/html owner=nginx-test state=directory
- name: touch web index
  shell: echo '<h1>welcome to ansible</h1>' > /data/webserver/html/index.html
- name: copy config
  copy: src=nginx_conf/nginx.conf dest=/etc/nginx/nginx.conf
- name: delete config
  file: name=/etc/nginx/sites-enabled/default state=absent
- name: copy subconfig
  copy: src=nginx_conf/nginx-define.conf dest=/etc/nginx/conf.d
```


```bash
vim roles/role_nginx/tasks/services.yaml
- name: start service
  service: name=nginx state=restarted enabled=yes
```

Now let's define the nginx's main config

roles/role_nginx/tasks/services.yaml
```conf
user nginx-test;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;
events {
        worker_connections 768;
}
http {
        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        ssl_prefer_server_ciphers on;
        access_log /var/log/nginx/access.log;
        gzip on;
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
```

Define the sub config

```conf
server {
  listen 10086;
  root         /data/webserver/html;
  location / {
  }
}
```

We need to specify the order of these tasks being executed. 

The include_tasks declaration order is the order the tasks will be executed.

```bash
vim roles/role_nginx/tasks/main.yaml
- include_tasks: groups.yaml
- include_tasks: users.yaml
- include_tasks: packages.yaml
- include_tasks: configs.yaml
- include_tasks: services.yaml
```


We also need a full orchestration file

```yaml
- hosts: web
  remote_user: root
  roles:
    - role: role_nginx
```


You can now deploy with one command

```bash
 ansible-playbook -l 10.0.0.16 nginx_role.yaml
```


## Case 2: A more advanced setup: LNMP


The structure goes like this:
10.0.0.16: nginx, php, wordpress

10.0.0.19: mysql

First, we still need to create the folder structure

```bash
mkdir /data/ansible/role/lnmp_case
cd /data/ansible/role/lnmp_case
mkdir -p lnmp_roles/{mysql,nginx,php,service,wordpress}/{tasks,files,templates}
```


The full folder structure looks like this

```
.
└── lnmp_roles
    ├── mysql
    │   ├── files
    │   ├── tasks
    │   └── templates
    ├── nginx
    │   ├── files
    │   ├── tasks
    │   └── templates
    ├── php
    │   ├── files
    │   ├── tasks
    │   └── templates
    ├── service
    │   ├── files
    │   ├── tasks
    │   └── templates
    └── wordpress
        ├── files
        ├── tasks
        └── templatse
```

### Nginx role
```bash
cat > lnmp_roles/nginx/tasks/group.yaml <<-eof
- name: add-nginx-group
  group: name=nginx gid=800 system=yes
eof

cat > lnmp_roles/nginx/tasks/user.yaml <<-eof
- name: add-nginx-user
  user: name=nginx group=800 system=yes uid=800 create_home=no
eof

cat > lnmp_roles/nginx/tasks/install.yaml <<-eof
- name: install-nginx
  apt: name=nginx,unzip state=present
eof

cat > lnmp_roles/nginx/tasks/main.yaml <<-eof
- include_tasks: group.yaml
- include_tasks: user.yaml
- include_tasks: install.yaml
eof 
```

### php

```bash
cat > lnmp_roles/php/tasks/group.yaml <<-eof
- name: add-php-group
  group: name=www-data gid=33 system=yes
eof

cat > lnmp_roles/php/tasks/user.yaml <<-eof
- name: add-php-user
  user: name=www-data group=33 system=yes uid=33 create_home=yes home=/var/www shell=/usr/sbin/nologin
eof

cat > lnmp_roles/php/tasks/install.yaml <<-eof
- name: install-php
  apt: name=php-fpm,php-mysqlnd,php-json,php-gd,php-xml,php-mbstring,php-zip state=present
eof

cat > lnmp_roles/php/tasks/main.yaml <<-eof
- include_tasks: group.yaml
- include_tasks: user.yaml
- include_tasks: install.yaml
eof
```


### wordpress

```bash
cat > lnmp_roles/wordpress/tasks/wp_get_code.yaml <<-eof
- name: wget-wordpress
  get_url: url=https://cn.wordpress.org/latest-zh_CN.zip dest=/var/www/html/wordpress.zip
eof

cat > lnmp_roles/wordpress/tasks/wp_unarchive.yaml <<-eof
- name: wp-unarchive
  unarchive: src=/var/www/html/wordpress.zip dest=/var/www/html/ owner=www-data group=www-data remote_src=yes
eof

cat > lnmp_roles/wordpress/tasks/wp_set_domain.yaml <<-eof
- name: set-wp-domain
  template: src=domain.conf.j2 dest=/etc/nginx/sites-enabled/{{ WP_DOMAIN }}.conf
- name: rm-default-conf
  shell: rm -rf /etc/nginx/sites-enabled/default
eof

cat > lnmp_roles/wordpress/templates/domain.conf.j2 <<-eof
server{
   listen {{ WP_PORT }};
   server_name {{ WP_DOMAIN }};
   include /etc/nginx/default.d/*.conf;
   root {{ WP_PATH }};
   index index.php index.html;
   location ~ \.php$ {
     include snippets/fastcgi-php.conf;
     fastcgi_pass unix:/run/php/php8.3-fpm.sock;
   }
}
eof

cat > lnmp_roles/wordpress/tasks/main.yaml <<-eof
- include_tasks: wp_get_code.yaml
- include_tasks: wp_unarchive.yaml
- include_tasks: wp_set_domain.yaml
eof
```


### Service

```bash
cat > lnmp_roles/service/tasks/service.yaml <<-eof  
- name: service
  service: name={{ item.name }} state={{ item.state }} enabled={{ item.enabled }}
  loop: "{{ SERVICE_LIST }}"
eof

cat > lnmp_roles/service/tasks/main.yaml <<-eof  
- include_tasks: service.yaml
eof
```


### MySql
```bash
cat > lnmp_roles/service/tasks/main.yaml <<-eof  
- include_tasks: service.yaml
eof

cat > lnmp_roles/mysql/tasks/user.yaml <<-eof
- name: add-mysql-user
  user: name=mysql group=306 system=yes uid=306 create_home=no
eof

cat > lnmp_roles/mysql/tasks/install.yaml <<-eof
- name: apt-install-mysql-server
  apt: name=mysql-server state=present update_cache=yes
  
- name: set-mysqld-conf-task-1
  lineinfile: path=/etc/mysql/mysql.conf.d/mysqld.cnf backrefs=yes regexp='^(bind-address.*)$' line='#\1'
  
- name: set-mysqld-conf-task-2
  lineinfile: path=/etc/mysql/mysql.conf.d/mysqld.cnf  line='skip-name-resolve'
- name: set-mysqld-conf-task-3
  lineinfile: path=/etc/mysql/mysql.conf.d/mysqld.cnf  line='default-authentication-plugin=mysql_native_password'  
eof

cat > lnmp_roles/mysql/tasks/restart.yaml <<-eof
- name: restart-mysql-service
  service: name=mysql enabled=yes state=restarted
eof

cat > lnmp_roles/mysql/tasks/copy_file.yaml <<-eof
- name: copy-mysql-file
  copy: src=files/grant.sql dest=/tmp/grant.sql
eof

cat > lnmp_roles/mysql/tasks/grant.yaml <<-eof
- name: mysql-client-init
  shell: mysql </tmp/grant.sql
eof

cat > lnmp_roles/mysql/files/grant.sql <<-eof
create database if not exists wordpress;
create user 'wp_user'@'10.0.0.%' identified by '123456';
grant all on wordpress.* to 'wp_user'@'10.0.0.%';
flush privileges;
eof

cat > lnmp_roles/mysql/tasks/main.yaml <<-eof
- include_tasks: group.yaml
- include_tasks: user.yaml
- include_tasks: install.yaml
- include_tasks: restart.yaml
- include_tasks: copy_file.yaml
- include_tasks: grant.yaml
eof
```


### Final Config

```yaml
cat > lnmp_wp.yaml <<-eof
- hosts: 10.0.0.16
  remote_user: root
  gather_facts: no
  vars:
    WP_PORT: 80
    WP_DOMAIN: blog.magedu.com
    WP_PATH: /var/www/html/wordpress
    SERVICE_LIST: [ {name: nginx, state: restarted, enabled: yes},{name: php8.3-fpm, state: started, enabled: yes} ]
  roles:
    - lnmp_roles/nginx
    - lnmp_roles/php
    - lnmp_roles/wordpress
    - lnmp_roles/service
eof
```


```yaml
cat > mysql.yaml <<-eof
- hosts: 10.0.0.19
  remote_user: root
  gather_facts: no
  roles:
   - lnmp_roles/mysql
eof
```

Now what we can do is to first deploy mysql, then deploy 


## The most advanced one: big ansible project
When you define task, you can use the following way to pass a variable to the task, this variable is only specific to that task

```yaml
- { role: role_nginx, nginx_port: 8888 }
```


You can also use condition to make the variable passing more flexible

```yaml
- { role: role_nginx, nginx_port: 10086, when: ansible_hostname == 'ubuntu24-16' }
```

This will only run the task for 10.0.0.16.

You can even use tag, then when you run ansible playbook you can use -t to tell what tag you want to run

```yaml
- { role: role_nginx, nginx_port: 200, tags: ['nginx', 'web'] }
- { role: role_nginx1, nginx_port: 100, tags: ['nginx1', 'web'] }
```


How do we define environment variable? 
1. the vars main.yaml
2. main yaml: vars field
3. main yaml: {role:xxx, variable: xxx} 

How do we customize project environment?
1. Add when in main yaml
2. Add tags in main yaml

