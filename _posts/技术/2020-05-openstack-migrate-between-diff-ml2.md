---
layout: post
title: 跨不同类型ML2迁移vm
category: 技术
date: 2020-05-07

---

更新历史：

- 2020.05 完成

------

###

### 环境：

```
openstack版本：Rocky
hapervisor：libvirt + kvm
网络:linuxbridge-agent 和openvswitch-agent 组合方式
```

控制节点：

neutron.conf

```
[DEFAULT]
debug = True
log_dir = /var/log/kolla/neutron
use_stderr = False
bind_host = x.x.21.11
bind_port = 9696
api_paste_config = /usr/share/neutron/api-paste.ini
endpoint_type = internalURL
api_workers = 5
metadata_workers = 5
rpc_workers = 3
rpc_state_report_workers = 3
metadata_proxy_socket = /var/lib/neutron/kolla/metadata_proxy
#interface_driver = linuxbridge
allow_overlapping_ips = true
core_plugin = ml2
service_plugins = lbaasv2,qos,router
transport_url = rabbit://openstack:xxxx
ipam_driver = internal
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
dhcp_agents_per_network = 3
timeout = 360
url_timeout = 360
```

ml2_conf.ini

```
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,openvswitch,l2population,genericswitch
extension_drivers = port_security,dns_domain_ports,qos

[ml2_type_vlan]
network_vlan_ranges = provider

[ml2_type_flat]
flat_networks = *

[ml2_type_vxlan]
vni_ranges = 1:1000
vxlan_group = 239.1.1.1

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[linux_bridge]
physical_interface_mappings = provider:p2p2

[ovs]
bridge_mappings = provider:br-ex
local_ip = x.x.21.11

[vxlan]
l2_population = true
local_ip = x.x.21.11


[agent]
extensions = qos
```

计算节点(compute-21-39)：

部署linuxbridge-agent

neutron.conf

```
[DEFAULT]
debug = False
log_dir = /var/log/kolla/neutron
use_stderr = False
bind_host = x.x.21.39
bind_port = 9696
api_paste_config = /usr/share/neutron/api-paste.ini
endpoint_type = internalURL
api_workers = 5
metadata_workers = 5
rpc_workers = 3
rpc_state_report_workers = 3
metadata_proxy_socket = /var/lib/neutron/kolla/metadata_proxy
interface_driver = linuxbridge
allow_overlapping_ips = true
core_plugin = ml2
service_plugins = lbaasv2,qos,router
transport_url = rabbit:xxxxx
ipam_driver = internal
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
dhcp_agents_per_network = 3

...
```

ml2_conf.ini

```
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population,genericswitch
extension_drivers = port_security,dns_domain_ports,qos

[ml2_type_vlan]
network_vlan_ranges = provider

[ml2_type_flat]
flat_networks = *

[ml2_type_vxlan]
vni_ranges = 1:1000
vxlan_group = 239.1.1.1

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[linux_bridge]
physical_interface_mappings = provider:p2p2

[vxlan]
l2_population = true
local_ip = x.x.21.39

[agent]
extensions = qos

```



计算节点(compute-24-16)：

neutron.conf

```
[DEFAULT]
debug = True
log_dir = /var/log/kolla/neutron
use_stderr = False
bind_host = x.x.24.16
bind_port = 9696
api_paste_config = /usr/share/neutron/api-paste.ini
endpoint_type = internalURL
api_workers = 5
metadata_workers = 5
rpc_workers = 3
rpc_state_report_workers = 3
metadata_proxy_socket = /var/lib/neutron/kolla/metadata_proxy
#interface_driver = openvswitch
allow_overlapping_ips = true
core_plugin = ml2
service_plugins = lbaasv2,qos,router
transport_url = rabbit://openstack:xxxx
ipam_driver = internal
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
dhcp_agents_per_network = 3

```

ml2_conf.ini

```
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,baremetal,l2population
extension_drivers = port_security,dns_domain_ports,qos

[ml2_type_vlan]
network_vlan_ranges = provider

[ml2_type_flat]
flat_networks = *

[ml2_type_vxlan]
vni_ranges = 1:1000
vxlan_group = 239.1.1.1

[securitygroup]
#firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
firewall_driver = openvswitch

[agent]
tunnel_types = vxlan
l2_population = true
arp_responder = true
extensions = qos

[ovs]
bridge_mappings = provider:br-ex
#datapath_type = system
#ovsdb_connection = tcp:127.0.0.1:6640
local_ip = x.x.24.16
```





### 问题简述：

​		从linuxbridge-agent的计算节点热迁移虚拟机到openvswitch-agent的节点时，虚拟机可以正常迁移，但是迁移后网络不通，网卡tap设备没有桥接到默认的集成网桥br-int上。



### 问题复现：



​		通过指定宿主机的方式，在计算节点compute-21-39上创建一台虚拟机：

```
openstack server create --image centos-76 --flavor C2-R4G-D80G --network network-71 --availability-zone nova:bcompute-21-39 yan-live-1109-1
```

​		虚拟机：

