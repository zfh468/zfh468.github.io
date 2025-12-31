---
title: keepalived构建nginx高可用集群
date: 2025-12-30 10:00:00 +0800
categories: [高可用集群]
tags: [keepalived + nginx]
---
**需求：部署基于nginx分发的高可用web集群**
- 分发器故障自动切换
- 数据服务器自动容错
- 任何机器宕机不中断web业务


**实验环境**

以下均为alma linux 9虚拟机

| 角色     | IP             |
| ------ | -------------- |
| master | 192.168.64.130 |
| backup | 192.168.64.134 |
| web1   | 192.168.64.143 |
| web2   | 192.168.64.138 |

**基本准备**

所有虚拟机关闭selinux，关闭防火墙
```
sudo setenforce 0  
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo systemctl disable firewalld
```




**实验步骤**

**一、配置nginx集群**

1.1、master和backup安装nginx和keepalived

```
dnf install -y nginx keepalived
```

master和backup修改nginx配置文件，内容一样
```
vim /etc/nginx/nginx.conf
```

内容
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
    
    # web负载均衡配置
    upstream web {
        # 3秒内失败2次，则认为此节点失效
        server 192.168.64.143 max_fails=2 fail_timeout=3;
        server 192.168.64.138 max_fails=2 fail_timeout=3;
    }

    server {
        listen       80;
        server_name  _;

        location / {
                # least_conn; #最少连接优先(比轮询更稳,但这里我使用轮询)

                # 默认轮询
                proxy_pass http://web;

                # 保留真实客户端信息
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;

                # 超时控制（防雪崩）
                proxy_connect_timeout 3s;
                proxy_send_timeout 10s;
                proxy_read_timeout 10s;

                # 失败自动切换,将同一个请求转发给upstream中的下一个后端节点
                proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
        }

    }

}
```

1.2、master和backup配置keepalived
```
vim /etc/keepalived/keepalived.conf
```

master的keepalived配置文件内容
```
global_defs {
    router_id NGINX_MASTER
}

# 定义检测脚本
vrrp_script chk_nginx {
    # 检查对应位置的脚本是否存在
    script "/etc/keepalived/check_nginx.sh"
    # 定义执行间隔为2秒
    interval 2
    # 1次失败判定异常
    fall 1
    rise 1
    # 降权重
    weight -20
}

#定义实例名称nginx
vrrp_instance nginx {
    # 定义主机状态
    state MASTER
    # 定义通信接口，VIP绑定的接口
    interface ens160
    # 定义VRID，同一个集群的主从设备的VRID要一致
    virtual_router_id 51
    # 定义优先级
    priority 100
    # 定义检查间隔，默认1秒
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 123456
    }
    
    # VIP
    virtual_ipaddress {
        192.168.64.200/24
    }

    track_script {
        # 调用在vrrp_script中定义的内容
        chk_nginx
    }
}
```


backup的keepalived配置文件内容（和master基本一致，只需要修改state和priority）
```
global_defs {
    router_id NGINX_BACKUP
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    fall 1
    rise 1
    weight -20
}

vrrp_instance nginx {
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

    track_script {
        chk_nginx
    }
}
```


master和backup上的`/etc/keepalived/check_nginx.sh`内容

```
#!/bin/bash

if ! pgrep -x nginx >/dev/null; then
    exit 1
fi

exit 0

```

给予脚本执行权限
```
chmod +x /etc/keepalived/check_nginx.sh
```

当检测不到nginx进程，keepalived会进行降权，VIP切换到备用设备



**二、集群高可用性测试**

2.1、web1和web2安装nginx(或者httpd也行，我这里选择nginx)
```
dnf install -y nginx
```

2.2、在master和backup启动nginx和keepalived服务
```
sudo systemctl enable --now nginx
sudo systemctl enable --now keepalived
```

2.3、检查VIP归属

```
ip a
```

只能在master看到VIP192.168.64.200

2.4、准备验证页面(web1和web2)

web1
```
echo "This is web1" > /usr/share/nginx/html/index.html
```

web2
```
echo "This is web2" > /usr/share/nginx/html/index.html
```

curl http://VIP 访问测试或者浏览器访问

反复尝试会返回不同的web页面

**在master上down掉nginx服务**

```
systemctl stop nginx
```

master执行ip a命令，VIP消失
backup出现VIP

继续浏览器或者curl命令访问VIP，网页仍然返回不同的web页面

**关闭master的keepalived服务(这里已经恢复之前关掉的nginx服务)**

```
systemctl stop keepalived
```

VIP会立刻迁移到backup

curl或者网页访问VIP，网页返回不同的网页


**数据服务器宕机测试**

停止一台web服务器的服务，这里选择web1

```
systemctl stop nginx
```

现在只会返回web2的网页

恢复web1的服务
```
systemctl start nginx
```

访问返回不同的网页


