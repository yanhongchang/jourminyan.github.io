---
layout: post
title: nova代码添加虚拟网卡挂卸载通知
category: 技术
date: 2020-04-02



---

更新历史：

- 2020.04 完成

------

###

**版本：**

​		queens



**原因：**

​		由于云管需要监听notificats.info的关于挂盘interface_attach和interface_detach的消息，但是nova代码中在操作相关动作时，并没有相关发送的消息，因此，需要对nova代码进行增加代码，具体如下：



**代码修改：**



---

nova/compute/manager.py

（1）def attach_interface()方法修改

原代码：

```python
    @wrap_exception()
    @wrap_instance_event(prefix='compute')
    @wrap_instance_fault
    def attach_interface(self, context, instance, network_id, port_id,
                         requested_ip, tag):
        """Use hotplug to add an network adapter to an instance."""
        if not self.driver.capabilities['supports_attach_interface']:
            raise exception.AttachInterfaceNotSupported(
                instance_uuid=instance.uuid)
        if (tag and not
            self.driver.capabilities.get('supports_tagged_attach_interface',
                                         False)):
            raise exception.NetworkInterfaceTaggedAttachNotSupported()

        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.INTERFACE_ATTACH,
            phase=fields.NotificationPhase.START)

        bind_host_id = self.driver.network_binding_host_id(context, instance)
        network_info = self.network_api.allocate_port_for_instance(
            context, instance, port_id, network_id, requested_ip,
            bind_host_id=bind_host_id, tag=tag)
        if len(network_info) != 1:
            LOG.error('allocate_port_for_instance returned %(ports)s '
                      'ports', {'ports': len(network_info)})
            # TODO(elod.illes): an instance.interface_attach.error notification
            # should be sent here
            raise exception.InterfaceAttachFailed(
                    instance_uuid=instance.uuid)
        image_meta = objects.ImageMeta.from_instance(instance)

        try:
            self.driver.attach_interface(context, instance, image_meta,
                                         network_info[0])
        except exception.NovaException as ex:
            port_id = network_info[0].get('id')
            LOG.warning("attach interface failed , try to deallocate "
                        "port %(port_id)s, reason: %(msg)s",
                        {'port_id': port_id, 'msg': ex},
                        instance=instance)
            try:
                self.network_api.deallocate_port_for_instance(
                    context, instance, port_id)
            except Exception:
                LOG.warning("deallocate port %(port_id)s failed",
                            {'port_id': port_id}, instance=instance)

            compute_utils.notify_about_instance_action(
                context, instance, self.host,
                action=fields.NotificationAction.INTERFACE_ATTACH,
                phase=fields.NotificationPhase.ERROR,
                exception=ex)

            raise exception.InterfaceAttachFailed(
                instance_uuid=instance.uuid)

        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.INTERFACE_ATTACH,
            phase=fields.NotificationPhase.END)

        return network_info[0]

```



修改为：

```python
@wrap_exception()
    @wrap_instance_event(prefix='compute')
    @wrap_instance_fault
    def attach_interface(self, context, instance, network_id, port_id,
                         requested_ip, tag):
        """Use hotplug to add an network adapter to an instance."""
        if not self.driver.capabilities['supports_attach_interface']:
            raise exception.AttachInterfaceNotSupported(
                instance_uuid=instance.uuid)
        if (tag and not
            self.driver.capabilities.get('supports_tagged_attach_interface',
                                         False)):
            raise exception.NetworkInterfaceTaggedAttachNotSupported()

        self._notify_about_instance_usage(context, instance,
                                          "interface_attach.start")
        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.INTERFACE_ATTACH,
            phase=fields.NotificationPhase.START)

        bind_host_id = self.driver.network_binding_host_id(context, instance)
        network_info = self.network_api.allocate_port_for_instance(
            context, instance, port_id, network_id, requested_ip,
            bind_host_id=bind_host_id, tag=tag)
        if len(network_info) != 1:
            LOG.error('allocate_port_for_instance returned %(ports)s '
                      'ports', {'ports': len(network_info)})
            # TODO(elod.illes): an instance.interface_attach.error notification
            # should be sent here
            raise exception.InterfaceAttachFailed(
                    instance_uuid=instance.uuid)
        image_meta = objects.ImageMeta.from_instance(instance)

        try:
            self.driver.attach_interface(context, instance, image_meta,
                                         network_info[0])
        except exception.NovaException as ex:
            port_id = network_info[0].get('id')
            LOG.warning("attach interface failed , try to deallocate "
                        "port %(port_id)s, reason: %(msg)s",
                        {'port_id': port_id, 'msg': ex},
                        instance=instance)
            try:
                self.network_api.deallocate_port_for_instance(
                    context, instance, port_id)
            except Exception:
                LOG.warning("deallocate port %(port_id)s failed",
                            {'port_id': port_id}, instance=instance)

            compute_utils.notify_about_instance_action(
                context, instance, self.host,
                action=fields.NotificationAction.INTERFACE_ATTACH,
                phase=fields.NotificationPhase.ERROR,
                exception=ex)

            raise exception.InterfaceAttachFailed(
                instance_uuid=instance.uuid)

        self._notify_about_instance_usage(context, instance,
                                          "interface_attach.end")
        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.INTERFACE_ATTACH,
            phase=fields.NotificationPhase.END)

        return network_info[0]

```