```
+--------------------------------------+---------------------------------------------------------------------------------+
| Property                             | Value                                                                           |
+--------------------------------------+---------------------------------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                                          |
| OS-EXT-AZ:availability_zone          | nova                                                                            |
| OS-EXT-SRV-ATTR:host                 | compute-21-39                                                   |
| OS-EXT-SRV-ATTR:hostname             | yan-live-1109-1                                                         |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | compute-21-39                                                   |
| OS-EXT-SRV-ATTR:instance_name        | instance-00002e02                                                               |
| OS-EXT-SRV-ATTR:kernel_id            |                                                                                 |
| OS-EXT-SRV-ATTR:launch_index         | 0                                                                               |
| OS-EXT-SRV-ATTR:ramdisk_id           |                                                                                 |
| OS-EXT-SRV-ATTR:reservation_id       | r-z0e30t18                                                                      |
| OS-EXT-SRV-ATTR:root_device_name     | /dev/vda                                                                        |
| OS-EXT-SRV-ATTR:user_data            | -                                                                               |
| OS-EXT-STS:power_state               | 1                                                                               |
| OS-EXT-STS:task_state                | -                                                                               |
| OS-EXT-STS:vm_state                  | active                                                                          |
| OS-SRV-USG:launched_at               | 2020-11-09T02:42:38.000000                                                      |
| OS-SRV-USG:terminated_at             | -                                                                               |
| accessIPv4                           |                                                                                 |
| accessIPv6                           |                                                                                 |
| config_drive                         |                                                                                 |
| created                              | 2020-11-09T02:42:06Z                                                            |
| description                          | yan-live-1109-1                                                         |
| flavor:disk                          | 80                                                                              |
| flavor:ephemeral                     | 0                                                                               |
| flavor:extra_specs                   | {"quota:disk_total_iops_sec": "500", "quota:disk_total_bytes_sec": "102400000"} |
| flavor:original_name                 | C2-R4G-D80G                                                        |
| flavor:ram                           | 4096                                                                            |
| flavor:swap                          | 0                                                                               |
| flavor:vcpus                         | 2                                                                               |
| hostId                               | 68799dffb190eedd34c1fef79a0c3739a171bc939d6907176690b0df                        |
| host_status                          | UP                                                                              |
| id                                   | 41948c89-c320-4512-b002-90227d422c59                                            |
| image                                | centos-76 (a3eaae59-9a3f-4667-9af1-c619a7e021b9)                      |
| key_name                             | -                                                                               |
| locked                               | False                                                                           |
| metadata                             | {}                                                                              |
| name                                 | yan-live-1109-1                                                         |
| os-extended-volumes:volumes_attached | []                                                                              |
| progress                             | 0                                                                               |
| security_groups                      | default                                                                         |
| status                               | ACTIVE                                                                          |
| tags                                 | []                                                                              |
| tenant_id                            | 28e6517b7d6d4064be1bc878b590c40c                                                |
| updated                              | 2020-11-09T02:42:38Z                                                            |
| user_id                              | b114d7969c0e465fbd15c2911ca4bb23                                                |
| ops-71 network                    | x.x.71.164                                                                    |
+--------------------------------------+---------------------------------------------------------------------------------+
```



网卡信息：

```
+-----------------------+--------------------------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                                    |
+-----------------------+--------------------------------------------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                                                     |
| allowed_address_pairs |                                                                                                                          |
| binding:host_id       | compute-24-16                                                                                            |
| binding:profile       | {}                                                                                                                       |
| binding:vif_details   | {"port_filter": true, "bridge_name": "br-int", "datapath_type": "system", "ovs_hybrid_plug": false}                      |
| binding:vif_type      | ovs                                                                                                                      |
| binding:vnic_type     | normal                                                                                                                   |
| created_at            | 2020-11-09T02:42:09Z                                                                                                     |
| description           |                                                                                                                          |
| device_id             | 41948c89-c320-4512-b002-90227d422c59                                                                                     |
| device_owner          | compute:nova                                                                                                             |
| dns_assignment        | {"hostname": "yan-live-1109-1", "ip_address": "1.1.1.2", "fqdn": ""} 
|
| dns_domain            |                                                                                                                          |
| dns_name              | yan-live-1109-1                                                                                                  |
| extra_dhcp_opts       |                                                                                                                          |
| fixed_ips             | {"subnet_id": "2171e4e8-bd87-46d8-94a7-5dc839a8c789", "ip_address": "1.1.1.2"}                                      |
| id                    | cfcfd97b-b530-4b53-81d9-920097311fec                                                                                     |
| mac_address           | fa:16:3e:92:8f:fe                                                                                                        |
| name                  |                                                                                                                          |
| network_id            | 0fe9c836-58c5-4473-9e8c-16a1db4d16ba                                                                                     |
| port_security_enabled | True                                                                                                                     |
| project_id            | 28e6517b7d6d4064be1bc878b590c40c                                                                                         |
| qos_policy_id         |                                                                                                                          |
| revision_number       | 18                                                                                                                       |
| security_groups       | fdc1196b-b345-40fb-bfd8-9c920892c203                                                                                     |
| status                | ACTIVE                                                                                                                   |
| tags                  |                                                                                                                          |
| tenant_id             | 28e6517b7d6d4064be1bc878b590c40c                                                                                         |
| updated_at            | 2020-11-09T02:44:04Z                                                                                                     |
+-----------------------+--------------------------------------------------------------------------------------------------------------------------+
```



​		等待虚拟机在计算节点compute-21-39上启动成功后，执行热迁移动作。从计算节点compute-21-39上迁移到compute-22-16上。虚拟机可以正常迁移成功。但是迁移后，虚拟机网卡为DOWN状态。在计算节点compute-24-16上查看ovs 桥接情况，发现迁移后的u 虚拟机网卡没有桥街到br-int的桥上，而是新建了一个ovs桥brq0fe9c836-58，并且网卡的tap设备tapcfcfd97b-b5桥接到新建的qbr-0fe9c836-58上。

计算节点查看ovs桥接情况：

```
9d8e6e41-c74d-4f29-9a1f-5729c7fb494a
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "qvoeb5fdfeb-8d"
            tag: 2
            Interface "qvoeb5fdfeb-8d"
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    Bridge "brq0fe9c836-58"
        Port "tapcfcfd97b-b5"
            Interface "tapcfcfd97b-b5"
        Port "tapb1542e7b-83"
            Interface "tapb1542e7b-83"
        Port "brq0fe9c836-58"
            Interface "brq0fe9c836-58"
                type: internal
    Bridge br-ex
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-ex
            Interface br-ex
                type: internal
        Port "enp4s0f1"
            Interface "enp4s0f1"
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
```



### 问题分析：

​		经过排查日志和分析nova的热迁移的代码发现，梳理流程如下：



​	又api接收到的热迁移热迁移后，由于指定了目标主机，所以nova-schduler的部分不需要执行过多的调度任务，因此跳过schduler部分。这里从nova-condector开始分析。

nova-conductor

nova.conductor.manager.py

```
@targets_cell
    @wrap_instance_event(prefix='conductor')
    def live_migrate_instance(self, context, instance, scheduler_hint,
                              block_migration, disk_over_commit, request_spec):
        self._live_migrate(context, instance, scheduler_hint,
                           block_migration, disk_over_commit, request_spec)
```



准备数据，构建迁移所需的任务LiveMigrationTask，并且调用execute()执行热迁移任务：

