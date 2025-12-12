docker网络主要用来解决容器联网问题，如果容器没有网络，就无法从网络中提供服务。

**网络管理命令帮助**
`docker network --help`

![[Pasted image 20251211232045.png]]

**docker网络类型**

创建容器的时候可以通过`--network`命令来指定容器的网络类型，如果不指定就会使用默认的`bridge`桥接网络。docker网络类型如下：

- bridge
- host
- none
- overlay
- macvlan
- ipvlan

在大多数情况下。bridge和overlay是使用最多的网络类型。


**bridge**

桥接网络是指容器通过桥接的方式将容器网卡桥接到宿主机的docker0网桥，然后通过宿主机防火墙的NAT表实现与外网的联系。

![[Pasted image 20251211232734.png]]
在上图时，还没有创建容器


下图创建了一个新的容器
![[Pasted image 20251212035143.png]]

此时出现了一个新的网卡`veth4f47e6a@if2`，它是容器网卡，每创建一个桥接网络的容器就会生成一个对应的网卡。


也可以在容器中执行命令查看
![[Pasted image 20251212041232.png]]



**查看容器网卡与docker0网卡的桥接信息**

```
#安装brctl命令
apt install -y bridge-utils
```

```
brctl show
```

![[Pasted image 20251212041929.png]]


**创建一个网络为bridge类型的容器，不指定默认也是这个类型**

```
docker run -d --network bridge --name nginx_test nginx:latest
```

**host**

容器和真机共用网卡及对应的端口，缺点是同一个端口只能宿主机或者某个容器使用，其他容器不能。

创建一个网络类型为host的容器

```
docker run -d --network host --name nginx_1 nginx:latest
```

**none**

容器只有lo网卡，是一个不能联网的本地容器

```
docker run -d --network none --name nginx_2 nginx:latest
```

## 实现网桥网络

**目的：** 不同的服务容器组使用不同的网桥，避免同一网络内容器太多，保持容器网络独立性。

如果担心新网桥联网问题，不用担心。创建网桥后，宿主机会自动帮你做NAT，所以不用担心联网问题

**查看网络-ls**

```
➜  ~ docker network ls                                             
NETWORK ID     NAME      DRIVER    SCOPE
5b2d4efa1f70   bridge    bridge    local
2a49adf13d91   host      host      local
9a2def573861   none      null      local
```

**字段说明：**
NETWORK ID    网桥ID
NAME                名称
DRIVER              网络类型
SCOPE                作用范围



**创建网桥-create**

```
docker network create -d bridge --subnet 192.168.1.0/24 --gateway 192.168.1.1 mydocker0
```


```
➜  ~ docker network ls
NETWORK ID     NAME        DRIVER    SCOPE
30b8e8a2a512   bridge      bridge    local
2a49adf13d91   host        host      local
295aed5208f4   mydocker0   bridge    local
9a2def573861   none        null      local
```

**修改网桥名字**

```
#1、关闭新建网桥，重命名前必须禁用
➜  ~ ip link set dev br-295aed5208f4 down     
#2、修改名字，将Linux网络接口br-295aed5208f4从自动生成的名称，重命名为更友好的mydocker0
➜  ~ ip link set dev br-295aed5208f4 name mydocker0
#3、启动网桥
➜  ~ ip link set dev mydocker1 up
#4、重启docker服务
➜  ~ systemctl restart docker 
```

**删除没有使用的网桥-prune**

```
➜  ~ docker network prune
WARNING! This will remove all custom networks not used by at least one container.
Are you sure you want to continue? [y/N] y
Deleted Networks:
mydocker0
```

**删除某个网桥-rm**

```
docker network rm [网桥名称]
```
**注意**：执行删除网桥命令时，该网桥不能被活动容器占用


**容器连接到网桥**

前提：该容器是桥接网络

`docker network connect 网桥 容器`

在连接到该网桥后，使用exec命令进入容器，再使用`ifconfig`或者`ip a`查看，发现会多一个网卡，网卡使用的就是连接网桥的网段

**容器断开网桥**

将容器网络从某一个网桥断开

`docker network disconnet 网桥 容器`


### 不同主机间容器的通信
 
**1、macvlan**

macvlan是一种跨主机的网络模型，作为一种驱动应用，docker macvlan只支持bridge模式

```
#1、 macvlan需要一块独立的网卡来使用，所以需要添加一块新的网卡，我这里使用的是虚拟机的网卡
docker network create -d macvlan \
    --subnet=192.168.1.0/24 \
    --ip-range=192.168.1.0/24 \ 
    -o parent=ens33 \
    mtacvlan-1

# -o parent=绑定网卡名称  指定用来给 macvlan 网络使用的物理网卡
# 要在所有需要使用 macvlan的主机上执行上面这条创建macvlan网络的命令，但要更改网关地址，避免IP冲突，我这里为了方便不指定网关，让docker自己发现网络中的路由


#2、运行一个使用该网络的容器
docker run -itd --network mtacvlan-1 ubuntu:latest bash
```

**2、overlay**

docker swarm用的就是这种网络类型，使用这个网络的前提是先进行`swarm init`

overlay是一种跨主机的全局网络模型，有一个专门的数据库存储网络分配信息，避免IP冲突，同时在内部还有一个小型的DNS，我们可以直接通过主机名进行访问

创建overlay网络(全局网络)：一台主机上创建自动同步

```
docker network create -d overlay overlay-1
```

启动容器测试：

```
docker run -it --name test1 --network overlay-1 ubuntu:latest /bin/bash

docker run -it --name test1 --network overlay-1 ubuntu:latest /bin/bash

# 验证：ping test1

```

**3、ipvlan**

如果macvlan遭到限制，ipvlan是一个很好的代替方案。

创建ipvlan网络
```
docker network create -d ipvlan \
    --subnet=192.168.1.0/24 \
    --ip-range=192.168.1.100/24 \
    --gateway=192.168.1.1 \
    -o parent=ens33 \
    ipvlan_net

```

将容器连接到ipvlan网络
```
docker run -itd --network ipvlan_net ubuntu:latest bash
```

ipvlan有两种模式：`l2`和`l3`

**1、L2**

默认为`l2`模式，容器之间的通信发生在数据链路层(L2)，它们共享宿主机的MAC地址。这是最常见的模式。

命令：

```
docker network create -d ipvlan \
    -o parent=ens33 \
    -o ipvlan_mode=l2 \
    <其他网络参数> \
    ipvlan_l2_net

```


**2、L3**

容器之间的通信发生在网络层(L3)，此模式需要手动配置子网之间的路由，提供更精细的流量控制和隔离，配置更为复杂。

命令：

```
docker network create -d ipvlan \
    -o parent=ens33 \
    -o ipvlan_mode=l3 \
    <其他网络参数> \
    ipvlan_l3_net

```

大多数场景使用默认的l2模式即可。

