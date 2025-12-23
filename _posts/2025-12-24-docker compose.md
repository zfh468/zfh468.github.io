**一、什么是docker compose**

`compose`是docker官方的开源项目，负责实现docker容器集群的快速编排，源代码在<https://github.com/docker/compose>

Dockerfile可以很方便的让用户定义一个单独的应用容器，但是在实际应用场景里，经常需要多个容器互相配合来完成某项任务。例如web服务本身，往往需要加上数据库容器，甚至负载均衡器，比如lnmp服务

`Compose`就是负责这个事情的，它允许用户通过一个单独的`docker-compose.yml`模板文件，以`yaml格式`来定义**一组关联的容器为一个项目**。


Compose中有两个重要概念：

**服务 service** ：
一个应用的容器，实际上可以包括若干个运行相同镜像的容器实例

**项目 project** ：
由一组关联的应用容器组成的一个完整业务单元，在docker-compose.yml中定义

现在的docker安装好以后就可以使用compose了

**二、命令**

**2.1、命令帮助**

```
docker compose --help
```

**帮助说明**

```
# Docker Compose 中文速查手册

#1、 基本提示
- -h 简写已弃用，请使用：
docker compose --help

---

#2、 用法（Usage）
docker compose [选项] 命令
# 使用 Docker 定义并运行多容器应用

---

#3、 常用选项（Options）
--all-resources             包含所有资源，即使未被服务使用
--ansi <never|always|auto> 控制是否输出 ANSI 控制字符，默认 auto
--compatibility             启用向后兼容模式
--dry-run                   演练模式执行，不真正执行
--env-file <文件>           指定替代的环境变量文件
-f, --file <文件>           指定 Compose 配置文件（可多个）
--parallel <整数>           最大并行度，-1 表示无限制，默认 -1
--profile <名称>            指定要启用的 profile
--progress <auto|tty|plain|json|quiet> 设置构建进度输出格式
--project-directory <目录>   指定项目工作目录（默认：第一个 Compose 文件所在路径）
-p, --project-name <名称>   指定 Compose 项目名称

---

#4、 管理类命令（Management Commands）
bridge    将 compose 文件转换为另一种模型（高级用途）

---

#5、 常用命令（Commands）
attach    连接到运行中服务容器的标准输入/输出/错误流
build     构建或重建服务镜像
commit    将容器当前状态保存为新镜像
config    解析、校验并输出规范化 compose 文件
cp        容器与本地文件系统之间复制文件/目录
create    创建容器但不启动
down      停止并删除容器、网络等
events    实时接收容器事件
exec      在运行中的容器里执行命令
export    导出容器文件系统为 tar 包
images    列出 compose 项目使用的镜像
kill      强制停止服务容器
logs      查看容器日志输出
ls        列出正在运行的 compose 项目
pause     暂停服务
port      查看端口映射
ps        列出容器状态
publish   发布 compose 应用
pull      拉取服务镜像
push      推送服务镜像
restart   重启服务容器
rm        删除已停止的服务容器
run       在服务上运行一次性命令
scale     调整服务实例数量
start     启动服务
stats     实时显示容器资源使用情况
stop      停止服务
top       显示容器运行进程
unpause   恢复已暂停的服务
up        创建并启动容器（最常用命令）
version   显示 Docker Compose 版本
wait      阻塞等待容器停止
watch     监控构建上下文文件变化，自动刷新容器

---

#6、 提示
docker compose <命令> --help
# 查看某个命令的详细帮助信息

```

 
 
 **三、常用 Docker Compose 场景:**

```

场景               命令示例                                   说明
创建并启动服务      docker compose up -d                        后台启动所有服务
停止并删除          docker compose down                          安全清理容器和网络
查看日志            docker compose logs -f web                   实时跟踪 web 服务日志
进入容器            docker compose exec web bash                 用于调试或运维操作
更新镜像            docker compose pull && docker compose up -d  镜像更新后重启服务
扩容服务            docker compose up -d --scale web=3          将 web 服务扩容到 3 个实例
查看资源占用        docker compose stats                         监控 CPU/内存/网络
拷贝文件            docker compose cp web:/app/logs ./logs      下载容器web的日志或配置，从容器的/app/logs目录复制到本地的/logs目录
```


