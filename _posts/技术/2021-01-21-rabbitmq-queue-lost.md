---
layout: post
title: rabbitmq队列镜像失效问题发现
category: 技术
date: 2021-01-21

---

更新历史：

- 2021.01-21

------

####

集群现状：

```
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$ rabbitmqctl cluster_status
Cluster status of node 'rabbit@x-x-x-135-193'
[{nodes,[{disc,['rabbit@x-x-x-135-193',
                'rabbit@x-x-x-135-194',
                'rabbit@x-x-x-135-195']}]},
 {running_nodes,['rabbit@x-x-x-135-194',
                 'rabbit@x-x-x-135-195',
                 'rabbit@x-x-x-135-193']},
 {cluster_name,<<"rabbit@x-x-x-135-193">>},
 {partitions,[]},
 {alarms,[{'rabbit@x-x-x-135-194',[]},
          {'rabbit@x-x-x-135-195',[]},
          {'rabbit@x-x-x-135-193',[]}]}]
```

1、非持久化队列场景

1）创建非持久队列yan，node 选择rabbit@x-x-x-135-193。

```
$ rabbitmqctl list_queues name durable  auto_delete |grep yan        
yan     false   false
```

2）在该队列publish 2个消息（1个为persistent消息、1个non-persistent消息)

```
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$ rabbitmqctl list_queues| grep yan
yan     2


web UI：
          Total	Ready	Unacked	In memory	Persistent	Transient, Paged Out
Messages 	2	2	0	2	0	0
```

3）关闭rabbit@x-x-x-135-193

```
docker stop rabbitmq
```



 查看队列镜像情况

```
$ rabbitmqctl list_queues name pid slave_pids synchronised_slave_pids |grep yan     
yan    <rabbit@x-x-x-135-194.2.633.0>      [<rabbit@x-x-x-135-195.3.8859.3>]        [<rabbit@x-x-x-135-195.3.8859.3>]
```

可以看到 队列yan 已经的master 切换到了rabbit@x-x-x-135-194了

web UI:

```
Features	
Policy		ha-all
Node			rabbit@x-x-x-135-194
Mirrors		rabbit@x-x-x-135-195
```

可以看到关闭的135-193不再作为mirrors node显示，mirrors node 只剩下135-195.

在 rabbit@x-x-x-135-194、 rabbit@x-x-x-135-195查看yan 队列的消息情况，可以正常查看2条信息，并且可以读取到。

```
$ rabbitmqctl list_queues| grep yan 
yan     2
```

Web UI:

```
Get messages
Warning: getting messages from a queue is a destructive action. 

Requeue:

Yes
Encoding:

Auto string / base64
 
Messages:
2
Message 1
The server reported 1 messages remaining.

Exchange	(AMQP default)
Routing Key	yan
Redelivered	●
Properties	
delivery_mode:	1
headers:	
Payload
3 bytes
Encoding: string
aaa
Message 2
The server reported 0 messages remaining.

Exchange	(AMQP default)
Routing Key	yan
Redelivered	●
Properties	
delivery_mode:	2
headers:	
Payload
3 bytes
Encoding: string
bbb
```

4）启动135-193 rabbitmq

```
Features	
Policy		ha-all
Node			rabbit@x-x-x-135-194
Mirrors		rabbit@x-x-x-135-195
					rabbit@x-x-x-135-193 (unsynchronised)
					Synchronise
```

发现135-193加会mirrors node中，但是状态是unsynchronised，表明没有自动同步，需哟点击下面的’Synchronise‘ 按钮同步。这里同步指的是消息的同步，后续再同步确定是否是消息。

5)关闭队列yan 的master node，这里是135-194

```
docker stop rabbitmq
```

 查看该队列的镜像情况

```
$ rabbitmqctl list_queues name pid slave_pids synchronised_slave_pids |grep yan
yan     <rabbit@x-x-x-135-195.3.8859.3>      [<rabbit@x-x-x-135-193.2.633.0>]        []
```

 可以看到队列的master node 又切换到了135-195，135-194退出mirrors node，mirrors node只剩下135-193，且没有未同步状态。

 查看该队列的消息情况：

```
$ rabbitmqctl list_queues| grep yan 
yan     2
```

 消息未丢失，可正常读取。

5）启动135-194节点的rabbitmq

 查看队列镜像情况

```
rabbitmqctl list_queues name pid slave_pids synchronised_slave_pids |grep yan
yan     <rabbit@x-x-x-135-195.3.8859.3>      [<rabbit@x-x-x-135-193.2.633.0>, <rabbit@x-x-x-135-194.2.633.0>]        []
```

web UI：

```
Features	
Policy		ha-all
Node			rabbit@x-x-x-135-195
Mirrors		rabbit@x-x-x-135-193 (unsynchronised)
					rabbit@x-x-x-135-194 (unsynchronised)
					Synchronise
```

 可以看到135-194 加回mirrors 中，但是也是处于unsynchronised。

6）此时关闭rabbit@x-x-x-135-195，如果消息丢失，那就可以说明，Synchronise是同步的内容是消息。

 关135-195节点rabbitmq

```
docker stop rabbitmq
```

 在135-193、135-194上查看队列yan

```
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$  rabbitmqctl list_queues| grep yan 
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$ 


(rabbitmq)[rabbitmq@x-x-x-135-194 /]$  rabbitmqctl list_queues| grep yan 
(rabbitmq)[rabbitmq@x-x-x-135-194 /]$ 
```

发现队列在剩下的节点无法查看到，说明这里Synchronise不是同步的消息，同步的是队列。

7）启动135-195的rabbitmq

