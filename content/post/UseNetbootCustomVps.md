---
title: "使用 netboot.xyz 定制 VPS 操作系统"
date: 2022-04-06T17:51:25+08:00
draft: false
typora-root-url: ../../static
---

> 前段时间使用 Oracle Cloud ARM VPS 测试内核，没有设置好 kconfig，导致 VPS 无法启动。Oracle Cloud 不支持重装，ARM 机器也非常难申请，所以想办法恢复。Google 的过程中意外发现了 netboot.xyz 这个工具。它可以解决有些 VPS 无法安装指定操作系统的问题（比如 Arch Linux）。
>

# 必要性

该方案只对 VPS **安装操作系统，且安装过程有特殊要求** 的需求有价值。比如我的 VPS 存储空间有限，希望使用 btrfs 文件系统的压缩特性。

假如只想重装或更换 VPS 的操作系统，可以 Google 现成的 dd 脚本，既简单又方便。

当然，即使没有上述需求，也可以考虑在 VPS 上配置 netboot.xyz。可以作为应急恢复模式使用。

# netboot.xyz

## 简介

[netboot.xyz](https://github.com/netbootxyz/netboot.xyz]) 可以通过网络直接引导常见的操作系统。它使用 iPXE，无需下载 ISO 镜像。

详细的使用文档详见[官方文档](https://netboot.xyz/docs/)。

![netboot.xyz](/images/UseNetbootCustomVps/netboot-example.gif)

## 快速使用

### 添加到 GRUB

常见的发行版都包含 `grub-imageboot`。安装后，将 `netboot.xyz.iso` 放到 `/boot/images`目录下，GRUB 就可以自动并完成配置，非常简单。

比如在 Debian/Ubuntu 下：

```bash
# Install grub-imageboot
apt install grub-imageboot

# Download netboot.xyz ISO
mkdir /boot/images
cd /boot/images
wget https://boot.netboot.xyz/ipxe/netboot.xyz.iso

# Update GRUB menu to include this ISO
update-grub2
reboot
```

重启后，就可以在 GRUB 菜单中看到选项了。

![](/images/UseNetbootCustomVps/netboot-GRUB.webp)

### 添加 UEFI 固件

如果机器使用 UEFI 启动方式，可以直接使用 UEFI 固件。

```bash
mkdir /boot/efi/EFI/rescue
cd /boot/efi/EFI/rescue

# for arm64 machine
wget https://boot.netboot.xyz/ipxe/netboot.xyz-arm64.efi

# for x86_64 machine
wget https://boot.netboot.xyz/ipxe/netboot.xyz-snp.efi
```

重启进入 BIOS，可以通过选择启动文件的方式，启动 netboot.xyz。

![](/images/UseNetbootCustomVps/OracleBIOS-5.webp)

#  安装记录

记录下 Oracle Cloud ARM 机器使用 netboot.xyz 安装 Arch Linux 的过程。

## 下载 netboot.xyz UEFI 固件

可以按照 **快速设置** 章节的方法，将固件放到 EFI 文件目录中。

```bash
mkdir /boot/efi/EFI/rescue
cd /boot/efi/EFI/rescue
wget https://boot.netboot.xyz/ipxe/netboot.xyz-arm64.efi
```

## 设置 Oracle 控制台

Oracle 没有提供 VNC 登录的功能，需要配置登录 Cloud Shell，就可以管理机器了。

除了命令行外，使用 Cloud Shell 还可以控制机器的 BIOS（开机 F2）。

创建好 Cloud Shell 后，可以直接复制登陆需要的 SSH 命令登陆。如下图所示：

![](/images/UseNetbootCustomVps/OracleConsole.webp)

## BIOS

重启 VPS，按 F2 可进入 BIOS。

选择 `Boot Maintenance Manager` --> `Boot From File`，然后按照上述文件路径选择 netboot.xyz  的启动固件。

![](/images/UseNetbootCustomVps/OracleBIOS-1.webp)

![](/images/UseNetbootCustomVps/OracleBIOS-2.webp)

![](/images/UseNetbootCustomVps/OracleBIOS-3.webp)

![](/images/UseNetbootCustomVps/OracleBIOS-4.webp)

![](/images/UseNetbootCustomVps/OracleBIOS-5.webp)

## netboot.xyz

选择固件后，就可以进入 netboot.xyz 了。如下图所示，netboot.xyz 会自动配置网络，并使用 iPXE 拉取相关镜像。

![](/images/UseNetbootCustomVps/netboot-1.webp)

除了引导操作系统外，netboot.xyz 还提供了一些功能。如下图所示。

这里选择 `Linux Network Installs`。

![](/images/UseNetbootCustomVps/netboot-2.webp)

针对 arm64 设备，netboot.xyz 没有提供 Arch Linux 的安装镜像。这里选择启动先引导相对轻量的 Alpine Linux，然后通过它再安装 Arch Linux。

假如机器是 x86_64 架构，可以直接选择 Arch Linux 镜像安装。

![](/images/UseNetbootCustomVps/netboot-3.webp)

![](/images/UseNetbootCustomVps/netboot-4.webp)

选择后，netboot.xyz 会使用 iPXE 拉取 Alpine Linux 的引导内核和镜像。

完成后就会进入登陆界面。使用 `root` 用户无需密码即可登陆。

![](/images/UseNetbootCustomVps/netboot-5.webp)

## 安装系统

### 配置 Alpine Linux 源

安装 Arch Linux ARM 过程中，需要用到 Alpine Linux `community` 仓库的软件包。

配置 Alpine Linux 源，将 `community` 仓库对应的第三行解注释，并更新仓库数据：

```bash
vi /etc/apk/repositories
# 删除第三行的 ‘#’

apk update
```

![](/images/UseNetbootCustomVps/netboot-6.webp)

### 安装需要的软件

```bash
apk add dosfstools e2fsprogs libarchive-tools pacman arch-install-scripts parted btrfs-progs
```

### 磁盘分区

Oracle Cloud ARM 默认包含两个分区，`/dev/sda15`是 EFI 分区，大小为 100MiB，即上文存放 netboot.xyz 固件的地方。 `/dev/sda1` 是数据盘，即 Linux 系统的根分区。空间是申请机器选择的磁盘大小减去EFI空间后的剩余。

我的机器磁盘空间一共是 100GiB，希望使用 `btrfs` 文件系统。**为了避免一些引导的麻烦**，我将剩余空间分为两份：

* `/dev/sda1` 划分了 1GiB，作为 `/boot` 分区，使用 `ext4` 文件系统，存放操作系统引导需要的文件（因为我需要经常换内核，所以空间稍大了些）。
* 剩余的空间全给到 `/dev/sda2` ，作为根分区，使用`btrfs` 文件系统。

划分好的磁盘如下所示：

![](/images/UseNetbootCustomVps/netboot-7.webp)

按需划好磁盘后，使用对应的工具，将所有分区格式化。

```bash
mkfs.vfat /dev/sda15
mkfs.ext4 /dev/sda1

#https://btrfs.wiki.kernel.org/index.php/Problem_FAQ#I_get_the_message_.22failed_to_open_.2Fdev.2Fbtrfs-control_skipping_device_registration.22_from_.22btrfs_dev_scan.22
mknod /dev/btrfs-control c 10 234 
mkfs.btrfs /dev/sda2
```

按照 Arch Linux ARM 的文档，并结合自己的需求，将分区挂载好：

```bash
mount /dev/sda2 /mnt -o compress-force=zstd
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
mkdir /mnt/boot/efi
mount /dev/sda15 /mnt/boot/efi
```

### 安装 Arch Linux ARM

与 Arch Linux 的安装过程不同，Arch Linux ARM 提供了一个最小环境的压缩包。

将其下载并解压到根分区（tmpfs）：

```bash
cd /
wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
cd /mnt
bsdtar -xpf /ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
```

完成这一步后，安装 Arch Linux ARM 与 Arch Linux 就没有太大的区别了。假如遇到问题，可以直接在 [Arch Linux Wiki](https://wiki.archlinux.org/) 检索。

后续步骤记录：

```bash
# 生成 fstab
genfstab -U /mnt >> /mnt/etc/fstab

# 切换 chroot 环境并初始化
arch-chroot /mnt/
pacman-key --init
pacman-key --populate archlinuxarm

# 安装必要的软件包
pacman -S base base-devel linux linux-firmware btrfs-progs zsh wget git vim

# EFI boot
pacman -Syu grub efibootmgr
grub-install --efi-directory=/boot/efi --bootloader-id=GRUB

# grub config
grub-mkconfig -o /boot/grub/grub.cfg

# 更改默认用户名/组为 leesheen，并设置密码
groupmod -n leesheen alarm
usermod -d /home/leesheen -l leesheen -m -c leesheen alarm
passwd leesheen
usermod -aG wheel leesheen

# modify /etc/sudoer
pacman -S sudo
vi /etc/sudoer
# 解注释 %wheel ALL=(ALL) ALL

# sshd 配置
vi /etc/ssh/sshd_config 
# 配置安全：关闭 ssh 密码登陆，解注释配置 "PasswordAuthentication no"
# 配置sshd 不断开，解注释配置 "ClientAliveInterval 60" 和 "ClientAliveCountMax 3"

# 添加 ssh key
su leesheen
mkdir ~/.ssh
vim ~/.ssh/authorized_keys
# 添加公钥

# 设置 root 用户密码
passwd

exit # leesheen
exit # chroot
umount -R /mnt

reboot
```

### 进一步配置

安装完成后，重启机器。在拥有私钥的机器上直接 ssh，即可进入系统了。

[在 2018 年写过一篇安装 Arch Linux 的文章](https://leesheen.github.io/post/InstallArchLinux/)，里面有详细的步骤，这里就不赘述了。

yay 在 Arch Linux ARM 上的安装，需要编译源码：

```bash
# 安装 yay
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

按照 Arch Linux 推荐，修改 Linux 部分参数：

```
# /etc/sysctl.d/10-network.conf

net.core.netdev_max_backlog = 16384
net.core.somaxconn = 8192
net.core.rmem_default = 1048576
net.core.rmem_max = 16777216
net.core.wmem_default = 1048576
net.core.wmem_max = 16777216
net.core.optmem_max = 65536
net.ipv4.tcp_rmem = 4096 1048576 2097152
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.udp_rmem_min = 8192
net.ipv4.udp_wmem_min = 8192
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_timestamps = 0
net.core.default_qdisc = cake
net.ipv4.tcp_congestion_control = bbr
net.ipv4.ip_local_port_range = 30000 65535
# net.ipv4.tcp_syncookies = 1
# net.ipv4.tcp_rfc1337 = 1
# net.ipv4.conf.default.log_martians = 1
# net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.icmp_echo_ignore_all = 1
net.ipv6.icmp.echo_ignore_all = 1
# net.ipv4.ping_group_range = 100 100
# net.ipv4.ping_group_range = 0 65535
```

```
# /etc/sysctl.d/20-vm.conf
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.vfs_cache_pressure = 50
vm.dirty_background_bytes = 4194304
vm.dirty_bytes = 4194304
vm.page-cluster = 0
```

### 备份

到此为止，Arch Linux ARM 安装就差不多完成了。

清理无用的文件，使用 btrfs 快照功能做一个备份：

```bash
# 清理pacman
pacman -Scc

# btrfs 备份（只读快照）
sudo mkdir /.snapshots
sudo btrfs subvolume snapshot -r / /.snapshots/root_20220406
```

# 总结

netboot.xyz 不仅帮助我在 VPS 上，按照自己的需求安装了 Arch Linux ARM，它的存在不再让我担心系统被搞挂，可以安心的在 VPS 上做内核测试了。

## 关于压缩

### btrfs

做这件事情的初衷，是需要在安装前启用 btrfs 的压缩功能（不然就直接用 dd 脚本了）。

使用 `-o compress=zstd` ，会默认使用 `level=3` 的压缩比。效果如何？下载并编译 Linux 内核代码，测试看看。

使用 `du -sh ~/Workdir/linux-kernel` 看到编译完，Linux 内核目录占用的空间是 22G。

使用 `compsize -x ~/Workdir/linux-kernel`，看看具体的压缩情况：

```
Processed 167289 files, 406612 regular extents (407509 refs), 69022 inline.
Type       Perc     Disk Usage   Uncompressed Referenced
TOTAL       41%      8.8G          21G          20G
none       100%      2.3G         2.3G         2.2G
zstd        34%      6.5G          18G          18G
```

原本总计需要占用 22G 左右的空间，压缩后只占了 8.8G，压缩比为 41%，**节省了一半多的空间**。

其中绝大部分的文件（18G）是可以压缩的，压缩比为 34%。

不能压缩的 2.3G 应该是 git 的元数据，因为这些文件已经被 git 压缩过了。确认下：

```
# compsize -x ~/Workdir/linux-kernel/.git/

Processed 71 files, 7727 regular extents (7727 refs), 56 inline.
Type       Perc     Disk Usage   Uncompressed Referenced
TOTAL       97%      2.4G         2.4G         2.4G
none       100%      2.0G         2.0G         2.0G
zstd        86%      357M         414M         414M
```

对于磁盘比较紧张，且磁盘性能不强的设备，使用 btrfs 还是有必要的。如果担心性能问题，也可以使用其他的压缩算法替换，比如 `lz4`。

### zRAM

类似空间紧张的问题，也会出现在内存上。可以使用 zRAM 压缩内存（原理参考 swap）。

使用 AUR `zramd` 这个包来管理 zRAM。

```bash
yay -S zramd
```

在内存比较大的机器上（比如 Oracle Cloud ARM 4core 24G 内存），我选择了 `lz4` 压缩算法，内存比较小的机器（比如轻量服务器 4core 4G 内存），我选择了压缩比更大的 `zstd` 压缩算法。大小都配置为物理内存的一半。

可以按需修改配置文件 `/etc/default/zramd`。

```
# See available algorithms by running "cat /sys/block/zramX/comp_algorithm"
ALGORITHM=zstd

# Max fraction of physical memory to use
FRACTION=0.5
```

完成配置文件更新后，启动 zramd。

```bash
systemctl enable --now zramd
```

使用 `zramctl` 查看状态。

```
NAME       ALGORITHM DISKSIZE DATA COMPR TOTAL STREAMS MOUNTPOINT
/dev/zram0 zstd          1.9G   4K   59B    4K       4 [SWAP]
```

使用 zRAM 作为 swap 后，`vm.page-cluster` 也需要修改为 0。详情可以参考[这篇文章](https://www.reddit.com/r/Fedora/comments/mzun99/new_zram_tuning_benchmarks/)。

