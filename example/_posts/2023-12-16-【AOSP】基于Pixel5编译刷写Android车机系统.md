---
layout: post
description: > 
  本文介绍了使用Pixel5手机作为车载Android开发设备，从源码拉取，到系统编译，刷写的全流程记录。并且当时处于WSL子系统环境，并非全功能Ubuntu系统。
image: 
  path: /assets/img/blog/blogs_pixel.png
  srcset: 
    1920w: /assets/img/blog/blogs_pixel.png
    960w:  /assets/img/blog/blogs_pixel.png
    480w:  /assets/img/blog/blogs_pixel.png
accent_image: /assets/img/blog/blogs_pixel.png
excerpt_separator: <!--more-->
sitemap: false
---
# 基于Pixel5编译刷写Android车机系统
## Pixel5使用体验
去年2022年中的时候网购了一台Pixel5的库存机，现在闲置成备用机了。

之所以有这个在手机上跑车机的想法，是因为笔者是车机Android应用层开发，想着谷歌手机原生支持那么好，能不能整一个Google Automotive的车机系统上去跑跑，顺便还可以学习学习AOSP源码、系统编译、系统apk集成、权限管理、CarService服务等等。

一看官方网站居然还真有定制，而且目前恰好支持Pixel4a和Pixel5，另外还有Pixel6，但是是Experimental实验性的，拿6代设备的朋友整活有风险。

2024-12-31更新：目前已经支持到了Pixel8手机

废话不多说，开始正经的经验记录！

## 系统环境准备
首先最低硬盘控件需要准备300G，低于这个数就很危险了。

### 打开Windows功能
注意Google的AOSP开源项目，谷歌宣称其开发和调试均是在Ubuntu14上进行的。强烈建议开发者也需要使用Ubuntu系统进行AOSP源码的拉取和编译。

不想把自己电脑刷成Ubuntu系统的话，也可以使用windows上的wsl虚拟机，这个也是需要win10及以上可以使用，直接通过微软Microsoft应用商店搜索Ubuntu即可下载安装。注意安装之前要在控制面板的“程序和功能”里打开“windows子系统选项”，重启系统后生效。
### WSL迁移其他盘与空间扩展
安装完成后，进行简单的username用户名和password设置就可以进入系统了，啊，还是熟悉的terminal指令。然后下一步我们需要将这个子系统的位置从C盘移出去。