**四、示例**

compose官方文档<https://docs.docker.com/reference/compose-file/#compose-file-structure-and-examples>，可参考写模板文件


**compose的使用**
1. 创建一个目录并进入
2. 创建一个python应用
3. 创建requirements.txt文件，里面是需要安装的python包
4. 创建Dockerfile文件
5. 创建docker-compose.yml文件
6. 使用compose构建并运行应用程序
7. 测试服务，浏览器访问ip:5000，每刷新一次就+1


```
mkdir test && cd test
```


app.py
```
import time
import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)

```

requirements.txt

```
flask
redis
```

Dockerfile

```
FROM python 
ADD . /code 
WORKDIR /code 
RUN pip install -r requirements.txt 
CMD ["python", "app.py"]
```

这将告诉Docker：

1. 从 Python 开始构建镜像，记得先拉取python镜像
2. 将当前目录 . 添加到 /code 镜像中的路径     
3. 将工作目录设置为 /code    
4. 安装 Python 依赖项     
5. 将容器的默认命令设置为 python app.py


docker-compose.yml

```
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/code

  redis:
    image: "redis"

```

此 Compose 文件定义了两个服务，web 和 redis 

该web服务：
1. 使用从 Dockerfile 当前目录中构建的镜像         
2. 将容器上的公开端口 5000 转发到主机上的端口 5000 
3. 我们使用 Flask Web 服务器的默认端口 5000         
4. 该 redis 服务使用从 Docker Hub 中提取的公共 Redis 映像


**使用compose后台构建并运行应用程序**
```
docker compose up -d
```



```
➜  test docker compose up -d
[+] Building 14.0s (11/11) FINISHED                                                                                                                                                          
 => [internal] load local bake definitions                                                                                                                                              0.0s
 => => reading from stdin 311B                                                                                                                                                          0.0s
 => [internal] load build definition from Dockerfile                                                                                                                                    0.0s
 => => transferring dockerfile: 140B                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:latest                                                                                                                        0.0s
 => [internal] load .dockerignore                                                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                                                         0.0s
 => [internal] load build context                                                                                                                                                       0.0s
 => => transferring context: 981B                                                                                                                                                       0.0s
 => [1/4] FROM docker.io/library/python:latest                                                                                                                                          0.0s
 => [2/4] ADD . /code                                                                                                                                                                   0.0s
 => [3/4] WORKDIR /code                                                                                                                                                                 0.0s
 => [4/4] RUN pip install -r requirements.txt                                                                                                                                          13.6s
 => exporting to image                                                                                                                                                                  0.2s 
 => => exporting layers                                                                                                                                                                 0.2s 
 => => writing image sha256:1dd9227a2b3cd058e98b3ad7640e7cb05b0e05068dee31686016bec65acbf60e                                                                                            0.0s 
 => => naming to docker.io/library/test-web                                                                                                                                             0.0s 
 => resolving provenance for metadata file                                                                                                                                              0.0s 
[+] Running 4/4                                                                                                                                                                              
 ✔ web                     Built                                                                                                                                                        0.0s 
 ✔ Network test_default    Created                                                                                                                                                      0.0s 
 ✔ Container test-web-1    Started                                                                                                                                                      0.3s 
 ✔ Container test-redis-1  Started                                                                                                                                                      0.3s 
➜  test docker compose ps   
NAME           IMAGE      COMMAND                   SERVICE   CREATED          STATUS          PORTS
test-redis-1   redis      "docker-entrypoint.s…"   redis     12 seconds ago   Up 12 seconds   6379/tcp
test-web-1     test-web   "python app.py"           web       12 seconds ago   Up 12 seconds   0.0.0.0:5000->5000/tcp, [::]:5000->5000/tcp
```

![[Pasted image 20251224041622.png]]


访问虚拟机ip:5000

![[Pasted image 20251224041748.png]]

