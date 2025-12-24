---
title: docker swarm
date: 2025-12-24 08:00:00 +0800
categories: [Linux,Docker]
tags: [Docker-swarm]
---
**一、什么是Swarm**

Swarm是Docker公司推出的用来管理docker集群的平台，几乎完全用go语言开发，源码在<https://github.com/docker/swarm>。

Docker Swarm和Docker Compose一样，都是Docker官方容器编排项目，但不同的是，Docker Compose是一个在单个服务器或主机上创建多个容器的工具，而Docker Swarm可以在多个服务器或主机上创建容器集群服务，对于微服务的部署，Docker Swarm更加适合。

从Docker 1.12.0版本开始。Swarm已经包含在Docker引擎里，并内置服务发现工具。

Swarm deamon只是一个调度器（Schedule）加路由器（router），Swarm自己不运行容器，它只是接收Docker客户端发来的请求，调度适合的节点来运行容器，这就意味着，即使Swarm因为某些原因挂掉了，集群中的节点也会照常运行，当Swarm重新恢复运行以后，它会重新收集集群信息。

**二、Swarm的几个关键概念**

**Swarm**

集群的管理和编排是使用嵌入docker引擎的SwarmKit，可以在docker初始化时启动swarm模式或加入已存在的swarm。

**Node**

一个节点是docker引擎集群的一个实例，还可以将它视为docker节点。

我们可以在单个主机或云服务器上运行一个或多个节点，但生产集群部署通常包括分布在多个物理和云计算机上的docker节点。

要部署应用程序到swarm，要把服务定义提交给管理器节点。

管理器节点将称为任务的工作单元分派给工作节点。

Manager还执行维护所需集群状态所需要的编排和集群管理功能，Manager节点选择单个领导者来执行编排任务，工作节点接受并执行从管理器节点分派的任务。

默认情况下，管理器节点还将服务作为工作节点运行，如果想可以把它们配置成仅运行管理器任务并且是仅管理器节点。代理程序在每个工作程序节点上运行，并报告分配给它的任务。

工作节点向管理节点通知其分配的任务的当前状态，以便管理器可以维持每个工作者的期望状态。

**Service**

一个服务是任务的定义，管理机或者工作节点上执行。它是集群的中心结构，是用户与群体交互的主要根源。创建服务时，需要指定要用的容器镜像。

**Task**

任务是在docker容器中执行的命令，Manager节点根据指定数量的任务副本分配任务给worker节点。


**三、相关命令**

主要分三种类型：

**3.1、集群管理命令-docker swarm**

子命令有`init , join , leave , update。`(docker swarm --help查看命令帮助)

**3.2、服务管理命令-docker service**

子命令有`create，inspect，update，remove，tasks`。（docker service --help查看帮助）

**3.3、节点管理命令-docker node**

子命令有`accept，promote，demote，inspect，update，tasks，ls，rm`。（docker node --help）


**四、swarm集群部署**

**4.1、环境准备**

swarm集群由manager和worker组成

实验环境：三台debian12虚拟机 其中一台作为manager管理节点


如果是rhel需要关掉selinux和防火墙


**4.2、修改主机名并配置/etc/hosts文件**

在选择作为manager的虚拟机执行
```
hostnamectl set-hostname manager
```

在选择作为工作节点的worker1的虚拟机执行
```
hostnamectl set-hostname worker1
```

在选择作为工作节点的worker1的虚拟机执行
```
hostnamectl set-hostname worker2
```

在每一台机子都要配置/etc/hosts文件
```
sudo tee /etc/hosts > /dev/null << 'EOF'

192.168.1.11 manager
192.168.1.14 worker1
192.168.1.15 worker2

EOF
```


随后重启使配置生效


**4.3.创建swarm集群**

**4.3.1、初始化集群**
```
#1、--advertise-addr参数表示其它swarm中的worker节点使用此ip地址与manager联系  
  
docker swarm init --advertise-addr 192.168.1.11
```

初始化集群以后，会输出一段命令，在节点主机输入这段命令就可以添加工作节点到swarm集群

```
➜  ~ docker swarm init --advertise-addr 192.168.1.11
Swarm initialized: current node (t636e4xd210arrlsx8mle9rwo) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1pecyn6fwx8uc2okic5350kxeoyfecliuf71yajf1xz765k43c-00jneatjat9ga17njo3052s2m 192.168.1.11:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```



**4.3.2、添加worker（node工作节点）到swarm**

在worker1和worker2执行
```
docker swarm join --token SWMTKN-1-1pecyn6fwx8uc2okic5350kxeoyfecliuf71yajf1xz765k43c-00jneatjat9ga17njo3052s2m 192.168.1.11:2377
```

**4.3.3、在管理节点manager验证加入情况**

执行
```
docker node ls
```

