---
layout: post
title: "把Ubuntu升级为Debian"
date: 2021-12-29 09:26:14 +0800
categories: 原创
---

这题目听着很有意思。

自从在知乎上听说Ubuntu的发布是基于Debian Testing开始，我就有了这个想法。因为根据这个发布模型来看，Ubuntu的package总是不早于Debian Testing。那么理论上，如果我把apt的源全部替换为Debian Testing的源是不应该存在问题的。`apt upgrade`之后，原有的package要么保持不变，要么会被更新。

对这个问题的探索也来自一个现实的问题。几个月后，Ubuntu 22.04 LTS就要发布了。在之前12.04到14.04，和14.04到16.04两次升级，我都遇到了GUI层面的崩溃，所以这次必须有所准备。当时一直没有研究过如何拯救，都是直接重装了事。这次有了KVM虚拟机，也恰好有机会试错学习。

# 配置Debian Testing源

这个很常规。将原来的`/etc/apt/sources.list`做一个备份，然后将文件完整替换一下。

```
deb http://mirrors.aliyun.com/debian/ testing main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ testing main non-free contrib
deb http://mirrors.aliyun.com/debian-security/ testing-security main
deb-src http://mirrors.aliyun.com/debian-security/ testing-security main
deb http://mirrors.aliyun.com/debian/ testing-updates main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ testing-updates main non-free contrib
deb http://mirrors.aliyun.com/debian/ testing-backports main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ testing-backports main non-free contrib
```

我这边试用过tsinghua、aliyun、huaweicloud三个源。商业公司的源明显比大学的快，其中华为的源最快。就这一点，给华为点赞。

要想用huaweicloud的源，只需要把上面的`http://mirrors.aliyun.com`替换成`https://repo.huaweicloud.com`即可。具体注意事项，还请参考[华为云镜像](https://mirrors.huaweicloud.com/home)

替换完成之后，还需要更新一下。

```
sudo apt update --allow-insecure-repositories
```

因为源发生了剧烈变更，无法完成校验。只能追加参数，忽略校验。


# 启动至命令行界面

在GUI下，替换apt源，并直接`apt upgrade`并不能成功，因为Ubuntu和Debian在GUI层面有很多不同。直接替换，会在再次启动时在完成用户登录的时候界面发生崩溃（登录音乐都可以听得到）。

所以此时应该让系统启动在命令行界面中，保证后续还能用命令行来操纵系统。

启动至命令行的方法是修改grub。首先以root权限打开`/etc/default/grub`文件。

参考命令：

```
sudo gedit /etc/default/grub  # 喜欢vim也可以把gedit换成vim
```

然后：
1. 把`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`这一行注释掉
2. 把`GRUB_CMDLINE_LINUX=""`改成`GRUB_CMDLINE_LINUX="text"`
3. 把`GRUB_TERMINAL="console"`这一行释放出来

接下来，需要让这个配置的修改生效。

```
sudo update-grub
sudo systemctl set-default multi-user.target
```

此时，重启系统即可进入命令行界面。记得把自己的用户名记下来:-)

# 升级package依赖

进入命令行界面后，输入用户名和密码完成登录。

这时就可以升级依赖。不过，为了避免无效升级，需要先卸载桌面环境。如果使用的是Ubuntu 20.04的主流版本，那么需要卸载的就是GNOME。如果是其他桌面环境，请自行查找相关资料。

```
sudo apt remove --purge gnome*
```

然后就可以正常升级package依赖了。

```
sudo apt upgrade
```

在升级过程中，会有报错。提示说某个依赖A想要替换某文件B，但是因为被其他package C依赖，所以不成功。这里需要把这个C去掉。

```
sudo dpkg -P <C>
```

然后继续执行

```
sudo apt update
sudo apt --fix-broken install
sudo apt upgrade
```

反复执行上述命令，直至upgrade完全成功。

此时，如果我们重启一次系统，调用`hostnamectl`就会发现系统完全成了Debian。但是重启过程中会出现一些小问题。我遇到有两点：
1. `/usr/bin/lsb_release`报错，是由于字符串格式不对。看下这个脚本内容，都是Python代码，完成一些console输出性的工作。所以干脆把main方法注释掉了，改成pass。
2. `/etc/profile/`里有一个文件调用不到某个脚本了，我也干脆把这一行注释掉了（生冷硬）。

# 安装桌面环境

命令行界面梳理干净以后就可以安装Debian的桌面环境了。

```
sudo apt install task-gnome-desktop
```

执行这个命令，会报错。因为依赖的组件版本号对不上，需要的依赖是debian体系的，实际安装的却是ubuntu体系的。因此得把报错的依赖删掉，让apt自行按需安装。

```
sudo apt remove --purge <D>
```

如此反复几次，直至`task-gnome-desktop`安装成功。

如果是笔记本电脑，最好安装一下Debian的相关配置。

```
sudo tasksel install laptop
```

最后，要让系统启动后进入GUI界面。

```
sudo systemctl set-default graphical.target
```

重启，大功告成！

# 最后

重启成功进入GNOME界面后，还有一些收尾工作要做。

```
sudo apt update
sudo apt full-upgrade # 在Debian Testing的Best Practice中要求谨慎使用
sudo apt autoremove
```

这是因为完成GUI依赖替换以后，原有的Ubuntu的依赖都不需要了，所以可以被清理掉。还有少数依赖因为莫种原因被限制升级了，如果使用中遇到什么问题，可以谨慎升级（full-upgrade）。

# 说明

以上流程在虚拟机测试通过，升级后的Debian系统可以正常浏览网页，执行常见shell命令。

但还没有在物理设备上验证。留作明年升级失败的应急方案吧。

# 参考资料

- [阿里云 开发者社区>镜像站>debian](https://developer.aliyun.com/mirror/debian)
- [Ubuntu(Debian):apt-get:处理repository数字签名无效、过期、没有签名:即如何强制apt-get update?](https://www.cnblogs.com/jinzhenshui/p/13446279.html)
- [How to Boot Ubuntu 20.04 into Text / Command Console](https://ubuntuhandbook.org/index.php/2020/05/boot-ubuntu-20-04-command-console/)
- [Debian Reference - 07.GUI System](https://www.debian.org/doc/manuals/debian-reference/ch07.en.html)
- [Installing GNOME Desktop Environment on Debian 10 Minimal Server](https://linuxhint.com/install_gnome_debian_10_minimal_server/)

