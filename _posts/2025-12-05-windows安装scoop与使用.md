---
title: windows10安装scoop与基本使用
date: 2025-12-05 10:00:00 +0800
categories: [Windows]
tags: [scoop]
---


scoop可以方便我们在Windows搭建各种环境，节省手动下载、配置的功夫，节约时间成本。

按顺序执行以下命令

1. 设置powershell的执行策略，不然不能运行程序
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

2. 下载安装scoop的脚本
```
irm get.scoop.sh -outfile 'install.ps1'
```

3. 指定scoop安装目录
```
.\install.ps1 -ScoopDir 'C:\Scoop'
```

4. 补充内容，可略

如果想用管理员身份安装scoop(我本人用的这种)
把第三步命令换成如下
```
.\install.ps1 -RunAsAdmin -ScoopDir 'C:\Scoop'
```

到这里scoop就安装好了，加下来进行一些scoop的基本设置

![windows安装scoop命令](/assets/img/Pasted image 20251124182506.png)

5. 添加bucket：extras和versions

但是需要先安装git
```
scoop install git
```

```
scoop bucket add extras
scoop bucket add versions
```

然后就可以安装各种语言环境和软件了
例如安装python只需要执行如下命令，该命令默认安装最新版python
```
scoop install python
```

还可以精准安装特定的python版本，其他语言同样如此。其他用法自行了解。



### scoop常用命令

1. 搜索软件
```
scoop search <app_name>
```

2. 安装软件
```
scoop install <app_name>
```

3. 查看软件信息
```
scoop info <app_name>
```

4. 查看已安装软件
```
scoop list
```

5. 卸载软件，-p删除配置文件
```
scoop uninstall <app_name>
```

6. 更新scoop自身和软件列表
```
scoop update
```

7. 更新指定的某个软件
```
scoop update <app_name>
```

8. 更新所有已安装软件
```
scoop update *
```

9. 检测scoop的问题，并获取解决建议
```
scoop checkup
```

10. 查看命令帮助
```
scoop help
```

11. 查看命令帮助说明
```
scoop help <command>
```

12. 删除所有安装宝缓存
```
scoop cache rm *
```

13. 删除所有软件旧版本
```
scoop cleanup *
```

14. 删除所有软件的旧版本并清除安装包缓存
```
scoop cleanup -k **
```

15. scoop代理设置

scoop默认使用系统代理，如果想自行指定代理，使用以下命令，但是只支持http协议，设置完查看代理使用`scoop config proxy`.

```
scoop config proxy 127.0.0.1:7890
```

取消代理
```
scoop config rm proxy
```

