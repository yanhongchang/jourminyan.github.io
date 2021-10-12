---
layout: post
title: rabbitmq 脑裂后服务异常分析
category: 技术
date: 2020-08-01



---

更新历史：

- 2020.08 完成

------



在网络演练中，由于网络异常导致openstack环境中消息通信的rabbitmq集群发生了脑裂现象，但是随着网络恢复，rabbitmq集群也恢复，但是出现了部分服务异常，日志报消息waiting timeout，例如neutron-dhcp-agent日志如下：

```
2021-01-21 20:52:56.310 6 WARNING neutron.common.rpc [req-26043aa8-976d-40f0-a913-ca8d9bb73619 - - - - -] Increasing timeout for get_active_networks_info calls to 120 seconds. Restart the agent to restore it to the default value.: MessagingTimeout: Timed out waiting for a reply to message ID 6e3133071dba42a3a2e9bb79278c90d8
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent [req-26043aa8-976d-40f0-a913-ca8d9bb73619 - - - - -] Unable to sync network state.: MessagingTimeout: Timed out waiting for a reply to message ID 6e3133071dba42a3a2e9bb79278c90d8
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent Traceback (most recent call last):
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/neutron/agent/dhcp/agent.py", line 191, in sync_state
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     enable_dhcp_filter=False)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/neutron/agent/dhcp/agent.py", line 698, in get_active_networks_info
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     host=self.host, **kwargs)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/neutron/common/rpc.py", line 185, in call
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     time.sleep(wait)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/oslo_utils/excutils.py", line 220, in __exit__
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     self.force_reraise()
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/oslo_utils/excutils.py", line 196, in force_reraise
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     six.reraise(self.type_, self.value, self.tb)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/neutron/common/rpc.py", line 162, in call
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     return self._original_context.call(ctxt, method, **kwargs)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/oslo_messaging/rpc/client.py", line 174, in call
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     retry=self.retry)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/oslo_messaging/transport.py", line 131, in _send
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     timeout=timeout, retry=retry)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 559, in send
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     retry=retry)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 548, in _send
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     result = self._waiter.wait(msg_id, timeout)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 440, in wait
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     message = self.waiters.get(msg_id, timeout=timeout)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent   File "/var/lib/kolla/venv/lib/python2.7/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 328, in get
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent     'to message ID %s' % msg_id)
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent MessagingTimeout: Timed out waiting for a reply to message ID 6e3133071dba42a3a2e9bb79278c90d8
2021-01-21 20:53:21.303 6 ERROR neutron.agent.dhcp.agent 
```



​		经过分析三个节点日志，发现在一段时间内由于网络反复抖动，导致出现不只一次的网络分区，然后在这段时间内三个节点根据配置的分区处理策略：cluster_partition_handling=autoheal出现了交替反复重启服务，具体见一下日志：



21:

```
=INFO REPORT==== 15-Jan-2021::01:57:12 ===
Autoheal request received from 'rabbit@work-160-35' when healing; ignoring

=WARNING REPORT==== 15-Jan-2021::01:57:12 ===
Autoheal: we were selected to restart; winner is 'rabbit@work-160-23'

=INFO REPORT==== 15-Jan-2021::01:57:12 ===
RabbitMQ is asked to stop...

-----

=WARNING REPORT==== 15-Jan-2021::02:08:58 ===
Autoheal: we were selected to restart; winner is 'rabbit@work-160-35'

=INFO REPORT==== 15-Jan-2021::02:08:58 ===
RabbitMQ is asked to stop...

```



23:

