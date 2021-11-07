---
layout: post
title: openstack虚拟机热迁移导致网络中断问题记录
category: 技术
date: 2020-12-02

---

更新历史：

- 2020.12 完成

------



### 1、环境说明

版本：openstack queens

虚拟机操作系统：centos74-mini-v1

租户网络：vlan网络



### 2、现象描述

在【生产】【测试】等环境进行vm热迁移时，会偶发正常和非正常两种问题：

2.1 迁移后网络正常，ping测试不中断。

2.2 迁移后网络异常， 异常现象又可以分为：

2.2.1 热迁移后vm网络中断时间太久，断网时间从10s+到几分钟，而后网络自动恢复正常联通。

2.2.2 热迁移vm网络中断，一直处于中断，不会自动恢复。



### 3、现象分析

我们先对2.2热迁移导致网络异常的问题进行分析，然后根据分析内容与2.1对比，更容易定位问题。

3.1 针对2.2.1vm热迁移后网络断网时间过久后自动恢复问题分析。

同环境下有些vm热迁移后网络中断后会自动恢复，而有些vm热迁移后网络中断不会自动恢复，怀疑这些虚拟机的差异性，是否由于虚拟机镜像差异导致不同现象。发现中断后自动恢复的vm使用的镜像都为centos76-mini-v0.1（后续简称centos76）。而热迁移后网络中断不会自动恢复的vm都是使用centos74-mini-v0.1（后续简称centos74）镜像。经过对比测试可以确定centos76镜像vm确实是会经过一段时间后网络恢复。而使用centos74镜像的vm在热迁移后网络中断并不会恢复。

针对不同镜像导致不同现象进一步分析，经过抓包后发现，使用centos76-mini-v0.1的vm，在热迁移后网络是中断状态，然后由于镜像中默认安装了chronyd NTP服务，该服务会定期和外部ntp服务器通信。

```
18:36:30.177264 IP 10.20.64.14.44751 > 111.230.189.174.ntp: NTPv4, Client, length 48
18:36:30.177944 IP 10.20.64.14.38620 > 10.20.128.16.domain: 58773+ PTR? 174.189.230.111.in-addr.arpa. (46)
18:36:30.203868 IP 10.20.128.16.domain > 10.20.64.14.38620: 58773 NXDomain 0/1/0 (103)
18:36:30.205666 IP 10.20.64.14.36888 > 10.20.128.16.domain: 23693+ PTR? 14.64.20.10.in-addr.arpa. (42)
18:36:30.205934 IP 10.20.128.16.domain > 10.20.64.14.36888: 23693 NXDomain* 0/1/0 (92)
18:36:30.211634 IP 10.20.64.14.45945 > 10.20.128.16.domain: 47244+ PTR? 16.128.20.10.in-addr.arpa. (43)
18:36:30.211809 IP 10.20.128.16.domain > 10.20.64.14.45945: 47244 NXDomain* 0/1/0 (93)
18:36:30.213620 IP 111.230.189.174.ntp > 10.20.64.14.44751: NTPv4, Server, length 48
18:36:33.083574 IP 10.20.64.14.43426 > sv1.ggsrv.de.ntp: NTPv4, Client, length 48
```

当网络中断后，由于vm内部向外部发包从而使得网络恢复通信。而centos74镜像的vm内部没有安装chronyd服务。因此该镜像启动的虚拟机在热迁移中断后无法自动恢复。那么为什么chronyd服务会导致网络恢复呢，这个问题我们放在后面再说。

致此，我们可以的到初步判断，是所有vm都会在热迁移后导致网络中断，只是由于centos76安装的chronyd服务，从而导致了触发了网络恢复。因此，我们可以得知网络中断和网络中断后恢复的问题是一个问题，就是热迁移后网络会中断。接下来我们可以聚焦排查热迁移后vm的网络为什么会中断的问题。



3.2 2.2.2热迁移后网络中断分析

