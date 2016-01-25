---
layout: post
title:  "搭建 yum 源"
categories: DevOps 
---

----------------

 对于企业的 Openstack 私有云，出于安全和某些因素的考虑，有些服务器无法访问公网，导致服务器无法更新某些 RPM 包，同时内部常有 Openstack 新特性开发需求、版本的维护与升级，因此非常有必要构建企业私有的 openstack yum 源。 构建 openstack yum 源有两个步骤：1.同步(下载)官方的源至企业 yum 服务器中；2. 重新创建 repo 并通过 nginx(apache)发布。

Overview
     以 Redhat Openstack 官网安装手册为例，安装 Openstack 需要用到两类共计 8 个 repo，
    1). Centos 源
       CentOS-Base.repo  CentOS-Debuginfo.repo  CentOS-Media.repo  CentOS-Vault.repo
    2). openstack 源及相关依赖源(epel, foreman, puppet)： 
        epel.repo  foreman.repo  puppetlabs.repo  rdo-release.repo

构建本地源步骤

1.yum源文件下载
(1).下载必要工具：

```
$ yum install wget createrepo

(2).下载yum源到本地：
[root@yumserver ~]# mkdir -p /wget-yum
[root@yumserver ~]# cd /wget-yum：

下载 Centos, foreman, epel, puppetlabs, openstack 源的所有 rpm。
$ wget -S -c -r -np -L http://mirrors.sohu.com/centos/6.5/
$ wget -S -c -r -np -L http://yum.theforeman.org/plugins/1.5/el6/
$ wget -S -c -r -np -L http://yum.theforeman.org/releases/1.5/el6/
$ wget -S -c -r -np -L http://mirrors.yun-idc.com/epel/6/
$ wget -S -c -r -np -L https://yum.puppetlabs.com/el/6/
$ wget -S -c -r -np -L https://repos.fedorapeople.org/repos/openstack/openstack-icehouse/

(3).删除不需要的软件包和文件：
$ find ./ -name index.html* | xarge rm -rf
$ find ./ -name fedora-20 | xarge rm -rf
$ find ./ -name fedora-19 | xarge rm -rf
$ find ./ -name i386 | xarge rm -rf

(4).调整目录结构：
[root@yumserver wget-yum]# mkdir foreman
$ mv yum.theforeman.org/plugins foreman/
$ mv yum.theforeman.org/releases foreman/
$ mv repos.fedorapeople.org/repos/openstack ./
$ mv yum.puppetlabs.com/el /puppetlabs/
$ mv mirrors.yun-idc.com/epel ./
$ mkdir puppetlabs
$ rm -rf mirrors.yun-idc.com
$ rm -rf yum.puppetlabs.com
$ rm -rf yum.theforeman.org
$ rm -rf repos.fedorapeople.org
[root@yumserver wget-yum]# ls
centos  epel  foreman  openstack  puppetlabs

2.nginx配置：
[root@yumserver wget-yum]# rpm -ivh http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm
[root@yumserver wget-yum]# yum -y install nginx
[root@yumserver wget-yum]# vi /etc/nginx/nginx.conf

[root@yumserver wget-yum]# /etc/init.d/nginx restart


3. release.rpm制作：
(1).下载icehouse源码包：
[root@yumserver ~]# wget https://repos.fedorapeople.org/repos/openstack/openstack-icehouse/rdo-release-icehouse-4.src.rpm

(2).创建padraig用户和组：
[root@yumserver ~]# groupadd -g 2000 padraig
[root@yumserver ~]# useradd -u 2000 -g padraig -m padraig -d /home -s /bin/bash

(3).解压rpm，并修改各个 .repo 文件的 url：
[root@yumserver ~]# rpm -i rdo-release-icehouse-4.src.rpm
修改 .repo 文件，以 rdo-release.repo 为例llllh
(3).解压rpm，并修改各个 .repo 文件的 url：
[root@yumserver ~]# rpm -i rdo-release-icehouse-4.src.rpm
修改 .repo 文件，以 rdo-release.repo 为例
```
(4).修改 .spec 文件内容：
[root@yumserver ~]# cd rpmbuild/
[root@yumserver rpmbuild]# 
SOURCES  SPECS
[root@yumserver ~]# cd SPECS
[root@yumserver SPECS]# vi rdo-release.spec 
URL:            https://github.com/redhat-openstack/rdo-release
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

#GPG Keys
install -Dpm 644 %{SOURCE1} %{buildroot}%{_sysconfdir}/pki/rpm-gpg/RPM-GPG-KEY-RDO-Icehouse
install -Dpm 644 %{SOURCE3} %{buildroot}%{_sysconfdir}/pki/rpm-gpg/RPM-GPG-KEY-foreman
install -Dpm 644 %{SOURCE5} %{buildroot}%{_sysconfdir}/pki/rpm-gpg/RPM-GPG-KEY-puppetlabs
install -Dpm 644 %{SOURCE7} %{buildroot}%{_sysconfdir}/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6

for repo in rdo-release foreman puppetlabs epel ; do

(5).修改SOURCES文件内容，并增加相应文件：
[root@yumserver SPECS]# cd ../SOURCES
[root@yumserver SOURCES]# ls
epel.repo  foreman.repo  puppetlabs.repo  rdo-release.repo  RPM-GPG-KEY-EPEL-6  RPM-GPG-KEY-foreman  RPM-GPG-KEY-puppetlabs  RPM-GPG-KEY-RDO-Icehouse

(6).重新打包rpm：
[root@yumserver SPECS]# yum -y install rpm-build
[root@yumserver SPECS]# pwd
/root/rpmbuild/SPECS
[root@yumserver SPECS]# rpmbuild -ba rdo-release.spec

4. 解决依赖关系，创建仓库：
[root@yumserver updates]# ls
repodata  x86_64
[root@yumserver updates]# pwd
yum-repo/openstack/openstack-icehouse/updates
[root@yumserver updates]# createrepo x86_64
 
5. repo 更新 RPM 包：
createrepo x86_64 --update

编译后的rpm源码包示例：
new-rdo-release-icehouse-4.0.src.rpm
```

troubleshooting:
 yum repolist 出现
[root@controller yum.repos.d]# yum repolist
Loaded plugins: axelget, fastestmirror, security
Loading mirror speeds from cached hostfile
http://openstack-yum-server/ceph/el6/x86_64/repodata/repomd.xml: [Errno 14] PYCURL ERROR 22 - "The requested URL returned error: 403 Forbidden"
Trying other mirror.
http://openstack-yum-server/ceph/el6/noarch/repodata/repomd.xml: [Errno 14] PYCURL ERROR 22 - "The requested URL returned error: 403 Forbidden"
Trying other mirror.
.....
 
解决方案：
1)
      每个 repo 配置新增 proxy=None
      [openstack-havana-updates]
      .....
      _proxy_=None
2)
     关闭防火墙， service iptables stop

注：此博文由陈尚华和本人所作。
