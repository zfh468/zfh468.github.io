---
title: Dockerfile
date: 2025-12-24 00:00:00 +0800
categories: [Linux,Docker]
tags: [docker-Dockerfile]
---
**一、基本概念**

**镜像：** `镜像`是由`多个镜像层`叠加起来的一个文件系统(通过`Union FS`与`AUFS`文件联合系统实现)，镜像层也可以理解为一个基本的镜像，每个镜像层之间通过`指针`的形式进行叠加。

**Dockerfile:** 一个包含用于组合镜像的命令的脚本，docker通过Dockerfile和构建环境的上下文来构建镜像。

Dockerfile每一条指令都会生成一层镜像。


**二、Dockerfile基础命令**

##### 2.1、FROM

**功能：** 指定基础镜像，并且**必须是第一条指令** ，后续的指令会运行在这个镜像上。

如果不以任何镜像为基础，写法为`FROM scratch`。

一个Dockerfile可以有多个FROM。

**语法：** 
`FROM 镜像名[:tag] [AS name]`
 AS用于多阶段构建

`FROM nginx `


##### 2.2、MAINTAINER/LABEL

作用：指定作者,仅作为元信息，不影响镜像构建或运行行为
语法：`MAINTAINER <author_name> <email>`

此命令已废弃，被`LABEL`代替


**LABEL用法：**

```
LABEL maintainer="name <test@gmail.com>"
```

或者更标准的OCI标签：
```
LABEL org.opencontainers.image.authors="name <test@gmail.com>"
```

一个dockerfile可以有多个LABEL，但是最好写在一行，太长需要换行使用`\`符号

LABEL会继承基础镜像中的LABEL，如果key相同，会出现值被覆盖的情况


LABEL除了设置维护者的基本信息外，还可以设置元数据，如下：
```
LABEL version="1.0"
```

当我们队docker build后的镜像使用docker inspect命令时就会看见这个LABEL

##### 2.3、RUN

功能：运行指定的命令，是构件容器时就运行的命令

**语法：** 

shell格式

`RUN command`      
或者

exec格式

`RUN ["executable","param1"]`

**特点：** 构建时执行，每条RUN生成一层


**示例：**
`RUN bash -c 'touch /test.txt'`
`RUN ["bash","-c","touch","test.txt"]`
`RUN mkdir /data`
`RUN ["mkdir","/data"]`

多行命令不要写多个RUN，Dockerfile的每一个指令都会建立一层，RUN书写时的换行符是`\`，多少个RUN就构建了多少层镜像，会造成镜像的臃肿、多层，不仅仅增加了部署时间，还容易出错

##### 2.4、ADD

一个复制命令，把文件复制到镜像中

**语法：** 
`ADD <src> <dest>`

src可以是一个本地文件或者本地压缩文件，还可以是一个url

dest路径的填写可以是容器里的绝对路径，也可以是相对于工作目录的相对路径

**示例** 
```
ADD test1.txt test1.txt
ADD test1.txt test1.txt.bak 
ADD test1.txt /mydir/ 
ADD data1 data1 
ADD zip.tar /myzip
```

**特点：** 自动解压.tar，支持URL，可以使用通配符`?`和`*`匹配一个或多个字符

绝大多数情况用COPY就行

##### 2.5、COPY

**语法：**
`COPY <src> <dest>`

**特点：** 只能拷贝本地文件，支持通配符，比ADD更好推荐使用

**用法：** 
```
COPY file.txt /opt/
COPY dir/ /opt/dir/
COPY ["file.txt","/opt/"]
```

##### 2.6、VOLUME

可以实现挂载功能，可以将本地文件夹或者其他容器里的文件夹挂载在这个容器里

不过，主机的目录只能在docker run的时候声明，这是为了保证镜像的可移植性

**示例：** 
```
VOLUME ["/data"]
VOLUME /data

VOLUME /var/log /var/db
VOLUME ["/var/log","var/db"]
```

**特点：** 声明匿名卷，运行时自动挂载

使用场景：需要持久化存储数据时，容器使用的是AUFS，这种文件系统不能持久化数据，当容器关闭后，所有的更改都会丢失，所以当数据需要持久化时用这个命令

##### 2.7、EXPOSE

功能：暴露容器运行时的监听端口给外部

但是EXPOSE不会让容器访问主机的端口

如果想让容器与主机的端口有映射关系，必须在容器启动时加上`-P` 参数，不过会使用随机的端口

`EXPOSE 80`
`EXPOSE 80/tcp`

作用：
- 文档性声明
- 不做端口映射

##### 2.8、WORKDIR

功能：设置工作目录，该目录不存在会自动创建

**语法：** 
`WORKDIR /app`

**特点：** 自动创建目录，可多次使用（相对路径叠加）

```
WORKDIR /app
WORKDIR src    # 实际是/app/src
```

##### 2.9、ENV（运行时变量）

功能：设置环境变量，变量会保存到容器里

**语法**

第一种方法可以一次设置多个环境变量

`ENV KEY=value`
或者
`ENV KEY value`

想使用该环境变量时只需要`${KEY}`
```
ENV my_version=1.0
RUN apt-get install -y mypackage=${my_version}
```


###### 2.10、CMD

功能：使用容器启动时要执行的命令，在构件时不运行

**用法：**
```
CMD ["executable","param"]
CMD command param
CMD ["param"]
```

**示例：**
```
CMD ["sh","-c","echo $HOME"]

