
#### 1、准备

kvm创建Windows虚拟机比Linux复杂一些

准备windows10的iso镜像

windows默认没有VirtIO驱动

如果不加载驱动，Windows 安装程序会找不到你创建的虚拟磁盘，因为它是 `virtio` 总线的


红帽官方提供的 VirtIO总线驱动iso
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/

virtio-win-0.1.285-1.noarch.rpm

```
rpm -ivh virtio-win-0.1.285-1.noarch.rpm
```

安装的驱动文件路径
```
ls -l /usr/share/virtio-win/
```





#### 2、创建Windows虚拟机

```
virt-install \
--name win10 \
--memory 4096 \
--vcpus 2 \
--disk path=/var/lib/libvirt/images/win10.qcow2,size=50,bus=virtio \
--os-variant win10 \
--network network=default,model=virtio \
--cdrom /cache/iso/Windows.iso \
--disk path=/usr/share/virtio-win/virtio-win.iso,device=cdrom \
--graphics vnc,listen=0.0.0.0,port=5999,password=123456 \
--boot cdrom,hd \
--noautoconsole
```


#### 3、 安装过程中的关键步骤


![[Pasted image 20260310135958.png]]

当通过 VNC 连接到虚拟机并开始安装时，会遇到“找不到驱动器”的情况。按以下步骤操作：

1. **加载驱动**：在选择安装位置界面，点击 **“加载驱动程序 (Load Driver)”**。
    
2. **浏览路径**：选择挂载的那个 `virtio-win` 光盘驱动器。
    
3. **点进去**：
    
    - **磁盘驱动**：选择 `viostor\w10\amd64`目录。
    ![[Pasted image 20260310140529.png]]
		按照上面的路径点击后，列表里会出现一个 **"Red Hat VirtIO SCSI controller"** 之类的选项，选中它点击下一页

![[Pasted image 20260310140718.png]]
		稍等片刻，出现50GB磁盘驱动器，继续安装即可，不用多说




**联网问题解决**

方法1

- 在刚才加载磁盘驱动的界面，点击 **“加载驱动程序 (Load Driver)”**。
    
- 再次点击 **“浏览”**，进入 `virtio-win` 光盘。
    
- **路径**：`NetKVM` -> `w10` -> `amd64`。
    
- 选择后点击确定，加载 **Red Hat VirtIO Ethernet Adapter**。
    




方法2

或者也可以选择没有internet连接
继续执行受限设置
![[Pasted image 20260310144410.png]]


    
- **一键安装**：
    
    - 在 Windows 里打开 `E:` 盘（virtio-win 光盘）。
        
    - 双击运行 **`virtio-win-gt-x64.exe`** 或 **`virtio-win-guest-tools.exe`**。
        
    - 一路点击“Next”，会一次性帮你装好网卡、显卡加速、内存优化等所有驱动。


