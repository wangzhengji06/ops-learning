## What is ldap?

LDAP is a protocol used to access and manage directory information, such as users, departments, and groups.

Each person usually has one unique directory entry, identified by a Distinguished Name, or DN:

uid=zhangsan,ou=People,dc=company,dc=com

A user can belong to multiple groups because different group entries can reference the same user DN:

vpn-users  → zhangsan
developers → zhangsan

So the user record is not duplicated. LDAP simply stores relationships between users and groups.

In enterprise systems, applications such as OpenVPN, OA systems, email, and internal websites can use LDAP to verify a user’s identity and check group membership.

In one sentence:

LDAP is a centralized directory service for managing users, identities, and group relationships.

## Terminology

DIT: directory tree

dc: root

ou: department / group


## Deploying ldap

```bash

# these two steps are very important
hostnamectl set-hostname ldap.magedu.com
echo "10.0.0.13 ldap.magedu.com" >> /etc/hosts


# Install the software
apt update
apt install -y slapd ldap-utils

# dc is created by default, now we need to create ou
vim base.ldif

dn: ou=users,dc=magedu,dc=com
objectClass: organizationalUnit
ou: users

dn: ou=groups,dc=magedu,dc=com
objectClass: organizationalUnit
ou: groups

# Import the ldif
ldapadd -x -D cn=admin,dc=magedu,dc=com -w Ldap@123456 -f base.ldif

# create group ou
vim group.ldif

dn: cn=dev-group,ou=groups,dc=magedu,dc=com
objectClass: posixGroup
objectClass: top
cn: dev-group
gidNumber: 10001

ldapadd -x -D cn=admin,dc=magedu,dc=com -w Ldap@123456 -f group.ldif

# create user ou
vim user.ldif

dn: uid=testuser,ou=users,dc=magedu,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Test User
sn: User
uid: testuser
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/testuser
loginShell: /bin/bash
shadowMax: 99999
shadowMin: 0
shadowWarning: 7
userPassword: {SSHA}T8Kvq0U1jqWp+NN+vOxPRt6wvxcEbXR5

ldapadd -x -D cn=admin,dc=magedu,dc=com -w Ldap@123456 -f user.ldif
```


Now you have created the group and user, you can start querying it

```bash
ldapsearch -x -H ldap://10.0.0.13 -b dc=magedu,dc=com "uid=testuser"
```

`ldapsearch` is typically used for verifying whether the command was successful. `ldapsearch -x -D user -w Password -b range "(filtered condition)"`

```bash
ldapwhoami -x -D uid=testuser,ou=users,dc=magedu,dc=com -w admin@123456
```

This is used to test whether the proper password and username has been assigned already


## Installing Client

```bash
apt install -y libnss-ldapd nslcd ldap-utils


vim /etc/pam.d/common-session
# this would add automatic create home folder option
session required        pam_mkhomedir.so skel=/etc/skel umask=0022
# end of pam-auth-update config
```


## The use of Schema
Scenario: I am banning some schema, then I am adding some new schema.


```bash
#first import the misc schema
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/misc.ldif
```


```bash
# now we can disable the setting here
cat > online_disable_misc_fixed.ldif <<-EOF
dn: cn={4}misc,cn=schema,cn=config
changetype: modify
replace: olcObjectClasses
olcObjectClasses: {0}( 2.16.840.1.113730.3.2.147 NAME 'inetLocalMailRecipient'
DESC 'Internet local mail recipient' SUP top AUXILIARY MUST cn MAY (
mailLocalAddress $ mailHost $ mailRoutingAddress ) )
olcObjectClasses: {1}( 1.3.6.1.4.1.42.2.27.1.2.5 NAME 'nisMailAlias' DESC 'NIS
mail alias' SUP top ABSTRACT MUST cn MAY rfc822MailMember )
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f online_disable_misc_fixed.ldif
```

We have added the misc schema, but disable some objectclass here. Now we can try to add to those diabled object class

```bash
cat > add_maillocal_direct.ldif <<-EOF
dn: uid=testuser,ou=users,dc=magedu,dc=com
changetype: modify
add: mailLocalAddress
mailLocalAddress: direct@example.com
EOF

# this line will be rejected
ldapmodify -x -D "cn=admin,dc=magedu,dc=com" -w "Ldap@123456" -f add_maillocal_direct.ldif
```

Now we can restore the objectclass, and start modifying the data

