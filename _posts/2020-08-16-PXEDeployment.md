---
layout:     post
title:      CentOS7生产环境配置PXE无人值守OS安装环境
subtitle:   基于UEFI引导的批量安装
date:       2020-07-05
author:     Lionado
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - PXE
    - CentOS
    - Batch Install
---

##  引言

为解决机器上架后大规模机器OS的系统标准化、自动化安装，特采用PXE+Anaconda的方式实现。  

* 目标：机器上架连接到LAN后启动电源即自动完成操作系统的安装并自动进入到服务状态。  
* 名词解释：  
  ** PXE：预启动执行环境（Preboot eXecution Environment，PXE，也被称为预执行环境)提供了一种使用网络接口（Network Interface）启动计算机的机制。这种机制让计算机的启动可以不依赖本地数据存储设备（如硬盘）或本地已安装的操作系统。PXE实现为C/S架构，client端集成在网卡的启动芯片中。  
  ** Anaconda： naconda是Red Hat、CentOS、Fedora等Linux的安装管理程序。它可以提供文本、图形等安装管理方式，并支持Kickstart等脚本提供自动安装的功能。

## 部署实现

### 部署架构

环境整体由三个服务协作完成：DHCP服务+TFTP服务+HTTP服务。整体实现流程如下：
![png1](https://raw.githubusercontent.com/WeiyiGeek/blogimage/master/2019/2019032505.png)  

1. PXE 客户端发送UDP广播请求
   PXE Client向DHCP发送请求 PXE Client从自己的PXE网卡启动，通过PXE BootROM(自启动芯片)会以UDP(简单用户数据报协议)发送一个广播请求，向本网络中的DHCP服务器索取IP。

2. DHCP服务器提供信息
   DHCP服务器收到客户端的请求，验证是否来至合法的PXE Client的请求，验证通过它将给客户端一个“提供”响应，这个“提供”响应中包含了为客户端分配的IP地址、pxelinux启动程序(TFTP)位置，以及配置文件所在位置。

3. PXE客户端请求下载启动文件
   客户端收到服务器的“回应”后，会回应一个帧，以请求传送启动所需文件，这些启动文件包括：pxelinux.0（ #引导文件，相当于grub）、pxelinux.cfg/default(#启动菜单文件)、vmlinuz（内核文件）、initrd.img（伪文件系统文件）等文件。

4. TFTP服务器响应客户端请求并传送文件
   当服务器收到客户端的请求后，他们之间之后将有更多的信息在客户端与服务器之间作应答, 用以决定启动参数;BootROM由TFTP通讯协议从Boot Server下载启动安装程序所必须的文件(pxelinux.0、pxelinux.cfg/default)，default文件下载完成后，会根据该文件中定义的引导顺序，启动Linux安装程序的引导内核。

5. 请求下载自动应答文件
   客户端通过pxelinux.cfg/default文件成功的引导Linux安装内核后，安装程序首先必须确定你通过什么安装介质来安装linux，如果是通过网络安装(NFS, FTP, HTTP)，则会在这个时候初始化网络，并定位安装源位置，接着会读取default文件中指定的自动应答文件ks.cfg所在位置，根据该位置请求下载该文件。

6. 客户端安装操作系统
   将ks.cfg文件下载回来后，通过该文件找到OS Server，并按照该文件的配置请求下载安装过程需要的软件包， OS Server和客户端建立连接后，将开始传输软件包，客户端将开始安装操作系统，安装完成后将提示重新引导计算机。  
     ![png2](https://raw.githubusercontent.com/WeiyiGeek/blogimage/master/2019/2019032506.png)  
     *补充问题：在第2步和第5步初始化2次网络了，这是由于PXE获取的是安装用的内核以及安装程序等，而安装程序要获取的是安装系统所需的二进制包引导以及配置文件。因此PXE模块和安装程序是相对独立的，PXE的网络配置并不能传递给安装程序，从而进行两次获取IP地址过程，但IP地址在DHCP的租期内是一样的。*

  *补充问题2：对于UEFI模式的安装，实际请求与上图Legacy模式的是不一样的，这里贴出实际的tftpd的日志予以说明：*

 ![img](file:///E:/企业微信文件存储/Image/2020-07/企业微信截图_15948979621286.png) 

*从上面可以看出，最初需求的是BOOTX64.EFI文件（网络攻略则写的是 grubx64.efi ，实际操作发现客户端下载后无法打开），但这是通过实际摸索发现的，BOOTX64.EFI客户端下载后会进一步下载grubx64.efi,需要在BIOS中关闭SecureBoot 模式。 通过google搜索，网络攻略中介绍如果打开 SecureBoot 模式，则应该是 shim.efi文件，但是实际操作发现客户端下载后无法打开。（相关攻略参见 http://blog.itpub.net/20747382/viewspace-2153053/ ）*





### 环境准备

将三个服务均部署在同一个服务器上，该服务器作为PXE安装的服务端，需要存储安装OS所需的OS光盘数据和kickstart配置文件，整体负责所有机器的IP地址分配和安装必需文件的下载。  
以下配置部署均在CentOS7.4下执行，serverIP为192.168.125.136。

#### 服务包的安装。

在这之前你需要配置好yum源。这里不做详述。  

```
yum -y install httpd tftp* xinetd syslinux
```

#### 服务配置与启动  

##### 固定DHCP服务器IP地址

必须固定ip，否则后面DHCP配置及其他配置均需要因为每次重新分配IP而修改。

##### DHCP

1. 编辑/etc/dhcp/dhcpd.conf，配置好PXE相关网络配置. 这里要注意：

   1.1 支持跨网段DHCP：如果DHCP服务器需要管理跨网段IP分配（即分配网段在本机网卡所在网段之外），需要配置好交换机的dhcp relay，同时hcp不要配置share-network，因为这样就不会根据relay传过来的via IP段来匹配subnet分配IP了；

   1.2 必须配置一个本机任意一个IP地址所在地址段的subnet，否则服务无法启动。

   1.3 同时支持两种启动模式，需要用到option architecture-type，需要先声明option（见下方实际配置）,否则会提示不被支持的option。

```
ddns-update-style interim;
ignore client-updates;
option time-offset   -18000;
default-lease-time 259200;
max-lease-time 777600;
option domain-name-servers 10.127.96.11;
option architecture-type code 93 = unsigned integer 16;##使用option architecture-type需要先声明这段



  subnet 10.127.19.0 netmask 255.255.255.0 {
          range dynamic-bootp 10.127.19.200 10.127.19.210;
          option routers 10.127.19.1;
          class "pxeclients" {
                  match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
                  next-server 10.127.64.56;
                  if option architecture-type = 00:06 {
                          filename "uefi/grubx.efi";
                  } else if option architecture-type = 00:07 {
                          filename "uefi/BOOTX64.EFI";
                  } else {
                          filename "legacy/pxelinux.0";
                  }
         }

  }

  subnet 10.127.64.0 netmask 255.255.255.0 {
          range dynamic-bootp 10.127.64.200 10.127.64.210;
          option routers 10.127.64.1;
  }

```

2. 启动hdcp服务  

```
systemctl start dhcpd && systemctl enable dhcpd
```

##### tftp

记住在tftp正常工作之前必需安装xinetd守护进程,因为tftp依赖于xinetd。

1. 激活tftp服务(最好打开debug日志用于排查问题，方法是在/etc/xinetd.d/tftp中server_args参数中加上-v)

```
sed -i '/disable/s/yes/no/g' /etc/xinetd.d/tftp
```

2. 启动服务

```
systemctl start xinetd && systemctl enable xinetd
systemctl start tftp &&  systemctl enable tftp
```

##### httpd

httpd用于ks.cfg和安装文件的下载。

1. 启动服务。

```
systemctl start httpd && systemctl enable httpd
```

2. 拷贝安装光盘的文件(可到19.29的/rhviso/目录下找到)到本地存储，否则直接使用会有IO性能问题。然后用本地镜像挂载的方式挂到/var/www/html/centos76下。  

   ```
   mount -o loop -t iso9660 /data/ISO/rhel-server-7.6-x86_64-dvd.iso /var/www/html/rhel76/
   ```

   

3. 将当前系统中的ks文件拷贝到/var/www/html/路径下。

```
cp /root/anaconda-ks.cfg /var/www/html/ks.cfg
chmod 666 /var/www/html/ks.cfg #默认是600，会导致下载403
```

4. 修改ks.cfg配置文件/var/www/html/ks.cfg（ks.cfg的作用是预先指定好需要的安装选项（包括系统镜像路径，安装组件，系统语言，网络配置，用户及密码等），当正式安装时PXE Client将会很据该文件去自动配置安装，从而避免了大规模部署时的大量重复操作。）。  
   主要修改的地方是将

```
# Use CDROM installation media

cdrom
```

修改为

```
# Use network installation

url --url="http://192.168.52.132/centos7"
```

从而指定PXE Client从哪里去获得镜像文件，ks.cfg文件修改后如下(手工分区，gpt等配置均没配，实际情况要更复杂)，

```
#version=DEVEL
# License agreement
eula --agreed
repo --name="Server-HighAvailability" --baseurl=file:///run/install/repo/addons/HighAvailability
repo --name="Server-ResilientStorage" --baseurl=file:///run/install/repo/addons/ResilientStorage
# System authorization information
auth --enableshadow --passalgo=sha512
# Use graphical install
text
# Run the Setup Agent on first boot
firstboot --disable
ignoredisk --only-use=sda
# Keyboard layouts
# old format: keyboard us
# new format:
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network  --bootproto=dhcp --device=eno1 --onboot=on --ipv6=ignore --activate
network  --device=bond0 --noipv6  --onboot=yes --bondslaves=em1,em2 --bondopts=mode=active-backup,balance-rr;primary=em1,miimon=80,updelay=60000 --activate
network --device=em1 --noipv6 --nodns --onboot=yes --activate
network --device=em2 --noipv6 --nodns --onboot=yes --activate
network  --bootproto=dhcp --device=bond0 --onboot=on --ipv6=auto
network  --hostname=localhost.localdomain
# Shutdown after installation
reboot
repo --name="monitor" --baseurl=http://10.127.46.29/monitor
repo --name="rhel-7-server-rpms" --baseurl=http://10.127.46.29/ocp.311/rhel-7-server-rpms
repo --name="rhel-7-server-extras-rpms" --baseurl=http://10.127.46.29/ocp.311/rhel-7-server-extras-rpms

# Use network installation
url --url="http://10.127.64.56/rhel76/"
#Root password
rootpw --iscrypted 05a53c00526986581b9192df06be2058806a9e2d56a02c59e4b26df9d480c25bade4ef0424eceff6393ba031f086a13c57c4d74f3375c696f63a96d8bb23872
# System services
services --enabled="chronyd"
# System timezone
timezone Asia/Shanghai --isUtc --nontp
user --name=node --lock
# System bootloader configuration
bootloader --append="rhgb quiet" --location=mbr --driveorder="sda" --boot-drive=sda
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot/efi --fstype="efi" --ondisk=sda --size=256 --fsoptions="defaults,uid=0,gid=0,umask=0077,shortname=winnt"
part /boot --asprimary --fstype="xfs" --ondisk=sda --size=1024
part pv.179 --fstype="lvmpv" --ondisk=sda --size=1000 --grow
volgroup vg_root --pesize=4096 pv.179
logvol /var/crash  --fstype="xfs" --size=1024 --name=lv_crash --vgname=vg_root
logvol /opt/cores  --fstype="xfs" --size=15360 --name=lv_cores --vgname=vg_root
logvol /var/log/audit  --fstype="xfs" --size=256 --name=lv_audit --vgname=vg_root
logvol /var  --fstype="xfs" --size=20480 --name=lv_var --vgname=vg_root
logvol swap  --fstype="swap" --size=4096 --name=lv_swap --vgname=vg_root
logvol /opt  --fstype="xfs" --size=15360 --name=lv_opt --vgname=vg_root
logvol /tmp  --fstype="xfs" --size=20480 --name=lv_tmp --vgname=vg_root
logvol /  --fstype="xfs" --size=20480 --name=lv_root --vgname=vg_root
logvol /home  --fstype="xfs" --size=2048 --grow --name=lv_home --vgname=vg_root

%pre
parted -s /dev/sda mklabel gpt
%end


%post --nochroot
cp -a /run/install/repo/isolinux/bootcustom.sh /mnt/sysimage/opt/
%end

%packages
@core
aide
audit-libs-devel
binutils-devel
bison
elfutils-devel
elfutils-libelf-devel
flex
gcc
gcc-c++
hmaccalc
kexec-tools
ksh
libaio
libstdc++
make
nc
ncurses-devel
net-tools
ntp
numactl-devel
patchutils
pciutils
python-devel
redhat-rpm-config
rpm-build
rpmdevtools
system-config-kdump
system-config-keyboard
system-config-kickstart
xmlto
yum-utils
zlib-devel

%end

%addon com_redhat_kdump --disable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end

```

##### 系统引导文件存储--UEFI模式

前面DHCP配置中已经通过条件判断同时支持了UEFI模式和Legacy模式了，因此下面需要将两种模式所需的文件都准备好。

1. 从RHEL7光盘中提取shim.efi和grubx64.efi。

   ```
   cp /var/www/html/rhel76/Packages/shim-0.9-2.el7.x86_64.rpm /tmp
   cp  /var/www/html/rhel76/Packages/grub2-efi-2.02-0.44.el7.x86_64.rpm /tmp
   rpm2cpio /tmp/shim-0.9-2.el7.x86_64.rpm | cpio -dimv
   rpm2cpio /tmp/grub2-efi-2.02-0.44.el7.x86_64.rpm| cpio -dimv
   mkdir -p  /var/lib/tftpboot/uefi/
   cp /tmp/boot/efi/EFI/redhat/shim*.efi /var/lib/tftpboot/uefi/
   cp /tmp/boot/efi/EFI/redhat/grubx64.efi /var/lib/tftpboot/uefi/
   chmod 555 /var/lib/tftpboot/uefi/grubx64.efi  #文件权限不对会导致客户端下载失败
   ```

2. 从RHEL7光盘拷贝内核文件。

   ```
   cp /var/www/html/rhel76/EFI/BOOT/BOOTX64.EFI /var/lib/tftpboot/uefi/
   cp /var/www/html/rhel76/isolinux/vmlinuz /var/lib/tftpboot/uefi/
   cp /var/www/html/rhel76/isolinux/initrd.img /var/lib/tftpboot/uefi
   ```

3. 配置grub.cfg系统启动文件. vi /var/lib/tftpboot/uefi/grub.cfg

   ```
   set timeout=5
   
   menuentry 'Install RHEL 7.6 via [UEFI] PXE+Kickstart' {
   
   linuxefi uefi/vmlinuz  inst.repo=http://10.127.64.56/rhel76 inst.ks=http://10.127.64.56/ks.cfg
   
   initrdefi uefi/initrd.img
   
   }
   ```

   

##### 系统引导文件存储--Lagecy模式

1. pxelinux.0文件放到ftp中，文件名要和dhcp配置文件内的一致。如果没有该文件则是因为syslinux没安装。

```
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/legacy/
```

2. copy光盘目录中的vmlinuz和initrd.img,这两个文件相当于系统启动时/boot目录下的启动文件,这个用来引导anacoda而不是根。

```
cp /var/www/html/rhel76/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/legacy/
```

3. copy pxe引导所需要的配置文件,splash.png:背景图.boot.msg启动标语,vesamenu.c32:显示同行界面用的程序.（非必须）

```
cp /var/www/html/rhel76/isolinux/{boot.msg,vesamenu.c32,splash.png} var/lib/tftpboot/legacy/
```

  4. pxe启动时显示配置文件信息,和光盘启动类似.

 ```
 mkdir /var/lib/tftpboot/legacy/pxelinux.cfg
 cp /var/www/html/rhel76/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default
 ```

  5. 配置default系统启动文件.在default配置文件/var/lib/tftpboot/pxelinux.cfg/default中找到下面标签:

 ```
 label linux
  menu label^Install CentOS 7
  kernel vmlinuz
  menu default
  appendinitrd=initrd.img inst.stage2=http://192.168.52.132/centos7\
 inst.ks=http://192.168.52.132/ks.cfg  quiet
 ```

6. 检查PXE Server的状态并启动PXE Client.

```
systemctl status dhcpd xnetd tftp httpd
```

以上三者结果都应该为active(running).
同时确认关闭防火墙。

```
service firewalld status   
```

结果都应该为inactive(dead)。  
检查待安装系统盘是否以及挂载到指定目录下。  
如果以上状态均正常，可以开始启动PXEClient，并将启动方式设置为网卡启动。

接下来可以启动客户端进入PXE模式开始自动部署了。



##### 主要问题记录：

1. dhcp分配的网段混乱无序
   解答： 不要用shared-network， 直接用subnet，注意本机所在网段至少配置一个，否则dhcp无法启动

2. pxelinux.0下载后无反应。从xinet日志看“Error code 8: User aborted the transfer”
   解答： 是因为客户端时uefi引导模式，而该模式不能使用pxelinux.0引导启动，需要支持uefi模式的PXE配置。 （可以同时支持uefi+legacy两种）

3. DHCP配置同时支持uefi+legacy两种引导安装，但是配置if arch =时提示“非法的option”。
   解答：DHCP加上下一条配置：
   option architecture-type code 93 = unsigned integer 16;

4. uefi模式下客户端无法下载shim文件，通过tcpdump抓包也能看到回包为空。（但是pxelinux.0文件可以正常下载）
   解答：dhcp的file由shimx64.efi改为BOOTX64.EFI，客户机BIOS中同时disable safe model ,就可以下载了！下载完后会去下载grubx64.efi文件，下载失败。 
   修改grubx64.efi权限为555，就可以下载了。后面就正常下载其他文件了！

5. 无法下载ks.cfg, 提示403
   解答：因为文件属性是600且属主是root， 改为666即可。

