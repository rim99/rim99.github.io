---
layout: post
title: "在Debian上使用KVM安装虚拟机"
date: 2021-12-28 16:04:03 +0800
categories: 原创
---

KVM原本是Sun提供的虚拟机API，最初实现在Solaris上。后来其他系统，例如FreeBSD、Linux均陆续实现了相同的API。

Linux系统的KVM实现位于内核层面，实现非常高效，效率损失在2%以下，因此在云计算服务商中的应用很广。可以说，前些年常见的VirtualBox、VMWare最终都会淹没在历史中。

# 预先检查

Linux的KVM目前只支持X86平台，也就是Intel的VT-x以及AMD的AMD-V虚拟技术。所以得先看看自己的CPU是否支持。

```
lscpu | egrep --color=auto 'vmx|svm'
```

如果返回结果有高亮输出，说明本机支持KVM。

# 安装用户态组件

KVM只是内核提供的一组API，可以虚拟CPU、内存等资源。但如果想要运行虚拟机，还需要用户态软件的参与。这就是QEMU。而QEMU并有提供图形化界面，所以还需要安装libvirt相关软件。这里选择的是virt-manager。

```
sudo apt install qemu-kvm virt-manager
```

刚刚安装完成时，virt-manager还不能立即使用，因为它依赖于libvirtd后台进程。

这一条命令可以确认该进程的状态。

```
sudo systemctl status libvirtd
```

如果没有启动，则需要执行

```
sudo systemctl start libvirtd
```

# 完成

剩下的操作就是在GUI中完成，和印象中VirtualBox里的操作差不多。下载想要的操作系统的ISO，启动安装就好啦。这下可以愉快的测试各种系统，还不担心破坏物理机系统了。


参考来源：
- [在Debian 10上安装KVM](http://linux.it.net.cn/m/view.php?aid=30189)
- [Kernel Virtual Machine - Main Page](https://www.linux-kvm.org/page/Main_Page)
- [QEMU和KVM的关系](https://zhuanlan.zhihu.com/p/48664113)
