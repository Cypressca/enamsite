---
title: 在Arch Linux上安装博通BCM4360网卡驱动
published: 2025-12-20
description: '博通BCM4360网卡驱动在最小化Linux下的安装流程。'
image: 'https://img.enamht.site/posts/1775049668/cover.jpg'
tags: [Linux]
category: '技术分享'
draft: false 
lang: ''
---

# 在Arch Linux上安装博通BCM4360网卡驱动

最近想在2015款MacBook Air上安装Debian 13，但是无论如何也没办法连上网。后来发现这台机器用的网卡是Broadcom臭名昭著的BCM4360，只能用闭源驱动broadcom-wl。正好也想试试Arch Linux，因此本文就在Arch Linux下安装这个网卡的驱动。本文并不保证操作过程中一定正确不出问题，详细内容请参考[Broadcom 无线 - ArchWiki ](https://wiki.archlinux.org.cn/title/Broadcom_wireless#broadcom-wl)。

## 准备

> [!NOTE]
>
> 相比于最小化安装的Debian 13，Arch Linux的好处就是在live环境可以联网，并且往系统里面直接安装软件。所以，如果是Debian的最小化安装，有可能需要另一台能联网的电脑，并用它来下载非自由固件，然后用U盘拷过去安装。

可能需要准备：安装Arch Linux的所有设备（例如写入Arch Linux镜像的U盘），一部有足够流量的手机（在这里使用的是iPhone），一根数据线以连接电脑和手机。

## 安装Arch Linux

安装Arch Linux的过程在此不做赘述。如果有需要可以参考[archlinux 简明指南](https://arch.icekylin.online/)、[安装ArchLinux](https://github.com/SHORiN-KiWATA/ShorinArchExperience-ArchlinuxGuide/wiki/安装ArchLinux)与[安装指南 - ArchWiki ](https://wiki.archlinux.org.cn/title/Installation_guide)。

由于Arch Linux没有BCM4360的驱动，因此无法通过无线网进行安装。因此可以使用手机进行有线网连接。以iOS 26为例，打开设置 > 个人热点 > 允许其他人加入后，打开蜂窝网络，将手机用数据线连接电脑，在手机上选择“信任”后，输入密码，即可实现有线连接。使用`ip a`或`ping baidu.com`来检验网络连接。

对于iPhone，在安装完成之后要在live环境下安装`ifuse`包。

```shell
pacstrap -S /mnt infuse
```

由于新系统中没有`infuse`包，除非在live环境下完成无线网的配置，否则为了保证在新系统中能够正常联网，有必要提前安装该依赖。

> [!NOTE]
>
> 没有用过Android设备进行有线网连接，但是根据[Broadcom 无线 - ArchWiki ](https://wiki.archlinux.org.cn/title/Broadcom_wireless#broadcom-wl)的说法，Android设备有线网共享似乎不需要额外的依赖。

## 安装驱动

Arch Linux正常安装完成之后，就可以安装网卡驱动了。

首先确认网卡确实为BCM4360。

```shell
lspci -vnn | grep -i broadcom
```

如果为BCM4360，此时应该没有**无线**接口。

```shell
ip link show
```

此时端口名称应该是没有以“wl”开头的接口。

BCM4360网卡使用的是broadcom-wl驱动。这种驱动有两种变体：broadcom-wl与broadcom-wl-dkms。本文使用dkms变体。

### 安装yay

由于broadcom-wl是不开源的，因此只能通过AUR获取。本文采用`yay`进行安装。首先添加archlinuxcn源。打开pacman配置文件。

```shell
sudo vim /etc/pacman.conf
```

在文件底部写入任意一个镜像源。

```shell
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch # 中国科学技术大学开源镜像站
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch # 清华大学开源软件镜像站
Server = https://mirrors.hit.edu.cn/archlinuxcn/$arch # 哈尔滨工业大学开源镜像站
Server = https://repo.huaweicloud.com/archlinuxcn/$arch # 华为开源镜像站
```

确保网络连接正常。更新源并安装签名与yay。

```shell
sudo pacman -Sy
sudo pacman -S archlinuxcn-keyring 
sudo pacman -S yay
```

### 安装驱动

安装网卡驱动与必要的依赖。

```shell
sudo pacman -S dkms linux-headers broadcom-wl-dkms
```

如果没有问题，这一步完成后驱动应该已经安装完成。

### 加载驱动

为了防止broadcom-wl驱动与其他驱动冲突，先卸载其他的驱动。

```shell
sudo modprobe -r b43 b43legacy ssb bcma #可以查看所有网卡驱动后卸载
```

> [!NOTE]
>
> 如果觉得可能是冲突的问题，还可以将其他驱动屏蔽。编辑配置文件：
>
>  ```shell
>  sudo nano /etc/modprobe.d/blacklist.conf
>  ```
> 
>  屏蔽有可能会出现冲突的驱动，例如：
> 
>  ```shell
>  blacklist b43
>  blacklist b43legacy
>  blacklist ssb
>  blacklist bcma
>  blacklist wl
>  ```

现在可以加载新驱动了。

```shell
sudo modprobe wl
```

一般来说我们希望开机自动加载驱动，因此我们添加wl到配置文件中。

```shell
echo "wl" > /etc/modules-load.d/wl.conf
```

### 检查驱动

检查模块是否正确加载。

```shell
lsmod | grep wl
```

如果出现`wl`，说明模块已经正确加载。

检查无线网口。

``` shell
ip link show
```

如果这个时候多出一个无线接口，那么说明驱动已经正确加载了。可以使用`nmcli`工具进行网络连接。如果暂时不能联网，可以尝试重启。
