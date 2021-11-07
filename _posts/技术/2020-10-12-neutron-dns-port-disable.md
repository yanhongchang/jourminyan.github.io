---
layout: post
Title: neutron的dns服务53端口屏蔽
category: 技术
date: 2020-10-12



---

更新历史：

- 2020.10 完成

------

###

#### 背景：

​		目前在用的几套openstack集群中，每个集群都存在一个真正公网意义的公网网络，该网络方式不是通过public类型网络方式，通过浮动ip的方式提供给虚拟机，而是直接通过虚拟机绑定该网络下的port来实现虚拟机配置外部公网需求。这样的好处就是省略的私网ip到工网IP的nat转换，网络流量更加简单清楚。



#### 问题：

​		由于将一段公网ip地址放在private 网络中，并且enable该网络的dhcp服务，那么，根据neutron.conf的dhcp_agents_per_network参数设置，在控制节点的通过dnsmasq启动三个dhcp服务和dns服务。这样就会将dns服务所需要的53端口暴漏在外网中，这样在安全性上就带来的风险。安全部门对该端口的暴漏比较在意。因此，需要进行关闭或者限制。



#### 解决思路：

##### 1、安全组限制方向

​		由于涉及到端口的限制，首先想到的就是通过对该提供53端口服务的port进行安全组限制，对端口添加ingress 方向的限制，限制source的cidr。经过测试和验证，发现不生效，经过neutron代码查看，发现安全组对owner_prefix=network的网络端口进行跳过。



```
def setup_arp_spoofing_protection(vif, port_details):
    ...
    if net.is_port_trusted(port_details):
        # clear any previous entries related to this port
        delete_arp_spoofing_protection([vif])
        LOG.debug("Skipping ARP spoofing rules for network owned port "
                  "'%s'.", vif)
        return
    _setup_arp_spoofing_protection(vif, port_details)
```



```
def is_port_trusted(port):
    """Used to determine if port can be trusted not to attack network.

    Trust is currently based on the device_owner field starting with 'network:'
    since we restrict who can use that in the default policy.json file.

    :param port: The port dict to inspect the 'device_owner' for.
    :returns: True if the port dict's 'device_owner' value starts with the
        networking prefix. False otherwise.
    """
    return port['device_owner'].startswith(
        constants.DEVICE_OWNER_NETWORK_PREFIX)
```

这样就无法对网络设备进行安全组设备。该思路pass。



##### 2、禁用dns服务

​	既然无法通过安全组对port进行端口限制，那么我们看这个dnsmasq提供的dns是否禁用，来彻底关闭53端口的暴漏？经过查阅资料和官方文档，发信可以通过设置dnsmasq的参数来实现只提供dhcp服务，而关闭dns服务。在neutron-dhcp-agent/dnsmasq.conf中通过port=0来彻底关闭dns服务，经过验证确实可以做到彻底关闭53端口。但是随着近一步分析，我们目前的dns服务使用方式是通过子网设置dns来使用，如果关闭dns服务，那么子网就无法提供dns服务，子网中配置的dns无法注入到该网络的虚拟机中，这样就影响了所有的subnet的dns服务。因此，这个思路也pass掉。



 ##### 3、关闭dhcp

​		既然上面两个思路都无法成功，因此，还有一个最后一个终极解决办法，那就是将该公网子网中dhcp 彻底disable掉。不使用dhcp进行ip的自动获取。而是人工对分配的port的虚拟机手工设置静态ip。由于目前的公网ip绑定流程确实是需要手工设置静态ip，因此，disable掉都dhcp服务不会给使用带来麻烦。



ps:

​		还有一种方式就是通过手工方式来对控制节点的3个暴露53端口的port进行iptables进行限制，理论上是可行的，不过需要手动添加很多的安全组chain和规则，比较复杂。而且在后期维护起来也比较困难。



#####参考：

https://blog.csdn.net/wuliangtianzu/article/details/81867540	

https://www.runscripts.com/support/guides/tools/openstack/neutron-and-dns

 https://docs.openstack.org/neutron/train/admin/config-dns-res.html

https://blog.csdn.net/weixin_37583747/article/details/80939142