```

=WARNING REPORT==== 15-Jan-2021::01:53:09 ===
Autoheal: we were selected to restart; winner is 'rabbit@work-160-35'

=ERROR REPORT==== 15-Jan-2021::01:53:09 ===
Channel error on connection <0.4656.5484> (x.x.x.23:48084 -> x.x.x.23:5672, vhost: '/', user: 'openstack'), channel 1:
operation basic.publish caused a channel exception not_found: no exchange 'reply_a86d30f61fcb4552a2860736133a8711' in vhost '/'

=INFO REPORT==== 15-Jan-2021::01:53:09 ===
RabbitMQ is asked to stop...

-----


=WARNING REPORT==== 15-Jan-2021::01:55:51 ===
Autoheal: we were selected to restart; winner is 'rabbit@work-160-35'

=INFO REPORT==== 15-Jan-2021::01:55:51 ===
RabbitMQ is asked to stop...


------

=WARNING REPORT==== 15-Jan-2021::01:57:11 ===
Autoheal: we were selected to restart; winner is 'rabbit@work-160-11'

=INFO REPORT==== 15-Jan-2021::01:57:11 ===
RabbitMQ is asked to stop...

--------


=INFO REPORT==== 15-Jan-2021::01:57:12 ===
Autoheal: I am the winner, waiting for ['rabbit@work-160-11'] to stop


----


=WARNING REPORT==== 15-Jan-2021::02:08:58 ===
Autoheal: we were selected to restart; winner is 'rabbit@work-160-35'


```



35:

```
 =INFO REPORT==== 15-Jan-2021::01:53:09 ===
*Autoheal: I am the winner, waiting for ['rabbit@work-160-23'] to stop
 
 
 
 =INFO REPORT==== 15-Jan-2021::01:55:51 ===
*Autoheal: I am the winner, waiting for ['rabbit@work-160-23'] to stop
 
 
 
 =INFO REPORT==== 15-Jan-2021::02:08:58 ===
*Autoheal: I am the winner, waiting for ['rabbit@work-160-23',
                                         'rabbit@work-160-11'] to stop
                                         
                                         
                                         
=INFO REPORT==== 15-Jan-2021::03:17:08 ===
RabbitMQ is asked to stop...


```

​		但是，按照做了队列的镜像配置后，消息会在不同node间做同步备份，但是这里为什么还会出现消息的丢失？随后又查看了队列的镜像配置，发现队列的镜像策略被下面列出的设置的其他policy覆盖掉了,具体复现见另一篇《rabbitmq队列镜像失效问题复现》

```
rabbitmqctl set_policy expiry ".*" '{"expires":1800000}' --apply-to queues
```

​	 	没有队列的镜像同步，又加上反复的重启，导致消息的丢失就可以理解了。



这时，可以通过两种方式恢复。

1）通过重新安装消息队列的集群来实现消息的恢复。（不推荐）

```
kolla-ansible deploy -i multinode --limit control -t rabbitmq --configdir ./ --passwords ./passwords.yml | tee rabbitmq.log
```

​	

2）	可以通过顺序关闭消息队列然后再逆序启动方式恢复。（推荐）

```
node-35:docker stop rabbitmq
node-23:docker stop rabbitmq
node-11:docker stop rabbitmq

node-11:docker start rabbitmq
node-23:docker start rabbitmq
node-35:docker start rabbitmq
```



另外，经过分析发现这里的网络分区处理策略不是很好，我们看下官方https://www.rabbitmq.com/partitions.html支持的分区策略：

```
RabbitMQ also offers three ways to deal with network partitions automatically: pause-minority mode, pause-if-all-down mode and autoheal mode. The default behaviour is referred to as ignore mode.

In pause-minority mode RabbitMQ will automatically pause cluster nodes which determine themselves to be in a minority (i.e. fewer or equal than half the total number of nodes) after seeing other nodes go down. It therefore chooses partition tolerance over availability from the CAP theorem. This ensures that in the event of a network partition, at most the nodes in a single partition will continue to run. The minority nodes will pause as soon as a partition starts, and will start again when the partition ends. This configuration prevents split-brain and is therefore able to automatically recover from network partitions without inconsistencies.

In pause-if-all-down mode, RabbitMQ will automatically pause cluster nodes which cannot reach any of the listed nodes. In other words, all the listed nodes must be down for RabbitMQ to pause a cluster node. This is close to the pause-minority mode, however, it allows an administrator to decide which nodes to prefer, instead of relying on the context. For instance, if the cluster is made of two nodes in rack A and two nodes in rack B, and the link between racks is lost, pause-minority mode will pause all nodes. In pause-if-all-down mode, if the administrator listed the two nodes in rack A, only nodes in rack B will pause. Note that it is possible the listed nodes get split across both sides of a partition: in this situation, no node will pause. That is why there is an additional ignore/autoheal argument to indicate how to recover from the partition.

In autoheal mode RabbitMQ will automatically decide on a winning partition if a partition is deemed to have occurred, and will restart all nodes that are not in the winning partition. Unlike pause_minority mode it therefore takes effect when a partition ends, rather than when one starts.
```



