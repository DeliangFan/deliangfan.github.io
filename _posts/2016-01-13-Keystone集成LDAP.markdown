---
layout: post
title:  "Keystone集成LDAP"
categories: OpenStack
---

--------------

基本配置如下：

- Linux: Ubuntu 14.04 LTS
- OpenStack: Kilo
- LDAP: slap 2.4.31

LDAP 的 DN(Distinguished Names) 默认由主机域名生成，本地的 DNS 设置如下：

```
root@ubuntu:~# cat /etc/hosts
10.10.1.100    keystone.com
127.0.0.1   localhost
```
----------------

#Install LDAP

```bash
sudo apt-get install slapd ldap-utils
```

安装完成后可通过以下命令和步骤完成 LDAP 的基本配置：

```
sudo dpkg-reconfigure slapd


* Omit OpenLDAP server configuration? No

* DNS domain name?  keystone.com

* Organization name?  admin

* Administrator password? YourPassword

* Use the password you configured during installation, or choose another one
  Database backend to use? HDB

* Remove the database when slapd is purged? No

* Move old database? Yes

* Allow LDAPv2 protocol? No
```

---------------

#Configure LDAP

由于 LDAP 的用户属性和 Keystone 默认的用户属性有所差异，所以 LDAP 需生成与 Keystone 中的 User 和 Group 相匹配的对象，可通过以下脚本(add_user_group.ldif)添加该对象，并生成两个用户。

```
# Users
dn: ou=users,dc=keystone,dc=com
ou: users
objectClass: organizationalUnit

# Group
dn: ou=groups,dc=keystone,dc=com
objectClass: organizationalUnit
ou: groups

# demo user
dn: cn=demo,ou=users,dc=keystone,dc=com
cn: demo
displayName: demo
givenName: demo
mail: demo@example.com
objectClass: inetOrgPerson
objectClass: top
sn: demo
uid: demo
userPassword: 123456

# admin user
dn: cn=admin,ou=users,dc=keystone,dc=com
cn: admin
displayName: admin
givenName: admin
mail: admin@example.com
objectClass: inetOrgPerson
objectClass: top
sn: admin
uid: admin
userPassword: 123456
```

Keystone 的配置文件如下：

```
[identity]
driver = keystone.identity.backends.ldap.Identity

[assignment]
driver = keystone.assignment.backends.sql.Assignment

[ldap]
url = ldap://keystone.com
query_scope = sub
user = "cn=admin,dc=keystone,dc=com"
password = 123456
tree_dn = "dc=keystone,dc=com"

user_tree_dn = "ou=users,dc=keystone,dc=com"
user_objectclass = inetOrgPerson
user_id_attribute = cn
user_name_attribute = cn
user_mail_attribute = mail
user_pass_attribute = userPassword
user_enabled_attribute = enabled

group_tree_dn = "ou=groups,dc=keystone,dc=com"
group_objectclass = groupOfUniqueNames
group_id_attribute = cn
group_name_attribute = cn
group_member_attribute = uniquemember
group_desc_attribute = description

user_allow_create = true
user_allow_update = true
user_allow_delete = true

group_allow_create = true
group_allow_update = true
group_allow_delete = true
```

--------------------

#Test

```
root@ubuntu:~# openstack user list
+--------------------+--------------------+
| ID                 | Name               |
+--------------------+--------------------+
| goodluck@gmail.com | goodluck@gmail.com |
| demo               | demo               |
| admin              | admin              |
+--------------------+--------------------+

root@ubuntu:~# openstack user show demo
+-----------+------------------+
| Field     | Value            |
+-----------+------------------+
| domain_id | default          |
| email     | demo@example.com |
| id        | demo             |
| name      | demo             |
+-----------+------------------+

root@ubuntu:~# openstack project create ldap_project
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | cbdf05b17cf54587b3b58a11f49252e7 |
| name        | ldap_project                     |
+-------------+----------------------------------+
```

------------

#Tips
TBD