针对热迁移后会导致外部ping中断的问题，通过在vm内部、迁移源宿主机、迁移目标主机进行抓包后发现，当热迁移后应该发送到目标节点的NIC的ping包没有发送到目标节点，而是还持续发送到源节点的NIC上。正常现象是迁移前ping包发送到源节点NIC卡上，迁移后应该发送到目标主机的网卡上。

为什么会出现这种现象呢？

我们先来分析下迁移后为什么外部ping包可以知道vm进行了迁移操作，由源节点迁移到了目标节点？经过查询资料和抓包后分析得知，并非外部交换设备会自动知道vm做了迁移操作，而是被告知，那是被谁告知呢？是由被迁移的vm告知，vm在迁移成功后，vm启动时，会发送5个arp广播，来通知外部设备，自己的最新位置。其他外部路由设备可以根据这个arp来更新路由表。vm内部发送arp如下：



```
20:06:07.498061 ARP, Request who-has bj-sjhl-yhc-live-migraaa-test.novalocal tell bj-sjhl-yhc-live-migraaa-test.novalocal, length 28
20:06:07.548276 ARP, Request who-has bj-sjhl-yhc-live-migraaa-test.novalocal tell bj-sjhl-yhc-live-migraaa-test.novalocal, length 28
20:06:07.698222 ARP, Request who-has bj-sjhl-yhc-live-migraaa-test.novalocal tell bj-sjhl-yhc-live-migraaa-test.novalocal, length 28
20:06:07.948369 ARP, Request who-has bj-sjhl-yhc-live-migraaa-test.novalocal tell bj-sjhl-yhc-live-migraaa-test.novalocal, length 28
20:06:08.298418 ARP, Request who-has bj-sjhl-yhc-live-migraaa-test.novalocal tell bj-sjhl-yhc-live-migraaa-test.novalocal, length 28
```



经过这个arp广播后，外部的ping包从交换机出来后就可以找到目标主机后，然后将ping包发送到目标主机的NIC上，由此NIC转发到vm中，完成ping联通。

那么，热迁移后为什么ping包还是会发送到源节点的NIC上呢？有一种原因可能是vm的arp广播包并没有到达交换设备，交换设备没有更新路由表，因此还会将vm的ping包发送到源节点。但是经过vm内部抓包发现vm启动时正常发送了arp广播，但是在目标节点的NIC上并没有抓到arp包，因此可以确定，问题出在了节点内部的虚拟网络上。

现在我们可以与2.1提到的热迁移后网络ping不断的情况。经过大量测试后，发现当vm迁移的目标主机上，如果该主机上存在vm所属网络对应的linuxbridge的网桥以及对应的vlan子接口，并且该子接口已经连接到该网桥上，这时，虚拟机在迁移前后ping测试时不中断的。而当目标主机不存在对应的linuxbidge网桥时，此时就会导致ping中断问题出现。

（我们来看下为什么有的主机会存在有对应的linuxbridge 网桥，有的主机不存在呢？这里是因为如果该节点曾经创建过属于该网络的虚拟机时，这个对应的linuxbridgee网桥会被创建出来。如果主机没有创建过属于该网络时的虚拟机时，主机是不会存在对应网络的网桥）

这里我们可以大致可以得知，外部交换设备无法得到vm启动时发送的arp广播，很可能是vm发送的5个arp广播，当由vm的VNIC发出后，由于外部的linuxbridge还没有准备好，子接口还没有挂在到该网桥，导致arp广播包无法正常到达目标主机的物理NIC，也就无法发送到外部网络上。当过了一段时间后，网桥准备完毕，并且vlan子接口已经挂在到该网桥上，这次在vm内部向外部发包时，就会让交换设备得知vm已经迁移到目标主机。这也就可以解释为什么chronyd可以触发网络恢复的现象了。

那么，为什么为出现这种情况外部网络没有准备好时，vm就启动了并且发送了arp？分析nova的live migration代码后，发现热迁移带代码里有机制控制vm启动的判断条件，但是这个判断条件应该是没有生效，导致外部网络没有准备好的情况下，启动了vm，从而提前发送了arp广播。此arp包无法到达交换机，被外部交换机感知到，因此，ping包还会被发送到源节点上。