```bash
cat > online_restore_misc_fixed.ldif <<'EOF'
dn: cn={4}misc,cn=schema,cn=config
changetype: modify
replace: olcObjectClasses
olcObjectClasses: {0}( 2.16.840.1.113730.3.2.147 NAME 'inetLocalMailRecipient' DESC 'Internet local mail recipient' SUP top AUXILIARY MAY ( mailLocalAddress $ mailHost $ mailRoutingAddress ) )
olcObjectClasses: {1}( 1.3.6.1.4.1.42.2.27.1.2.5 NAME 'nisMailAlias' DESC 'NIS mail alias' SUP top STRUCTURAL MUST cn MAY rfc822MailMember )
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f online_restore_misc_fixed.ldif

cat > add_misc_aux_restore.ldif <<-EOF
dn: uid=testuser,ou=users,dc=magedu,dc=com
changetype: modify
add: objectClass
objectClass: inetLocalMailRecipient
EOF

ldapmodify -x -D "cn=admin,dc=magedu,dc=com" -w "Ldap@123456" -f add_misc_aux_restore.ldif
```


## The creation of Schema

Now I want to create my own schema
```bash
cat > custom_employee.ldif <<'EOF'
dn: cn=custom_employee,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: custom_employee
olcAttributeTypes: {0}( 1.3.6.1.4.1.4203.1.1.1.100 NAME 'employeeID' DESC '企业员工工号' EQUALITY caseIgnoreMatch SUBSTR caseIgnoreSubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 SINGLE-VALUE )
olcAttributeTypes: {1}( 1.3.6.1.4.1.4203.1.1.1.101 NAME 'deptCode' DESC '部门编码' EQUALITY caseIgnoreMatch SUBSTR caseIgnoreSubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 SINGLE-VALUE )
olcObjectClasses: {0}( 1.3.6.1.4.1.4203.1.2.2.100 NAME 'customEmployee' DESC '企业员工对象类' SUP inetOrgPerson STRUCTURAL MUST employeeID MAY deptCode )
EOF

```

Now I want to load the schema

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f custom_employee.ldif
```

Now I can add a new employee to the ldap

```bash
ldapadd -x -D "cn=admin,dc=magedu,dc=com" -w "Ldap@123456" -f add_employee.ldif
```

Now I want to add the password to this user.

```bash
vim add_auth_attr.ldif

dn: uid=emp001,ou=users,dc=magedu,dc=com
changetype: modify
# 1. 添加posixAccount对象类（每个add操作独立，分隔符单独一行）
add: objectClass
objectClass: posixAccount
-
# 2. 添加shadowAccount对象类（分隔符“-”单独一行，无多余字符）
add: objectClass
objectClass: shadowAccount
-
# 3. 补充posixAccount必填属性
add: uidNumber
uidNumber: 10011
-
add: gidNumber
gidNumber: 10011
-
add: homeDirectory
homeDirectory: /home/emp001
-
add: loginShell
loginShell: /bin/bash
-
# 4. 补充密码属性（加密串正确）
add: userPassword
userPassword: {SSHA}VXh01qQbOKj+nbLBDS8Pog/5BQkc1p89
-
# 5. 补充shadowAccount属性
add: shadowLastChange
shadowLastChange: 19500
-
add: shadowMax
shadowMax: 90

```

Now we can execute what we just created

```bash
ldapmodify -x -D "cn=admin,dc=magedu,dc=com" -w "Ldap@123456" -f add_auth_attr.ldif
```


The problem here is maybe you havent create the group yet, what you need to do is to create a group on the client

```bash
groupadd -g 10011 mygroup
```

## Verification for the ldap

We can add ssl to make sure that the password it not transferred freely in the network.

First, we create the certificate
```bash
mkdir -p /etc/ldap/ssl
chown openldap:openldap /etc/ldap/ssl
chmod 700 /etc/ldap/ssl
```

Generate the private key
```bash
openssl genrsa -out /etc/ldap/ssl/ldap.key 4096
chmod 600 /etc/ldap/ssl/ldap.key
chown openldap:openldap /etc/ldap/ssl/ldap.key
```

Create the self-siugned certificate.
```bash
openssl req -new -x509 -key /etc/ldap/ssl/ldap.key -out /etc/ldap/ssl/ldap.crt -days 365 \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=Magedu/OU=IT/CN=ldap.magedu.com"
```

Now we can create the file for adding the tls setting

```bash
cat > ldap_tls.ldif << EOF
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/ssl/ldap.crt
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/ssl/ldap.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/ssl/ldap.key
EOF

# now we can load the three more properties to the config object class.
ldapmodify -Y EXTERNAL -H ldapi:/// -f ldap_tls.ldif

# sadly you also have to modify this
sed -i 's/^SLAPD_SERVICES=.*/SLAPD_SERVICES="ldap:\/\/\/ldaps:\/\/\/ ldapi:\/\/\/"/' /etc/default/slapd

# And you also have to scp the public key to the client, and add the dns( or just /etc/hosts)
ldapsearch -x -H ldaps://ldap.magedu.com:636 -D "cn=admin,dc=magedu,dc=com" -w "Ldap@123456" -b dc=magedu,dc=com | grep dn:
```


Now you are done with the ssl certificate verification.

## ACL
What is the point? To make sure that there are different privleges for different roles.

The ACL rules are executed from up to down.

some keywords:
```
stop: once the rule is matched, stop looking further
continue: keep matching.
none: no visit
auth: I feel like there is no difference with none.
read: can read
write: can write
```

Let's show some examples for the ACL rules.

```
# Rule 1: anaonymous users cannot access any resource
access to  *
	by anonymous none stop
	