增加的内容：

```python
5833,5834d5832
<         self._notify_about_instance_usage(context, instance,
<                                           "interface_attach.start")
5878,5879d5875
<         self._notify_about_instance_usage(context, instance,
<                                           "interface_attach.end")
```



(2) def detach_interface()

原代码

```python
    @wrap_exception()
    @wrap_instance_event(prefix='compute')
    @wrap_instance_fault
    def detach_interface(self, context, instance, port_id):
        """Detach a network adapter from an instance."""
        network_info = instance.info_cache.network_info
        condemned = None
        for vif in network_info:
            if vif['id'] == port_id:
                condemned = vif
                break
        if condemned is None:
            raise exception.PortNotFound(_("Port %s is not "
                                           "attached") % port_id)

        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.INTERFACE_DETACH,
            phase=fields.NotificationPhase.START)

        try:
            self.driver.detach_interface(context, instance, condemned)
        except exception.NovaException as ex:
            # If the instance was deleted before the interface was detached,
            # just log it at debug.
            log_level = (logging.DEBUG
                         if isinstance(ex, exception.InstanceNotFound)
                         else logging.WARNING)
            LOG.log(log_level,
                    "Detach interface failed, port_id=%(port_id)s, reason: "
                    "%(msg)s", {'port_id': port_id, 'msg': ex},
                    instance=instance)
            raise exception.InterfaceDetachFailed(instance_uuid=instance.uuid)
        else:
            try:
                self.network_api.deallocate_port_for_instance(
                    context, instance, port_id)
            except Exception as ex:
                with excutils.save_and_reraise_exception():
                    # Since this is a cast operation, log the failure for
                    # triage.
                    LOG.warning('Failed to deallocate port %(port_id)s '
                                'for instance. Error: %(error)s',
                                {'port_id': port_id, 'error': ex},
                                instance=instance)

        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.INTERFACE_DETACH,
            phase=fields.NotificationPhase.END)
```



修改后代码：

```python
    @wrap_exception()
    @wrap_instance_event(prefix='compute')
    @wrap_instance_fault
    def detach_interface(self, context, instance, port_id):
        """Detach a network adapter from an instance."""
        network_info = instance.info_cache.network_info
        condemned = None
        for vif in network_info:
            if vif['id'] == port_id:
                condemned = vif
                break
        if condemned is None:
            raise exception.PortNotFound(_("Port %s is not "
                                           "attached") % port_id)

        self._notify_about_instance_usage(context, instance,
                                          "interface_detach.start")
        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.INTERFACE_DETACH,
            phase=fields.NotificationPhase.START)

        try:
            self.driver.detach_interface(context, instance, condemned)
        except exception.NovaException as ex:
            # If the instance was deleted before the interface was detached,
            # just log it at debug.
            log_level = (logging.DEBUG
                         if isinstance(ex, exception.InstanceNotFound)
                         else logging.WARNING)
            LOG.log(log_level,
                    "Detach interface failed, port_id=%(port_id)s, reason: "
                    "%(msg)s", {'port_id': port_id, 'msg': ex},
                    instance=instance)
            raise exception.InterfaceDetachFailed(instance_uuid=instance.uuid)
        else:
            try:
                self.network_api.deallocate_port_for_instance(
                    context, instance, port_id)
            except Exception as ex:
                with excutils.save_and_reraise_exception():
                    # Since this is a cast operation, log the failure for
                    # triage.
                    LOG.warning('Failed to deallocate port %(port_id)s '
                                'for instance. Error: %(error)s',
                                {'port_id': port_id, 'error': ex},
                                instance=instance)

        self._notify_about_instance_usage(context, instance,
                                          "interface_detach.end")
        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.INTERFACE_DETACH,
            phase=fields.NotificationPhase.END)
```



增加的内容为：

```python
<         self._notify_about_instance_usage(context, instance,
<                                           "interface_detach.start")
5934,5935c5928
<         self._notify_about_instance_usage(context, instance,
<                                           "interface_detach.end")
```





----

/nova/rpc.py

原代码：