```
docker start rabbitmq
```

 查看队列yan的情况

```
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$  rabbitmqctl list_queues| grep yan 
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$ 


(rabbitmq)[rabbitmq@x-x-x-135-194 /]$  rabbitmqctl list_queues| grep yan 
(rabbitmq)[rabbitmq@x-x-x-135-194 /]$ 

(rabbitmq)[rabbitmq@x-x-x-135-195 /]$ rabbitmqctl list_queues| grep yan
(rabbitmq)[rabbitmq@x-x-x-135-195 /]$ 
```

Web UI:

 可以看到队列yan，但是state 没有显示。说明这个队列异常了。

经过查阅rbabitmq官方文档 https://www.rabbitmq.com/ha.html，发现如果配置了镜像队列策略，是否为自动同步队列还有参数控制，参数如下：

```
ha-sync-mode:
				manual: 手动同步 （默认）
				automatic: 	自动同步
```

可以看到默认的同步方式是手动同步，这样当一个节点关机后，由于队列时非持久化的，队列丢失。然后再启动时，会生成队列的mirrors ，但是此时应该是原数据，队列实体并咩有同步，需要人工手动同步，如果不同步，队列就不是高可用的了。

 如果使用automatic的话，根据官方文档，建议和ha-sync-batch-size一起用，因为自动同步时，当消息量非常大时，会对网络带宽有非常大的影响。

------

接下来验证一下为队列设置expires导致时导致队列的HA失效。

1）先创建一个测试队列，模拟openstack的队列。创建非持久队列yan，node 依旧选择rabbit@x-x-x-135-193。

```
$ rabbitmqctl list_queues name durable  auto_delete |grep yan        
yan     false   false
[rabbitmq@x-x-x-135-193 /]$  rabbitmqctl list_queues name pid slave_pids synchronised_slave_pids| grep yan
yan_expiry      <rabbit@x-x-x-135-193.2.12086.0>     [<rabbit@x-x-x-135-195.1.13944.0>, <rabbit@x-x-x-135-194.2.23752.1>]    [<rabbit@x-x-x-135-195.1.13944.0>, <rabbit@x-x-x-135-194.2.23752.1>]
```

可以看到队列已经同步到135-194、135-195节点上。

2）在该队列publish 2个消息（1个为persistent消息、1个non-persistent消息)

```
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$ rabbitmqctl list_queues| grep yan
yan_expiry      2

(rabbitmq)[rabbitmq@x-x-x-135-194 /]$ rabbitmqctl list_queues| grep yan
yan_expiry      2

(rabbitmq)[rabbitmq@x-x-x-135-195 /]$ rabbitmqctl list_queues| grep yan
yan_expiry      2
```

1. 设置expiry 参数

查看现在的策略列表：

```
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$ rabbitmqctl list_policies
Listing policies
/       ha-all  all     .*      {"ha-mode":"all"}       0
```

节点x-x-x-135-193上设置如下策略：

```
135-193 /]$ rabbitmqctl set_policy expiry ".*" '{"expires":1800000}' --apply-to queues
Setting policy "expiry" for pattern ".*" to "{\"expires\":1800000}" with priority "0"
```

查看设置完后的策略列表：

```
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$ rabbitmqctl list_policies
Listing policies
/       ha-all  all     .*      {"ha-mode":"all"}       0
/       expiry  queues  .*      {"expires":1800000}     0
```

查看队列yan_expiry的镜像情况：

```
rabbitmqctl list_queues name pid slave_pids synchronised_slave_pids| grep yan 
yan_expiry      <rabbit@x-x-x-135-193.2.12086.0>
```

Web UI:

```
Features	
Policy	expiry
Node	rabbit@x-x-x-135-193
Mirrors	
```

可以看到队列的mirrors 节点已经没有了。目前这种状况，就不再是HA集群，属于普通集群。按照rabbitmq 集群的特性，如果关闭队列实体所属的节点x-x-x-135-193的rabbitmq，其他节点无法读取和消费该队列的消息。下面验证一下：

关闭x-x-x-135-193的rabbitmq

```
docker stop rabbitmq
(rabbitmq)[rabbitmq@x-x-x-135-194 /]$ rabbitmqctl list_queues| grep yan_expiry
(rabbitmq)[rabbitmq@x-x-x-135-194 /]$

(rabbitmq)[rabbitmq@x-x-x-135-195 /]$ rabbitmqctl list_queues| grep yan_expiry
(rabbitmq)[rabbitmq@x-x-x-135-195 /]$
```

可以看到已经无法查询到yan_expiry的队列。当然也无法消费和读取。

按照普通集群的特性，如果恢复x-x-x-135-193的rabbitmq队列中的消息，如果队列是非持久化的，那就会丢失。消息也会丢失。验证一下：

启动x-x-x-135-193的rabbitmq

```
docker start rabbitmq
```

查看队列：

```
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$ rabbitmqctl list_queues| grep yan_expiry
(rabbitmq)[rabbitmq@x-x-x-135-193 /]$

(rabbitmq)[rabbitmq@x-x-x-135-194 /]$ rabbitmqctl list_queues| grep yan_expiry
(rabbitmq)[rabbitmq@x-x-x-135-194 /]$

(rabbitmq)[rabbitmq@x-x-x-135-195 /]$ rabbitmqctl list_queues| grep yan_expiry
(rabbitmq)[rabbitmq@x-x-x-135-195 /]$
```

确实是队列丢失，消息（不管是持久化的还是非持久化）也丢失。

因此，可以确定在执行了如下策略后，会影响到队列的镜像策略。
