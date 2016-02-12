---
layout: post
title:  "Heat 支持查询 Autoscaling Group 下的虚拟机"
categories: OpenStack
---

---------------

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

上述的大意是如何通过 Heat API 获取 Autoscaling Group 里的虚拟机列表。

---------------

# Autoscaling 介绍

以 Heat 推荐的 Autoscaling Group 的模板为例子，采用该模板创建 Stack 后，查询该 Stack 包含的 resouce 如下，可知 asg 即为 AutoScalingGroup。

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

查询相关虚拟机

~~~ bash
$ nova list
+----------+-------------+--------+------------+
| ID       | Name        | Status | Task State | 
+----------+-------------+
| 22c...0a |
| 3f0...c8 |
+----------+
~~~

----------------

# Nested Stack

在讲解 nested_stack 之前，先问读者一个问题：请问刚刚一共创建了多少个 stack？

一个！明明只有一个嘛！它的名字就叫 fuck ！......

貌似言之有理，再查询数据库看看，奇葩出现了，咋跑出了四个 stack，而且四个 stack 的名字均以 fuck 开头，有着几分相似和某些规律。再看看 owner_id 和 id 的关系，机智的仁兄，你应该发现些嘘头了吧。
1. stack-list 查询到的 stack, 其 owner_id 均未 NULL
2. nested_stack 的 owner_id 为父 stack 的 id


看，熟悉的虚拟机现身了。从上不难发现另外一条规律，nested_stack 的 id 和父 stack 的某个 resource id 完全一致。事实上，Heat 有以下资源支持 nested_stack
OS::Heat::InstanceGroup，OS::Heat::ResourceGroup，OS::Heat::AutoScalingGroup，AWS::AutoScaling::AutoScalingGroup。

对于初步了解 heat 的同学来说，nested_stack 比较陌生晦涩，更多的资料请移步 [wiki](https://wiki.openstack.org/wiki/Heat/NestedStacks)

----------------

# 解决方案

我们可以按照以上方法查询 Autoscaling Group 下得虚拟机信息，但是频繁的 CLI 查询操作繁琐、效率低下，用户体验极差。最好的方式是查询 Autoscaling Group 资源时，可以返回其旗下的虚拟机列表。如下：

由于代码实现简单，核心代码为如下，详情见该 [commit](https://github.com/DeliangFan/heat/commit/63d35793c47784b4ff0e980a0148eaf96139c853)：

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
