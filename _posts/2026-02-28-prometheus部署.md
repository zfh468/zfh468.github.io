---
title: prometheus部署
date: 2026-02-28 00:00:00 +0800
categories: [prometheus + grafana]
tags: [prometheus deploy]
---
### 一、下载并安装Prometheus

官方下载地址<https://prometheus.io/download/>

1.1、下载Prometheus主程序包
```
wget https://github.com/prometheus/prometheus/releases/download/v3.5.1/prometheus-3.5.1.linux-amd64.tar.gz
```


1.2、安装并启动Prometheus
```
# 解压
tar xf prometheus-3.5.1.linux-amd64.tar.gz -C /usr/local/ && cd /usr/local/prometheus-3.5.1.linux-amd64/

# 启动
./prometheus --config.file=prometheus.yml &
```


1.3、防火墙放行9090端口
```
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
```

1.4、web访问Prometheus

浏览器访问http://ip:9090，看到 Prometheus的UI代表部署成功
![](/assets/img/Pasted image 20260228063641.png)


点击status > Target health可以看到被监控的主机，默认情况只监控自己

![](/assets/img/Pasted image 20260228063905.png)

访问http://ip:9090/metrics，看抓取的监控数据
![](/assets/img/Pasted image 20260228064046.png)

prometheus把监控的数据都统一存放在一起,然后生成一个web页面,用户可以通过web页面查看相关的数据,这些数据遵循了时序数据库的格式,也就是key=value的形式.这些数据就是我们的监控指标

这些数据需要进行可视化，否则不容易看



**监控项查询**


假设我们需要查询cpu当前使用状态

点击Query，在下方输入`process_cpu_seconds_total`，点击Execute，再点击Graph可以查看图表


报警功能使用Grafana就好