```
    def _live_migrate(self, context, instance, scheduler_hint,
                      block_migration, disk_over_commit, request_spec):
        destination = scheduler_hint.get("host")

        def _set_vm_state(context, instance, ex, vm_state=None,
                          task_state=None):
            request_spec = {'instance_properties': {
                'uuid': instance.uuid, },
            }
            scheduler_utils.set_vm_state_and_notify(context,
                instance.uuid,
                'compute_task', 'migrate_server',
                dict(vm_state=vm_state,
                     task_state=task_state,
                     expected_task_state=task_states.MIGRATING,),
                ex, request_spec)

        migration = objects.Migration(context=context.elevated())
        migration.dest_compute = destination
        migration.status = 'accepted'
        migration.instance_uuid = instance.uuid
        migration.source_compute = instance.host
        migration.migration_type = 'live-migration'
        if instance.obj_attr_is_set('flavor'):
            migration.old_instance_type_id = instance.flavor.id
            migration.new_instance_type_id = instance.flavor.id
        else:
            migration.old_instance_type_id = instance.instance_type_id
            migration.new_instance_type_id = instance.instance_type_id
        migration.create()

        task = self._build_live_migrate_task(context, instance, destination,
                                             block_migration, disk_over_commit,
                                             migration, request_spec)
        try:
                    task.execute()
        except (exception.NoValidHost,
                exception.ComputeHostNotFound,
                exception.ComputeServiceUnavailable,
                exception.InvalidHypervisorType,
                exception.InvalidCPUInfo,
                exception.UnableToMigrateToSelf,
                exception.DestinationHypervisorTooOld,
                exception.InvalidLocalStorage,
                exception.InvalidSharedStorage,
                exception.HypervisorUnavailable,
                exception.InstanceInvalidState,
                exception.MigrationPreCheckError,
                exception.MigrationSchedulerRPCError) as ex:
            with excutils.save_and_reraise_exception():
                # TODO(johngarbutt) - eventually need instance actions here
                _set_vm_state(context, instance, ex, instance.vm_state)
                migration.status = 'error'
                migration.save()
        except Exception as ex:
            LOG.error('Migration of instance %(instance_id)s to host'
                      ' %(dest)s unexpectedly failed.',
                      {'instance_id': instance.uuid, 'dest': destination},
                      exc_info=True)
            # Reset the task state to None to indicate completion of
            # the operation as it is done in case of known exceptions.
            _set_vm_state(context, instance, ex, vm_states.ERROR,
                          task_state=None)
            migration.status = 'error'
            migration.save()
            raise exception.MigrationError(reason=six.text_type(ex))
```



```
    def _build_live_migrate_task(self, context, instance, destination,
                                 block_migration, disk_over_commit, migration,
                                 request_spec=None):
        return live_migrate.LiveMigrationTask(context, instance,
                                              destination, block_migration,
                                              disk_over_commit, migration,
                                              self.compute_rpcapi,
                                              self.servicegroup_api,
                                              self.scheduler_client,
                                              request_spec)
```



接下来代码会执行到 nova.conductor.tasks.live_migrate.py

```
    def _execute(self):
        self._check_instance_is_active()
        self._check_instance_has_no_numa()
        self._check_host_is_up(self.source)

        if should_do_migration_allocation(self.context):
            self._source_cn, self._held_allocations = (
                # NOTE(danms): This may raise various exceptions, which will
                # propagate to the API and cause a 500. This is what we
                # want, as it would indicate internal data structure corruption
                # (such as missing migrations, compute nodes, etc).
                migrate.replace_allocation_with_migration(self.context,
                                                          self.instance,
                                                          self.migration))

        if not self.destination:
            # Either no host was specified in the API request and the user
            # wants the scheduler to pick a destination host, or a host was
            # specified but is not forcing it, so they want the scheduler
            # filters to run on the specified host, like a scheduler hint.
            self.destination, dest_node = self._find_destination()
        else:
            # This is the case that the user specified the 'force' flag when
            # live migrating with a specific destination host so the scheduler
            # is bypassed. There are still some minimal checks performed here
            # though.
            source_node, dest_node = self._check_requested_destination()
            # Now that we're semi-confident in the force specified host, we
            # need to copy the source compute node allocations in Placement
            # to the destination compute node. Normally select_destinations()
            # in the scheduler would do this for us, but when forcing the
            # target host we don't call the scheduler.
            # TODO(mriedem): In Queens, call select_destinations() with a
            # skip_filters=True flag so the scheduler does the work of claiming
            # resources on the destination in Placement but still bypass the
            # scheduler filters, which honors the 'force' flag in the API.
            # This raises NoValidHost which will be handled in
            # ComputeTaskManager.
            scheduler_utils.claim_resources_on_destination(
                self.context, self.scheduler_client.reportclient,
                self.instance, source_node, dest_node,
                source_node_allocations=self._held_allocations)

            # dest_node is a ComputeNode object, so we need to get the actual
            # node name off it to set in the Migration object below.
            dest_node = dest_node.hypervisor_hostname

        self.instance.availability_zone = (
            availability_zones.get_host_availability_zone(
                self.context, self.destination))

        self.migration.source_node = self.instance.node
        self.migration.dest_node = dest_node
        self.migration.dest_compute = self.destination
        self.migration.save()

        # TODO(johngarbutt) need to move complexity out of compute manager
        # TODO(johngarbutt) disk_over_commit?
        return self.compute_rpcapi.live_migration(self.context,
                host=self.source,
                instance=self.instance,
                dest=self.destination,
                block_migration=self.block_migration,
                migration=self.migration,
                migrate_data=self.migrate_data)
```



_execute() 执行检查目标节点，例如检查目标节点是否正常，hypervisor的类型是否允许迁移等。检查正确后，通过rpcapi将任务发送到迁移的source host计算节点 compute-21-16上nova-compute。

```
 # TODO(johngarbutt) need to move complexity out of compute manager
        # TODO(johngarbutt) disk_over_commit?
        return self.compute_rpcapi.live_migration(self.context,
                host=self.source,
                instance=self.instance,
                dest=self.destination,
                block_migration=self.block_migration,
                migration=self.migration,
                migrate_data=self.migrate_data)
```

计算节点nova-compute接收到热迁移任务：

