
RHEL系使用firewalld，Ubuntu、Debian等使用ufw


## 一、firewalld使用

服务管理：
```
systemctl start firewalld
systemctl stop firewalld
systemctl restart firewalld
systemctl status firewalld
```

开机自启：
```
systemctl enable firewalld
systemctl disable firewalld
```


firewalld信息查看
```
# 1、查看默认区域
firewall-cmd --get-default-zone

# 2、查看所有区域
firewall-cmd --get-zones

# 3、查看当前活动区域
firewall-cmd --get-active-zones

# 4、查看某个区域规则
firewall-cmd --zone=public --list-all

# 查看所有规则
firewall-cmd --list-all
```


端口控制：
```
# 临时开放端口，重启后失效
firewall-cmd --add-port=80/tcp

# 指定区域
firewall-cmd --zone=public --add-port=8080/tcp

# 永久开放端口
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload
```

删除端口：
```
firewall-cmd --zone=public --remove-port=80/tcp --permanent
firewall-cmd --reload
```


服务控制：
```
# 1、查看所有可用服务
firewall-cmd --get-services

# 2、开放HTTP服务
firewall-cmd --add-service=http --permanent
firewall-cmd --reload

# 3、删除HTTP服务
firewall-cmd --remove-service=http --permanent
firewall-cmd --reload
```


区域管理：
```
# 1、设置默认区域
firewall-cmd --set-default-zone=public

# 2、把网卡加入区域
firewall-cmd --zone=internal --change-interface=eth0

# 3、把IP加入区域
firewall-cmd --zone=trusted --add-source=192.168.1.0/24 --permanent
```

富规则：
```
# 1、允许某IP访问
firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="192.168.1.10" port port="22" protocol="tcp" accept'
  
# 2、拒绝某IP
firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="1.2.3.4" reject'
  
```


开启nat转发：

开启IP转发
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

端口转发
```
firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8080
firewall-cmd --reload
```

做完nat转发后，访问这台机器80端口的流量，自动转发到这台机器8080端口。

如果要结合ip转发可以使用`--toaddr`参数。


## 二、ufw使用

服务管理：
```
ufw enable
ufw disable
ufw reload
ufw status
ufw status verbose
```

默认策略：
```
ufw default deny incoming
ufw default allow outgoing
```

开放端口：
```
ufw allow 80
ufw allow 22/tcp
ufw allow 443
```

基于IP的访问控制：
```
# 1、允许某IP访问22端口
ufw allow from 192.168.1.10 to any port 22

# 2、拒绝某IP
ufw deny from 1.2.3.4
```


删除ufw规则：

先查看规则编号
```
ufw status numbered
```

删除对应规则编号
```
ufw delete [编号]
```



## 三、selinux

selinux有三种运行模式：
- `Enforcing`，强制模式，也是默认的模式。纪录违规行为，并直接拦截。
- `Permissive`，宽容模式，不拦截操作，但会把行为写进日志。
- `Disabled`，禁用。

一般来说，如果程序没有问题跑不通，大概率是selinux的问题，可以考虑禁用，修改文件`/etc/selinux/config`，将SELINUX=enforing 改为disabled即可。


常用相关命令：
```
# 查看selinux状态
getenforce或sestatus

# 临时切换工作模式
setenforce 0   # 切换到Permissive
setenforce 1   # 切换到Enforcing
```

