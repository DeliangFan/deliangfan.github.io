---
layout: post
title:  "OpenStack日志搜集分析之ELK"
categories: OpenStack
---
------------------
集群达到一定规模时，日志管理和分析显得日益重要，良好统一的日志管理和分析平台有助于快速定位问题。Fuel 便集成了 ELK。
Logstash：负责日志的收集，处理和储存。
Elasticsearch：负责日志检索和分析。
Kibana：负责日志的可视化。
优点
迅速查询全局 ERROR 级别的日志。
 某个 API 的请求频率。 
 通过 request ID, 可得到一个 API 整个流程的日志。

规划与设计
部署架构
控制节点作为日志服务器，存储计算节点的 OpenStack 相关日志。logstash 部署于所有节点，收集本节点下所需收集的日志，然后以网络(node/http)方式输送给控制节点的 elasticsearch，kibana 作为 web portal 提供展示日志信息，如下图：
日志格式
为了提供快速直观的检索功能，对于每一条 OpenStack 日志，我们希望它能包含以下属性，用于过滤日志信息：
Host: 如 controller01，compute01 等
Service Name: 如 nova-api, neutron-server 等
Module: 如 nova.filters 
Log Level: 如 DEBUG, INFO, ERROR 等
Log date
以上属性可以通过 logstash 实现，通过提前日志的关键信息，从而获上述几种属性。

安装配置
安装手册可参考为：http://www.icyfire.me/2014/11/13/logstash-es-kibana.html
配置，OpenStack logstash 的配置为： https://github.com/osops/tools-logging/blob/master/logstash/basic/logstash.conf


