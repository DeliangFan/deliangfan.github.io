---
layout: post
title:  "An Introduction to Heat"
categories: OpenStack
---

![heat_log](http://7xp2eu.com1.z0.glb.clouddn.com/heat_logo.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[原图出处](http://www.slideshare.net/openstackindia/introduction-to-openstack-heat)


# Overview

[Heat](https://wiki.openstack.org/wiki/Heat) 从 Havana 起成为 OpenStack 的正式项目，它的定位相当于 AWS 的 CloudFormation，为多种云平台上的云服务乃至虚拟机的应用程序提供编排的功能。所谓的编排，就是按照特定的顺序执行某些操作。我们以用户需要创建一个 instance 和 volume，然后挂载 volume 至 instance 为例，一般的步骤为：

~~~
调用 Nova 创建 instance --> 调用 Cinder 创建 volume --> 调用 nova 挂载 volume 至 instance
~~~

如果利用 Heat，用户只需在一个文本的 template 中定义上述的资源和操作步骤，然后调用 heat stack-create 即可，Heat 会自动的解析 template 里内容和依赖关系，最后按照顺序调用相关组件的 API 依次创建 instance、volume 并挂载 volume。利用 Heat，用户可以非常便捷的为用户构建 IT 设施和部署应用，极大的简化了中间繁杂的步骤。

Heat 的架构如下，它由 4 部分组成：

- heat-api: 提供原生的 Restful API- heat-api-cfn: 兼容 AWS CloudFormation API
- heat-api-cloudwatch: 兼容 AWS CloudWatch API，可用于接收 ceilometer 的告警- heat-engine: 最核心部分，解析 template，处理逻辑业务，完成资源的创建和部署

![Heat work flow](http://7xp2eu.com1.z0.glb.clouddn.com/heat_work_flow.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[原图出处](http://gigaroom.blogspot.com/2014/12/cloud-orchestrator-between-it-and-telco.html?view=magazine)

------------------

# Orchestration

Orchestration 是 Heat 最重要的功能，它把所有的云服务，如 instance、flavor、volume、network、ip 乃至 volumeattachment、softwaredeployment 都抽象为 resource，每种的 resource 都有一些自身的 properties，比如 flavor 类型的 resource，它有 ram、vcpu 和 disk 等 properties。Template 用文本定义一系列的 resource，Heat 解析 template 的 resource、resource 之间的依赖关系以及 resource 的 properties，然后生成一个 stack，stack 表示这些 resource 的集合，最后根据依赖关系，依次完成各种资源的创建。

- template: 文本模块，定义各种 resource，支持 AWS 和 HOT 格式的模板
- stack: stack 是一组 resource 的集合，heat 根据 template 创建 stack
- resource: 云服务，如 instance, flavor 等
- properties: 表示 resource 的某些属性

例如，下述 [template](https://github.com/openstack/heat-templates/blob/master/hot/vm_with_cinder.yaml) 创建一个 instance 和 volume，并把 volume 挂载到 instance：

~~~ yaml
heat_template_version: 2013-05-23

description: >
  A HOT template that holds a VM instance with an attached
  Cinder volume.  The VM does nothing, it is only created.

resources:
  my_instance:
    type: OS::Nova::Server
    properties:
      image: your_image_id
      flavor: m1.small
      networks: [{network: your_network_id }]

  my_volume:
    type: OS::Cinder::Volume
    properties:
      size: 10

  vol_attachment:
    type: OS::Cinder::VolumeAttachment
    properties:
      instance_uuid: { get_resource: my_instance }
      volume_id: { get_resource: my_vol }
      mountpoint: /dev/vdb
~~~

![volume_instance_attachment](http://7xp2eu.com1.z0.glb.clouddn.com/volume_instance_attachment_heat.png)

Heat 支持多种类型的 resource，用户也可以加入自己定义的 resource：

~~~ bash
AWS::AutoScaling::AutoScalingGroup
......
AWS::S3::Bucket
OS::Ceilometer::Alarm
OS::Ceilometer::CombinationAlarm
OS::Cinder::EncryptedVolumeType
OS::Cinder::Volume
OS::Cinder::VolumeAttachment
OS::Glance::Image
OS::Heat::AutoScalingGroup
......
OS::Heat::WaitConditionHandle
OS::Keystone::Endpoint
......
OS::Neutron::LBaaS::PoolMember
OS::Neutron::LoadBalancer
......
OS::Nova::Server
......
~~~

--------

# Autoscaling

------------

# Software Configuration and Deployment