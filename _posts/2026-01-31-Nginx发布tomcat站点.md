之前访问tomcat站点都是类似http://ip/jpress这样的URL，比较麻烦，可以让tomcat站点通过nginx发布。

发布方法有URL重写和反向代理。这里只写反向代理的内容。


**一、部署tomcat网站**

在一台Linux主机部署两个tomcat站点。这里要用到多实例的内容。

tomcat1
```
[root@192 webapps]# cd /opt/tomcat1/webapps/
[root@192 webapps]# mkdir haha1
[root@192 webapps]# echo "this is a page!" > haha1/index.html
```

tomcat2
```
[root@192 webapps]# cd /opt/tomcat2/webapps/
[root@192 webapps]# mkdir haha2
[root@192 webapps]# echo "this is b page!" > haha2/index.html
```

安装nginx
```
dnf install -y nginx
```

```
vim /etc/nginx/nginx.conf
```

nginx配置文件内容
```
http {

    server {
        listen 80;
        server_name www.a.com;

        location / {
            proxy_pass http://127.0.0.1:8080/haha1/;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

    server {
        listen 80;
        server_name www.b.com;

        location / {
            proxy_pass http://127.0.0.1:8081/haha2/;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

如果不使用域名的配置
```
server {
    listen 80;
    server_name 192.168.1.7;

    location /haha1/ {
        proxy_pass http://127.0.0.1:8080/haha1/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /haha2/ {
        proxy_pass http://127.0.0.1:8081/haha2/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```


重启nginx
```
systemctl restart nginx
```

linux修改待测试机的hosts文件（不使用域名的话没必要）

如果用windows也要修改C:\Windows\System32\drivers\etc\hosts文件

```
vim /etc/hosts
```

```
192.168.1.7 a.com www.a.com
192.168.1.7 b.com www.b.com
```


关闭selinux，nginx默认不能访问8080/8081这种后端端口

**二、打开浏览器直接访问站点测试**

浏览器分别访问设置好的域名
http://www.a.com
http://www.b.com

查看是否能访问到对应的网站内容，能看到说明实验成功
![[Pasted image 20260205022937.png]]