因为安装位置默认在C盘，而一份源码下载和编译后至少需要300G的空间，所以为了windows系统的流畅运行，我们最好不要将其挤在C盘，使用工具将其迁移到其他盘下面。为了完成这个操作，我们需要下载一个第三方工具 LxRunOffline，这个是由国人开发的 WSL 工具，其可以弥补官方工具的不足，比如说他可以实现将任何发行版的 Linux 以 WSL 形式安装到 Windows 10 中，增强 WSL 发行版管理功能，甚至可以实现 WSL 系统备份和恢复，这样无论是学习 Linux 还是进行开发工作都要比以往操作更为方便。
```
# 以管理员权限打开PowerShell，首先关闭wsl虚拟机
wsl --shutdown

#切到LxRunOfflin目录下，查看系统里wsl有哪些
.\LxRunOffline.exe list


#迁移wsl,需要十几分钟，完成后会生成虚拟硬件磁盘ext4.vhdx文件
.\LxRunOffline.exe move -n Ubuntu-20.04 -d f:\wsl_ubuntu20

#迁移完成,查看迁移后路径
.\LxRunOffline.exe get-dir -n Ubuntu-20.04
```
完成后还有一个问题，WSL默认只支持最大256G的硬盘空间，我们下载源码编译后很有可能就会超过256G，那么WSL就会报错，编译等操作也会中断。想要将WSL的最大硬盘空间突破这个限制，需要通过扩展 VHD 大小来解决：
```
#关闭wsl
wsl --shutdown

#查看wsl版本
 wsl -l -v
  NAME            STATE           VERSION
* Ubuntu-20.04    Stopped         2

#进入disk命令行
diskpart

#选择虚拟磁盘
DISKPART> Select vdisk file=f:\wsl_ubuntu20\ext4.vhdx

#查看VHD的详细信息
DISKPART> detail vdisk

#扩展vdisk空间，xxx为空间大小，以MB为单位，默认为256000,我拓展到了1000000
DISKPART> expand vdisk maximum=xxx

#退出DISKPART，进入wsl
DISKPART> exit
$wsl

#查看分区
$df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb        251G  991M  238G   1% /
tools           200G   53G  148G  27% /init
none            4.9G     0  4.9G   0% /dev
tmpfs           4.9G     0  4.9G   0% /sys/fs/cgroup
none            4.9G  4.0K  4.9G   1% /run
...

#在wsl中操作，使wsl知道磁盘大小限制已经更改
$sudo mount -t devtmpfs none /dev
# 将none挂载到/dev目录下，若返回'mount: /dev: none already mounted on /dev.'，可忽略
$mount | grep ext4
# 得到none挂载到/dev目录下的磁盘路径名
# 本句命令返还的信息 '/dev/sdX' 即为磁盘路径名,X可能是a,b,c等,xxx为前面分配的vhd大小，M为MB单位
$ sudo resize2fs /dev/sdb 1000000M
resize2fs 1.44.1 (24-Mar-2018)
Filesystem at /dev/sdb is mounted on /; on-line resizing required
old_desc_blocks = 32, new_desc_blocks = 123
The filesystem on /dev/sdb is now 256000000 (4k) blocks long.
# 重新查看分区配置
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb        961G 1000M  918G   1% /
tools           200G   53G  148G  27% /init
none            4.9G     0  4.9G   0% /dev
```
## WSL拉取同步Android源码
上面WSL移出C盘和硬盘空间扩展完成之后，Ubuntu环境准备完成，即可开始Android源码的拉取了，注意拉代码前一定要提前下载这些辅助工具，以免正式开始后缺工具，手忙脚乱。
### 代码拉取前的程序安装
注意不要习惯性的将Ubuntu换源阿里或者中科大，我们直接使用WSL上自带的默认软件源，否则有些官方工具的安装会产生链式依赖问题，在Ubuntu18及以上终端输入：
```
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig
```
另外别忘了安装java，后面编译，还有开发系统应用生成Android系统platform签名，需要用到java的keytool工具。
```
sudo apt install oracle-java8-installer
```
### WSL不可使用adb，刷机流程更改
使用wsl的话，我们虽然可以使用usbipd这个工具来配置，访问windows电脑连接的usb设备，但是不可以识别手机，也不可以在wsl上使用adb进行调试刷机。所以我最终采用的方案是Ubuntu编译，将编译产物同步到windows，再在windows上连接手机，最后进行设备刷写推送。
```
# 这个目录就是windows的文件夹在wsl的挂载同步，可以以此作为两个系统的文件同步区域
cd mnt/d/Pixel5

# 复制编译产物到Windows下的文件夹
cp -r /aaos/build/product/XXXX   /mnt/d/Pixel5
```
### 使用repo进行源码拉取同步
首先明确一点，Pixel 5手机其支持的车机版本只有一个，我们必须使用 Android 12，和build SP1A.210812.016.A1，对应AOSP分支为 android-12.0.0_r3

Android的AOSP源码使用repo来进行版本管理，repo是Google开发的用于管理Android版本库的一个工具，repo是使用Python对git进行了一定的封装，并不是用于取代git，它简化了对多个Git版本库的管理。用repo管理的版本库都需要使用git命令来进行操作。