问题根源锁定，就是在热迁移后，虚拟机启动后，由于所属于网桥的物理设备还未被挂载上，而此时源节点的vm被stoped，导致新起的vm无法被感知到。出现断ping问题。



下面梳理迁移流程中，发现在nova配置文件中有个参数

```
CONF.compute.live_migration_wait_for_vif_plug
```

找到这个参数如获至宝，以为这个参数就可以避免迁移过程中出现的竞争关系。

根据官方文档介绍，该参数时为了解决vm热迁移时可能会出现的竞争关系。具体如下：

```
Determine if the source compute host should wait for a ``network-vif-plugged``
event from the (neutron) networking service before starting the actual transfer
of the guest to the destination compute host.

Note that this option is read on the destination host of a live migration.
If you set this option the same on all of your compute hosts, which you should
do if you use the same networking backend universally, you do not have to
worry about this.

Before starting the transfer of the guest, some setup occurs on the destination
compute host, including plugging virtual interfaces. Depending on the
networking backend **on the destination host**, a ``network-vif-plugged``
event may be triggered and then received on the source compute host and the
source compute can wait for that event to ensure networking is set up on the
destination host before starting the guest transfer in the hypervisor.

```

然而。。。

经过测试发现，根本屁用不顶，而且发现这个参数在live-migrations还有个未被发现的bugs。具体见社区：https://bugs.launchpad.net/neutron/+bug/1880389。（7月份提的bugs，到现在还没有被处理，看来openstack社区也走下坡路了）。