```
@wrap_exception()
    @wrap_instance_event(prefix='compute')
    @wrap_instance_fault
    def live_migration(self, context, dest, instance, block_migration,
                       migration, migrate_data):
        """Executing live migration.

        :param context: security context
        :param dest: destination host
        :param instance: a nova.objects.instance.Instance object
        :param block_migration: if true, prepare for block migration
        :param migration: an nova.objects.Migration object
        :param migrate_data: implementation specific params

        """
        self._set_migration_status(migration, 'queued')
        # NOTE(Kevin_Zheng): Submit the live_migration job to the pool and
        # put the returned Future object into dict mapped with migration.uuid
        # in order to be able to track and abort it in the future.
        self._waiting_live_migrations[instance.uuid] = (None, None)
        try:
            future = self._live_migration_executor.submit(
                self._do_live_migration, context, dest, instance,
                block_migration, migration, migrate_data)
            self._waiting_live_migrations[instance.uuid] = (migration, future)
        except RuntimeError:
            # ThreadPoolExecutor.submit will raise RuntimeError if the pool
            # is shutdown, which happens in _cleanup_live_migrations_in_pool.
            LOG.info('Migration %s failed to submit as the compute service '
                     'is shutting down.', migration.uuid, instance=instance)
            self._set_migration_status(migration, 'error')
            raise exception.LiveMigrationNotSubmitted(
                migration_uuid=migration.uuid, instance_uuid=instance.uuid)
```



设置任务的入执行队列后 通过_do_live_migration()来时执行热迁移任务：

```
    def _do_live_migration(self, context, dest, instance, block_migration,
                           migration, migrate_data):
        # NOTE(danms): We should enhance the RT to account for migrations
        # and use the status field to denote when the accounting has been
        # done on source/destination. For now, this is just here for status
        # reporting
        self._set_migration_status(migration, 'preparing')
        source_bdms = objects.BlockDeviceMappingList.get_by_instance_uuid(
                context, instance.uuid)

        class _BreakWaitForInstanceEvent(Exception):
            """Used as a signal to stop waiting for the network-vif-plugged
            event when we discover that
            [compute]/live_migration_wait_for_vif_plug is not set on the
            destination.
            """
            pass

        events = self._get_neutron_events_for_live_migration(instance)
        try:
            if ('block_migration' in migrate_data and
                    migrate_data.block_migration):
                block_device_info = self._get_instance_block_device_info(
                    context, instance, bdms=source_bdms)
                disk = self.driver.get_instance_disk_info(
                    instance, block_device_info=block_device_info)
            else:
                disk = None

            deadline = CONF.vif_plugging_timeout
            error_cb = self._neutron_failed_live_migration_callback
            # In order to avoid a race with the vif plugging that the virt
            # driver does on the destination host, we register our events
            # to wait for before calling pre_live_migration. Then if the
            # dest host reports back that we shouldn't wait, we can break
            # out of the context manager using _BreakWaitForInstanceEvent.
            with self.virtapi.wait_for_instance_event(
                    instance, events, deadline=deadline,
                    error_callback=error_cb):
                with timeutils.StopWatch() as timer:
                    migrate_data = self.compute_rpcapi.pre_live_migration(
                        context, instance,
                        block_migration, disk, dest, migrate_data)
                LOG.info('Took %0.2f seconds for pre_live_migration on '
                         'destination host %s.',
                         timer.elapsed(), dest, instance=instance)
                wait_for_vif_plugged = (
                    'wait_for_vif_plugged' in migrate_data and
                    migrate_data.wait_for_vif_plugged)
                if events and not wait_for_vif_plugged:
                    raise _BreakWaitForInstanceEvent
        except _BreakWaitForInstanceEvent:
            if events:
                LOG.debug('Not waiting for events after pre_live_migration: '
                          '%s. ', events, instance=instance)
            # This is a bit weird, but we need to clear sys.exc_info() so that
            # oslo.log formatting does not inadvertently use it later if an
            # error message is logged without an explicit exc_info. This is
            # only a problem with python 2.
            if six.PY2:
                sys.exc_clear()
        except exception.VirtualInterfacePlugException:
            with excutils.save_and_reraise_exception():
                LOG.exception('Failed waiting for network virtual interfaces '
                              'to be plugged on the destination host %s.',
                              dest, instance=instance)
                self._cleanup_pre_live_migration(
                    context, dest, instance, migration, migrate_data)
        except eventlet.timeout.Timeout:
            msg = 'Timed out waiting for events: %s'
            LOG.warning(msg, events, instance=instance)
            if CONF.vif_plugging_is_fatal:
                self._cleanup_pre_live_migration(
                    context, dest, instance, migration, migrate_data)
                raise exception.MigrationError(reason=msg % events)
        except Exception:
            with excutils.save_and_reraise_exception():
                LOG.exception('Pre live migration failed at %s',
                              dest, instance=instance)
                self._cleanup_pre_live_migration(
                    context, dest, instance, migration, migrate_data)

        # NOTE(Kevin_Zheng): Pop the migration from the waiting queue
        # if it exist in the queue, then we are good to moving on, if
        # not, some other process must have aborted it, then we should
        # rollback.
        try:
            self._waiting_live_migrations.pop(instance.uuid)
        except KeyError:
            LOG.debug('Migration %s aborted by another process, rollback.',
                      migration.uuid, instance=instance)
            migrate_data.migration = migration
            self._rollback_live_migration(context, instance, dest,
                                          migrate_data, 'cancelled')
            self._notify_live_migrate_abort_end(context, instance)
            return

        self._set_migration_status(migration, 'running')
        if migrate_data:
            migrate_data.migration = migration

        # NOTE(mdbooth): pre_live_migration will update connection_info and
        # attachment_id on all volume BDMS to reflect the new destination
        # host attachment. We fetch BDMs before that to retain connection_info
        # and attachment_id relating to the source host for post migration
        # cleanup.
        post_live_migration = functools.partial(self._post_live_migration,
                                                source_bdms=source_bdms)

        LOG.debug('live_migration data is %s', migrate_data)
        try:
            self.driver.live_migration(context, instance, dest,
                                       post_live_migration,
                                       self._rollback_live_migration,
                                       block_migration, migrate_data)
        except Exception:
            LOG.exception('Live migration failed.', instance=instance)
            with excutils.save_and_reraise_exception():
                # Put instance and migration into error state,
                # as its almost certainly too late to rollback
                self._set_migration_status(migration, 'error')
                # first refresh instance as it may have got updated by
                # post_live_migration_at_destination
                instance.refresh()
                self._set_instance_obj_error_state(context, instance,
                                                   clean_task_state=True)
```



热迁移的任务，在执行过程中，分为两个步骤，pre_live_migrate 和post_live_migrate，这里我们先看pre_live_migrate()

