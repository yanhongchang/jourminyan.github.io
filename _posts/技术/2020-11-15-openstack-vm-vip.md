---
layout: post
title: openstack虚拟机使用vip
category: 技术
date: 2020-11-15




---

更新历史：

- 2020.11 完成

------

####

1、openstack 正常方式创建两台虚拟机，镜像指定centos7.6

2、同网段创建一个port，占用一个ip，用这个ip作为后续的vip。

3、对两台虚拟机的port 进行处理

​	1） 清除port的安全组规则。

```
openstack port set --no-security-group ef404269-a5ff-4000-98b5-8185b2946619
openstack port set --no-security-group 73f83bcb-c654-498a-8177-1e534cfd7c75
```

​	2)	禁用port的安全组规则

```
 openstack port set --disable-port-security ef404269-a5ff-4000-98b5-8185b2946619
 openstack port set --disable-port-security 73f83bcb-c654-498a-8177-1e534cfd7c75
```

​	（或者放行vrrp协议的安全组也可实现）



4、然后在虚拟机配置haproxy和keepalived就可以正常使用。

