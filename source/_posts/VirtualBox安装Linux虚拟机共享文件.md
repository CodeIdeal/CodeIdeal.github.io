title: VirtualBox 安装虚拟机共享文件
date: 2016-11-15 15:01:06
---
#### VirtualBox 安装虚拟机共享文件

需要手动mount 上所共享的文件才可以在虚拟机中看见。

因为，当VirtualBox把文件夹共享过来后并没有挂载到虚拟机中，需要手动：

```bash
mount -t vboxsf <共享文件夹名> <链接到的目录>

eg：mount -t vboxsf share ~/share
```

将共享文件夹挂载至主机