```
    def pre_live_migration(self, context, instance, block_migration, disk,
                           migrate_data):
        """Preparations for live migration at dest host.

        :param context: security context
        :param instance: dict of instance data
        :param block_migration: if true, prepare for block migration
        :param disk: disk info of instance
        :param migrate_data: A dict or LiveMigrateData object holding data
                             required for live migration without shared
                             storage.
        :returns: migrate_data containing additional migration info
        """
        LOG.debug('pre_live_migration data is %s', migrate_data)

        migrate_data.old_vol_attachment_ids = {}
        bdms = objects.BlockDeviceMappingList.get_by_instance_uuid(
            context, instance.uuid)
        network_info = self.network_api.get_instance_nw_info(context, instance)
        self._notify_about_instance_usage(
            context, instance, "live_migration.pre.start",
            network_info=network_info)
        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.LIVE_MIGRATION_PRE,
            phase=fields.NotificationPhase.START, bdms=bdms)

        connector = self.driver.get_volume_connector(instance)
        try:
            for bdm in bdms:
                if bdm.is_volume and bdm.attachment_id is not None:
                    # This bdm uses the new cinder v3.44 API.
                    # We will create a new attachment for this
                    # volume on this migration destination host. The old
                    # attachment will be deleted on the source host
                    # when the migration succeeds. The old attachment_id
                    # is stored in dict with the key being the bdm.volume_id
                    # so it can be restored on rollback.
                    #
                    # Also note that attachment_update is not needed as we
                    # are providing the connector in the create call.
                    attach_ref = self.volume_api.attachment_create(
                        context, bdm.volume_id, bdm.instance_uuid,
                        connector=connector, mountpoint=bdm.device_name)

                    # save current attachment so we can detach it on success,
                    # or restore it on a rollback.
                    # NOTE(mdbooth): This data is no longer used by the source
                    # host since change I0390c9ff. We can't remove it until we
                    # are sure the source host has been upgraded.
                    migrate_data.old_vol_attachment_ids[bdm.volume_id] = \
                        bdm.attachment_id

                    # update the bdm with the new attachment_id.
                    bdm.attachment_id = attach_ref['id']
                    bdm.save()

            block_device_info = self._get_instance_block_device_info(
                                context, instance, refresh_conn_info=True,
                                bdms=bdms)

            # The driver pre_live_migration will plug vifs on the host. We call
            # plug_vifs before calling ensure_filtering_rules_for_instance, to
            # ensure bridge is set up.
            migrate_data = self.driver.pre_live_migration(context,
                                           instance,
                                           block_device_info,
                                           network_info,
                                           disk,
                                           migrate_data)
            LOG.debug('driver pre_live_migration data is %s', migrate_data)
            # driver.pre_live_migration is what plugs vifs on the destination
            # host so now we can set the wait_for_vif_plugged flag in the
            # migrate_data object which the source compute will use to
            # determine if it should wait for a 'network-vif-plugged' event
            # from neutron before starting the actual guest transfer in the
            # hypervisor
            migrate_data.wait_for_vif_plugged = (
                CONF.compute.live_migration_wait_for_vif_plug)

            # NOTE(tr3buchet): setup networks on destination host
            self.network_api.setup_networks_on_host(context, instance,
                                                             self.host)

            # Creating filters to hypervisors and firewalls.
            # An example is that nova-instance-instance-xxx,
            # which is written to libvirt.xml(Check "virsh nwfilter-list")
            # This nwfilter is necessary on the destination host.
            # In addition, this method is creating filtering rule
            # onto destination host.
            self.driver.ensure_filtering_rules_for_instance(instance,
                                                network_info)
        except Exception:
            # If we raise, migrate_data with the updated attachment ids
            # will not be returned to the source host for rollback.
            # So we need to rollback new attachments here.
            with excutils.save_and_reraise_exception():
                old_attachments = migrate_data.old_vol_attachment_ids
                for bdm in bdms:
                    if (bdm.is_volume and bdm.attachment_id is not None and
                            bdm.volume_id in old_attachments):
                        self.volume_api.attachment_delete(context,
                                                          bdm.attachment_id)
                        bdm.attachment_id = old_attachments[bdm.volume_id]
                        bdm.save()

        # Volume connections are complete, tell cinder that all the
        # attachments have completed.
        for bdm in bdms:
            if bdm.is_volume and bdm.attachment_id is not None:
                self.volume_api.attachment_complete(context,
                                                    bdm.attachment_id)

        self._notify_about_instance_usage(
                     context, instance, "live_migration.pre.end",
                     network_info=network_info)
        compute_utils.notify_about_instance_action(
            context, instance, self.host,
            action=fields.NotificationAction.LIVE_MIGRATION_PRE,
            phase=fields.NotificationPhase.END, bdms=bdms)

        LOG.debug('pre_live_migration result data is %s', migrate_data)
        return migrate_data
```

通过代码和注释可以看到这个pre_live_migrate在dest host节点compute-21-16上进行的，基本做了网卡接入时的网络相关检查和准备以及磁盘块设备的相关信息收取等，这里需要注意的时，虽然代码的注释上写了he driver pre_live_migration will plug vifs on the host，但是这里确没有进行vifs的plug操作，只是对vif所需的桥设备进行检查和确认。



做完pre_live_migrate后，source host节点compute-21-39调用本节点上的hypervisor driver继续进行热迁移操作：

```
    def live_migration(self, context, instance, dest,
                       post_method, recover_method, block_migration=False,
                       migrate_data=None):
        """Spawning live_migration operation for distributing high-load.

        """

        # 'dest' will be substituted into 'migration_uri' so ensure
        # it does't contain any characters that could be used to
        # exploit the URI accepted by libivrt
        if not libvirt_utils.is_valid_hostname(dest):
            raise exception.InvalidHostname(hostname=dest)

        self._live_migration(context, instance, dest,
                             post_method, recover_method, block_migration,
                             migrate_data)
```



