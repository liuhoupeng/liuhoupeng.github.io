---
layout:     post
title:      一种RabbitMQ网络分区问题的解决方案
category: 技术
description: 一种RabbitMQ网络分区问题的解决方案
---
# 背景：
RabbitMQ是实现AMQP(高级消息队列协议)的消息中间件。本文研究在日常测试中发现的RabbitMQ的一个问题——网络分区。叙述从发现问题，分析问题到解决问题的每一步思考，最后实现一套RabbitMQ检测恢复服务，保证RabbitMQ服务的可靠性。
# 发现问题——网络测试结果异常：
## 1.场景复现
基本环境信息：

配置项 | 说明
---|---
RabbitMQ版本 | rabbitmq-server-3.6.6
erlang版本 | erlang-R16B-03.17.el7
RabbitMQ集群节点数量 | 3
操作系统 | CentOS 7.2	
测试步骤以及现象：
- 在集群某两个个RabbitMQ节点执行管理网网卡闪断操作，30次，持续大概一分半钟；
- 闪断结束后，查看集群健康状态，通过RabbitMQ的web管理界面看到警告；

![image](http://note.youdao.com/favicon.ico)
- 通过rabbitmqctl cluster_status命令查到的确发生网络分区，并且无法自动恢复，集群服务不可用；

## 2.初步结论
环境网络异常会导致某一个或者某几个RabbitMQ节点不能与另外节点取得联系，那么Mnesia会认为另外一部分节点服务异常，同理另外一部分节点也会如此认为，在此情况下，可能出现网络分区的问题。

# 分析问题——网络分区从中作梗：
## 1.网络分区
官网对此的描述是RabbitMQ clusters do not tolerate network partitions well，即RabbitMQ集群无法很好的应对网络分区情况。RabbitMQ 会将信息保存在 Erlang 的分布式数据库 Mnesia 中。而和网络分区相关的许多细节问题都和 Mnesia 的行为相关。Mnesia 判定某个 node 失效的根据是，如果其他 node 无法连接该 node 的时间达到net_ticktime定义的时间以上。当这两个 node 恢复到能联系上的状态时，都会认为对端 node 已 down 掉了，此时 Mnesia 将会判定发生了网络分区。产生类似如下日志：
```
=ERROR REPORT==== 15-Oct-2012::18:02:30 ===
Mnesia(rabbit@smacmullen): ** ERROR ** mnesia_event got
    {inconsistent_database, running_partitioned_network, hare@smacmullen}
```
集群信息表示如下：
```
# rabbitmqctl cluster_status
Cluster status of node rabbit@smacmullen ...
[{nodes,[{disc,[hare@smacmullen,rabbit@smacmullen]}]},
 {running_nodes,[rabbit@smacmullen,hare@smacmullen]},
 {partitions,[{rabbit@smacmullen,[hare@smacmullen]},
              {hare@smacmullen,[rabbit@smacmullen]}]}]
```

## 2.网络分区可能产生的影响
当发生网络分区时，可能会产生两个或多个分区，同时认为其他分区里面的节点已经不可用。由于网络分区而被割裂的镜像队列最终会在每个分区中产生一个master,每个分区均能够独立工作(如果达到集群工作条件)，也可能发生其他未定义和奇怪的行为。另外，当网络分区情况得到恢复后，问题依旧存在，需要手动按照步骤进行修复。
## 3.需求分析
针对上述场景以及可能出现的问题，列出如下需求：
- 实时监测每个RabbitMQ节点的状态，并上报；
- 根据上报的状态整理出核心问题场景，针对每种问题场景，通过一定手段自动恢复集群，保证集群可用；
- 需要在RabbitMQ集群中选取一个主节点执行上两步的工作，如果被选取的节点异常或者网络故障，能够及时切换到备用节点，实现主备切换；
# 解决问题——主从监控实时恢复：
## 1.解决思路
总体架构图如下：

![image](http://note.youdao.com/favicon.ico)

首先借鉴已有的RabbitMQ恢复检测脚本，发现存在的问题是必须要在集群每个节点运行，当某个节点因为网络故障不能通讯时，该脚本失效。所以首要要解决的问题就是选取一个”中心节点”，通过中心节点管理整个集群服务。  

***注*：中心节点是选取RabbitMQ集群的某一个节点作为恢复检测脚本运行的主节点，集群中另外节点也有脚本存在，但是没有当即运行。当中心节点发生网络故障时，会自动切换到备用节点继续进行监控。**  

然后针对选取”中心节点”这一需求，原本想引入etcd、zookeeper等服务发现工具，但引入新组件又会造成不可控的行为，最终想到通过Keepalived的主备切换来达到选举中心节点的效果。  

中心节点选举出来后，就需要针对各个场景进行检测和恢复了。经过分析，影响RabbitMQ集群状态因素主要有三个：网络状态，单节点服务状态，网络分区状态。因此将异常场景分为以上三种。针对异常场景需要能够检测到并且恢复，借用之前已有的设计思路，单节点服务状态以及网络分区状态通过RabbitMQ自带的API去获取；网络状态通过socket来获取。”中心节点”每次检测结束会远程各个节点生成状态文件，xinted将状态文件暴露出49203服务端口供haproxy进行判断，通过haproxy实时反馈状态结果。最后通过Keepalived来实现主备的监控切换。  

状态检测脚本会每隔20s检测一次集群状态，并将状态反馈给haproxy，haproxy的配置如下：
```
listen rabbitmq 192.168.101.100:5672
    mode tcp
    balance source
    timeout client 999d
    timeout server 999d
    option tcpka
    option httpchk
    server 192.168.101.53 192.168.101.53:5672 check port 49203 inter 20s rise 2 fall 3 on-marked-down shutdown-sessions
    server 192.168.101.54 192.168.101.54:5672 check port 49203 inter 20s rise 2 fall 3 on-marked-down shutdown-sessions
        server 192.168.101.55 192.168.101.55:5672 check port 49203 inter 20s rise 2 fall 3 on-marked-down shutdown-sessions
```
haproxy每隔20s检测集群节点的49203端口，如果检查到三次失败则判定RabbitMQ服务异常。这里并没有针对RabbitMQ的5672端口进行单独的检测，是因为一方面haproxy多进程会对RabbitMQ产生影响，另一方面，当出现网络分区时，5672端口也是正常的，所以无法检测到此种异常。最后根据返回的节点数据状态，针对三种场景，执行不同的恢复方法。

## 2.状态检测的实现
节点状态数据结构如下：
```
cluster_status = {
   '192.168.101.53':
       {'is_network_ok':True,    #反应节点网络状态
        'is_service_ok':True, #反应节点服务状态
        'is_no_partition':True #反应节点分区状态
       },
    '192.168.101.54':
       {'is_network_ok':True,
        'is_service_ok':True,
        'is_no_partition':True
       },   
   '192.168.101.55':
       {'is_network_ok':True,
        'is_service_ok':True,
        'is_no_partition':True
        }
    }
```
三状态的优先级依次降低，即满足网络状态异常，则不去判断剩下状态，直接记录该节点网络状态异常。否则如果单节点服务异常，则不会判断网络分区状态，记录该节点服务异常。最后如果满足网络正常以及服务正常，则去判断是否有网络分区发生。
```
is_network_ok = self.node_api.networkcheck()
        is_service_ok = True
        if is_network_ok:
            is_service_ok = self.node_api.healthchecks()
        is_no_partition = True
        if is_network_ok and is_service_ok:
            is_no_partition = self.node_api.partitions(self.host_index)
```
通过调取healthchecks/node这个API来获取单节点服务状态，正常状态返回值如下：

![image](http://note.youdao.com/favicon.ico)
通过调取api/nodes这个API来获取单节点网络分区状态，正常状态返回值中，partition的属性值为空，如下：  

![image](http://note.youdao.com/favicon.ico)  
最后根据每个节点的状态，远程ssh各个节点，写入xinted需要检测的状态文件/tmp/.rabbitmq_healthchecks。
## 3.异常恢复的实现
通过上述过程，得到一个集群节点状态的返回对象cluster_status，针对此状态进行进一步判断并根据特定场景执行特定恢复步骤。  

首先根据状态优先级，将各节点加入到不同的列表当中：
```
for host in cluster_hosts:
    if not cluster_status[host]["is_network_ok"]:
        network_failed_nodes.append(host)
    elif not cluster_status[host]["is_service_ok"]:
        service_failed_nodes.append(host)
    elif not cluster_status[host]["is_no_partition"]:
            partition_failed_nodes.append(host)
```
如此分配的好处在于每个节点如果发生异常，只会对应唯一一种异常场景，从而可以根据唯一的异常场景进行相关恢复动作。  

然后将所有异常场景归纳为如下三类：
- 网络异常
- 服务异常
- 分区异常  
针对网络异常，默认该节点不采取任何操作直到网络恢复再进行判断；  
针对服务异常，如果服务异常节点数量小于集群节点总数量的一半，则执行重启异常节点RabbitMQ服务的命令，如果超过集群节点总数量的一半，则认为集群不可用(事实上目前配置默认此情况集群不可用)，会执行重启集群所有节点服务的脚本；  
针对分区异常，只要任意节点发生网络分区，则会按照指定方法执行分区恢复脚本。
**注：分区恢复方法参考官网描述http://www.rabbitmq.com/partitions.html**
## 4.主从监控的实现  

由于检测恢复服务是针对整个集群进行的，所以同一时刻只允许一台机器上运行该服务。另外当节点网络异常时，调用RabbitMQ的相关API会超时，导致判断异常，因此通过Keepalived的形式，给rabbitmq集群的检测恢复服务做了一个主备，当网络异常时可以自动切换到备节点上面继续检测集群服务。Keepalived的配置如下(单节点)：
```
vrrp_instance rabbitmq {
  state BACKUP
  interface ens32
  virtual_router_id 66
  unicast_src_ip 192.168.100.53
  unicast_peer {
  }
  priority 100
  advert_int 1
  notify /etc/keepalived/script/notify_rabbitmq.sh
}
```
主要使用到了Keepalived的notify功能，notify后面接的脚本，会当该节点状态切换时执行。初始时选取一个主节点，notify脚本的作用是当该节点切换为master时，开启检测恢复的服务，当切换为backup时，停掉检测恢复服务，实现主从监控。
# 效果展示：

## 1.Keepalived服务
该节点切换为backup状态，不会有检测恢复服务运行  
![image](http://note.youdao.com/favicon.ico)
## 2.监控日志
每隔20s会执行一次检测，正常检测结束会显示如下信息  

![image](http://note.youdao.com/favicon.ico)  

当出现异常时，会提示如下类似信息：  

![image](http://note.youdao.com/favicon.ico)  

然后开始根据异常场景执行恢复动作： 

![image](http://note.youdao.com/favicon.ico)  
# 总结：
此设计方案，能够基本解决RabbitMQ网络异常时的所出现的所有情况，当集群遇到由于网络异常而产生的问题时，能够保证集群自动恢复，或者保证集群可用。另一方面，针对集群可能出现的其他异常，此方案也能探测出异常情况，从而自动恢复。

# 参考资料：
[1] https://my.oschina.net/moooofly/blog/424660  
[2] http://www.rabbitmq.com/partitions.html