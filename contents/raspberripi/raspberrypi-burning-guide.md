# 树莓派烧录指南

## 镜像下载

镜像下载地址为: https://cdimage.ubuntu.com/releases

清华mirrors: https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cdimage/releases

在此目录内选择需要的版本进行下载，树莓派镜像一半名称内带有 `raspi` 字样，ubuntu 18 区分树莓派2、3、4使用不同镜像，ubuntu 20树莓派2、3、4均使用同一个镜像

- ubuntu 20.04: https://cdimage.ubuntu.com/releases/focal/release/ubuntu-20.04.1-preinstalled-server-arm64+raspi.img.xz
- ubuntu 18.04 树莓派4: http://cdimage.ubuntu.com/releases/18.04.4/release/ubuntu-18.04.5-preinstalled-server-arm64+raspi4.img.xz
- ubuntu 18.04 树莓派3: http://cdimage.ubuntu.com/releases/18.04.4/release/ubuntu-18.04.5-preinstalled-server-arm64+raspi3.img.xz

## 系统镜像制作及安装

推荐使用树莓派imager官方烧录工具： https://downloads.raspberrypi.org/imager

或者手动烧录：

```sh
# 镜像下载
$ wget http://cdimage.ubuntu.com/releases/18.04.4/release/ubuntu-18.04.4-preinstalled-server-arm64+raspi3.img.xz
# 格式化SD卡为 MS-DOS （FAT32）格式，非必须要求
$ sudo diskutil eraseDisk FAT32 "UBUNTU" MBRFormat  /dev/disk2
$ sudo diskutil unmountDisk /dev/disk2
# 烧写启动数据
$ sudo sh -c 'gunzip -c ~/Downloads/ubuntu-18.04.4-preinstalled-server-arm64+raspi3.img.xz | sudo dd of=/dev/disk2  bs=32m'
0+40048 records in
0+40048 records out
2624517120 bytes transferred in 913.454427 secs (2873178 bytes/sec)
```

## cloudinit 配置

ubuntu server 提供了 cloud-init 进行服务器配置，可以预先进行配置服务器初始状态，例如ip设置，ssh配置等。而无需在树莓派启动后再进行设置。

使用树莓派烧录工具烧录完成后，重新插拔SD卡后，电脑能识别到一个名称为system-boot 的FAT分区。 打开可以看到许多文件。

> 查看`README`文件以了解cloud-init的配置内容与初始化流程的执行方式。

- 其中 `meta-data` 用于配置 配置 cloudinit 数据源，目前无需对该文件进行更改。
- 其中 `user-meta` 用于配置 ssh file 等,我们主要用它来配置ssh初始密码,hostname等。
- 其中 `network-config` 用于配置服务器网络。

可参考下文进行配置：
- 设置主机名称
- 设置密码永不过期
- 创建账户ubuntu,密码password
- 设置账户root,密码password
- 设置允许root ssh登录
- 设置静态网络配置

user-meta:

```yaml
#cloud-config

# Enable password authentication with the SSH daemon
ssh_pwauth: true

# Hostname
hostname: ubuntu-001

# On first boot, set the (default) ubuntu user's password to "ubuntu" and
# expire user passwords
chpasswd:
  expire: false
  list:
    - ubuntu:password
    - root:password
runcmd:
  - sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/g' /etc/ssh/sshd_config
  - service ssh restart
```

network-config:

```yaml
# This file contains a netplan-compatible configuration which cloud-init
# will apply on first-boot. Please refer to the cloud-init documentation and
# the netplan reference for full details:
#
# https://cloudinit.readthedocs.io/
# https://netplan.io/reference
#

version: 2
ethernets:
  eth0:
    optional: true
    addresses:
      - 192.168.2.31/16
    gateway4: 192.168.0.1
    nameservers:
      addresses:
        - 61.139.2.69
        - 223.5.5.5
```