​	这里[https://blog.csdn.net/u013256816/article/details/73757884]对这几种策略做了很好的说明：

```
ignore的配置是当网络分区的时候，RabbitMQ不会自动做任何处理，即需要手动处理。

pause_minority 当发生网络分区时，集群中的节点在观察到某些节点down掉时，会自动检测其自身是否处于少数派（小于或者等于集群中一般的节点数）。少数派中的节点在分区发生时会自动关闭，当分区结束时又会启动。这里的关闭是指RabbitMQ application关闭，而Erlang VM并不关闭，这个类似于执行了rabbitmqctl stop_app命令。处于关闭的节点会每秒检测一次是否可连通到剩余集群中，如果可以则启动自身的应用，相当于执行rabbitmqctl start_app命令

在pause_if_all_down模式下，RabbitMQ会自动关闭不能和list中节点通信的节点。语法为{pause_if_all_down, [nodes], ignore|autoheal}，其中[nodes]即为前面所说的list。如果一个节点与list中的所有节点都无法通信时，自关闭其自身。如果list中的所有节点都down时，其余节点如果是ok的话，也会根据这个规则去关闭其自身，此时集群中所有的节点会关闭。如果某节点能够与list中的节点恢复通信，那么会启动其自身的RabbitMQ应用，慢慢的集群可以恢复。

在autoheal模式下，当认为发生网络分区时，RabbitMQ会自动决定一个获胜的（winning）分区，然后重启不在这个分区中的节点以恢复网络分区。一个获胜的分区是指客户端连接最多的一个分区。如果产生一个平局，既有两个或者多个分区的客户端连接数一样多，那么节点数最多的一个分区就是获胜的分区。如果此时节点数也一样多，将会以一种特殊的方式来挑选获胜分区。
```



这里autoheal和pause_minority的最大区分就是，前者是根据客户端的链接数决定那个是master，而后者是根据集群的node数来决定master。这里，如果使用autoheal的话，很可能会把一个分区中只有节点的node选为master，然后把其他两个node的app重启了。

又查阅了kolla-ansible的源码，确实autoheal会引起不必要的服务中断，具体见：https://bugs.launchpad.net/kolla-ansible/+bug/1837761，在Train版本进行了修复：https://github.com/openstack/kolla-ansible/commit/cdfc1c23442f43b6d86010cbd5147802c5214934。

​		后修改autoheal为pause_minority，可发现模拟网路分区后，再恢复，服务可自动恢复，无需再重启rabbtiqm集群。



---

参考：

https://blog.csdn.net/u013256816/article/details/53588206 RabbitMQ Network Partitions

https://www.pianshen.com/article/28411383079/ RabbitMQ消息不丢失补偿方案

[理解 OpenStack 高可用（HA）（5）：RabbitMQ HA](https://www.cnblogs.com/sammyliu/p/4730517.html)

https://www.cnblogs.com/huligong1234/p/13549450.html#31%E9%80%9A%E8%BF%87%E5%91%BD%E4%BB%A4%E8%A1%8C%E9%85%8D%E7%BD%AE. RabbitMQ高可用-镜像模式部署使用

https://www.cnblogs.com/me-sa/archive/2012/11/12/rabbitmq_ram_or_disk_node.html

https://www.cnblogs.com/frankyou/p/5283825.html

https://www.jianshu.com/p/7cf2ad01c422

https://cloud.tencent.com/developer/article/1631148

https://zhhuabj.blog.csdn.net/article/details/105847301

https://blog.csdn.net/hncscwc/article/details/109444406

https://www.xiexianbin.cn/software/rabbitmq/2017-04-24-rabbitmq-cluster/index.html

https://www.oschina.net/question/2680454_2176038

https://www.rabbitmq.com/ha.html

https://www.cnblogs.com/cwp-bg/p/8397639.html

生产环境建议：

https://github.com/openstack/kolla-ansible/commit/cdfc1c23442f43b6d86010cbd5147802c5214934

