---
layout: post
title:  "搭建 yum 源"
categories: DevOps 
---

----------------
&nbsp;&nbsp;&nbsp;&nbsp;

> 注：此博文由陈尚华和本人整理所作。

对于企业的 Openstack 私有云，出于安全和某些因素的考虑，有些服务器无法访问公网，导致服务器无法更新某些 RPM 包，同时常有私有特性开发需求、版本的维护与升级，因此非常有必要构建企业私有 yum repo，主要有三个步骤。
 
- 下载官方的 repository
- 制作 release rpm
- nginx 发布

以 Redhat 官网安装手册为例，安装 Openstack 需要用到两类共计 8 个 repo，

- Centos 源：CentOS-Base.repo，CentOS-Debuginfo.repo，CentOS-Media.repo，CentOS-Vault.repo
- OpenStack 源及相源(epel, foreman, puppet)：epel.repo，foreman.repo，puppetlabs.repo，rdo-release.repo

环境

- Linux: Centos 6.5
- OpenStack: Icehouse

----------------

# 下载官网 repository

安装必要工具：

~~~ 
$ yum install wget createrepo rpm-build
~~~ 

下载 Centos, foreman, epel, puppetlabs, OpenStack 源的所有 rpm。

~~~ 
$ wget -S -c -r -np -L http://mirrors.sohu.com/centos/6.5/
$ wget -S -c -r -np -L http://yum.theforeman.org/plugins/1.5/el6/
$ wget -S -c -r -np -L http://yum.theforeman.org/releases/1.5/el6/
$ wget -S -c -r -np -L http://mirrors.yun-idc.com/epel/6/
$ wget -S -c -r -np -L https://yum.puppetlabs.com/el/6/
$ wget -S -c -r -np -L https://repos.fedorapeople.org/repos/openstack/openstack-icehouse/
~~~ 

删除不需要的文件：

~~~ 
$ find ./ -name index.html* | xarge rm -rf
$ find ./ -name fedora-20 | xarge rm -rf
$ find ./ -name fedora-19 | xarge rm -rf
$ find ./ -name i386 | xarge rm -rf
~~~ 

调整目录结构，使其更为直观可读：

~~~ 
$ mkdir foreman puppetlabs
$ mv yum.theforeman.org/plugins foreman/
$ mv yum.theforeman.org/releases foreman/
$ mv repos.fedorapeople.org/repos/openstack ./
$ mv yum.puppetlabs.com/el /puppetlabs/
$ mv mirrors.yun-idc.com/epel ./
$ rm -rf mirrors.yun-idc.com
$ rm -rf yum.puppetlabs.com
$ rm -rf yum.theforeman.org
$ rm -rf repos.fedorapeople.org
~~~ 

调整后目录如下：

~~~ 
$ ls
centos  epel  foreman  openstack  puppetlabs
~~~ 

依次重新制作 repository，以 OpenStack 为例：

~~~ 
$ createrepo openstack/openstack-icehouse/updates
$ createrepo openstack/openstack-icehouse/epel-6
~~~ 

如果有新 rpm，则加入 updates 目录中，并执行：

~~~ 
$ createrepo openstack/openstack-icehouse/updates --update
~~~ 

----------------

# Release RPM 制作：

创建padraig用户和组，用于编译制作 rpm：

~~~ 
$ groupadd -g 2000 padraig
$ useradd -u 2000 -g padraig -m padraig -d /home -s /bin/bash
~~~ 

下载并解压 Icehouse 源码包：

~~~ 
$ wget https://repos.fedorapeople.org/repos/openstack/openstack-icehouse/rdo-release-icehouse-4.src.rpm
$ rpm -i rdo-release-icehouse-4.src.rpm
~~~ 

修改各个 *.repo 文件的 url，以 rdo-release.repo 为例，更新 url 如下：

~~~ 
[openstack-icehouse]
name=OpenStack Icehouse Repository
baseurl=http://<your_yum_server>/openstack/openstack-icehouse/epel-6/
enabled=1
skip_if_unavailable=0
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-RDO-Icehouse
priority=98
~~~ 

更新 rpmbuild/SPECS/rdo-release.spec 文件内容：

~~~  
......

Source0:        rdo-release.repo
Source1:        RPM-GPG-KEY-RDO-Icehouse
Source2:        foreman.repo
Source3:        RPM-GPG-KEY-foreman
Source4:        puppetlabs.repo
Source5:        RPM-GPG-KEY-puppetlabs
Source6:        epel.repo
Source7:        RPM-GPG-KEY-EPEL-6
 
%install
install -p -D -m 644 %{SOURCE0} %{buildroot}%{_sysconfdir}/yum.repos.d/rdo-release.repo
install -p -D -m 644 %{SOURCE2} %{buildroot}%{_sysconfdir}/yum.repos.d/foreman.repo
install -p -D -m 644 %{SOURCE4} %{buildroot}%{_sysconfdir}/yum.repos.d/puppetlabs.repo
install -p -D -m 644 %{SOURCE6} %{buildroot}%{_sysconfdir}/yum.repos.d/epel.repo

# GPG Keys
install -Dpm 644 %{SOURCE1} %{buildroot}%{_sysconfdir}/pki/rpm-gpg/RPM-GPG-KEY-RDO-Icehouse
install -Dpm 644 %{SOURCE3} %{buildroot}%{_sysconfdir}/pki/rpm-gpg/RPM-GPG-KEY-foreman
install -Dpm 644 %{SOURCE5} %{buildroot}%{_sysconfdir}/pki/rpm-gpg/RPM-GPG-KEY-puppetlabs
install -Dpm 644 %{SOURCE7} %{buildroot}%{_sysconfdir}/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6

for repo in rdo-release foreman puppetlabs epel ; do
......
~~~ 

更新 rpmbuild/SOURCES/ 文件内容如下：

~~~ 
$ ls rpmbuild/SOURCES/
epel.repo  foreman.repo  puppetlabs.repo  rdo-release.repo  RPM-GPG-KEY-EPEL-6  RPM-GPG-KEY-foreman  RPM-GPG-KEY-puppetlabs  RPM-GPG-KEY-RDO-Icehouse
~~~ 

重新编译制作 release rpm：

~~~ 
$ rpmbuild -ba rpmbuild/SPECS/rdo-release.spec
~~~ 

----------------

#  Nginx 发布：

安装 Nginx：

~~~ 
$ rpm -ivh http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm
$ yum -y install nginx
~~~ 

/etc/nginx/conf.d/default.conf 的配置可参考如下：

~~~ 
server {  
    listen       80;  
    server_name  <your_server_name>;  
    location / {  
        root <your_yum_path>;  
        autoindex on; 
        index  index.html index.htm;  
    }  
    error_page   500 502 503 504  /50x.html;  
    location = /50x.html {  
        root   /usr/share/nginx/html;  
    }  
}
~~~ 

重启 nginx：

~~~ 
$ /etc/init.d/nginx restart
~~~ 

----------------

# Troubleshooting:

yum repolist 出现

~~~ 
$ yum repolist
...
http://openstack-yum-server/ceph/el6/x86_64/repodata/repomd.xml: [Errno 14] PYCURL ERROR 22 - "The requested URL returned error: 403 Forbidden"
Trying other mirror.
.....
~~~ 

解决方案：

- 每个 repo 配置新增 proxy=None

~~~ 
[openstack-icehouse-updates]
.....
_proxy_=None
~~~ 

- 关闭防火墙， service iptables stop


