---
title: HAProxy + keepalived
date: 2025-12-26 013:00:00 +0800
categories: [高可用集群,HAProxy]
tags: [HAProxy高可用集群]
---
目前我的架构是：通过haproxy将来自用户的请求转发到后端的web站点。

问题是，如果HAProxy挂了，整个服务就挂了（单点故障）。

计划增加一台LB备用，两台LB都要安装**HAProxy + keepalived**。


**节点规划**

| 角色     | 主机名      | IP             |
| ------ | -------- | -------------- |
| LB1（主） | haproxy  | 192.168.64.139 |
| LB2（备） | haproxy2 | 192.168.64.137 |
| web    | web1     | 192.168.64.136 |
| web    | web2     | 192.168.64.134 |
| VIP    |          | 192.168.64.200 |


**两台LB都要做的准备**

1、关闭防火墙，禁用selinux
```
systemctl disable --now firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

2、安装haproxy+keepalived
```
dnf install -y haproxy keepalived
```

在前面的实验里，LB1已经安装过haproxy了

3、HAProxy配置（两台LB都一样）

注意：HAProxy要绑定VIP，不是本机IP

```
vim /etc/haproxy/haproxy.cfg
```

只需要修改前端部分的监听地址
```
frontend main
	# 监听地址，这里由之前的 *:80 改成 192.168.64.200:80
    bind 192.168.64.200:80
    acl url_static       path_beg       -i /static /images /javascript /stylesheets
    acl url_static       path_end       -i .jpg .gif .png .css .js

    use_backend static          if url_static
    default_backend             app

```

4、keepalived核心配置


4.1、LB1（master）配置
```
vim /etc/keepalived/keepalived.conf
```


要配置配置router_id、state、interface、priority、auth_pass、虚拟IP。

LB1配置文件内容
```
global_defs {
    router_id LB1
}

vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 123456
    }

    virtual_ipaddress {
        192.168.64.200/24
    }
}
```


LB2(BACKUP)的keepalived配置文件内容（简洁版）
```
global_defs {
    router_id LB2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 90
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 123456
    }

    virtual_ipaddress {
        192.168.64.200/24
    }
}
```



虚拟IP、auth_pass、virtual_router_id必须一致！！！！


两台LB先启动HAProxy
```
systemctl restart haproxy
```

如果在LB2（BACKUP）绑定VIP后，启动HAProxy时失败，原因是在备用机上还没有VIP，HAProxy试图绑定，导致系统内核拒绝。

解决方法

允许HAProxy绑定“不存在的IP”

在两台LB上执行：
```
sysctl -w net.ipv4.ip_nonlocal_bind=1
```

如果想永久生效：
```
echo "net.ipv4.ip_nonlocal_bind = 1" >> /etc/sysctl.conf
sysctl -p
```

原理：允许进程绑定一个当前未分配到本地的IP

这样，在master存在VIP可以正常监听，backup没有VIP也可以提前监听。

然后就可以重启HAProxy了。




再启动keepalived
```
systemctl enable --now keepalived
```

检查VIP是否生效，在master上执行：
```
ip addr | grep 192.168.64.200
```

你会看到VIP绑定在配置的网卡上，我这里是ens160。
backup是没有VIP的，除非master挂掉。


服务访问测试

curl访问VIP
```
[root@localhost ~]# curl http://192.168.64.200
This is Web1
[root@localhost ~]# curl http://192.168.64.200
This is Web2
[root@localhost ~]# curl http://192.168.64.200
This is Web1
[root@localhost ~]# curl http://192.168.64.200
This is Web2
[root@localhost ~]# curl http://192.168.64.200
This is Web1
```

得到以上响应表示配置成功



故障切换测试


在LB1（master）执行：
```
systemctl stop keepalived
```

在LB2（backup）执行：
```
ip addr | grep 192.168.64.200
```

此时VIP出现在LB2（BACKUP）

VIP已经漂移到LB2

LB1的keepalived虽然停了，但是HAProxy的负载均衡没受影响，所以现在多次访问VIP的网页还会显示web1和web2的内容。


keepalived职责：
- 基于VRRP协议
- 进行VIP的迁移和主备节点的选举
- 不处理任何业务流量

HAProxy的职责：
- 真正监听VIP:port
- 接收客户端请求
- 按照算法转发到后端


总结：keepalived只负责VIP的高可用，不参与实际的流量转发，真正做负载均衡和转发的是HAProxy。

所以就算keepalived停止，只要VIP还在当前节点，且HAProxy进程仍在运行并监听端口，业务访问就不会被中断。


**keepalived 绑定 haproxy 状态**

为什么这么做：为了实现真正的高可用。keepalived不知道HAProxy是否还能继续对外服务，所以必须将HAProxy的状态告诉keepalived，否则VIP可能会停在一个不能工作的节点上，导致服务中断。

```
global_defs {
    router_id LB1
}

# 定义健康检查脚本：同时检查进程和端口响应
vrrp_script chk_haproxy {
    # 尝试访问本地80端口，如果端口不通或进程不存在，返回非0状态
    script "/usr/bin/timeout 2 bash -c '</dev/tcp/127.0.0.1/80' || pidof haproxy"
    interval 2     # 每2秒检查一次
    weight -60     # 故障扣60分 (150-60=90)，确保低于备机的140
    fall 2         # 连续失败2次才判定为故障
    rise 1         # 成功1次即恢复
}

vrrp_instance VI_1 {
    state MASTER          # 主节点
    interface ens160
    virtual_router_id 51
    priority 100          # 初始优先级 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 123456
    }

    virtual_ipaddress {
        192.168.64.200/24
    }

    # 引用上方定义的脚本
    track_script {
        chk_haproxy
    }
}
```

LB2只需要修改state和priority。

LB1和LB2重启keepalived服务。
```
systemctl restart keepalived
```


测试HAProxy进程故障

在LB1停止HAProxy进程
```
systemctl stop haproxy
```

等主节点的优先级降低以后，VIP会转移到备用节点。

```
ip addr | grep 192.168.64.200
```
