---
layout: post
title:  "Nova 中的协程 -- 同步 (二)"
categories: OpenStack
---

![synchronize](http://7xp2eu.com1.z0.glb.clouddn.com/synchronize.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[图片来源请见](http://blog.takipi.com/5-things-you-didnt-know-about-synchronization-in-java-and-scala/)


- Nova 版本：Icehouse
- Eventlet 版本：0.18.2

----------

# Overview

同步是一个与并发相随的永恒话题，也是编程的一个难题。对 [race condition](https://en.wikipedia.org/wiki/Race_condition) 处理不当，轻者影响性能，重则死锁。协程被称为用户态的线程，为系统提供并发；但是只要有并发，就可能存在竞态条件，所以和线程一样，协程也必须要有处理竞态条件的能力。

线程常用的同步方式为：

- Lock: 确保同一时间只有一个线程访问数据。其中互斥锁最为常用
- Semaphore: 信号量，值为 1 时，相当于 lock。
- Condition Variable: 条件变量，允许线程按照特性顺序执行

与线程类似，eventlet 为协程提供如下同步方式：

- Lock: eventlet.semaphore.Semaphore(value=1)
- Semaphore: [eventlet.semaphore.Semaphore](http://eventlet.net/doc/modules/semaphore.html) 
- Condition Variable: [eventlet.event.Event](http://eventlet.net/doc/modules/event.html)


----------


# Synchronization in Nova

Nova 使用的同步方式是互斥锁，它支持两种类型的锁：

- 协程锁: semaphore.Semaphore() 实现
- 进程锁: Nova 在 semaphore.Semaphore() 基础之上自身实现的跨进程的文件锁

这两种锁由 lock(nova/openstack/common/lockutils.py) 函数实现，其中 external 参数决定该锁是协程锁还是进程锁：

~~~ python
def lock(name, lock_file_prefix=None, external=False, lock_path=None):
    """Context based lock

    This function yields a `threading.Semaphore` instance (if we don't use
    eventlet.monkey_patch(), else `semaphore.Semaphore`) unless external is
    True, in which case, it'll yield an InterProcessLock instance.

    :param lock_file_prefix: The lock_file_prefix argument is used to provide
    lock files on disk with a meaningful prefix.

    :param external: The external keyword argument denotes whether this lock
    should work across multiple processes. This means that if two different
    workers both run a a method decorated with @synchronized('mylock',
    external=True), only one of them will execute at a time.

    :param lock_path: The lock_path keyword argument is used to specify a
    special location for external lock files to live. If nothing is set, then
    CONF.lock_path is used as a default.
    """
    with _semaphores_lock:
        try:
            sem = _semaphores[name]
        except KeyError:
            sem = threading.Semaphore()
            _semaphores[name] = sem

    with sem:
        LOG.debug(_('Got semaphore "%(lock)s"'), {'lock': name})

        # NOTE(mikal): I know this looks odd
        if not hasattr(local.strong_store, 'locks_held'):
            local.strong_store.locks_held = []
        local.strong_store.locks_held.append(name)

        try:
            if external and not CONF.disable_process_locking:
                LOG.debug(_('Attempting to grab file lock "%(lock)s"'),
                          {'lock': name})

                # We need a copy of lock_path because it is non-local
                local_lock_path = lock_path or CONF.lock_path
                if not local_lock_path:
                    raise cfg.RequiredOptError('lock_path')

                if not os.path.exists(local_lock_path):
                    fileutils.ensure_tree(local_lock_path)
                    LOG.info(_('Created lock path: %s'), local_lock_path)

                def add_prefix(name, prefix):
                    if not prefix:
                        return name
                # NOTE(mikal): the lock name cannot contain directory
                # separators
                lock_file_name = add_prefix(name.replace(os.sep, '_'),
                                            lock_file_prefix)

                lock_file_path = os.path.join(local_lock_path, lock_file_name)

                try:
                    lock = InterProcessLock(lock_file_path)
                    with lock as lock:
                        LOG.debug(_('Got file lock "%(lock)s" at %(path)s'),
                                  {'lock': name, 'path': lock_file_path})
                        yield lock
                finally:
                    LOG.debug(_('Released file lock "%(lock)s" at %(path)s'),
                              {'lock': name, 'path': lock_file_path})
            else:
                yield sem

        finally:
            local.strong_store.locks_held.remove(name)
~~~


--------------


# Lock Usage in Nova

nova 还基于上节的 lock 函数定义许多其它和锁相关的函数，这些函数的原理和功能大抵相同：

- synchronized(name, lock_file_prefix=None, external=False, lock_path=None)，见 nova/openstack/common/lockutils.py
- synchronized_with_prefix(lock_file_prefix)，见 nova/openstack/common/lockutils.py
- synchronized = lockutils.synchronized_with_prefix('nova-')，见 nova/utils.py

例如协程锁：

~~~ python
def run_instance(self, context, instance, request_spec,
                 filter_properties, requested_networks,
                 injected_files, admin_password,
                 is_first_time, node, legacy_bdm_in_spec):

    if filter_properties is None:
        filter_properties = {}

    @utils.synchronized(instance['uuid'])
    def do_run_instance():
        self._run_instance(context, request_spec,
                filter_properties, requested_networks, injected_files,
                admin_password, is_first_time, node, instance,
                legacy_bdm_in_spec)
    do_run_instance()
~~~

例如进程锁，它多用于镜像文件处理过程中：

~~~ python
def create_image(self, prepare_template, base, size, *args, **kwargs):
    filename = os.path.split(base)[-1]

    @utils.synchronized(filename, external=True, lock_path=self.lock_path)
    def create_lvm_image(base, size):
        base_size = disk.get_disk_size(base)
        self.verify_base_size(base, size, base_size=base_size)
        resize = size > base_size
        size = size if resize else base_size
        libvirt_utils.create_lvm_image(self.vg, self.lv,
                                       size, sparse=self.sparse)
        images.convert_image(base, self.path, 'raw', run_as_root=True)
        if resize:
            disk.resize2fs(self.path, run_as_root=True)
        ......
~~~


