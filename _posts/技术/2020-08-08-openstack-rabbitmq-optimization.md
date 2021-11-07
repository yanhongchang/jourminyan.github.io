---
layout: post
title: openstack虚拟机热迁移导致网络中断问题记录
category: 技术
date: 2020-08-08

---

更新历史：

- 2020.08 完成

------

###

在规模500+台的openstack集群下，发现3节点rabbitmq消息队列的cpu负载一只居高不下，一直在500%上下，之前也排查过，发现整个集群会和三个节点的rabbitmq集群会创建1W+的队列，理解是这个规模的私有云集群对rabbitmq有一定压力，理解会有一定的负载，但是后来发现当消息队列集群整个集群在重启后，发现cpu负载竟然达到1200%！这个负载绝对不合理而且是不能接受的，因此决定还是要花点时间查查。

​		

​		下面通过一个测试环境来开始rabbitmq集群的调优，测试环境具体如下：

```
openstack version: openstack-pike
openstack cluster: controller:3 + compute+2
linux system: CentOS Linux release 7.4.1708 (Core) 

rabbtimq: rabbitmq-server-3.6.16-1.el7.noarch
rabbitmq status:

Status of node 'rabbit@work-135-193'
[{pid,216},
 {running_applications,
     [{rabbitmq_management,"RabbitMQ Management Console","3.6.16"},
      {rabbitmq_management_agent,"RabbitMQ Management Agent","3.6.16"},
      {rabbitmq_web_dispatch,"RabbitMQ Web Dispatcher","3.6.16"},
      {rabbit,"RabbitMQ","3.6.16"},
      {os_mon,"CPO  CXC 138 46","2.4.2"},
      {mnesia,"MNESIA  CXC 138 12","4.14.3"},
      {cowboy,"Small, fast, modular HTTP server.","1.0.4"},
      {ranch,"Socket acceptor pool for TCP protocols.","1.3.2"},
      {ssl,"Erlang/OTP SSL application","8.1.3.1"},
      {amqp_client,"RabbitMQ AMQP Client","3.6.16"},
      {rabbit_common,
          "Modules shared by rabbitmq-server and rabbitmq-erlang-client",
          "3.6.16"},
      {xmerl,"XML parser","1.3.14"},
      {compiler,"ERTS  CXC 138 10","7.0.4.1"},
      {inets,"INETS  CXC 138 49","6.3.9"},
      {syntax_tools,"Syntax tools","2.1.1"},
      {recon,"Diagnostic tools for production use","2.3.2"},
      {public_key,"Public key infrastructure","1.4"},
      {asn1,"The Erlang ASN1 compiler version 4.0.4","4.0.4"},
      {cowlib,"Support library for manipulating Web protocols.","1.0.2"},
      {crypto,"CRYPTO","3.7.4"},
      {rabbitmq_clusterer,"Declarative RabbitMQ clustering",[]},
      {sasl,"SASL  CXC 138 11","3.0.3"},
      {stdlib,"ERTS  CXC 138 10","3.3"},
      {kernel,"ERTS  CXC 138 10","5.2"}]},
 {os,{unix,linux}},
 {erlang_version,
     "Erlang/OTP 19 [erts-8.3.5.3] [source] [64-bit] [smp:4:4] [async-threads:64] [hipe] [kernel-poll:true]\n"},
 {memory,
     [{connection_readers,2427344},
      {connection_writers,8440},
      {connection_channels,36800},
      {connection_other,4914648},
      {queue_procs,2832},
      {queue_slave_procs,0},
      {plugins,10008160},
      {other_proc,14976408},
      {metrics,432144},
      {mgmt_db,9603360},
      {mnesia,1224168},
      {other_ets,2525568},
      {binary,200227216},
      {msg_index,44832},
      {code,24977885},
      {atom,1041593},
      {other_system,22712098},
      {allocated_unused,44616088},
      {reserved_unallocated,0},
      {total,135667712}]},
 {alarms,[]},
 {listeners,
     [{clustering,25672,"::"},{amqp,5672,"10.25.135.193"},{http,15672,"::"}]},
 {vm_memory_calculation_strategy,rss},
 {vm_memory_high_watermark,0.4},
 {vm_memory_limit,6663389184},
 {disk_free_limit,50000000},
 {disk_free,147162267648},
 {file_descriptors,
     [{total_limit,1048476},
      {total_used,85},
      {sockets_limit,943626},
      {sockets_used,83}]},
 {processes,[{limit,1048576},{used,934}]},
 {run_queue,0},
 {uptime,8670},
 {kernel,{net_ticktime,60}}]


rabbitmqctl cluster_status
Cluster status of node 'rabbit@work-135-193'
[{nodes,[{disc,['rabbit@work-135-193',
                'rabbit@work-135-194',
                'rabbit@work-135-195']}]},
 {running_nodes,['rabbit@work-135-194',
                 'rabbit@work-135-195',
                 'rabbit@work-135-193']},
 {cluster_name,<<"rabbit@work-135-193">>},
 {partitions,[]},
 {alarms,[{'rabbit@work-135-194',[]},
          {'rabbit@work-135-195',[]},
          {'rabbit@work-135-193',[]}]}]

```



​		因为我们使用了rabbitmq的queue mirror特性，在所有节点配置了所有队列的mirror queue，因此怀疑是否是队列镜像消耗了资源，然后查阅rabbitmq官方的文档：https://www.rabbitmq.com/ha.html，官方的建议是无需在集群的所有节点配置mirror queue，否则会有没必要的资源浪费。

```
Note that mirroring to all nodes is rarely necessary and will result in unnecessary resource waste.
```