由于影响使用了，所以还是需要处理，自力更生，继续排查。因为live_migration_wait_for_vif_plug这个参数本意是在热迁移过程中等到目标节点的vif被plug的信号后开始调用driver下的libvirt执行热迁移操作，但是根据测试和日志分析，发现在热迁移过程中，是可以收到network-vif-plugged的信号。但是这个信号根本就TM不是dest的发出来的。而且在目标节点执行pre_live_migraiton时，nova调用neutron的接口更新port的profile信息，具体更新信息为： {u'port': {u'binding:profile': {u'migrating_to': u'bj-sjhl-os-compute-test-21-55'}，在更新这个操作时，导致的触发了network-vif-plugged信号发出，然后被source host源节点的热迁移等待被激活，然后开始执行热迁移操作。你说坑die不？更坑还在后面，进一步梳理了port_update 流程后，思考是否可以在更新 这个port的profile 信息时，能否不触发这个network-vif-plugged？可以，我们修改了neutron代码后，只更新了port的profile信息，不触发network-vif-plugged信号。但是，这时，会发现，由于无法获取vif-plugged信号，然后热迁移动作就被ka在等待中，一直等到timeout...，由于不能执行热迁移操作，然后，虚拟机的tap设备无法在dest节点上被scan到，当然也无法触发network-vif-plugged信号。。。

好吧，此路不通，我们另寻他路。



既然是有热迁移和桥接的物理设备有竞争关系，那我们就消除这个竞赛关系。而且在梳理流程中，发现在pre_live_migraitons阶段，在dest node上有一步pre_live_migrations_plug_vifs动作

nova/virt/libvirt/driver.py

```
self._pre_live_migration_plug_vifs(
            instance, network_info, migrate_data)
```

根据方法的命名，以及日志打印输出，正常人都以为vif的tap设备以及挂载到bridge设备上，但是测试下来发现，然而并没有。而是在这个步骤中，只是在dest node上将bridge 桥设备准备好了。这里因该为什么不确定一下物理设备是否挂在到bridge上？合理吗？后来想了想，觉得也没毛病，kvm虚拟机在创建过程中，只要确定bridge设备正常，就可以正常启动vm，所以人家没必要再进一步确定是否桥设备的物理设备被挂载上。



不过，这个问题，对kvm没毛病，但是对咱们live-migration就会出现上面的问题。所以要需要对这个pre_live_migrations_plug_vifs里面的操作，增强一下。在ensure_bridge时，再ensure一下物理设备是否挂载上了。

经过view NOVA的代码，发现这个ensure_bridge动作使通过os_vif项目操作linuxbridge等设备，而且在os_vif中不仅有ensure_bridge，而且还有ensure_vlan_bridge.

os_vif/vif_plug_linux_bridge/linux_bridge.py

```
    def plug(self, vif, instance_info):
        """Ensure that the bridge exists, and add VIF to it."""
        network = vif.network
        bridge_name = vif.bridge_name
        if not network.multi_host and network.should_provide_bridge:
            mtu = network.mtu or self.config.network_device_mtu
            if network.should_provide_vlan:
                iface = self.config.vlan_interface or network.bridge_interface
                linux_net.ensure_vlan_bridge(network.vlan,
                                             bridge_name, iface, mtu=mtu)
            else:
                iface = self.config.flat_interface or network.bridge_interface
                # only put in iptables rules if Neutron not filtering
                install_filters = not vif.has_traffic_filtering
                linux_net.ensure_bridge(bridge_name, iface,
                                        filtering=install_filters, mtu=mtu)
```



哈哈，这个就是我们想要的。在确定桥设备时，顺便确认下vlan的子接口设备。但是想要用ensure_vlan_bridge，还需要更多的参数。经过分析需要增加如下三个参数

```
should_provide_vlan = True   // 是否创建vlan子接口
vlan = *    // vlan ID
bridge_interface = bond1    // 桥接的物理设备
```

参数确定了，那这些参数从哪里来？经过代码view,确定了时从nova instance caehe 的network_info.数据库中数据如下：

```
[{"profile": {}, "ovs_interfaceid": null, "preserve_on_delete": true, "network": {"bridge": "brq5ca4a059-d7", "subnets": [{"ips": [{"meta": {}, "version": 4, "type": "fixed", "floating_ips": [], "address": "10.20.64.30"}], "version": 4, "meta": {"dhcp_server": "10.20.64.11"}, "dns": [{"meta": {}, "version": 4, "type": "dns", "address": "10.20.128.16"}, {"meta": {}, "version": 4, "type": "dns", "address": "10.20.128.18"}], "routes": [], "cidr": "10.20.64.0/24", "gateway": {"meta": {}, "version": 4, "type": "gateway", "address": "10.20.64.1"}}], "meta": {"tunneled": false, "should_create_bridge": true, "tenant_id": "71cd3c900ad84fbbbee325d0ccba48f0", "mtu": 1500, "physical_network": "provider", "injected": false}, "id": "5ca4a059-d7cd-45e2-8787-8b1df5ca19af", "label": "xesops-64"}, "devname": "tap07f694d6-39", "vnic_type": "normal", "qbh_params": null, "meta": {}, "details": {"port_filter": true}, "address": "fa:16:3e:80:a7:2f", "active": true, "type": "bridge", "id": "07f694d6-39fc-4f74-952b-9250c0c5611d", "qbg_params": null}]
```

发现并没有这些参数，因此我们需要增加。那在哪里增加呢？需要找到更新instance cache，梳理后发现，在构建network_model时，将参数加进去

Nova/network/neutronv2/api.py



```
def _nw_info_build_network(self, context, port, networks, subnets):
        # TODO(stephenfin): Pass in an existing admin client if available.
        neutron = get_client(context, admin=True)
        network_name = None
        network_mtu = None
        for net in networks:
            if port['network_id'] == net['id']:
                network_name = net['name']
                tenant_id = net['tenant_id']
                network_mtu = net.get('mtu')
                break
        else:
            tenant_id = port['tenant_id']
            LOG.warning("Network %(id)s not matched with the tenants "
                        "network! The ports tenant %(tenant_id)s will be "
                        "used.",
                        {'id': port['network_id'], 'tenant_id': tenant_id})

        bridge = None
        ovs_interfaceid = None
        # Network model metadata
        should_create_bridge = None
        vif_type = port.get('binding:vif_type')
        port_details = port.get('binding:vif_details', {})
        if vif_type in [network_model.VIF_TYPE_OVS,
                        network_model.VIF_TYPE_AGILIO_OVS]:
            bridge = port_details.get(network_model.VIF_DETAILS_BRIDGE_NAME,
                                      CONF.neutron.ovs_bridge)
            ovs_interfaceid = port['id']
        elif vif_type == network_model.VIF_TYPE_BRIDGE:
            bridge = port_details.get(network_model.VIF_DETAILS_BRIDGE_NAME,
                                      "brq" + port['network_id'])
            should_create_bridge = True
        elif vif_type == network_model.VIF_TYPE_DVS:
            # The name of the DVS port group will contain the neutron
            # network id
            bridge = port['network_id']
        elif (vif_type == network_model.VIF_TYPE_VHOSTUSER and
         port_details.get(network_model.VIF_DETAILS_VHOSTUSER_OVS_PLUG,
                          False)):
            bridge = port_details.get(network_model.VIF_DETAILS_BRIDGE_NAME,
                                      CONF.neutron.ovs_bridge)
            ovs_interfaceid = port['id']
        elif (vif_type == network_model.VIF_TYPE_VHOSTUSER and
         port_details.get(network_model.VIF_DETAILS_VHOSTUSER_FP_PLUG,
                          False)):
            bridge = port_details.get(network_model.VIF_DETAILS_BRIDGE_NAME,
                                      "brq" + port['network_id'])

        # Prune the bridge name if necessary. For the DVS this is not done
        # as the bridge is a '<network-name>-<network-UUID>'.
        if bridge is not None and vif_type != network_model.VIF_TYPE_DVS:
            bridge = bridge[:network_model.NIC_NAME_LEN]

        physnet, tunneled = self._get_physnet_tunneled_info(
            context, neutron, port['network_id'])
        network = network_model.Network(
            id=port['network_id'],
            bridge=bridge,
            injected=CONF.flat_injected,
            label=network_name,
            tenant_id=tenant_id,
            mtu=network_mtu,
            physical_network=physnet,
            tunneled=tunneled
            )
        network['subnets'] = subnets

        if should_create_bridge is not None:
            network['should_create_bridge'] = should_create_bridge
        return network, ovs_interfaceid
```



修改后：

```
    def _nw_info_build_network(self, context, port, networks, subnets):
        # TODO(stephenfin): Pass in an existing admin client if available.
        neutron = get_client(context, admin=True)
        network_name = None
        network_mtu = None
        network_type = None
        segmentation_id = None
        for net in networks:
            if port['network_id'] == net['id']:
                network_name = net['name']
                tenant_id = net['tenant_id']
                network_mtu = net.get('mtu')
                segmentation_id = net.get('provider:segmentation_id')
                network_type = net.get('provider:network_type')
                break
        else:
            tenant_id = port['tenant_id']
            LOG.warning("Network %(id)s not matched with the tenants "
                        "network! The ports tenant %(tenant_id)s will be "
                        "used.",
                        {'id': port['network_id'], 'tenant_id': tenant_id})

        bridge = None
        ovs_interfaceid = None
        # Network model metadata
        should_create_bridge = None
        should_create_vlan = None
        vif_type = port.get('binding:vif_type')
        port_details = port.get('binding:vif_details', {})
        if vif_type in [network_model.VIF_TYPE_OVS,
                        network_model.VIF_TYPE_AGILIO_OVS]:
            bridge = port_details.get(network_model.VIF_DETAILS_BRIDGE_NAME,
                                      CONF.neutron.ovs_bridge)
            ovs_interfaceid = port['id']
        elif vif_type == network_model.VIF_TYPE_BRIDGE:
            bridge = port_details.get(network_model.VIF_DETAILS_BRIDGE_NAME,
                                      "brq" + port['network_id'])
            should_create_bridge = True
            if network_type == 'vlan':
                should_create_vlan = True
        elif vif_type == network_model.VIF_TYPE_DVS:
            # The name of the DVS port group will contain the neutron
            # network id
            bridge = port['network_id']
        elif (vif_type == network_model.VIF_TYPE_VHOSTUSER and
         port_details.get(network_model.VIF_DETAILS_VHOSTUSER_OVS_PLUG,
                          False)):
            bridge = port_details.get(network_model.VIF_DETAILS_BRIDGE_NAME,
                                      CONF.neutron.ovs_bridge)
            ovs_interfaceid = port['id']
        elif (vif_type == network_model.VIF_TYPE_VHOSTUSER and
         port_details.get(network_model.VIF_DETAILS_VHOSTUSER_FP_PLUG,
                          False)):
            bridge = port_details.get(network_model.VIF_DETAILS_BRIDGE_NAME,
                                      "brq" + port['network_id'])

        # Prune the bridge name if necessary. For the DVS this is not done
        # as the bridge is a '<network-name>-<network-UUID>'.
        if bridge is not None and vif_type != network_model.VIF_TYPE_DVS:
            bridge = bridge[:network_model.NIC_NAME_LEN]

        physnet, tunneled = self._get_physnet_tunneled_info(
            context, neutron, port['network_id'])
        network = network_model.Network(
            id=port['network_id'],
            bridge=bridge,
            injected=CONF.flat_injected,
            label=network_name,
            tenant_id=tenant_id,
            mtu=network_mtu,
            bridge_interface=CONF.neutron.bridge_interface,
            vlan=segmentation_id,
            physical_network=physnet,
            tunneled=tunneled
            )
        network['subnets'] = subnets

        if should_create_bridge is not None:
            network['should_create_bridge'] = should_create_bridge
        if should_create_vlan is not None:
            network['should_create_vlan'] = should_create_vlan
        return network, ovs_interfaceid
```



并且在需要在nova.conf添加bridge_interface 参数项：

nova/conf/neutron.py

```
    cfg.IntOpt('http_retries',
               default=3,
               min=0,
               help="""
Number of times neutronclient should retry on any failed http call.

0 means connection is attempted only once. Setting it to any positive integer
means that on failure connection is retried that many times e.g. setting it
to 3 means total attempts to connect will be 4.

Possible values:

* Any integer value. 0 means connection is attempted only once
"""),
]
```



修改后：

```
    cfg.IntOpt('http_retries',
               default=3,
               min=0,
               help="""
Number of times neutronclient should retry on any failed http call.

0 means connection is attempted only once. Setting it to any positive integer
means that on failure connection is retried that many times e.g. setting it
to 3 means total attempts to connect will be 4.

Possible values:

* Any integer value. 0 means connection is attempted only once
"""),
    cfg.StrOpt('bridge_interface',
               default='bond1',
               help="""
the default physical interface for vlan bridge.

Possible values:

* 
"""),
]
```



nova.conf

添加参数：

```
[neutron]
bridge_interface = bond1
```



到这里我们就把所有的参数准备好了，然后以为可以愉快的跑起来了，发现还是被卡住了。然后一趟就是4个坑。

1） 权限问题。

发现通过os_vif/vif_plug_linux_bridge/linux_net.py的ensure_vlan_bridge.py时，遇到了权限问题，提示没有权限操作。然后就日了狗，刚开始还以为是rootwrap方式设置权限，然后修改了/etc/rootwrap.d/network.filter，发现不行，然后一顿尝试，没成功。最后还是研究了下目前的权限方式，才发现目前常用组件的权限都从rootwrap更新为oslo-privsep,通过这种方式，可以更加高效、精准的控制权限。（所以说，还是要熟悉版本的release note，要不就容易走冤枉路）。

然后发现这里可能还有一个bug，由于ensure_vlan_bridge估计没有在用，连ensure_vlan_bridge的权限装饰器位置都放错了地方。



```
@privsep.vif_plug.entrypoint
def ensure_vlan_bridge(vlan_num, bridge, bridge_interface,
                       net_attrs=None, mac_address=None,
                       mtu=None):
                      

@lockutils.synchronized('nova-lock_vlan', external=True)
def _ensure_vlan_privileged(vlan_num, bridge_interface, mac_address, mtu):
```



修改为：

```
@lockutils.synchronized('nova-lock_vlan', external=True)
def ensure_vlan_bridge(vlan_num, bridge, bridge_interface,
                       net_attrs=None, mac_address=None,
                       mtu=None):
                      

@privsep.vif_plug.entrypoint
def _ensure_vlan_privileged(vlan_num, bridge_interface, mac_address, mtu):
```





2）vlan子接口名称规则不一致。

默认vlan子接口这里会创建为名称vlan+VLAN_ID的方式，但是，linuxbridge-agent 在scan到tap设备后，也会检查桥上物理接口，但是它检查的名称是bridge_interface+.+VLAN_ID，这样就会重复创建子接口。所以这里需要修改命名规则，和neutron-linuxbridge-agent保持一致。



```
interface = 'vlan%s' % vlan_num
```

修改为：

```
interface = '%s.%s' % (bridge_interface, vlan_num)
```



3）参数问题

```
@privsep.vif_plug.entrypoint
def ensure_vlan_bridge(vlan_num, bridge, bridge_interface,
                       net_attrs=None, mac_address=None,
                       mtu=None):
    """Create a vlan and bridge unless they already exist."""
    interface = _ensure_vlan_privileged(vlan_num, bridge_interface,
                                        mac_address, mtu=mtu)
    _ensure_bridge_privileged(bridge, interface, net_attrs)

    _ensure_bridge_filtering(bridge, None)
    return interface

```

修改为：

```
@privsep.vif_plug.entrypoint
def ensure_vlan_bridge(vlan_num, bridge, bridge_interface,
                       net_attrs=None, mac_address=None,
                       mtu=None):
    """Create a vlan and bridge unless they already exist."""
    interface = _ensure_vlan_privileged(vlan_num, bridge_interface,
                                        mac_address, mtu=mtu)
    _ensure_bridge_privileged(bridge, interface, net_attrs, True,
                              mtu=mtu)
    _ensure_bridge_filtering(bridge, None)
    return interface


```



4) 设置set_mtu 异常。

在创建完vlan子接口后，发现在设置设备mtu时，报：

```

```

百思不得其解， 1500正常值，怎么会出现这个问题。然后看子接口设备，mtu也正常。所以，这里直接跳过设置mtu的步骤。

```
def _ensure_vlan_privileged(vlan_num, bridge_interface, mac_address, mtu):
    """Create a vlan unless it already exists.

    This assumes the caller is already annotated to run
    with elevated privileges.
    """
    interface = '%s.%s' % (bridge_interface, vlan_num)
    if not device_exists(interface):
        LOG.debug('Starting VLAN interface %s', interface)
        ip_lib.add(interface, 'vlan', link=bridge_interface,
                   vlan_id=vlan_num, check_exit_code=[0, 2, 254])
        # (danwent) the bridge will inherit this address, so we want to
        # make sure it is the value set from the NetworkManager
        if mac_address:
            ip_lib.set(interface, address=mac_address,
                       check_exit_code=[0, 2, 254])
        ip_lib.set(interface, state='up', check_exit_code=[0, 2, 254])
        # NOTE(vish): set mtu every time to ensure that changes to mtu get
        #             propogated
        _set_device_mtu(interface, mtu)

    return interface
```

去掉 _set_device_mtu(interface, mtu)

```
#_set_device_mtu(interface, mtu)
```



综上，我们修改如下：

/os_vif/vif_plug_linux_bridge/linux_net.py

修改前：

```
@privsep.vif_plug.entrypoint
def ensure_vlan_bridge(vlan_num, bridge, bridge_interface,
                       net_attrs=None, mac_address=None,
                       mtu=None):
    """Create a vlan and bridge unless they already exist."""
    interface = _ensure_vlan_privileged(vlan_num, bridge_interface,
                                        mac_address, mtu=mtu)
    # _ensure_bridge_privileged(bridge, interface, net_attrs)
    _ensure_bridge_privileged(bridge, interface, net_attrs, True,
                              mtu=mtu)
    _ensure_bridge_filtering(bridge, None)
    return interface


@lockutils.synchronized('nova-lock_vlan', external=True)
def _ensure_vlan_privileged(vlan_num, bridge_interface, mac_address, mtu):
    """Create a vlan unless it already exists.

    This assumes the caller is already annotated to run
    with elevated privileges.
    """
    interface = 'vlan%s' % vlan_num
    if not device_exists(interface):
        LOG.debug('Starting VLAN interface %s', interface)
        ip_lib.add(interface, 'vlan', link=bridge_interface,
                   vlan_id=vlan_num, check_exit_code=[0, 2, 254])
        # (danwent) the bridge will inherit this address, so we want to
        # make sure it is the value set from the NetworkManager
        if mac_address:
            ip_lib.set(interface, address=mac_address,
                       check_exit_code=[0, 2, 254])
        ip_lib.set(interface, state='up', check_exit_code=[0, 2, 254])
        # NOTE(vish): set mtu every time to ensure that changes to mtu get
        #             propogated
        _set_device_mtu(interface, mtu)

    return interface


@lockutils.synchronized('nova-lock_bridge', external=True)
def ensure_bridge(bridge, interface, net_attrs=None, gateway=True,
                  filtering=True, mtu=None):
    _ensure_bridge_privileged(bridge, interface, net_attrs, gateway,
                              filtering=filtering, mtu=mtu)
    if filtering:
        _ensure_bridge_filtering(bridge, gateway)
```





修改后：

```
@lockutils.synchronized('nova-lock_vlan', external=True)
def ensure_vlan_bridge(vlan_num, bridge, bridge_interface,
                       net_attrs=None, mac_address=None,
                       mtu=None):
    """Create a vlan and bridge unless they already exist."""
    interface = _ensure_vlan_privileged(vlan_num, bridge_interface,
                                        mac_address, mtu=mtu)
    # _ensure_bridge_privileged(bridge, interface, net_attrs)
    _ensure_bridge_privileged(bridge, interface, net_attrs, True,
                              mtu=mtu)
    _ensure_bridge_filtering(bridge, None)
    return interface


@privsep.vif_plug.entrypoint
def _ensure_vlan_privileged(vlan_num, bridge_interface, mac_address, mtu):
    """Create a vlan unless it already exists.

    This assumes the caller is already annotated to run
    with elevated privileges.
    """
    interface = '%s.%s' % (bridge_interface, vlan_num)
    if not device_exists(interface):
        LOG.debug('Starting VLAN interface %s', interface)
        ip_lib.add(interface, 'vlan', link=bridge_interface,
                   vlan_id=vlan_num, check_exit_code=[0, 2, 254])
        # (danwent) the bridge will inherit this address, so we want to
        # make sure it is the value set from the NetworkManager
        if mac_address:
            ip_lib.set(interface, address=mac_address,
                       check_exit_code=[0, 2, 254])
        ip_lib.set(interface, state='up', check_exit_code=[0, 2, 254])
        # NOTE(vish): set mtu every time to ensure that changes to mtu get
        #             propogated
        #_set_device_mtu(interface, mtu)

    return interface


@lockutils.synchronized('nova-lock_bridge', external=True)
def ensure_bridge(bridge, interface, net_attrs=None, gateway=True,
                  filtering=True, mtu=None):
    _ensure_bridge_privileged(bridge, interface, net_attrs, gateway,
                              filtering=filtering, mtu=mtu)
    if filtering:
        _ensure_bridge_filtering(bridge, gateway)
```



到这里我们就可以正常的在pre_live_migrarions时，把bridge的物理接口挂载到桥上。



经过测试，live_migration 在目标节点没有网校和vlan子接口时，热迁移不再出现丢包断网问题。

但是由于修改了很对公共代码，不知这些修改对其他操作是否有影响，还需全量覆盖测试，或者提交社区，通过社区力量review修改的代码，确定不会引起其他问题，才能更新生产环境。