节点加入成功
```
➜  ~ docker node ls                                 
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
t636e4xd210arrlsx8mle9rwo *   manager    Ready     Active         Leader           28.3.0
t9of34buwfb02z4o48oh2hhwu     worker1    Ready     Active                          28.3.0
4syfcql73g6eqp8x3i3faqoh5     worker2    Ready     Active                          28.3.0
```

**4.3.4、在Swarm中部署服务(Nginx为例)**

**4.3.4.1、创建自定义网络再部署服务**

在docker创建服务时，如果不指定，它会给服务使用默认的网络，虽然可以使用，但是不建议

最好的办法是为你的服务手动创建一个自定义网络，创建服务时，使用--network将它们连接到这个网络。这样的好处是可以实现服务间的可靠通信和网络隔离。

```
➜  ~ docker network create -d overlay nginx_net 
qyiwkbxujclfq0lwr0lwv0pdl
```

查看网络是否创建成功
```
➜  ~ docker network ls | grep nginx_net        
qyiwkbxujclf   nginx_net         overlay   swarm
```

**4.3.4.2、部署服务**

在manager和worker节点上使用上面的覆盖网络创建nginx服务

使用nginx镜像创建了一个具有一个副本(--replicas 1,此参数可以指定服务由多少个实例组成)的nginx服务

另外，不需要提前在节点上下载nginx镜像，这个命令执行后会自动下载这个容器镜像

```
➜  ~ docker service create --replicas 1 --network nginx_net --name my_nginx1 -p 80:80 n
ginx
jfm8wyz4lmvb2iq3wes3lda7d
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service jfm8wyz4lmvb2iq3wes3lda7d converged 
```

**4.3.4.3、查看正在运行的服务列表**

```
➜  ~ docker service ls                                                                     
ID             NAME        MODE         REPLICAS   IMAGE          PORTS
jfm8wyz4lmvb   my_nginx1   replicated   1/1        nginx:latest   *:80->80/tcp
```

**4.3.4.4、查询swarm中服务的信息**

加上--pretty输出内容更容易看，不加可以输出更详细的信息

```
➜  ~ docker service inspect --pretty my_nginx1

ID:             jfm8wyz4lmvb2iq3wes3lda7d
Name:           my_nginx1
Service Mode:   Replicated
 Replicas:      1
Placement:
UpdateConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         nginx:latest@sha256:fb01117203ff38c2f9af91db1a7409459182a37c87cced5cb442d1d8fcc66d19
 Init:          false
Resources:
Networks: nginx_net 
Endpoint Mode:  vip
Ports:
 PublishedPort = 80
  Protocol = tcp
  TargetPort = 80
  PublishMode = ingress 
```

**4.3.4.5、查询哪个节点正在运行该服务**

```
➜  ~ docker service ps my_nginx1
ID             NAME          IMAGE          NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
p7fjpx6npyc6   my_nginx1.1   nginx:latest   manager   Running         Running 8 minutes ago 
```

**4.3.4.6、swarm动态扩容**

nginx服务扩容到4个副本

如果想要减少服务，只需要修改scale my_nginx1的值就可以了

```
➜  ~ docker service scale my_nginx1=4
my_nginx1 scaled to 4
overall progress: 4 out of 4 tasks 
1/4: running   [==================================================>] 
2/4: running   [==================================================>] 
3/4: running   [==================================================>] 
4/4: running   [==================================================>] 
verify: Service my_nginx1 converged 
```

检查
```
➜  ~ docker service ps my_nginx1     
ID             NAME          IMAGE          NODE      DESIRED STATE   CURRENT STATE                ERROR     PORTS
p7fjpx6npyc6   my_nginx1.1   nginx:latest   manager   Running         Running 12 minutes ago                 
wdcxq7rrij71   my_nginx1.2   nginx:latest   worker2   Running         Running about a minute ago             
koosg2jci8qj   my_nginx1.3   nginx:latest   worker1   Running         Running about a minute ago             
sjuz7jomem0d   my_nginx1.4   nginx:latest   worker1   Running         Running about a minute ago
```

和创建服务一样，增加scale数之后，将会创建新的容器，这些新启动的容器也会经历从准备到运行的过程，大概一分钟左右，服务应该就会启动完成，这时候可以看一下nginx服务中的容器


**4.3.4.7、升级镜像/升级业务/回滚业务**

```
docker service update --image nginx:new my_nginx1
```

把swarm集群中名为my_nginx1的service使用的镜像，更新为nginx:new，并触发一次滚动更新，保证服务不会中断，而且如果升级更新失败，会自动回滚

**4.3.4.8、删除服务**

```
docker service rm my_nginx1
```


**4.3.4.9、节点管理**

停止节点worker1
```
docker node update --availability drain worker1
```

重新激活节点
```
docker node update --availability active worker1
```