​			于是，就修改了配置文件，将queue mirror的数量从所有节点修改成了精确数量的配置。

/etc/kolla/rabbitmq/definitions.json

修改前：



```
"policies":[
    {"vhost": "/", "name": "ha-all", "pattern": ".*", "apply-to": "all", "definition": {"ha-mode":"all"}, "priority":0}  ]
```



修改后：

```
 "policies":[
    {"vhost": "/", "name": "ha-all", "pattern": ".*", "apply-to": "all", "definition": {"ha-mode":"exactly", "ha-params":2}, "priority":0}  ]
```



查看策略：

```
(rabbitmq)[rabbitmq@work-135-193 /]$ rabbitmqctl list_policies 
Listing policies
/       ha-all  all     .*      {"ha-mode":"exactly","ha-params":2}     0
```



可以看到策略已经修改过来了。这里解释下策略的含义，ha-mode的exactly就是精确、确切的意思，ha-params:N是在N个节点上做队列的mirror。这条策略的意思是说，当一个队列被创建后，还需再N节点再mirror一个队列，满足ha-params参数的配置。这个N设置为2。



然后查看三个节点，发现确实是一个队列只有在两个node上又mirror，然后在没有mirror的节点，负载确实十分明显。然而在有mirror的node上，还是cpu 负载几乎咩有变化。这就说明，做了mirror的node，确实是会吃CPU，而且修改了mirror的数量并没有根本解决CPU负载过高的问题。

​	

​		继续排查，通过top —H 查看beam.smp的进程，看看到底是什么在消耗CPU资源

```
top -H -p 4597


  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                                                        
 4710 42439     20   0 3973368 104992   4388 S  6.0  0.6   0:12.14 1_scheduler                                                                                                                    
 4713 42439     20   0 3973368 104992   4388 S  4.7  0.6   0:10.25 4_scheduler                                                                                                                    
 4711 42439     20   0 3973368 104992   4388 S  4.3  0.6   0:08.14 2_scheduler                                                                                                                    
 4712 42439     20   0 3973368 104992   4388 R  4.0  0.6   0:07.01 3_scheduler                                                                                                                    
     
```



可以看到cpu几乎都消耗在了scheduler上，而且4个调度，有三个都是Sleep状态，然后本着不懂就问度娘（或bing）的精神，学习前人的经验。在这里[https://blog.csdn.net/u010657094/article/details/106392113]发现确实rabbitmq的调度器会有busy_wait（忙等待）机制，当调度器完成任务后，会持续等待一段时间后才会进行sleep状态，当处于busy_wait时还是会持续占用CPU，导致CPU负载过高。

该文指出可以通过

```
rabbitmq-diagnostics runtime_thread_stats
```

查看具体线程运行状况，但是这里我们奈何版本过低，没有rabbitmq-diagnostics。不过，这里提到可以优化busy_wait的时间来优化COPU负载，通过+sbwt降低或者取消忙等待。由于文章中提到的版本跟我们的不一致，这里有针对目前的版本在这里[http://erlang.org/documentation/]找到对应的erlang docs [http://erlang.org/documentation/doc-8.3/erts-8.3/doc/html/erl.html]参数文档。

```
+sbwt none|very_short|short|medium|long|very_long
Sets scheduler busy wait threshold. Defaults to medium. The threshold determines how long schedulers are to busy wait when running out of work before going to sleep.
```

默认是medium，这里我们修改为none。就是取消忙等待，但是配置这个后，这里[]可能会出现部分调度器睡死现象，需要结合+sub true,将负载均衡到所有调度器上，避免出现调度器睡死现象。



因此，我们这里优化两个参数：

```
+sbwt none
+sub true
```



配置文件如下：

/etc/kolla/rabbitmq/rabbitmq-env.conf

```
RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS="-pa /usr/lib/rabbitmq/lib/rabbitmq_server-3.6/plugins/rabbitmq_clusterer-3.6.x.ez/rabbitmq_clusterer-3.6.x-667f92b0/ebin"
```



修改为：

```
RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS="+sbwt none +sub true -pa /usr/lib/rabbitmq/lib/rabbitmq_server-3.6/plugins/rabbitmq_clusterer-3.6.x.ez/rabbitmq_clusterer-3.6.x-667f92b0/ebin"
```



重启rabbtimq服务或者rabbitmq容器后，生效。



top 查看CPU负载：

```
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                                                        
 7506 42439     20   0 4006560 128660   4372 S  2.7  0.8   0:03.85 1_scheduler                                                                                                                    
 7508 42439     20   0 4006560 128660   4372 S  2.0  0.8   0:02.35 3_scheduler                                                                                                                    
 7507 42439     20   0 4006560 128660   4372 S  1.7  0.8   0:03.69 2_scheduler                                                                                                                    
 7510 42439     20   0 4006560 128660   4372 S  1.3  0.8   0:03.53 4_scheduler  
```



负载确实是下来了，降低50%+，试运行一段时间看看效果。



参考：

https://news.tianyancha.com/ll_5mrqf4j1wm.html

https://blog.csdn.net/u010657094/article/details/106392113

https://docs.openstack.org/kolla-ansible/latest/reference/message-queues/rabbitmq.html

https://www.rabbitmq.com/runtime.html#cpu 官方cpu优化

https://docs.openstack.org/kolla-ansible/latest/reference/message-queues/rabbitmq.html openstack 优化rabbitmq 传参	

https://blog.csdn.net/u010657094/article/details/106392113

https://www.itdks.com/home/course/detail?id=13676 rabbitmq 调优