```
    def _live_migration(self, context, instance, dest, post_method,
                        recover_method, block_migration,
                        migrate_data):
        """Do live migration.

        :param context: security context
        :param instance:
            nova.db.sqlalchemy.models.Instance object
            instance object that is migrated.
        :param dest: destination host
        :param post_method:
            post operation method.
            expected nova.compute.manager._post_live_migration.
        :param recover_method:
            recovery method when any exception occurs.
            expected nova.compute.manager._rollback_live_migration.
        :param block_migration: if true, do block migration.
        :param migrate_data: a LibvirtLiveMigrateData object

        This fires off a new thread to run the blocking migration
        operation, and then this thread monitors the progress of
        migration and controls its operation
        """

        guest = self._host.get_guest(instance)

        disk_paths = []
        device_names = []
        if (migrate_data.block_migration and
                CONF.libvirt.virt_type != "parallels"):
            disk_paths, device_names = self._live_migration_copy_disk_paths(
                context, instance, guest)

        opthread = utils.spawn(self._live_migration_operation,
                                     context, instance, dest,
                                     block_migration,
                                     migrate_data, guest,
                                     device_names)

        finish_event = eventlet.event.Event()
        self.active_migrations[instance.uuid] = deque()

        def thread_finished(thread, event):
            LOG.debug("Migration operation thread notification",
                      instance=instance)
            event.send()
        opthread.link(thread_finished, finish_event)

        # Let eventlet schedule the new thread right away
        time.sleep(0)

        try:
            LOG.debug("Starting monitoring of live migration",
                      instance=instance)
            self._live_migration_monitor(context, instance, guest, dest,
                                         post_method, recover_method,
                                         block_migration, migrate_data,
                                         finish_event, disk_paths)
        except Exception as ex:
            LOG.warning("Error monitoring migration: %(ex)s",
                        {"ex": ex}, instance=instance, exc_info=True)
            raise
        finally:
            LOG.debug("Live migration monitoring is all done",
                      instance=instance)

```

source host的 hypervisor driver，先读取了被迁移的instance的虚拟机xml文件，然后调用_live_migration_operation通过hypervisor调用底层的libvirtd的接口开始执行迁移操作。

```
    def _live_migration_operation(self, context, instance, dest,
                                  block_migration, migrate_data, guest,
                                  device_names):
        """Invoke the live migration operation

        :param context: security context
        :param instance:
            nova.db.sqlalchemy.models.Instance object
            instance object that is migrated.
        :param dest: destination host
        :param block_migration: if true, do block migration.
        :param migrate_data: a LibvirtLiveMigrateData object
        :param guest: the guest domain object
        :param device_names: list of device names that are being migrated with
            instance

        This method is intended to be run in a background thread and will
        block that thread until the migration is finished or failed.
        """
        try:
            if migrate_data.block_migration:
                migration_flags = self._block_migration_flags
            else:
                migration_flags = self._live_migration_flags

            serial_listen_addr = libvirt_migrate.serial_listen_addr(
                migrate_data)
            if not serial_listen_addr:
                # In this context we want to ensure that serial console is
                # disabled on source node. This is because nova couldn't
                # retrieve serial listen address from destination node, so we
                # consider that destination node might have serial console
                # disabled as well.
                self._verify_serial_console_is_disabled()

            # NOTE(aplanas) migrate_uri will have a value only in the
            # case that `live_migration_inbound_addr` parameter is
            # set, and we propose a non tunneled migration.
            migrate_uri = None
            if ('target_connect_addr' in migrate_data and
                    migrate_data.target_connect_addr is not None):
                dest = migrate_data.target_connect_addr
                if (migration_flags &
                    libvirt.VIR_MIGRATE_TUNNELLED == 0):
                    migrate_uri = self._migrate_uri(dest)

            new_xml_str = None
            if CONF.libvirt.virt_type != "parallels":
                # If the migrate_data has port binding information for the
                # destination host, we need to prepare the guest vif config
                # for the destination before we start migrating the guest.
                get_vif_config = None
                if 'vifs' in migrate_data and migrate_data.vifs:
                    # NOTE(mriedem): The vif kwarg must be built on the fly
                    # within get_updated_guest_xml based on migrate_data.vifs.
                    # We could stash the virt_type from the destination host
                    # into LibvirtLiveMigrateData but the host kwarg is a
                    # nova.virt.libvirt.host.Host object and is used to check
                    # information like libvirt version on the destination.
                    # If this becomes a problem, what we could do is get the
                    # VIF configs while on the destination host during
                    # pre_live_migration() and store those in the
                    # LibvirtLiveMigrateData object. For now we just use the
                    # source host information for virt_type and
                    # host (version) since the conductor live_migrate method
                    # _check_compatible_with_source_hypervisor() ensures that
                    # the hypervisor types and versions are compatible.
                    get_vif_config = functools.partial(
                        self.vif_driver.get_config,
                        instance=instance,
                        image_meta=instance.image_meta,
                        inst_type=instance.flavor,
                        virt_type=CONF.libvirt.virt_type,
                        host=self._host)
                new_xml_str = libvirt_migrate.get_updated_guest_xml(
                    # TODO(sahid): It's not a really good idea to pass
                    # the method _get_volume_config and we should to find
                    # a way to avoid this in future.
                    guest, migrate_data, self._get_volume_config,
                    get_vif_config=get_vif_config)

            # NOTE(pkoniszewski): Because of precheck which blocks
            # tunnelled block live migration with mapped volumes we
            # can safely remove migrate_disks when tunnelling is on.
            # Otherwise we will block all tunnelled block migrations,
            # even when an instance does not have volumes mapped.
            # This is because selective disk migration is not
            # supported in tunnelled block live migration. Also we
            # cannot fallback to migrateToURI2 in this case because of
            # bug #1398999
            #
            # TODO(kchamart) Move the following bit to guest.migrate()
            if (migration_flags & libvirt.VIR_MIGRATE_TUNNELLED != 0):
                device_names = []

            # TODO(sahid): This should be in
            # post_live_migration_at_source but no way to retrieve
            # ports acquired on the host for the guest at this
            # step. Since the domain is going to be removed from
            # libvird on source host after migration, we backup the
            # serial ports to release them if all went well.
            serial_ports = []
            if CONF.serial_console.enabled:
                serial_ports = list(self._get_serial_ports_from_guest(guest))

            LOG.debug("About to invoke the migrate API", instance=instance)
            guest.migrate(self._live_migration_uri(dest),
                          migrate_uri=migrate_uri,
                          flags=migration_flags,
                          migrate_disks=device_names,
                          destination_xml=new_xml_str,
                          bandwidth=CONF.libvirt.live_migration_bandwidth)
            LOG.debug("Migrate API has completed", instance=instance)

            for hostname, port in serial_ports:
                serial_console.release_port(host=hostname, port=port)
        except Exception as e:
            with excutils.save_and_reraise_exception():
                LOG.error("Live Migration failure: %s", e, instance=instance)

        # If 'migrateToURI' fails we don't know what state the
        # VM instances on each host are in. Possibilities include
        #
        #  1. src==running, dst==none
        #
        #     Migration failed & rolled back, or never started
        #
        #  2. src==running, dst==paused
        #
        #     Migration started but is still ongoing
        #
        #  3. src==paused,  dst==paused
        #
        #     Migration data transfer completed, but switchover
        #     is still ongoing, or failed
        #
        #  4. src==paused,  dst==running
        #
        #     Migration data transfer completed, switchover
        #     happened but cleanup on source failed
        #
        #  5. src==none,    dst==running
        #
        #     Migration fully succeeded.
        #
        # Libvirt will aim to complete any migration operation
        # or roll it back. So even if the migrateToURI call has
        # returned an error, if the migration was not finished
        # libvirt should clean up.
        #
        # So we take the error raise here with a pinch of salt
        # and rely on the domain job info status to figure out
        # what really happened to the VM, which is a much more
        # reliable indicator.
        #
        # In particular we need to try very hard to ensure that
        # Nova does not "forget" about the guest. ie leaving it
        # running on a different host to the one recorded in
        # the database, as that would be a serious resource leak

        LOG.debug("Migration operation thread has finished",
                  instance=instance)
```



