**一、软件包获取**

tomcat官网：<https://tomcat.apache.org/>

jdk:<https://www.oracle.com/middleeast/java/technologies/downloads/>

我这里选择使用jdk21，tomcat10.1.52

**二、安装jdk**

点击下载下来的exe下载文件

安装完成后点击关闭
![[Pasted image 20260131025231.png]]

可以使用cmd键入java --version检查安装是否成功，显示具体信息代表成功
![[Pasted image 20260131025409.png]]

**配置JAVA_HOME**

哪怕安装好了java，不配置好环境变量，tomcat启动也会失败，它只认系统环境变量里的JAVA_HOME或JRE_HOME

哪怕java --version能跑，tomcat也有可能出现这样的问题

设置环境变量
```
设置
→ 系统
→ 关于
→ 高级系统设置
→ 环境变量
```

点击系统变量，新建，变量名为JAVA_HOME，变量值为jdk的安装目录，我的安装目录为`C:\Program Files\Java\jdk-21.0.10`
![[Pasted image 20260131032032.png]]

重新打开一个新的cmd
执行：
```
echo %JAVA_HOME%
```

你应该看到输出了jdk的安装目录
![[Pasted image 20260131032229.png]]

到现在jdk就已经完全安装好了


**三、安装tomcat**

解压tomcat压缩包并进入bin目录下，这里有许多tomcat的工具

启动tomcat
执行startup.bat
![[Pasted image 20260131032502.png]]


访问tomcat默认网站

打开浏览器输入http://127.0.0.1:8080
![[Pasted image 20260131032535.png]]
看到这个页面代表部署tomcat成功