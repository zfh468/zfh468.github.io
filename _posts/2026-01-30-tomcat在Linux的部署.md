**一、软件包下载**

tomcat官网：<https://tomcat.apache.org/>

jdk:<https://www.oracle.com/middleeast/java/technologies/downloads/>

alma linux ，jdk21，tomcat10.1.52

**二、安装jdk**

```
# 1、查看当前是否安装过jdk
[root@192 ~]# java --version
-bash: java: 未找到命令

# 2、系统提示没找到命令，意味着没有安装过，所以我们要安装，这里我提前下载好了安装包
[root@192 ~]# ls
anaconda-ks.cfg  apache-tomcat-10.1.52.zip  jdk-21_linux-x64_bin.rpm

# 3、安装jdk21
[root@192 ~]# rpm -ivh jdk-21_linux-x64_bin.rpm 
警告：jdk-21_linux-x64_bin.rpm: 头 V3 RSA/SHA256 Signature, 密钥 ID 8d8b756f: NOKEY
Verifying...                          ################################# [100%]
准备中...                          ################################# [100%]
正在升级/安装...
   1:jdk-21-2000:21.0.10-8            ################################# [100%]
   
# 4、验证jdk安装，安装成功
[root@192 ~]# java --version
java 21.0.10 2026-01-20 LTS
Java(TM) SE Runtime Environment (build 21.0.10+8-LTS-217)
Java HotSpot(TM) 64-Bit Server VM (build 21.0.10+8-LTS-217, mixed mode, sharing)
[root@192 ~]#
```

**三、安装tomcat**

```
# 1、安装tomcat
[root@192 ~]# unzip apache-tomcat-10.1.52.zip -d /opt/

[root@192 ~]# cd /opt/
[root@192 opt]# ls
apache-tomcat-10.1.52
[root@192 opt]# mv apache-tomcat-10.1.52 tomcat1
[root@192 opt]# cd tomcat1/
[root@192 tomcat1]# ls
bin           CONTRIBUTING.md  logs       RELEASE-NOTES  webapps
BUILDING.txt  lib              NOTICE     RUNNING.txt    work
conf          LICENSE          README.md  temp
[root@192 tomcat1]# cd bin

# 2、启动tomcat
[root@192 bin]# sh startup.sh 
Cannot find ./catalina.sh
The file is absent or does not have execute permission
This file is needed to run this program
[root@192 bin]# pwd
/opt/tomcat1/bin
# 给予catalina.sh执行权
[root@192 bin]# chmod +x catalina.sh
[root@192 bin]# sh startup.sh 
Using CATALINA_BASE:   /opt/tomcat1
Using CATALINA_HOME:   /opt/tomcat1
Using CATALINA_TMPDIR: /opt/tomcat1/temp
Using JRE_HOME:        /usr
Using CLASSPATH:       /opt/tomcat1/bin/bootstrap.jar:/opt/tomcat1/bin/tomcat-juli.jar
Using CATALINA_OPTS:   
Tomcat started.

# tomcat的两个端口
# 8005 是关闭tomcat使用的端口，可以使用telnet serverip 8005 ，然后输入大写的SHUTDOWN关闭tomcat，所以建议更改端口或改命令
# 8080 是连接端口
[root@192 bin]# ss -tunlp | grep java
tcp   LISTEN 0      1      [::ffff:127.0.0.1]:8005            *:*    users:(("java",pid=2119,fd=52)) 
tcp   LISTEN 0      100                     *:8080            *:*    users:(("java",pid=2119,fd=44)) 

```

**四、访问默认首页**

![](/assets/img/Pasted image 20260131034645.png)

tomcat的访问端口是8080