```python
allowed_legacy_notification_event_types = [
        'aggregate.addhost.end',
        'aggregate.addhost.start',
        'aggregate.create.end',
        'aggregate.create.start',
        'aggregate.delete.end',
        'aggregate.delete.start',
        'aggregate.removehost.end',
        'aggregate.removehost.start',
        'aggregate.updatemetadata.end',
        'aggregate.updatemetadata.start',
        'aggregate.updateprop.end',
        'aggregate.updateprop.start',
        'compute.instance.create.end',
        'compute.instance.create.error',
        'compute.instance.create_ip.end',
        'compute.instance.create_ip.start',
        'compute.instance.create.start',
        'compute.instance.delete.end',
        'compute.instance.delete_ip.end',
        'compute.instance.delete_ip.start',
        'compute.instance.delete.start',
        'compute.instance.evacuate',
        'compute.instance.exists',
        'compute.instance.finish_resize.end',
        'compute.instance.finish_resize.start',
        'compute.instance.live.migration.abort.start',
        'compute.instance.live.migration.abort.end',
        'compute.instance.live.migration.force.complete.start',
        'compute.instance.live.migration.force.complete.end',
        'compute.instance.live_migration.post.dest.end',
        'compute.instance.live_migration.post.dest.start',
        'compute.instance.live_migration._post.end',
        'compute.instance.live_migration._post.start',
        'compute.instance.live_migration.pre.end',
        'compute.instance.live_migration.pre.start',
        'compute.instance.live_migration.rollback.dest.end',
        'compute.instance.live_migration.rollback.dest.start',
        'compute.instance.live_migration._rollback.end',
        'compute.instance.live_migration._rollback.start',
        'compute.instance.pause.end',
        'compute.instance.pause.start',
        'compute.instance.power_off.end',
        'compute.instance.power_off.start',
        'compute.instance.power_on.end',
        'compute.instance.power_on.start',
        'compute.instance.reboot.end',
        'compute.instance.reboot.error',
        'compute.instance.reboot.start',
        'compute.instance.rebuild.end',
        'compute.instance.rebuild.error',
        'compute.instance.rebuild.scheduled',
        'compute.instance.rebuild.start',
        'compute.instance.rescue.end',
        'compute.instance.rescue.start',
        'compute.instance.resize.confirm.end',
        'compute.instance.resize.confirm.start',
        'compute.instance.resize.end',
        'compute.instance.resize.error',
        'compute.instance.resize.prep.end',
        'compute.instance.resize.prep.start',
        'compute.instance.resize.revert.end',
        'compute.instance.resize.revert.start',
        'compute.instance.resize.start',
        'compute.instance.restore.end',
        'compute.instance.restore.start',
        'compute.instance.resume.end',
        'compute.instance.resume.start',
        'compute.instance.shelve.end',
        'compute.instance.shelve_offload.end',
        'compute.instance.shelve_offload.start',
        'compute.instance.shelve.start',
        'compute.instance.shutdown.end',
        'compute.instance.shutdown.start',
        'compute.instance.snapshot.end',
        'compute.instance.snapshot.start',
        'compute.instance.soft_delete.end',
        'compute.instance.soft_delete.start',
        'compute.instance.suspend.end',
        'compute.instance.suspend.start',
        'compute.instance.trigger_crash_dump.end',
        'compute.instance.trigger_crash_dump.start',
        'compute.instance.unpause.end',
        'compute.instance.unpause.start',
        'compute.instance.unrescue.end',
        'compute.instance.unrescue.start',
        'compute.instance.unshelve.start',
        'compute.instance.unshelve.end',
        'compute.instance.update',
        'compute.instance.volume.attach',
        'compute.instance.volume.detach',
        'compute.libvirt.error',
        'compute.metrics.update',
        'compute_task.build_instances',
        'compute_task.migrate_server',
        'compute_task.rebuild_server',
        'HostAPI.power_action.end',
        'HostAPI.power_action.start',
        'HostAPI.set_enabled.end',
        'HostAPI.set_enabled.start',
        'HostAPI.set_maintenance.end',
        'HostAPI.set_maintenance.start',
        'keypair.create.start',
        'keypair.create.end',
        'keypair.delete.start',
        'keypair.delete.end',
        'keypair.import.start',
        'keypair.import.end',
        'network.floating_ip.allocate',
        'network.floating_ip.associate',
        'network.floating_ip.deallocate',
        'network.floating_ip.disassociate',
        'scheduler.select_destinations.end',
        'scheduler.select_destinations.start',
        'servergroup.addmember',
        'servergroup.create',
        'servergroup.delete',
        'volume.usage',
    ]
```



修改后代码：