CMD ["echo","$HOME"]
```

注意，这里面包括参数的一定要使用双引号，不能是单引号，原因是参数传递后，docker解析的是一个`JSON Array`

**特点：** 只有一个生效（最后一个），可以被docker run覆盖

##### 2.11、ENTRYPOINT

**语法：**
`ENTRYPOINT ["executable","param"]`
`ENTRYPOINT command param1 param2`

**特点：** 不易被覆盖，常用于工具容器

**CMD + ENTRYPOINT组合规则**

如果在Dockerfile同时写了`ENTRYPOINT`和`CMD`，并且`CMD`指令不是一个完整的可执行命令，那么CMD指定的内容会被作为ENTRYPOINT的参数，如下：

```
ENTRYPOINT ["python"]
CMD ["app.py"]
```

实际执行：
`python app.py`


如果我们在Dockerfile里同时写了`ENTRYPOINT`和`CMD`，并且CMD是一个完整的指令，那么它们两个会互相覆盖，谁在最后谁生效，如下：

```
FROM ubuntu
ENTRYPOINT ["top","-b"]
CMD ls -la
```

实际执行：
`ls -la`

而top -b不会执行

##### 2.12、ARG（构建时变量）
**特点：** 只在build阶段有效，不会进入最终镜像

使用：
```
ARG VERSION
RUN echo $VERSION
```

构建时传参：
```
docker build --build-arg VERSION=2.0 .
```

ARG不等于ENV


##### 2.13、SHELL

**语法：** 
`SHELL ["bash","-c"]`

改变默认shell：
- 默认：/bin/sh -c
- 影响后续RUN


##### 2.14、USER
`USER root`
`USER appuser`

**特点：** 影响后续RUN/CMD，安全加固常用

##### 2.15、ONBUILD

作用：作为父镜像被触发

例如，我们在父镜像的Dockerfile里编写：
```
FROM grandfather
ONBUILD echo "hello!"
```

然后将该镜像构建好以后，再构建子镜像
```
FROM father
```

构建子镜像的时候，就会打印出`hello!`。



**三、镜像构建-docker build**

基本命令

```
docker build .     
```

在这里`.`代表当前目录，指定Dockerfile的路径PATH构建上下文，docker只能访问上下文内的文件

常用参数：

**-t，--tag（给镜像命名）**

```
docker build -t name .
```

或者更完整的:
```
docker build -t name:tag .
```

**-f（指定Dockerfile）**

dockerfile文件名为test

```
docker build -f test .

```


**四、Dockerfile示例**

1、创建目录，存放dockerfile使用的文件
2、在此目录创建dockerfile文件
3、在此目录使用docker build创建镜像
4、使用创建的镜像启动容器

准备启动文件

```
vim apache-run.sh
```

```
#!/bin/bash
rm -rf /var/run/apache2/*
exec /usr/sbin/apachectl -D FOREGROUND
```

准备网页测试文件

```
vim index.html
```

```
hello! this is a test page!
```


**Dockerfile**

```
# 1. 基础镜像
FROM ubuntu:22.04

# 2. 镜像元信息（替代 MAINTAINER）
LABEL maintainer="rains"

# 3. 拷贝网页文件
COPY apache-run.sh /apache-run.sh
COPY index.html /var/www/html/index.html

# 4. 安装 httpd
RUN apt update && \
	apt install -y apache2 && \
	rm -rf /var/lib/apt/lists/*

# 5. 赋予脚本执行权限
RUN chmod +x /apache-run.sh

# 6. 声明端口
EXPOSE 80

# 7. 设置工作目录（可选，但语义清晰）
WORKDIR /

# 8. 容器启动命令
CMD ["/bin/bash", "/apache-run.sh"]

```


构建镜像


![](assets/img/Pasted image 20251224023337.png)

镜像构建成功

运行容器检验

```
docker run -d -p 80:80 --name apache-test ubuntu-apache:v1
```

```
curl http://localhost
```


![[Pasted image 20251224023818.png]]