通过这里可以看到迁移到目标计算节点后，网卡的tap设备挂在到哪个网桥是根据instance的xml文件决定的。我们通过上面代码可以看到在这里会对guest的xml文件进行更新，但是根据目前问题的现象可以推测出，关于vif的xml配置内容并没有更新，还是和迁移的source节点上通过linuxbridge-agent管理的vif信息一致，才会导致，虚拟机被迁移后挂在的桥设备名称和源节点的linuxbridge的桥名称:brqxxxxxx-xx一致。



这里我们着重看下xml的更新部分做了哪些工作：

```
new_xml_str = None
            if CONF.libvirt.virt_type != "parallels":
                # If the migrate_data has port binding information for the
                # destination host, we need to prepare the guest vif config
                # for the destination before we start migrating the guest.
                get_vif_config = None
                if 'vifs' in migrate_data and migrate_data.vifs:
                    # NOTE(mriedem): The vif kwarg must be built on the fly
                    # within get_updated_guest_xml based on migrate_data.vifs.
                    # We could stash the virt_type from the destination host
                    # into LibvirtLiveMigrateData but the host kwarg is a
                    # nova.virt.libvirt.host.Host object and is used to check
                    # information like libvirt version on the destination.
                    # If this becomes a problem, what we could do is get the
                    # VIF configs while on the destination host during
                    # pre_live_migration() and store those in the
                    # LibvirtLiveMigrateData object. For now we just use the
                    # source host information for virt_type and
                    # host (version) since the conductor live_migrate method
                    # _check_compatible_with_source_hypervisor() ensures that
                    # the hypervisor types and versions are compatible.
                    get_vif_config = functools.partial(
                        self.vif_driver.get_config,
                        instance=instance,
                        image_meta=instance.image_meta,
                        inst_type=instance.flavor,
                        virt_type=CONF.libvirt.virt_type,
                        host=self._host)
                new_xml_str = libvirt_migrate.get_updated_guest_xml(
                    # TODO(sahid): It's not a really good idea to pass
                    # the method _get_volume_config and we should to find
                    # a way to avoid this in future.
                    guest, migrate_data, self._get_volume_config,
                    get_vif_config=get_vif_config)
```



可以看到在self.vif_driver.get_config对vif的配置进行了更新，再看：

nova.virt.libvirt.vif.py

```
def get_config(self, instance, vif, image_meta,
               inst_type, virt_type, host):
    vif_type = vif['type']
    vnic_type = vif['vnic_type']

    # instance.display_name could be unicode
    instance_repr = utils.get_obj_repr_unicode(instance)
    LOG.debug('vif_type=%(vif_type)s instance=%(instance)s '
              'vif=%(vif)s virt_type=%(virt_type)s',
              {'vif_type': vif_type, 'instance': instance_repr,
               'vif': vif, 'virt_type': virt_type})

    if vif_type is None:
        raise exception.InternalError(
            _("vif_type parameter must be present "
              "for this vif_driver implementation"))

    # Try os-vif codepath first
    vif_obj = os_vif_util.nova_to_osvif_vif(vif)
    if vif_obj is not None:
        return self._get_config_os_vif(instance, vif_obj, image_meta,
                                       inst_type, virt_type, host,
                                       vnic_type)

    # Legacy non-os-vif codepath
    vif_slug = self._normalize_vif_type(vif_type)
    func = getattr(self, 'get_config_%s' % vif_slug, None)
    if not func:
        raise exception.InternalError(
            _("Unexpected vif_type=%s") % vif_type)
    return func(instance, vif, image_meta,
                inst_type, virt_type, host)
```



nova.network.os_vif_util.py

构建vif_obj

```
def nova_to_osvif_vif(vif):
    """Convert a Nova VIF model to an os-vif object

    :param vif: a nova.network.model.VIF instance

    Attempt to convert a nova VIF instance into an os-vif
    VIF object, pointing to a suitable plugin. This will
    return None if there is no os-vif plugin available yet.

    :returns: a os_vif.objects.vif.VIFBase subclass, or None
      if not supported with os-vif yet
    """

    LOG.debug("Converting VIF %s", vif)

    funcname = "_nova_to_osvif_vif_" + vif['type'].replace(".", "_")
    func = getattr(sys.modules[__name__], funcname, None)

    if not func:
        raise exception.NovaException(
            "Unsupported VIF type %(type)s convert '%(func)s'" %
            {'type': vif['type'], 'func': funcname})

    try:
        vifobj = func(vif)
        LOG.debug("Converted object %s", vifobj)
        return vifobj
    except NotImplementedError:
        LOG.debug("No conversion for VIF type %s yet",
                  vif['type'])
        return None

```



