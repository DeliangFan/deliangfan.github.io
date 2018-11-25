---
layout: post
title:  "Heat 支持查询 Autoscaling Group 下的虚拟机"
categories: OpenStack
---


OpenStack mailing list 有这么一封邮件：

> [Openstack] heat autoscaling group/instance relationships
>
>
> I'm trying to figure out how to determine all instances that were created
as part of a given autoscaling group. I want to take a given autoscaling
group and list all of its instances. So far I can't figure out how to do
this. The instances themselves have a tag called "AutoScalingGroupName"
(mystack-MyServerGroup-f3r72ifsj2jq for example) but the value of that
doesn't seem to map to anything. A resource-show on the autoscaling group
doesn't seem to show any identifier that maps to the autoscaling group
name.
>
>Any ideas on how I can do this? Thanks!

上述的大意是如何获取 [autoscaling group](https://wiki.openstack.org/wiki/Heat/AutoScaling) 里的虚拟机列表。

---------------

# Autoscaling Group 介绍

以 Heat 推荐的 autoscaling group 的 [template](https://github.com/openstack/heat-templates/blob/master/hot/autoscaling.yaml) 为例，采用该模板创建 stack 后，查询该 stack 包含的 resource 如下，可知 asg 即为 autoscaling group。

~~~ bash
$ heat resource-list asg_stack
+------------------+----------------------------+-----------------+---------------+
| resource_name    | resource_type              | resource_status | updated_time  |
+------------------+----------------------------+-----------------+---------------+
| monitor          | OS::Neutron::HealthMonitor | CREATE_COMPLETE | 2014-04-26... |
| pool             | OS::Neutron::Pool          | CREATE_COMPLETE | 2014-04-26... |
| lb               | OS::Neutron::LoadBalancer  | CREATE_COMPLETE | 2014-04-26... |
| lb_floating_ip   | OS::Neutron::FloatingIP    | CREATE_COMPLETE | 2014-04-26... |
| asg              | OS::Heat::AutoScalingGroup | CREATE_COMPLETE | 2014-04-26... |
| scaledown_policy | OS::Heat::ScalingPolicy    | CREATE_COMPLETE | 2014-04-26... |
| scaleup_policy   | OS::Heat::ScalingPolicy    | CREATE_COMPLETE | 2014-04-26... |
| cpu_alarm_low    | OS::Ceilometer::Alarm      | CREATE_COMPLETE | 2014-04-26... |
| cpu_alarm_high   | OS::Ceilometer::Alarm      | CREATE_COMPLETE | 2014-04-26... |
+------------------+----------------------------+-----------------+---------------+
~~~

继续查询 asg 详情，未发现任何和虚拟机相关的信息，而事实上，autoscaling group 作为一群虚拟机的集合，用户非常希望获取它所包含的虚拟机信息。如果 Heat 未提供该 API，那么用户只能通过 Nova API 查询相关的虚拟机，对用户来说，无疑增加了操作的复杂度。

~~~ bash
$ heat resource-show asg_stack asg
+------------------------+---------------------------------------------------------------------+
| Property               | Value                                                               |
+------------------------+---------------------------------------------------------------------+
| description            |                                                                     |
| links                  | http://ip:8004/v1/tenant_id/stacks/asg_stack/stack_id/resources/asg |
|                        | http://ip:8004/v1/tenant_id/stacks/asg_stack/stack_id               |
| logical_resource_id    | asg                                                                 |
| physical_resource_id   | 525954ea-a309-44a7-9e40-d793b7c754a                                 |
| required_by            | scaleup_policy                                                      |
|                        | scaledown_policy                                                    |
| resource_name          | asg                                                                 |
| resource_status        | CREATE_COMPLETE                                                     |
| resource_status_reason | state changed                                                       |
| resource_type          | OS::Heat::AutoScalingGroup                                          |
| updated_time           | 2014-04-26T09:33:04Z                                                |
+------------------------+---------------------------------------------------------------------+
~~~

事实上，经 Nova 查询到的虚拟机如下：

~~~ bash
$ nova list
+----------+---------------+--------+------------+-------------+-----+
| ID       | Name          | Status | Task State | Power State | ... |
+----------+---------------+--------+------------+-------------+-----+
| 22c...0a | as-47g2...de8 | ACTIVE | -          | Running     | ... |
| 3f0...c8 | as-47g2...ds3 | ACTIVE | -          | Running     | ... |
+----------+---------------+--------+------------+-------------+-----+
~~~

----------------

# Nested Stack

讲解 [nested stack](https://wiki.openstack.org/wiki/Heat/NestedStacks) 之前，先问一个问题：请问一共创建了多少个 stack？

一个！它的名字就叫 asg_stack ！......

貌似言之有理，再查询数据库看看，却出现四个名字均以 asg_stack 开头的 stack，并且 owner_id 和 id 有如下关系：

- heat stack-list 查询到的 stack, 其 owner_id 为 NULL
- nested stack 的 owner_id 和父 stack 的 id 一致

~~~ bash
mysql> select id,name,owner_id where deleted_at is Null;
+-----------+------------------------------------+-----------+
| id        | name                               | owner_id  |
+-----------+------------------------------------+-----------+
| 5259...4a | asg_stack-asg-ll5qfhea47g2         | 8dd9...c3 |
| 8dd9...c3 | asg_stack                          | NULL      |
| 9986...04 | asg_stack-asg-ll5qfhea47g2-ue...iu | 5259...4a |
| cad9...7a | asg_stack-asg-ll5qfhea47g2-4s...cr | 5259...4a |
+-----------+------------------------------------+-----------+
~~~

4 个 stack 的关系如下图所示：

![Image](http://wsfdl.oss-cn-qingdao.aliyuncs.com/Nested_stack.png?imageView2/1/w/450/h/400/q/100)

继续查询 asg_stack-asg-ll5qfhea47g2-4s...cr 这个 stack 包含的资源：
 
~~~ bash
$ heat resource-list 99860cfb-1110-4eb6-89be-0dfff14b3a04
+---------------+-------------------------+-----------------+---------------+
| resource_name | resource_type           | resource_status | updated_time  |
+---------------+-------------------------+-----------------+---------------+
| server        | OS::Nova::Server        | CREATE_COMPLETE | 2014-04-26... |
| member        | OS::Neutron::PoolMember | CREATE_COMPLETE | 2014-04-26... |
+---------------+-------------------------+-----------------+---------------+
~~~

从上不难发现另外一条规律，nested stack 的 id 和父 stack 的某个 resource_id 完全一致。事实上，Heat 的以下资源支持 nested stack：

- OS::Heat::InstanceGroup
- OS::Heat::ResourceGroup
- OS::Heat::AutoScalingGroup
- AWS::AutoScaling::AutoScalingGroup 

----------------

# 解决方案

理解了 nested stack，很容易实现查询 autoscaling group 资源时返回其旗下的虚拟机列表。如下：

~~~ bash
$ heat resource-show asg_stack asg
+------------------------+---------------------------------------------------------------------+
| Property               | Value                                                               |
+------------------------+---------------------------------------------------------------------+
| description            |                                                                     |
| instances              | [{'name': 'asg_stack-asg-ll5qfhea47g2...5w', 'uuid': '43de...e3'},  |
|                        |  {'name': 'asg_stack-asg-ll5qfhea47g2...4d', 'uuid': '43a4...7d'}]  |
| links                  | http://ip:8004/v1/tenant_id/stacks/asg_stack/stack_id/resources/asg |
|                        | http://ip:8004/v1/tenant_id/stacks/asg_stack/stack_id               |
| logical_resource_id    | asg                                                                 |
| physical_resource_id   | 525954ea-a309-44a7-9e40-d793b7c754a                                 |
| required_by            | scaleup_policy                                                      |
|                        | scaledown_policy                                                    |
| resource_name          | asg                                                                 |
| resource_status        | CREATE_COMPLETE                                                     |
| resource_status_reason | state changed                                                       |
| resource_type          | OS::Heat::AutoScalingGroup                                          |
| updated_time           | 2014-04-26T09:33:04Z                                                |
+------------------------+---------------------------------------------------------------------+
~~~

由于实现简单，详情见该 [commit](https://github.com/DeliangFan/heat/commit/63d35793c47784b4ff0e980a0148eaf96139c853)，核心代码为如下：

~~~ python
def _get_scaling_group_instances(self, resource):
    instance_list = []

    if hasattr(resource, 'nested') and callable(resource.nested):
        nested_stack = resource.nested()
    else:
        nested_stack = None

    if nested_stack:
        for r in nested_stack.values():
            instance_list.extend(self._get_scaling_group_instances(r))
    else:
        if resource.type() in INSTANCE_RESOURCES:
            instance_name = resource.physical_resource_name()
            instance_uuid = resource.resource_id
            instance_info = {'instance_name': instance_name,
                             'instance_uuid': instance_uuid}
            return [instance_info]

    return instance_list

@request_context
def describe_stack_resource(self, cnxt, stack_identity, resource_name):
    s = self._get_stack(cnxt, stack_identity)
    stack = parser.Stack.load(cnxt, stack=s)

    if cfg.CONF.heat_stack_user_role in cnxt.roles:
        if not self._authorize_stack_user(cnxt, stack, resource_name):
            logger.warning(_("Access denied to resource %s")
                           % resource_name)
            raise exception.Forbidden()

    if resource_name not in stack:
        raise exception.ResourceNotFound(resource_name=resource_name,
                                         stack_name=stack.name)

    resource = stack[resource_name]
    if resource.id is None:
        raise exception.ResourceNotAvailable(resource_name=resource_name)

    if resource.type() in SCALING_GROUP_RESOURCES:
        instance_list = self._get_scaling_group_instances(resource)
        resource.instance_list = instance_list

    return api.format_stack_resource(resource)
~~~