# Rule 2: all the verified users can read user's mail and telephone number
access to filter=(objectClass=inetOrgPerson) attrs=mail, telephoneNumber
	by users read continue
	
# Rule 3: Admin has all the right to do everything 
access to *
	by userdn = "cn=admin,dc=magedu,dc=com" write stop
```


## Customize ACL rules

Starts from writing a file, again
```bash
cat > acl_rules.ldif <<'EOF'
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to attrs=userPassword by dn.exact="cn=admin,dc=magedu,dc=com" write by self write by anonymous auth by * none
olcAccess: {1}to dn.subtree="ou=users,dc=magedu,dc=com" by dn.exact="cn=admin,dc=magedu,dc=com" write by self read by anonymous search by * none
olcAccess: {2}to * by dn.exact="cn=admin,dc=magedu,dc=com" write by * none
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f acl_rules.ldif

# You can now try to modify using admin account
cat > modify_emp001_mail.ldif << EOF
dn: uid=emp001,ou=users,dc=magedu,dc=com
changetype: modify
replace: mail
mail: emp001-new@example.com
EOF

# this command should work
ldapmodify -x -D "cn=admin,dc=magedu,dc=com" -w "Ldap@123456" -f modify_emp001_mail.ldif

# Now test whether the change was in place
ldapsearch -x -D "cn=admin,dc=magedu,dc=com" -w "Ldap@123456" -b "uid=emp001,ou=users,dc=magedu,dc=com" | grep mail
```


Now let's add a normal user

```bash
cat > add_emp002.ldif <<'EOF'
dn: uid=emp002,ou=users,dc=magedu,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: customEmployee
objectClass: posixAccount
objectClass: shadowAccount
cn: 王五
sn: 五
uid: emp002
# 自定义业务属性
employeeID: EMP2025002
deptCode: DEPT002
mail: emp002@example.com
# 系统认证必需属性
uidNumber: 10012
gidNumber: 10012
homeDirectory: /home/emp002
loginShell: /bin/bash
# 密码与过期属性
userPassword: {SSHA}1VaEntGPEwPQ6NuiskKFncw3E397PVxm
shadowLastChange: 20430
shadowMax: 90
EOF
```


Now we can finally restore the acl
```bash
cat > replace_acl_keep_default.ldif <<-EOF
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none
olcAccess: {1}to attrs=shadowLastChange by self write by * read
olcAccess: {2}to * by * read
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f replace_acl_keep_default.ldif SASL/EXTERNAL authentication started
```

## Prover and Cosumer

Provide does all the heavy work of modification etc, while cosumers just simply provide the read functions for the clients.

Let's use this setup: Let's make the 10.0.0.13 as the provider, and 10.0.0.16 as the consumer.

Let's start with 10.0.0.13

### First the provider node

```bash
#Make sure you install the software and configure the dns
vim /etc/hosts
10.0.0.13  ldap-master.magedu.com  ldap-master
10.0.0.16  ldap-slave.magedu.com  ldap-slaev

#Enable the syncprov
slapcat -b cn=config | grep -i syncprov
#Load it 
ldapmodify -Y EXTERNAL -H ldapi:/// -f syncprov.ldif

# Add the master sync property
cat > master-sync.ldif << EOF
# 为 mdb 数据库添加 syncprov 覆盖层
dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f master-sync.ldif

# Create a user
root@ldap-master:~# cat > repl-user.ldif <<-eof
dn: cn=repl,dc=magedu,dc=com
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: repl
description: OpenLDAP Replication User for Master-Slave
userPassword: {SSHA}IQlMgX4xSelJMdT5SKp9qPiqtS4RW9LM
eof

ldapadd -x -D cn=admin,dc=magedu,dc=com -W -f repl-user.ldif
```

### Now the consumer node

```bash
#install the software for the clients
#also dont forgot to install the logs as well
apt install rsyslog -y

# Add the slave ldif config
cat > slave-sync.ldif <<'EOF'
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSyncrepl
olcSyncrepl: {0}rid=001 provider=ldap://ldap-master.magedu.com:389/ binddn="cn=repl,dc=magedu,dc=com" bindmethod=simple credentials="Repl@123456" searchbase="dc=magedu,dc=com" scope=sub type=refreshAndPersist retry="60 +"
-
add: olcUpdateRef
olcUpdateRef: ldap://ldap-master.magedu.com:389/
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f slave-sync.ldif
```

And it is done!

I will skip the ansible part, but just know that you can use ansible to deploy the ldap
