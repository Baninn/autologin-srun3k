# autologin-srun3k 
# 运行环境:
校园网环境:河南工业大学haut.edu.cn srun3k登陆 

   (其它学校使用srun3K方案 应该 类似可能需要修改认证服务器地址)

运行环境:openwrt chaos_calmer 15.05.1 （Lenovo Y1 测试通过 ）

   至少需要安装 python3 (python3-base python3-codecs python3-email ), coreutils-nohup
         
# 如何使用：
## 准备环境
这里是Y1 OpenWrt Chaos Calmer 15.05原版固件
<u>https://oldwiki.archive.openwrt.org/toh/lenovo/lenovo_y1_v1</u>

刷完固件建议安装sftp 
```
opkg update
opkg install vsftpd openssh-sftp-server
```
由于python太大，路由器肯定放不下，所以要放到外接U盘里
挂载U盘方法如下
```
   opkg update
   opkg install kmod-usb-core
   opkg install kmod-usb2                #安装usb2.0 
   opkg install kmod-usb-ohci            #安装usb ohci控制器驱动
   opkg install kmod-usb-storage         #安装usb存储设备驱动
   opkg install kmod-fs-ext3             #安装ext3分区格式支持组件
   opkg install kmod-fs-vfat             #挂载FAT
   opkg install ntfs-3g                  #挂载NTFS
   opkg install mount-utils              #挂载卸载工具
   opkg install block-mount
   opkg install luci-app-samba           #SAMBA网络共享服务
   /etc/init.d/samba enable              #启用并开始SAMBA共享
   /etc/init.d/samba restart
```
重启路由器 登陆路由器设置界面 可以看到多了*挂载点*和*网络共享*两个选项
SSH到路由器，打开/etc/hotplug.d/block/10-mount文件(如果不存在请新建)
修改为如下内容
```
   \#!/bin/sh
   \# Copyright (C) 2009 OpenWrt.org  (C) 2010 OpenWrt.org.cn
   blkdev=`dirname $DEVPATH`
   if [ `basename $blkdev` != "block" ]; then
   ​    device=`basename $DEVPATH`
   ​    case "$ACTION" in
   ​        add)
   ​                mkdir -p /mnt/$device
   ​                \# vfat & ntfs-3g check
   ​                if  [ `which fdisk` ]; then
   ​                        isntfs=`fdisk -l | grep $device | grep NTFS`
   ​                        isvfat=`fdisk -l | grep $device | grep FAT`
   ​                        isfuse=`lsmod | grep fuse`
   ​                        isntfs3g=`which ntfs-3g`
   ​                else
   ​                        isntfs=""
   ​                        isvfat=""
   ​                fi 
   ​                \# mount with ntfs-3g if possible, else with default mount
   ​                if [ "$isntfs" -a "$isfuse" -a "$isntfs3g" ]; then
   ​                        ntfs-3g -o nls=utf8 /dev/$device /mnt/$device
   ​                elif [ "$isvfat" ]; then
   ​                        mount -t vfat -o iocharset=utf8,rw,sync,umask=0000,dmask=0000,fmask=0000 /dev/$device /mnt/$device
   ​                else
   ​                        mount /dev/$device /mnt/$device
   ​                fi
     if [ -f /dev/${device}/swapfile ]; then
      mkswap /dev/${device}/swapfile
      swapon /dev/${device}/swapfile
     fi
   ​                ;;
   ​        remove)
     if [ -f /dev/${device}/swapfile ]; then
      swapoff /dev/${device}/swapfile
     fi
   ​                umount /dev/$device
   ​                ;;
   ​    esac
   fi
```
这段脚本可实现自动挂载,但不能开机自动挂载
在启动项中添加
```
mount -t ntfs-3g /dev/sda1 /mnt/sda1
```
实现开机自动挂载U盘
进入*网络共享*中，添加共享目录例如/mnt/sda1，权限为777(即完全访问)。
就可以从资源管理器访问U盘了
若要把安装包安装在U盘里，需要设置环境变量:
编辑/etc/profile
添加两行
```
export LD_LIBRARY_PATH="/mnt/usb/optware/usr/lib:/mnt/usb/optware/lib"
export PATH=/usr/bin:/usr/sbin:/bin:/sbin:/mnt/usb/optware/usr/bin:/mnt/usb/optware/usr/sbin
```
让修改后的profile立即生效
```
source /etc/profile
```

还要修改一下/etc/opkg.conf
在最前面添加
```
dest root /
dest ram /tmp
lists_dir ext /var/opkg-lists
option overlay_root /overlay
dest usb /mnt/sda1/opkg

arch all 100
arch ramips_24kec 200
arch ramips 300
arch mips 400
arch unkown 500
```