```
# VIF_TYPE_OVS = 'ovs'
def _nova_to_osvif_vif_ovs(vif):
    vnic_type = vif.get('vnic_type', model.VNIC_TYPE_NORMAL)
    profile = objects.vif.VIFPortProfileOpenVSwitch(
        interface_id=vif.get('ovs_interfaceid') or vif['id'],
        datapath_type=vif['details'].get(
            model.VIF_DETAILS_OVS_DATAPATH_TYPE))
    if vnic_type == model.VNIC_TYPE_DIRECT:
        profile = objects.vif.VIFPortProfileOVSRepresentor(
            interface_id=vif.get('ovs_interfaceid') or vif['id'],
            representor_name=_get_vif_name(vif),
            representor_address=vif["profile"]['pci_slot'])
        obj = _get_vif_instance(
            vif,
            objects.vif.VIFHostDevice,
            port_profile=profile,
            plugin="ovs",
            dev_address=vif["profile"]['pci_slot'],
            dev_type=os_vif_fields.VIFHostDeviceDevType.ETHERNET)
        if vif["network"]["bridge"] is not None:
            obj.network.bridge = vif["network"]["bridge"]
    elif _is_firewall_required(vif) or vif.is_hybrid_plug_enabled():
        obj = _get_vif_instance(
            vif,
            objects.vif.VIFBridge,
            port_profile=profile,
            plugin="ovs",
            vif_name=_get_vif_name(vif),
            bridge_name=_get_hybrid_bridge_name(vif))
    else:
        obj = _get_vif_instance(
            vif,
            objects.vif.VIFOpenVSwitch,
            port_profile=profile,
            plugin="ovs",
            vif_name=_get_vif_name(vif))
        if vif["network"]["bridge"] is not None:
            obj.bridge_name = vif["network"]["bridge"]
    return obj

```



我们来看下这个vif具体包含哪些内容：

```
vif={
    "profile": {
        
    },
    "ovs_interfaceid": null,
    "preserve_on_delete": false,
    "network": {
        "bridge": "brq0fe9c836-58",
        "label": "ops-71",
        "meta": {
            "tunneled": false,
            "should_create_bridge": true,
            "tenant_id": "0088e55d8e5c44868637e0bfb3181609",
            "mtu": 1500,
            "physical_network": "provider",
            "injected": false
        },
        "id": "0fe9c836-58c5-4473-9e8c-16a1db4d16ba",
        "subnets": [
            {
                "ips": [
                    {
                        "meta": {
                            
                        },
                        "type": "fixed",
                        "version": 4,
                        "address": "x.x.71.35",
                        "floating_ips": [
                            
                        ]
                    }
                ],
                "version": 4,
                "meta": {
                    "dhcp_server": "x.x.71.4"
                },
                "dns": [
                    {
                        "meta": {
                            
                        },
                        "type": "dns",
                        "version": 4,
                        "address": "x.x.70.66"
                    },
                    {
                        "meta": {
                            
                        },
                        "type": "dns",
                        "version": 4,
                        "address": "x.x.70.92"
                    },
                    {
                        "meta": {
                            
                        },
                        "type": "dns",
                        "version": 4,
                        "address": "x.x.70.95"
                    }
                ],
                "routes": [
                    
                ],
                "cidr": "x.x.71.0/24",
                "gateway": {
                    "meta": {
                        
                    },
                    "type": "gateway",
                    "version": 4,
                    "address": "x.x.71.1"
                }
            }
        ]
    },
    "devname": "tapcfcfd97b-b5",
    "qbh_params": null,
    "vnic_type": "normal",
    "meta": {
        
    },
    "details": {
        "port_filter": true,
        "bridge_name": "br-int",
        "datapath_type": "system",
        "ovs_hybrid_plug": false
    },
    "address": "fa:16:3e:74:0a:5f",
    "active": false,
    "type": "ovs",
    "id": "4e6f8e26-6b81-4f03-81b5-02fb510660a1",
    "qbg_params": null
}
```



然后通过上面的代码可以看到，vifobj的bridge_name被设置成：

```
obj.bridge_name = vif["network"]["bridge"]
```

vifobj的devname保持不变:

```
"devname": "tapcfcfd97b-b5"
```

到这里我们可以看到，在def _live_migration_operation（）进行了更新xml动作，但是关于vif的信息并没有改变，还是和source节点的保持一致，这样就会导致迁移后的instance的网卡被挂在到了qbrxxxxxxx-xx上，而不是br-int上



因此，我们这里对nova.network.os_vif_util.py的_nova_to_osvif_vif_ovs()进行代码更新，当vif的 type为ovs时，将vifobj的bridge_name和devname更新，具体如下：

```1
def _nova_to_osvif_vif_ovs(vif):
    vnic_type = vif.get('vnic_type', model.VNIC_TYPE_NORMAL)
    profile = objects.vif.VIFPortProfileOpenVSwitch(
        interface_id=vif.get('ovs_interfaceid') or vif['id'],
        datapath_type=vif['details'].get(
            model.VIF_DETAILS_OVS_DATAPATH_TYPE))
    if vnic_type == model.VNIC_TYPE_DIRECT:
        profile = objects.vif.VIFPortProfileOVSRepresentor(
            interface_id=vif.get('ovs_interfaceid') or vif['id'],
            representor_name=_get_vif_name(vif),
            representor_address=vif["profile"]['pci_slot'])
        obj = _get_vif_instance(
            vif,
            objects.vif.VIFHostDevice,
            port_profile=profile,
            plugin="ovs",
            dev_address=vif["profile"]['pci_slot'],
            dev_type=os_vif_fields.VIFHostDeviceDevType.ETHERNET)
        if vif["network"]["bridge"] is not None:
            obj.network.bridge = vif["network"]["bridge"]
    elif _is_firewall_required(vif) or vif.is_hybrid_plug_enabled():
        obj = _get_vif_instance(
            vif,
            objects.vif.VIFBridge,
            port_profile=profile,
            plugin="ovs",
            vif_name=_get_vif_name(vif),
            bridge_name=_get_hybrid_bridge_name(vif))
    else:
        obj = _get_vif_instance(
            vif,
            objects.vif.VIFOpenVSwitch,
            port_profile=profile,
            plugin="ovs",
            vif_name=_get_vif_name(vif))
        if vif["type"] == 'ovs':
            if vif["details"]["bridge_name"] is not None:
                obj.bridge_name = vif["details"]["bridge_name"]
                obj.vif_name = obj.vif_name.replace('tap', 'qvo')
        elif vif["network"]["bridge"] is not None:
            obj.bridge_name = vif["network"]["bridge"]
    return obj
```



将：

```
if vif["network"]["bridge"] is not None:
    obj.bridge_name = vif["network"]["bridge"]
```



更改的内容：

```
 if vif["type"] == 'ovs':
     if vif["details"]["bridge_name"] is not None:
         obj.bridge_name = vif["details"]["bridge_name"]
         obj.vif_name = obj.vif_name.replace('tap', 'qvo')
 elif vif["network"]["bridge"] is not None:
       obj.bridge_name = vif["network"]["bridge"]
```



这里需要对source host和destination host节点代码都需要更新，然后迁移后，虚拟机正常，网卡正确挂在br-int上，网络正常。