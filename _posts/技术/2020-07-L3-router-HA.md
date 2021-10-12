---
layout: post
title: L3 router HA 多活问题
category: 技术
date: 2020-07-01

---

更新历史：

- 2020.07 

------



版本：rocky

开启HA router：l3_ha = true



问题说明：

​    创建ha router后，查看 neutron l3-agent-list-hosting-router d952bcdf-7aa4-4d48-a3d1-b09f305e043e

```
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+-----------------------------------+----------------+-------+----------+
| id | host | admin_state_up | alive | ha_state |
+--------------------------------------+-----------------------------------+----------------+-------+----------+
| ebca7a4b-9dcf-42d4-aa27-fff9514a24bb | bj-sjhl-os-controller-gray-161-13 | True | :-) | standby |
| 0b719192-2877-47d3-8578-e81d7c6a59c3 | bj-sjhl-os-controller-gray-161-11 | True | :-) | standby |
| 81594cd5-96a4-45d4-a551-631c4091859a | bj-sjhl-os-controller-gray-161-12 | True | :-) | active |
+--------------------------------------+-----------------------------------+----------------+-------+----------+
```





可以看到只有一个状态是active，查看active借点的router namespace，可以看到keepalived的心跳vip

```root@bj-sjhl-os-controller-gray-161-12 jumpadmin]# ip netns exec qrouter-d952bcdf-7aa4-4d48-a3d1-b09f305e043e ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: ha-f2e02f37-7c@if59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:67:8f:9d brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 169.254.192.6/18 brd 169.254.255.255 scope global ha-f2e02f37-7c
valid_lft forever preferred_lft forever
inet 169.254.0.175/24 scope global ha-f2e02f37-7c
valid_lft forever preferred_lft forever
inet6 fe80::f816:3eff:fe67:8f9d/64 scope link
valid_lft forever preferred_lft forever
3: qg-9d745ddf-30@if61: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:1f:39:ab brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.20.67.15/24 scope global qg-9d745ddf-30
valid_lft forever preferred_lft forever
inet 10.20.67.17/32 scope global qg-9d745ddf-30
valid_lft forever preferred_lft forever
inet6 fe80::f816:3eff:fe1f:39ab/64 scope link nodad
valid_lft forever preferred_lft forever
4: qr-22b5bb14-06@if63: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
link/ether fa:16:3e:35:93:f4 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.20.64.1/24 scope global qr-22b5bb14-06
valid_lft forever preferred_lft forever
```



查看其他两个节点的router namespace时，可以看到都属于standby状态：



```[root@bj-sjhl-os-controller-gray-161-11 jumpadmin]# ip netns exec qrouter-d952bcdf-7aa4-4d48-a3d1-b09f305e043e ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: ha-bd86b8ad-b1@if58: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:82:61:96 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 169.254.192.4/18 brd 169.254.255.255 scope global ha-bd86b8ad-b1
valid_lft forever preferred_lft forever
inet6 fe80::f816:3eff:fe82:6196/64 scope link
valid_lft forever preferred_lft forever
3: qg-9d745ddf-30@if60: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:1f:39:ab brd ff:ff:ff:ff:ff:ff link-netnsid 0
4: qr-22b5bb14-06@if62: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:35:93:f4 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```



但是当手动stop 掉 active节点的neutron_l3_agent后，发现会有双活情况出现，

```[root@bj-sjhl-os-controller-gray-161-11 ~]# neutron l3-agent-list-hosting-router d952bcdf-7aa4-4d48-a3d1-b09f305e043e
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+-----------------------------------+----------------+-------+----------+
| id | host | admin_state_up | alive | ha_state |
+--------------------------------------+-----------------------------------+----------------+-------+----------+
| ebca7a4b-9dcf-42d4-aa27-fff9514a24bb | bj-sjhl-os-controller-gray-161-13 | True | :-) | standby |
| 0b719192-2877-47d3-8578-e81d7c6a59c3 | bj-sjhl-os-controller-gray-161-11 | True | :-) | active |
| 81594cd5-96a4-45d4-a551-631c4091859a | bj-sjhl-os-controller-gray-161-12 | True | :-) | active |
+--------------------------------------+-----------------------------------+----------------+-------+----------+=
```



keepalived的vip出现在两个节点的namespace中:

```root@bj-sjhl-os-controller-gray-161-11 ~]# ip netns exec qrouter-d952bcdf-7aa4-4d48-a3d1-b09f305e043e ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: ha-bd86b8ad-b1@if58: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:82:61:96 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 169.254.192.4/18 brd 169.254.255.255 scope global ha-bd86b8ad-b1
valid_lft forever preferred_lft forever
inet 169.254.0.175/24 scope global ha-bd86b8ad-b1
valid_lft forever preferred_lft forever
inet6 fe80::f816:3eff:fe82:6196/64 scope link
valid_lft forever preferred_lft forever
3: qg-9d745ddf-30@if60: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:1f:39:ab brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.20.67.15/24 scope global qg-9d745ddf-30
valid_lft forever preferred_lft forever
inet6 fe80::f816:3eff:fe1f:39ab/64 scope link nodad
valid_lft forever preferred_lft forever
4: qr-22b5bb14-06@if62: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:35:93:f4 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.20.64.1/24 scope global qr-22b5bb14-06
valid_lft forever preferred_lft forever
inet6 fe80::f816:3eff:fe35:93f4/64 scope link nodad
```



\---------



```[root@bj-sjhl-os-controller-gray-161-12 jumpadmin]# ip netns exec qrouter-d952bcdf-7aa4-4d48-a3d1-b09f305e043e ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: ha-f2e02f37-7c@if59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:67:8f:9d brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 169.254.192.6/18 brd 169.254.255.255 scope global ha-f2e02f37-7c
valid_lft forever preferred_lft forever
inet 169.254.0.175/24 scope global ha-f2e02f37-7c
valid_lft forever preferred_lft forever
inet6 fe80::f816:3eff:fe67:8f9d/64 scope link
valid_lft forever preferred_lft forever
3: qg-9d745ddf-30@if61: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether fa:16:3e:1f:39:ab brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.20.67.15/24 scope global qg-9d745ddf-30
valid_lft forever preferred_lft forever
inet6 fe80::f816:3eff:fe1f:39ab/64 scope link nodad
valid_lft forever preferred_lft forever
4: qr-22b5bb14-06@if63: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
link/ether fa:16:3e:35:93:f4 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.20.64.1/24 scope global qr-22b5bb14-06
valid_lft forever preferred_lft forever
```



经过搜索社区关于vrouter的bug列表，发现当前版本存在如下bug：

https://review.opendev.org/#/c/643460/

https://review.opendev.org/#/c/719978/

https://review.opendev.org/#/c/717741/



由于目前环境中的网络架构还不适合通过l3方式挂载浮动ip的问题，暂不合入，后期使用l3时再参考合入。