顺便写一下如何换源
顺便装一下https
参考 https://github.com/Entware/Entware/wiki/Using-HTTPS-for-opkg

```
opkg install wget ca-certificates
```
固件的默认源
```
src/gz barrier_breaker_base http://downloads.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/base
src/gz barrier_breaker_luci http://downloads.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/luci
src/gz barrier_breaker_management http://downloads.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/management
src/gz barrier_breaker_packages http://downloads.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/packages
src/gz barrier_breaker_routing http://downloads.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/routing
src/gz barrier_breaker_telephony http://downloads.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/telephony
```
USTC反代 http://mirrors.ustc.edu.cn/
```
src/gz barrier_breaker_base https://openwrt.proxy.ustclug.org/chaos_calmer/15.05/ramips/mt7620/packages/base
src/gz barrier_breaker_luci https://openwrt.proxy.ustclug.org/chaos_calmer/15.05/ramips/mt7620/packages/luci
src/gz barrier_breaker_management https://openwrt.proxy.ustclug.org/chaos_calmer/15.05/ramips/mt7620/packages/management
src/gz barrier_breaker_packages https://openwrt.proxy.ustclug.org/chaos_calmer/15.05/ramips/mt7620/packages/packages
src/gz barrier_breaker_routing https://openwrt.proxy.ustclug.org/chaos_calmer/15.05/ramips/mt7620/packages/routing
src/gz barrier_breaker_telephony https://openwrt.proxy.ustclug.org/chaos_calmer/15.05/ramips/mt7620/packages/telephony
```
还有一个源
```
src/gz barrier_breaker_base https://archive.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/base
src/gz barrier_breaker_luci https://archive.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/luci
src/gz barrier_breaker_management https://archive.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/management
src/gz barrier_breaker_packages https://archive.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/packages
src/gz barrier_breaker_routing https://archive.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/routing
src/gz barrier_breaker_telephony https://archive.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/telephony
```
装python https://www.echoteen.com/openwrt-python.html

## 安装Python3

我们使用`opkg -d usb install xxx`即可将程序安装至`/mnt/sdb1/opkg`。

```
# 先需要安装libc，需要下载下来安装
## 建立文件夹
mkdir -p /mnt/sda1/opkg/src
cd /mnt/sda1/opkg/src
wget https://archive.openwrt.org/chaos_calmer/15.05/ramips/mt7620/packages/base/libc_0.9.33.2-1_ramips_24kec.ipk
## 安装libc，最好安装到根下
opkg install libc_0.9.33.2-1_ramips_24kec.ipk
 
# 接着安装Python3
opkg -d usb install libreadline
opkg -d usb install python3
opkg -d usb install python3-openssl
opkg -d usb install python3-base
opkg -d usb install python3-codecs
opkg -d usb install python3-email
opkg -d usb install coreutils-nohup

# 环境变量
export PATH=$PATH:/mnt/sda1/opkg/usr/bin
echo 'export PATH=$PATH:/mnt/sdb1/opkg/usr/bin' >> /etc/profile
 
# 别名
echo "alias opintall='opkg -d usb install'" >> /etc/profile
```
## 配置autologin-srun3k
下载至路由器端
```
wget https://github.com/ygqsgm/autologin-srun3k/raw/master/autologin.py    
```
修改 学号，密码

```
VIM autologin.py

nohup python3 autologin.py &
```

原理简介：

1.模拟校园网客户端发包(HTTP POST登陆，以及 UDP 心跳包)

2.通过读取/proc/net/arp 0x2 -1 获取在线人数>0时登陆

## 添加路由器启动项
启动项一直添加不成功，试了好几天

后来看系统日志，找不到python

去网上查，意思是开机的时候，python的环境变量还没有加载

在前面加上PATH=$PATH:/mnt/sda1/opkg/usr/bin

然后在输出的脚本里看到python3: can't load library 'libpython3.4.so.1.0'

然后又查了一下，是lib也没有加载

添上export LD_LIBRARY_PATH=/mnt/sda1/opkg/usr/lib

最终启动项为
```
mount -t ntfs-3g /dev/sda1 /mnt/sda1
PATH=$PATH:/mnt/sda1/opkg/usr/bin
export LD_LIBRARY_PATH=/mnt/sda1/opkg/usr/lib
sleep 6 && nohup python3 -u /mnt/sda1/autologin.py >/dev/null 2>&1 &
```
完成，下线重启试试吧
