**目标**

部署基于LVS DR模式的web高可用集群，实现：
- 数据服务器容错
- 分发器故障切换
- 任何机器宕机不中断web服务

**一、实验环境**

五台安装alma Linux的虚拟机，一台客户端，两台web服务器（Real Server），两台LVS分发器(主备)

| 角色名称   | 接口名称   | IP地址           |
| ------ | ------ | -------------- |
| VIP    |        | 192.168.64.100 |
| lvs1   | ens160 | 192.168.64.130 |
| lvs2   | ens160 | 192.168.64.137 |
| rs1    | ens160 | 192.168.64.143 |
| rs2    | ens160 | 192.168.64.138 |
| client | ens160 | 192.168.64.136 |

**1.1、所有节点环境基础配置**

关闭防火墙&selinux
```
systemctl disable --now firewalld
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

**二、Real Server（RS）配置(web服务器)**

在rs1/rs2操作

**2.1、安装httpd，定制测试页面**

rs1
```
dnf install -y httpd
echo "This is RS1" > /var/www/html/index.html
systemctl enable --now httpd
```


rs2
```
dnf install -y httpd
echo "This is RS2" > /var/www/html/index.html
systemctl enable --now httpd
```

**2.2、配置DR模式的步骤（rs1&rs2）**

2.2.1、绑定VIP到lo
```
ip addr add 192.168.64.100/32 dev lo
ip link set lo up
```

2.2.2、禁止RS响应ARP
```
cat >> /etc/sysctl.conf <<EOF
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
# rp_filter如果是1，会直接丢包
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.lo.rp_filter = 0
EOF

sysctl -p
```

RS要禁止ARP，防止RS抢VIP，保证VIP只有LVS响应


**三、LVS节点配置(master&backup)**

在master&backup上操作

3.1、安装keepalived和ipvsadm
```
dnf install -y ipvsadm keepalived
```

3.2、开启内核转发(必须)
```
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```


**四、keepalived配置**

4.1、master配置`/etc/keepalived/keepalived.conf`


```
global_defs {
   router_id LVS_MASTER
}

vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.64.100/32 dev ens160 label ens160:vip
    }
}
# VIP
virtual_server 192.168.64.100 80 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    protocol TCP
	
	# rs1
    real_server 192.168.64.143 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
	# rs2
    real_server 192.168.64.138 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
}
```

4.2、backup的keepalived配置

注意，backup节点同样要配置virtual_server，不然在主节点故障时，虽然VIP会迁移，但是backup缺少LVS转发表，业务仍然不可用。


```
global_defs {
   router_id LVS_BACKUP
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.64.100/32 dev ens160 label ens160:vip
    }
}

virtual_server 192.168.64.100 80 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    protocol TCP

    real_server 192.168.64.143 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.64.138 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
}

```



**五、启动与验证**

5.1、启动keepalived(master&backup)

```
systemctl enable --now keepalived
```

5.2、master上确认VIP是否存在
```
ip addr | grep 192.168.64.100
```


5.3、查看lvs转发表(master)
```
ipvsadm -Ln
```

![[Pasted image 20260101090430.png]]



5.4、访问测试，浏览器/curl/或者elinks测试

客户端使用elinks访问VIP测试

alma linux官方源没有elinks，在EPEL中有。

先安装EPEL源
```
dnf install -y epel-release
```

验证EPEL安装
```
dnf repolist | grep epel
```

安装elinks
```
dnf install -y elinks
```

elinks测试
```
elinks http://192.168.64.100 --dump
```

返回不同的页面
![[Pasted image 20260101090725.png]]


5.5、keepalived故障切换验证（重要！）

在master停掉keepalived服务

准备验证：
- VIP是否迁移到backup
- 客户端访问是否仍然正常

如果访问不受影响，高可用集群部署成功

```
systemctl stop keepalived
```

检查master的VIP是否已释放
![[Pasted image 20260101093026.png]]
这里master的VIP已经消失

在backup检查VIP，已经迁移到backup
![[Pasted image 20260101092913.png]]

再elinks访问
![[Pasted image 20260101091422.png]]
如果还能如图上返回正常网页内容，可进行下一步测试。


5.6、验证数据服务器容错

随便停止一台rs服务器的web服务，这里我选择rs1
```
systemctl stop httpd
```

在客户端访问
![[Pasted image 20260101093431.png]]

用户只能访问到rs2的页面.


如果想对vrrp协议有进一步的了解，可以使用tcpdump命令，能看到vrrp的相关发送消息
```
tcpdump -nn -vvv vrrp
```
![[Pasted image 20260101093731.png]]

图中数据包内容解释：

192.168.64.137(backup)这台机器现在正在以`VRRP master`的身份，每一秒向`224.0.0.18`发送一次VRRP通告，声明它是VRID=51的主节点，优先级=90，VIP=192.168.64.100。

`224.0.0.18`是VRRP官方保留组播地址，所有VRRP节点都会监听这个地址，backup就靠监听这个组播包判断master是否还存活。

vrrp本质就一句话：master不停广播：我每一秒告诉你们一次，我还活着，VIP归我！！！

到这里，在alma linux上，我们基于LVS + Keepalived部署了DR模式高可用负载均衡集群，实现了VIP的故障自动迁移与后端web服务的无感知切换。



