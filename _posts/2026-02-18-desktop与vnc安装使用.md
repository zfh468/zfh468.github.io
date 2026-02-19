
**一、Linux桌面安装**

```
dnf groupinstall "Server with GUI" -y
```

设置默认启动为图形界面
```
systemctl set-default graphical
reboot
```

重启后
![](/assets/img/Pasted image 20260220001839.png)



**二、vnc安装与配置**

vnc说常用的远程桌面协议，常用于远程桌面连接，通过它，你可以在Windows、mac以及另一台Linux主机远程控制你的Linux桌面。


2.1、tiger vnc安装
```
dnf install epel-release -y
dnf update -y
dnf install tigervnc-server -y
```


2.2、设置vnc连接密码

切换到想远程登录的用户
```
vncpasswd
# 会询问是否设置 "view-only" 密码，通常选 n。
```



VNC 默认使用 5900 + 桌面号 的端口。如果开启了 :1，则需要开放 5901。



2.3、firewalld防火墙配置
```
firewall-cmd --permanent --add-service=vnc-server
# 或者直接开端口
firewall-cmd --permanent --add-port=5901/tcp
firewall-cmd --reload

```

2.4、启动vnc
```
vncserver :1
```



2.5、在windows安装vnc viewer，输入ip地址:5901，再输入vnc密码连接

![](/assets/img/Pasted image 20260220020655.png)


连接成功
![](/assets/img/Pasted image 20260220020742.png)



**三、vnc管理命令**

1、启动vnc
```
# 启动 1 号桌面（对应端口 5901），一旦断开连接就关掉
vncserver :1 -autokill
```

2、查看vnc
```
vncserver -list
```

3、关闭vnc
```
vncserver -kill :1
```