```python
allowed_legacy_notification_event_types = [
        'aggregate.addhost.end',
        'aggregate.addhost.start',
        'aggregate.create.end',
        'aggregate.create.start',
        'aggregate.delete.end',
        'aggregate.delete.start',
        'aggregate.removehost.end',
        'aggregate.removehost.start',
        'aggregate.updatemetadata.end',
        'aggregate.updatemetadata.start',
        'aggregate.updateprop.end',
        'aggregate.updateprop.start',
        'compute.instance.create.end',
        'compute.instance.create.error',
        'compute.instance.create_ip.end',
        'compute.instance.create_ip.start',
        'compute.instance.create.start',
        'compute.instance.delete.end',
        'compute.instance.delete_ip.end',
        'compute.instance.delete_ip.start',
        'compute.instance.delete.start',
        'compute.instance.evacuate',
        'compute.instance.exists',
        'compute.instance.finish_resize.end',
        'compute.instance.finish_resize.start',
        'compute.instance.live.migration.abort.start',
        'compute.instance.live.migration.abort.end',
        'compute.instance.live.migration.force.complete.start',
        'compute.instance.live.migration.force.complete.end',
        'compute.instance.live_migration.post.dest.end',
        'compute.instance.live_migration.post.dest.start',
        'compute.instance.live_migration._post.end',
        'compute.instance.live_migration._post.start',
        'compute.instance.live_migration.pre.end',
        'compute.instance.live_migration.pre.start',
        'compute.instance.live_migration.rollback.dest.end',
        'compute.instance.live_migration.rollback.dest.start',
        'compute.instance.live_migration._rollback.end',
        'compute.instance.live_migration._rollback.start',
        'compute.instance.pause.end',
        'compute.instance.pause.start',
        'compute.instance.power_off.end',
        'compute.instance.power_off.start',
        'compute.instance.power_on.end',
        'compute.instance.power_on.start',
        'compute.instance.reboot.end',
        'compute.instance.reboot.error',
        'compute.instance.reboot.start',
        'compute.instance.rebuild.end',
        'compute.instance.rebuild.error',
        'compute.instance.rebuild.scheduled',
        'compute.instance.rebuild.start',
        'compute.instance.rescue.end',
        'compute.instance.rescue.start',
        'compute.instance.resize.confirm.end',
        'compute.instance.resize.confirm.start',
        'compute.instance.resize.end',
        'compute.instance.resize.error',
        'compute.instance.resize.prep.end',
        'compute.instance.resize.prep.start',
        'compute.instance.resize.revert.end',
        'compute.instance.resize.revert.start',
        'compute.instance.resize.start',
        'compute.instance.restore.end',
        'compute.instance.restore.start',
        'compute.instance.resume.end',
        'compute.instance.resume.start',
        'compute.instance.shelve.end',
        'compute.instance.shelve_offload.end',
        'compute.instance.shelve_offload.start',
        'compute.instance.shelve.start',
        'compute.instance.shutdown.end',
        'compute.instance.shutdown.start',
        'compute.instance.snapshot.end',
        'compute.instance.snapshot.start',
        'compute.instance.soft_delete.end',
        'compute.instance.soft_delete.start',
        'compute.instance.suspend.end',
        'compute.instance.suspend.start',
        'compute.instance.trigger_crash_dump.end',
        'compute.instance.trigger_crash_dump.start',
        'compute.instance.unpause.end',
        'compute.instance.unpause.start',
        'compute.instance.unrescue.end',
        'compute.instance.unrescue.start',
        'compute.instance.unshelve.start',
        'compute.instance.unshelve.end',
        'compute.instance.update',
        'compute.instance.volume.attach',
        'compute.instance.volume.detach',
        'compute.instance.interface_attach.start',
        'compute.instance.interface_attach.end',
        'compute.instance.interface_detach.start',
        'compute.instance.interface_detach.end',
        'compute.libvirt.error',
        'compute.metrics.update',
        'compute_task.build_instances',
        'compute_task.migrate_server',
        'compute_task.rebuild_server',
        'HostAPI.power_action.end',
        'HostAPI.power_action.start',
        'HostAPI.set_enabled.end',
        'HostAPI.set_enabled.start',
        'HostAPI.set_maintenance.end',
        'HostAPI.set_maintenance.start',
        'keypair.create.start',
        'keypair.create.end',
        'keypair.delete.start',
        'keypair.delete.end',
        'keypair.import.start',
        'keypair.import.end',
        'network.floating_ip.allocate',
        'network.floating_ip.associate',
        'network.floating_ip.deallocate',
        'network.floating_ip.disassociate',
        'scheduler.select_destinations.end',
        'scheduler.select_destinations.start',
        'servergroup.addmember',
        'servergroup.create',
        'servergroup.delete',
        'volume.usage',
    ]
```



增加内容为：

```python
341,344d340
<         'compute.instance.interface_attach.start',
<         'compute.instance.interface_attach.end',
<         'compute.instance.interface_detach.start',
<         'compute.instance.interface_detach.end',
```