下载repo工具：
```
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```
建议在home的个人文件夹下，建立放置源码的工作目录，一开机ls就是它了：
```
# 新建文件夹
mkdir aaos_on_phone
# 切换工作目录
cd aaos_on_phone
```
为了下载速度能拉满，我没有使用谷歌的官方仓库来拉取同步代码，而是改为使用清华大学的镜像网站，内容是相同的：清华大学开源软件镜像站 | [Tsinghua Open Source Mirror](https://mirrors.tuna.tsinghua.edu.cn/)打开后，可以看到第一个就是AOSP项目。

![tsinghua](/assets/img/blog/blogs_tsinghua.png){:width="600" height="300" loading="lazy"}

在新建好的工作目录下，使用如下命令通过repo工具拉取AOSP源码，笔者没有WI-FI，直接使用手机流量来硬刚的，大概需要70个G左右，耗时2小时。
```
# 初始化repo仓库，拉取某一个特定的分支
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-12.0.0_r3
# 开始同步拉取代码
repo sync
```
经过漫长的等待之后，打开工作目录，应该是下面的目录结构，因为我已经编译过，还加入了设备build文件，所以会多一点东西：

## 特定设备的二进制文件下载解包
源码拉取完成后，需下载特定设备的专有二进制文件和补丁程序，在如下网站找到对应设备与安卓版本的二进制包Nexus 和 Pixel 设备的驱动程序二进制文件 [ |  Google Play services  |  Google for Developers](https://developers.google.cn/android/drivers?hl=zh-cn#redfinsp1a.210812.016.a1)

对于Pixel 5，需要找到：适用于 Android 12.0.0 (SP1A.210812.016.A1) 的 Pixel 5 二进制文件。

下载完毕之后，将此文件copy到源码目录进行解压：
```
# 复制供应商映像和高通的驱动二进制文件到源码目录
cp mnt/d/Downloads/extract-google_devices-redfin.sh /home/stephen/aaos
cp mnt/d/Downloads/extract-qcom-redfin.sh /home/stephen/aaos

# 解压缩两个文件
curl --output - https://dl.google.com/dl/android/aosp/google_devices-redfin-sp1a.210812.016.a1-8813b219.tgz  | tar -xzvf -
tail -n +315 extract-google_devices-redfin.sh | tar -zxvf -

curl --output - https://dl.google.com/dl/android/aosp/qcom-redfin-sp1a.210812.016.a1-8d32b5b1.tgz | tar -xzvf -
tail -n +315 extract-qcom-redfin.sh | tar -xzvf -
```
## 开始编译源码
### WSL运行内存分配
源码和设备二进制文件准备完成后，可能有的朋友就按耐不住要开始编译了，其实还有很重要的一个步骤。

源码的编译是非常非常耗性能的，特别是内存，默认分配的是物理机一半的运行内存，对于编译源码是不太够的，所以我们要对WSL子系统进行一些特殊的性能配置，在个人用户文件夹下，新建一个  .wslconfig  文件，里面配置的字段含义可以参考微软官方文档：[WSL 中的高级设置配置 | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config#wslconfig)
```
[wsl2]

# Limits VM memory to use no more than 24 GB, this can be set as whole numbers using GB or MB
memory=24GB
# Sets the VM to use 6 virtual processors
processors=6
# Sets amount of swap storage space to 8GB, default is 25% of available RAM
swap=0
# Turn on default connection to bind WSL 2 localhost to Windows localhost
localhostForwarding=true

# 一些实验性的配置
# Enable experimental features
[experimental]
autoMemoryReclaim=gradual  
networkingMode=mirrored
dnsTunneling=true
firewall=true
autoProxy=true
```
配置完成后，笔者电脑是32G内存，核显显存分出去1个G，虚拟机分配24G，合理设置保证性能同时不会使windows系统其他功能可用内存太局促。

切换到wsl的源码目录，准备开始编译。

### launch起编系统
名词解释：
* Makefile → Android平台编译系统，用Makefile写出来的一个独立项目，定义了编译规则，实现自动化编译，将分散在数百个Git库中的代码整合起来，统一编译，而且把产物分门别类地输出到一个目录，打包成手机ROM，还可以生成应用开发时使用的SDK、NDK等。
* Android.mk → 定义一个模块的必要参数，使模块随着平台编译，简单点说就是告诉系统以什么规则编译源代码，并生成对应目标文件；
* kati → Google专门为Android研发的小工具，基于Golang和C++，作用是：将Android中的Makefile转换为Ninja文件
* Ninja → 致力于速度的小型编译系统，把Makefile看做高级语言，那它就是汇编，文件后缀为.ninja；
* Android.bp → 替换Android.mk的配置文件；
* Blueprint → 解析Android.bp文件翻译成Ninja语法文件；
* Soong → Makefile编译系统的替代品，负责解析Android.bp文件，并将之转换为Ninja文件；

起编的命令不多，只有两三行：
```
# 预声明环境命令
. build/envsetup.sh
# 编译Pixel系统，target选择aosp_redfin_car
lunch <target>
# 开始make编译，新版上直接一个m即可
m
# 构建与汽车相关的软件包
m android.hardware.automotive.audiocontrol@1.0-service android.hardware.automotive.vehicle@2.0-service
```
每次开始编译开始的第一个命令便是. build/envsetup.sh。在文件envsetup.sh声明了当前会话终端可用的命令，这里需要注意的是当前会话终端，也就意味着每次新打开一个终端都必须再一次执行这些指令。build/envsetup.sh文件存在的意义就是，设置一些环境变量和shell函数为后续的编译工作做准备。

而后的lunch操作执行的其实就是build/envsetup.sh脚本中的lunch函数，选择一个版本进行编译，一般可选user，userdebug，eng三种版本，其上的权限是逐步升级的。如果launch后没有参数，那么会出现一列版本可供选择，选择对应版本前的数字即可。

最后m开始起编，过程很长，笔者第一次编译晚上11点开始，等了两小时才到40%，于是放下电脑睡觉去，早上醒来就编完了。在编译过程中，以前只在论坛文章里看到的那些类，现在全部在命令行里一个个闪现出来参与编译，站在上层应用开发者的角度来看，就很神奇。

### 源码单编某个模块
除了系统整体进行编译，我们也可以对单个应用模块进行编译，编完的apk可以push推送到系统对应文件夹下，完成单个模块的置换。
```
source build/envsetup.sh
lunch aosp_bonito-eng
#进入模块目录
cd package/apps/Setting

#编译单独模块的可选指令如下：
#mm → 编译当前目录下的模块，不编译依赖模块
#mmm → 编译指定目录下的模块，不编译依赖模块
#mma → 编译当前目录下的模块及其依赖项
#mmmma → 编译指定路径下所有模块，切包含依赖
mm

#编译成功会提示生成文件的存放路径，除了生成Setting.odex外，还会在
#priv-app/Settings目录下生成Settings.apk，可直接adb push或adb install
#安装APK验证效果，也可以使用make snod命令重新打包生成system.img，运行模拟器查看
```

## 开始刷机流程
### AOSP编译产物
经过make编译后的产物，都位于源码的 /out目录 ，该目录下我们主要关注下面几个目录：

/out/host：Android开发工具的产物，包含SDK各种工具，比如adb，dex2oat，aapt等。
/out/target/common：通用的一些编译产物，包含Java应用代码和Java库；
/out/target/product/[product_name]：针对特定设备的编译产物以及平台相关C/C++代码和二进制文件；
在/out/target/product/[product_name]目录下，有几个重量级的镜像文件：

* system.img:挂载为根分区，主要包含Android OS的系统文件；
* ramdisk.img:主要包含init.rc文件和配置文件等；
* userdata.img:被挂载在/data，主要包含用户以及应用程序相关的数据；
当然还有boot.img，reocovery.img等镜像文件，这里就不介绍了。

查看/aaos/out/target/product/redfin文件夹下关于Pixel 5设备特定的文件：
```
stephen@CODE01:~/aaos/out/target/product/redfin$ ls
android-info.txt                           misc_info.txt
apex                                       module-info.json
appcompat                                  module-info.json.rsp
boot-debug.img                             obj
boot-test-harness.img                      obj_arm
boot.img                                   previous_build_config.mk
bootloader.img                             product
build_fingerprint.txt                      product.img
build_thumbprint.txt                       radio.img
clean_steps.mk                             ramdisk
data                                       ramdisk-debug.img
debug_ramdisk                              ramdisk-test-harness.img
dexpreopt_config                           ramdisk.img
dtb.img                                    recovery
dtbo.img                                   root
fake_packages                              super_empty.img
gen                                        symbols
installed-files-product.json               system
installed-files-product.txt                system.img
installed-files-ramdisk-debug.json         system_ext
installed-files-ramdisk-debug.txt          system_ext.img
installed-files-ramdisk.json               system_other
installed-files-ramdisk.txt                system_other.img
installed-files-recovery.json              test_harness_ramdisk
installed-files-recovery.txt               testcases
installed-files-root.json                  userdata.img
installed-files-root.txt                   vbmeta.img
installed-files-system-other.json          vbmeta_system.img
installed-files-system-other.txt           vendor
installed-files-system_ext.json            vendor.img
installed-files-system_ext.txt             vendor_boot-debug.img
installed-files-vendor-ramdisk-debug.json  vendor_boot-test-harness.img
installed-files-vendor-ramdisk-debug.txt   vendor_boot.img
installed-files-vendor-ramdisk.json        vendor_debug_ramdisk
installed-files-vendor-ramdisk.txt         vendor_ramdisk
installed-files.json                       vendor_ramdisk-debug.img
installed-files.txt                        vendor_ramdisk.img
kernel
```
确认无问题后，我把整个文件夹全部转到mnt挂载的windows目录下，准备好手机设备后即可刷写了。
```
cp -r ~/aaos/out/target/product/redfin /mnt/d/Pixel5
```
设置设备，刷写镜像文件
首先打开pixel 5的开发者选项里的USB调试模式，也需要打开OEM锁：
```
adb reboot bootloader
fastboot flashing unlock
```
在编译产物的文件夹，执行以下指令。开始清空设备数据，刷写车机系统，完成后推送汽车相关文件：
```
fastboot -w flashall
```
```
# 这些命令也可以制作成sh脚本，每次刷完机都执行一遍即可，免去手动输入
# 等刷写完毕并主屏幕显示后，再推送汽车专用文件
adb root
adb remount
adb reboot
# 每次刷写新系统都需要执行上面三步，使文件系统重新挂载生效
# 就可以使windows的shell获取root权限
adb root
adb remount
adb sync vendor
adb reboot
```
等手机再次reboot重启后就是下面的动画和launcher界面了：
![Google_mvi](/assets/img/blog/blogs_aaos.png){:width="400" height="300" loading="lazy"}

## 后续
刷完了系统，不光是走完了一次体验，还需要找到可以学习的角度，深入改动系统代码，通过定制系统，达到需要的效果。